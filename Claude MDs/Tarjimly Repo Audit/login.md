# Login Flow Analysis — Tarjimly Web

**Files read:**
- `src/middleware.ts`
- `src/lib/auth/auth-utils.ts`, `AuthContext.tsx`, `ProtectedRoute.tsx`
- `src/lib/cookies.ts`
- `src/lib/firebase.ts`
- `src/app/login/requests.tsx`, `src/app/login/page.tsx`
- `src/app/enterprise/login/page.tsx`
- `src/app/verify-email/page.tsx`, `src/app/blacklisted/page.tsx`
- `src/config.ts`

---

## Overview

Unlike TWB Platform's single `login()` route handler with four entry conditions, Tarjimly splits login across **two nearly-identical page components** (`/login` and `/enterprise/login`) that call **two different backend endpoints** but share the same post-login machinery (`AuthContext`). There is no server-side OAuth exchange, no SAML hook, and no Google Sign-In in this repo (unlike TWB) — authentication is email/password only, against the external backend.

| Entry point | Backend endpoint | Notes |
|---|---|---|
| `/login` (regular host) | `POST /api/v3/auth/login` | |
| `/enterprise/login` (any non-regular host) | `POST /api/v3/auth/enterprise-login` | Sends an extra `host` header |
| `/login` on an enterprise host | — | `middleware.ts` rewrites this to `/enterprise/login` before it renders |
| `/enterprise/login` on a regular host | — | `middleware.ts` rewrites this to `/login` |

---

## Path 1 — Regular Login

```
Browser ──POST email+password──► loginRequest() [src/app/login/requests.tsx]
                                        │
                                POST /api/v3/auth/login  (proxied by middleware.ts
                                                            to https://tarjim.ly/api/v3/auth/login)
                                credentials: 'include'
                                        │
                                        ▼
                          Response body: { token, firebase_token, tarjimly_id, error? }
                                        │
                                        ▼
                          AuthContext.login()
                            1. setAuthCookie(response.token)
                               → parseCookieValue() strips "connect.sid=" prefix
                               → document.cookie = "connect.sid=<value>; path=/; max-age=86400"
                               (client-JS-set, NOT HttpOnly — see Landmines below)
                            2. getUserMetadata()  [GET /api/mobile/v2/users/metadata]
                               → { user_role, organizations[], is_email_verified, tarjimly_id, ... }
                            3. Determine isEnterprise from organizations[0].type === 'enterprise'
                            4. setUserTypeCookie('enterprise' | 'regular')
                               setRegularUserType('aidworker' | 'translator')
                               (both plain, non-HttpOnly cookies — UI routing only)
                            5. If isEnterpriseDomain() && !isEnterprise:
                                 clearAllAuthCookies(); return error
                                 ("You are not authorized to access this application")
                            6. signInWithCustomToken(auth, response.firebase_token)
                               → Firebase Auth session established (Firestore access)
                               → identifyUser() [PostHog], identifyLogRocket() [LogRocket]
                               → updateUserAppVersion()  [POST /api/v3/users/me/app-version]
                            7. If !userMetadata.is_email_verified:
                                 router.push('/verify-email')
                               else:
                                 router.push(getRedirectUrlForUser(user_role, isEnterprise))
                                   → aidworker  → /user/jobs
                                   → translator → /translator/jobs
```

Note step 6 is wrapped in its own `try/catch` — a Firebase setup failure is logged to console but does **not** block login or the redirect. A user can end up fully backend-authenticated (valid `connect.sid`, redirected past login) with a broken Firebase session, which would silently break any Firestore-backed real-time feature (active-call status, etc.) until next reload.

---

## Path 2 — Enterprise Login

Structurally identical to Path 1, with three differences:

1. Calls `enterpriseLoginRequest()` → `POST /api/v3/auth/enterprise-login`, with an extra `'host': getApiHost()` header (`us.tarjim.ly` for enterprise, resolved client-side).
2. Only reachable via `/enterprise/login`, which `middleware.ts` treats as the canonical login page for any host not in `REGULAR_HOSTS`.
3. Same authorization check runs in reverse: if `!isEnterpriseDomain() && isEnterprise`, the user is logged into their enterprise account while browsing a regular-host page — `AuthContext`'s `useEffect` on mount (not the login function itself) detects this mismatch and force-clears cookies + redirects to `/login`.

Both regular and enterprise logins converge on the exact same `getRedirectUrlForUser` / email-verification logic.

---

## Path 3 — Session Rehydration (Page Load / Refresh)

