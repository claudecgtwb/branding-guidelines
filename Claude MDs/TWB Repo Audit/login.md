# Login Flow Analysis

**Files read:**
- `SOLAS-Match/ui/RouteHandlers/UserRouteHandler.class.php` — `login()` at line 1010
- `SOLAS-Match/ui/DataAccessObjects/UserDao.class.php` — `login()`, `requestAuthCode()`, `loginWithAuthCode()` at lines 887–1002
- `SOLAS-Match/api/DataAccessObjects/UserDao.class.php` — `apiLogin()` at lines 112–155
- `SOLAS-Match/api/v0/Users.php` — `getAuthCode()` at line 610, `getAccessToken()` at line 693
- `SOLAS-Match/api/dispatcher.php` — `sendResponse()` at line 123
- `SOLAS-Match/Common/lib/UserSession.class.php` — session helpers

---

## Overview

The `login()` function is a single route handler that handles **four distinct entry conditions** depending on HTTP method and query/post parameters:

| Condition | How triggered |
|---|---|
| `POST` with `login` field | Username + password form submission |
| `POST` with `credential` field | Google Sign-In button (Google Identity Services JWT) |
| `GET` with `?code=` query param | OAuth2 authorization-code callback (loops back after Google Sign-In, or external OAuth client) |
| `GET` with `?ReturnTo=` query param | SAML SP redirect — stores the return URL, falls through to render the login form |

---

## Path 1 — Username & Password

```
Browser ──POST email+password──► UserRouteHandler::login()
                                        │
                                        ▼
                               UI UserDao::login()
                                        │
                               POST to /api/v0/users/login
                               with ?client_id=…&client_secret=…
                               body: protobuf Login{email, password}
                                        │
                                        ▼
                               API UserDao::apiLogin()
                                  1. look up user by email
                                  2. check: exists, verified, not banned, password matches
                                  3. return user_id
                                        │
                                        ▼
                               API Users::login() endpoint
                                  OAuth2 password grant (league/oauth2-server)
                               Response:
                                  body: serialized User protobuf (JSON)
                                  header: X-Custom-Token: base64(serialized OAuthResponse)
                                        │
                                        ▼
                               UI UserDao::login() reads X-Custom-Token
                               → UserSession::setAccessToken($oauthResponse)
                                        │
                                        ▼
                               UserRouteHandler::login()
                               → UserSession::setSession($user->getId())
                                    stores user_id in encrypted session cookie
                               → post-login redirect (see below)
```

The password is sent from UI to API over localhost HTTP (same server), not user-facing HTTPS. The API call includes the OAuth `client_id` and `client_secret` as plain query parameters — this is safe only because the call never leaves the server.

---

## Path 2 — Google Sign-In

Google Identity Services posts a JWT `credential` field directly to the `/login` URL (a "one-tap" or button sign-in, not a redirect-based OAuth2 flow):

```
Browser ──POST credential (Google JWT)──► UserRouteHandler::login()
                                                │
                                         CSRF: verify g_csrf_token double-submit cookie
                                                │
                                         Google_Client::verifyIdToken($post['credential'])
                                         Extract: email, given_name, family_name
                                                │
                                         UserDao::set_google_user_details(email, first, last)
                                         (saves to GoogleUserDetails table for use during auto-registration)
                                                │
                                         UserDao::requestAuthCode($email)
                                         Returns URL:
                                           {siteApi}/api/v0/users/{email}/auth/code/
                                             ?client_id=…
                                             &redirect_uri=https://…/login
                                             &response_type=code
                                                │
                                         302 redirect browser to that URL ──────────────────┐
                                                                                             │
                                                          ┌──────────────────────────────────┘
                                                          ▼
                                             API Users::getAuthCode()
                                               1. Look up user by email
                                               2. If no user → auto-register:
                                                    - password = md5(email) (placeholder, never used)
                                                    - finishRegistration()
                                                    - language pref = English
                                                    - copy first/last name from GoogleUserDetails
                                                    - set terms_accepted = 1 (pending Code of Conduct)
                                               3. Check user is not banned
                                               4. Generate OAuth2 authorization code
                                                    via league/oauth2-server authorization_code grant
                                               5. 302 redirect to redirect_uri?code={authCode}
                                                    which is back to: https://…/login?code={authCode}
```

This brings us to Path 3.

---

## Path 3 — OAuth Authorization Code Callback

This GET lands on `/login?code={authCode}`. It is triggered by:
- The Google Sign-In flow (Path 2 above)
- Any external OAuth2 client that has gone through the `getAuthCode` endpoint

```
Browser ──GET /login?code={authCode}──► UserRouteHandler::login()
                                               │
                                        UI UserDao::loginWithAuthCode($authCode)
                                          POST to /api/v0/users/authCode/login/
                                          body: client_id, client_secret, redirect_uri, code
                                               │
                                               ▼
                                        API Users::getAccessToken()
                                          league/oauth2-server completes authorization_code flow
                                          → access_token, token_type, expires, expires_in
                                          Look up user by access token
                                          Strip password/nonce fields
                                          Log successful login attempt
                                        Response:
                                          body: serialized User protobuf
                                          header: X-Custom-Token: base64(serialized OAuthResponse)
                                               │
                                               ▼
                                        UI UserDao::loginWithAuthCode() reads X-Custom-Token
                                        → UserSession::setAccessToken($oauthResponse)
                                               │
                                               ▼
                                        UserRouteHandler::login()
                                        → UserSession::setSession($user->getId())
                                        → post-login redirect (see below)
```

---

## SAML SSO Hook

The SAML integration is handled outside this repo (a SimpleSAMLphp service provider or equivalent). Its only contact point with this code is:

1. **SAML SP redirects unauthenticated user** to `/login?ReturnTo={saml_return_url}`
2. `login()` on GET with `ReturnTo` stores the URL in `$_SESSION['return_to_SAML_url']` and renders the login form.
3. After any successful login (password or Google), the code checks:
   ```php
   if (!$request_url) {          // no session referer
       if (!empty($_SESSION['return_to_SAML_url'])) {
           $request_url = $_SESSION['return_to_SAML_url'];
       }
   }
   unset($_SESSION['return_to_SAML_url']);
   ```