There is no server-side session check on initial render — `AuthContext`'s `useEffect` runs client-side on every mount:

```
On mount:
  if hasAuthCookie():                          // connect.sid present?
    userMetadata = getUserMetadata()           // GET /api/mobile/v2/users/metadata
    if userMetadata:
      isEnterprise = organizations[0]?.type === 'enterprise'
      setIsEnterpriseUser(isEnterprise)
      setIsEmailVerified(userMetadata.is_email_verified)

      if isEnterpriseDomain() && !isEnterprise:
        clearAllAuthCookies()
        if not on a public route: redirect to /login
        return
      if !isEnterpriseDomain() && isEnterprise:
        clearAllAuthCookies()
        if not on a public route: redirect to /enterprise/login
        return

      setUser({ tarjimly_id })
    else:
      handleAuthError()   // clears cookies, does NOT redirect if already on a public route
```

This means **every page load makes a round trip to `/api/mobile/v2/users/metadata`** before the app knows who's logged in — there's no client-side JWT decode or cached-claims shortcut. `middleware.ts` only checks for the *presence* of the `connect.sid` cookie (not its validity) before allowing a request past the coarse redirect checks — actual validation happens in this client-side `getUserMetadata()` call, which fails against the real backend if the cookie is invalid or expired.

---

## Path 4 — Password Reset

Two near-identical flows, both hitting the **`/api/mobile/v2` prefix** (not `/v3`, unlike login):

```
Request reset:
  POST /api/mobile/v2/auth/reset-password   { email }
  header: host: getApiHost()   (so the backend knows which tenant's reset email to send)

Submit new password:
  POST /api/mobile/v2/auth/new-password     { token, password, request_id: requestId }
  header: host: getApiHost()
```