4. If a SAML URL was stored, the browser is redirected back to it (the SAML SP's assertion consumer or intermediate step) instead of the default post-login destination.

The SAML return URL typically carries session state for the SP (e.g., `SimpleSAMLphp`'s `AuthState` parameter). This mechanism is purely a redirect hand-off — no SAML assertion is verified here.

---

## Post-Login Redirect Logic

Both the password path and the OAuth callback path share identical redirect logic (duplicated in code at lines 1066–1094 and 1172–1203):

```
1. If $_SESSION['ref'] (pre-set referer, e.g. session timed out mid-use)
       → redirect there

2. Else if $_SESSION['return_to_SAML_url']
       → redirect there (SAML hand-off)

3. Else: role-based default
   ├── If site admin / org admin / project officer:
   │     - terms_accepted == 1 (new Google user, never accepted Code of Conduct)
   │           → /user/{id}/googleregister  (CoC acceptance page)
   │     - terms_accepted < 3
   │           → update_terms_accepted(3)   (promote to full acceptance)
   │     - has NGO orgs → /org/{org_id}/home_ngo
   │     - otherwise   → /home
   │
   └── If regular linguist:
         - no native locale set          → /user/{id}/user-private-profile  (complete profile)
         - has a post_login_message      → show message + /user/{id}/user-private-profile
         - hasn't seen tutorial          → /virtual-tour.html
         - otherwise                     → /home
```

---

## The "Confusing" Internal OAuth Loop — Explained

The platform uses OAuth2 internally as the authentication layer between the UI app and the API app (they are two separate Slim processes on the same server). This means:

- **Password login** triggers a direct OAuth2 **password grant** (client posts credentials, gets token back immediately).
- **Google Sign-In** triggers a full OAuth2 **authorization code grant** — the API generates a code, redirects the browser back to `/login?code=…`, and then the UI exchanges the code for a token. This adds a browser round-trip, but means the UI never handles the Google JWT itself beyond verifying it — all account creation and token issuance happens in the API.

The result is that `login()` handles *both* the initial entry point (POST with Google JWT) *and* the callback from its own API (GET with `?code=`). It is not an external OAuth2 server — it is the platform talking to itself.

**Why is this confusing?**

- `requestAuthCode()` constructs a URL to the API but **does not call it** — it returns the URL string to the route handler, which does the redirect. The method name implies an action but it only builds a string.
- `loginWithAuthCode()` calls an API endpoint named `authCode/login` which internally maps to `Users::getAccessToken()` — the naming does not match the function name.
- The route `/api/v0/users/authCode/login/` accepts a GET (per the dispatcher) but `loginWithAuthCode()` POSTs to it. This is a discrepancy worth verifying in production.
- The `X-Custom-Token` header carrying the OAuth token is completely non-standard. No client would discover this without reading the source.

---

## Integration Assessment

### Option A — Direct API login (simplest)

An external platform can authenticate a user against the TWB Platform by POSTing to the internal API login endpoint:

```
POST /api/v0/users/login/?client_id={id}&client_secret={secret}
Content-Type: application/json   (or protobuf)
Body: { email, password }

Response:
  200 body: User object (JSON/protobuf)
  header X-Custom-Token: base64(protobuf OAuthResponse { token, token_type, expires, expires_in })
```

The OAuth `client_id` and `client_secret` must be pre-registered in `oauth_clients` / `oauth_client_endpoints` tables. The access token from `X-Custom-Token` can then be passed as `Authorization: Bearer {token}` on subsequent API calls.

**Constraints:** Requires the external platform to handle the non-standard `X-Custom-Token` response header and deserialize a base64-encoded protobuf.

### Option B — Authorization Code flow (for SSO from another web app)

Register the external platform as an OAuth2 client in the DB. Then:

1. Redirect user to `/api/v0/users/{email}/auth/code/?client_id=…&redirect_uri={your_callback}&response_type=code`
2. The API creates the user account if it does not exist, generates a code, and redirects to `{your_callback}?code={authCode}`.
3. Exchange the code at `POST /api/v0/users/authCode/login/` with `client_id`, `client_secret`, `redirect_uri`, `code`.
4. Response returns user + token in the same format as Option A.

**Watch out:** `getAuthCode()` auto-registers new users with a `md5(email)` dummy password and `terms_accepted=1`. An external platform driving users through this flow will silently create TWB Platform accounts. Decide whether that is desirable.

### Option C — SAML SP redirect (for SSO via SAML)

If the external platform is a SAML Service Provider:

1. Redirect unauthenticated users to `{twb_platform}/login?ReturnTo={saml_return_url}`.
2. The user logs in (by any method — password or Google).
3. The TWB Platform redirects back to `{saml_return_url}`.

**This is the thinnest possible integration.** The TWB Platform is acting as a SAML Identity Provider front-end — but there is no SAML assertion issued here. The `ReturnTo` redirect is a plain HTTP redirect with no assertion, attributes, or signature. A genuine SAML2 integration would require a proper IdP (e.g., SimpleSAMLphp in IdP mode) fronting the TWB Platform, not this URL parameter hook.

---

## Recommendations for Migration / Integration

1. **Consolidate the duplicate post-login redirect block.** Lines 1066–1094 and 1172–1203 are identical. Extract to a private `redirectAfterLogin(Response $response, $user, $request_url): Response` method before migration to avoid the logic drifting out of sync.

2. **Replace `X-Custom-Token` with a standard response body.** Any new API layer (Next.js route handlers) should return the token in the JSON body. The base64-encoded protobuf header is opaque to every standard HTTP client.

3. **Replace the OAuth2 password grant between UI and API.** In Next.js, the UI server components call service functions directly — there is no internal HTTP round-trip. The OAuth layer between them can be dropped entirely. Keep OAuth2 only for genuinely external clients.

4. **For SAML: implement a real IdP assertion.** The `ReturnTo` mechanism gives a successful login redirect but no SAML assertion. If the external platform needs to receive user attributes (email, name, role) it will need a proper SAML2 IdP response. SimpleSAMLphp can be configured to use the TWB Platform's user DB as its authentication source.

5. **New-user auto-registration in `getAuthCode` is a side-effect risk.** The API silently creates accounts for any email passed to `getAuthCode`. If an external platform drives Google Sign-In for an email that does not belong to a TWB linguist, a stub account will be created. Consider adding an allowlist check or moving registration to an explicit step.

6. **Terms acceptance state must be respected by integrations.** A user with `terms_accepted == 1` has not accepted the Code of Conduct. The platform redirects them to the `googleregister` page. An external integration bypassing this check could give access to platform features before the user has agreed to the terms.