`/forgot-password` and `/reset-password` (regular) vs `/enterprise/reset-password` (enterprise) are separate pages but call the exact same `requestPasswordReset` / `submitPasswordReset` functions — the only per-tenant difference is which page renders and the `host` header value. There is no password-reset auto-registration behavior (unlike TWB's `getAuthCode`) — reset only works for existing accounts.

---

## Email Verification Gate

Handled entirely client-side, inside `ProtectedRoute` (`lib/auth/ProtectedRoute.tsx`):

```
EmailVerificationCheck (runs on every protected page):
  if not loading and authenticated:
    if !isEmailVerified and pathname is not /verify-email or /blacklisted:
      redirect to /verify-email
    if isEmailVerified and pathname is /verify-email:
      redirect to /
```

`/verify-email` itself presumably offers a resend action (`resendVerificationEmail()`, `GET /api/mobile/v2/users/resend-verification-email`) and a way to re-check status (`refreshEmailVerificationStatus()` in `AuthContext`, which just re-fetches `getUserMetadata()`). There is no client-side polling for verification completion visible in the traced files — the user must trigger a refresh (e.g. by navigating) after clicking the email link.

---

## Blacklist Gate

Also client-side, also inside `ProtectedRoute`, running in parallel with the email-verification check:

```
BlacklistCheck (runs on every protected page):
  { data } = useSWR('/api/blacklist-status', getBlacklistStatus)
  if data.isBlacklisted:
    redirect to /blacklisted
```

`/blacklisted` itself re-checks status on mount and bounces non-blacklisted users back to `/login` — it's not a static "you're banned" page, it's a live-checked gate. As with email verification, there is no evidence in this repo that blacklist status is re-validated by the backend on every API call — if that's true only of this client-side check, a user could theoretically continue issuing API calls directly (bypassing the UI) unless the backend independently enforces the block. This should be confirmed against the backend, not assumed.

---

## Cookie & Token Mechanics — Detail

Three cookies drive all client-side routing decisions, all set via plain `document.cookie` (`js-cookie` or raw string), none `HttpOnly`:

| Cookie | Set by | Value | Purpose |
|---|---|---|---|
| `connect.sid` | `setAuthCookie()`, parsed from the login response body | Opaque backend session token (Express-session format) | The actual credential sent on every proxied API call (`credentials: 'include'`) |
| `user_type` | `setUserTypeCookie()` | `'enterprise'` \| `'regular'` | UI routing only — decides which login/redirect logic applies |
| `regular_user_type` | `setRegularUserType()` | `'aidworker'` \| `'translator'` | UI routing only — decides post-login redirect target |

`middleware.ts` reads all three cookies directly (`request.cookies.get(...)`) to make its coarse routing decisions (protected-enterprise-route bounce, missing-user-type redirect to login) **before** any React code runs — this is the only server-adjacent (edge middleware) auth check in the app, and it only checks *presence*, never validity.

**Security note:** because `connect.sid` is written by client JavaScript rather than arriving as a `Set-Cookie: HttpOnly` response header, it is readable (and writable) by any script running on the page — including a successful XSS payload. This is a meaningfully different risk posture than a conventional session-cookie setup, and worth flagging explicitly if Tarjimly's auth is ever consolidated with TWB's (which at least sets its session cookie server-side, even though its encryption scheme has its own separately-flagged issues — see the TWB audit's `login.md` and `login-unification.md`).

---

## Firebase Custom-Token Sign-In — Detail

Immediately after a successful backend login (in both regular and enterprise flows), the client calls:

```ts
await signInWithCustomToken(auth, response.firebase_token)
```

This establishes a **separate** Firebase Auth session, used exclusively so Firestore security rules can authorize real-time reads/writes (active call/session documents). It is not used for backend API authorization — the `connect.sid` cookie remains the credential for all `/api/*` calls. The two identities (backend session, Firebase session) are provisioned together at login but have independent lifecycles: nothing in the traced code re-establishes the Firebase session if it expires mid-app-use without a full re-login, and a Firebase-setup failure at login time is caught and swallowed (logged, not surfaced to the user).

---

## Integration Assessment (for a merge with TWB Platform)

### Option A — Keep both auth systems, bridge at the edge

Tarjimly's `connect.sid` + Firebase pair and TWB's encrypted `slim_session` cookie are structurally incompatible (opaque Express session vs. hand-rolled AES-CBC payload) and neither is a standards-based token today. A merge would need a translation layer regardless of which system "wins."

### Option B — Adopt TWB's proposed GIP (per `TWB Repo Audit/login-unification.md`)

That proposal already anticipates federated per-platform user IDs via custom OIDC claims. For Tarjimly specifically, migrating to a GIP-issued token would mean:
- Replacing the client-JS-set `connect.sid` cookie with a proper `HttpOnly`, server-set session (fixing the XSS-exposure issue as a side effect).
- Deciding whether Firebase custom-token sign-in continues to exist as a Tarjimly-specific side channel (likely yes, since it's tied to Firestore, not to primary auth) — GIP would need to still hand back a `firebase_token`-equivalent, or Tarjimly's backend would mint one after establishing the user's identity via GIP.
- Reconciling `TarjimlyRole` (`aidworker`/`translator`) and `OrganizationTypeEnum` (`non_profit`/`enterprise`) with TWB's role bitmask (`LINGUIST`/`NGO_ADMIN`/etc.) — there's a rough conceptual overlap (translator ≈ LINGUIST, aidworker's enterprise-org staff ≈ NGO_ADMIN/PROJECT_OFFICER) but no existing mapping.
- Deciding how host-based tenancy (`REGULAR_HOSTS` allowlist) interacts with GIP's custom claims — today tenancy is inferred purely from the request's `Host` header, independent of anything in the user's token.

### Option C — Leave both auth systems standalone, merge only at the UI/product layer

If the merge is primarily about presenting a unified product experience (e.g., the "hybrid layout" HTML work already underway in `branding-guidelines/`) rather than a unified backend, no auth changes are strictly required yet — but any shared session or single-sign-on expectation between the two apps would need one of the above eventually.

---

## Recommendations

1. **Fix the non-`HttpOnly` session cookie** regardless of merge plans — this is a standalone security issue. The backend should set `connect.sid` via a proper `Set-Cookie: HttpOnly; Secure; SameSite=Lax` header on the login response rather than returning it as a JSON string for the client to parse and reassign.
2. **Confirm backend-side enforcement of email verification and blacklist status.** If these are *only* checked client-side today, a direct API caller (mobile app reverse-engineering, or a script) could bypass both gates entirely.
3. **Consolidate `/api/mobile/v2` and `/api/v3` auth endpoints** before or during any merge — having login on `/v3` but logout and password-reset only on `/v2` is a maintenance hazard independent of the TWB merge.
4. **Map `TarjimlyRole`/`OrganizationTypeEnum` to TWB's role bitmask** early in merge planning — this is the Tarjimly-side equivalent of TWB's role table and will be needed for any unified identity or permissions model.
5. **Decide Firebase's fate in a merged identity system** before implementing GIP — it's currently load-bearing for real-time features (not just an optional analytics add-on), so it can't simply be dropped in favor of a generic OIDC token.
