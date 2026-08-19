# Advisory: Unified Login via GIP for TWB Platform & Tarjimly

**Context:** TWB Platform login flow analysed in `login.md`. This advisory evaluates the proposal to migrate both platforms to a central Global Identity Platform (GIP).

---

## The Proposal (as understood)

1. GIP becomes the canonical identity store; TWB migrates its users there.
2. Users with the same email across TWB and Tarjimly are prompted to reset their password on next login, with a clear message that this becomes a shared credential. Google login offered as an alternative.
3. On login or signup via either system, that system writes its own internal user ID back into the GIP user record ("user payload").
4. Existing GIP users hit an onboarding screen on first access to either platform to supply any platform-specific required fields — OR each platform silently pulls what it needs via a GIP profile API.

**Short verdict:** The direction is right. The proposal covers the UX surface well but leaves several technical load-bearing questions unanswered. The risks are manageable if addressed before implementation begins.

---

## What the Proposal Gets Right

### Password migration via reset is the only viable option

TWB's passwords are hashed with `hash_hmac('sha512', password + per_user_nonce, site_key)` — a site-wide HMAC key means the hashes cannot be verified by any system that doesn't have that key, and cannot be safely exported. You cannot bulk-migrate TWB passwords to GIP. Prompting users to reset on first login is the correct call. It is also a natural moment to communicate the change in a trust-building way.

### Federated identity with per-platform user records is the right architecture

Storing each platform's internal user ID back in GIP ("add to the user payload") is the standard federated identity pattern — GIP is the IdP, each platform keeps its own user table indexed by a `gip_user_id` foreign key. This is how Login-with-Google works: Google owns the identity, each app owns its own user record. It is clean, well-understood, and avoids a single monolithic user object that both platforms must agree on.

### Google login already works in TWB

The Google Identity Services flow is already live in TWB Platform. Any user who currently uses Google sign-in will have a smooth migration path — their identity is already Google-managed. They just need to be told that GIP now mediates it.

---

## Gaps and Risks That Need Resolving

### 1. What protocol does GIP speak? (The most important question)

The proposal does not specify this, but everything else depends on it. The right answer is **OIDC (OpenID Connect)** on top of OAuth2. Reasons:

- It supports **custom claims** in the ID token, which is the mechanism for storing `twb_user_id` and `tarjimly_user_id` in the user payload.
- TWB already has an OAuth2 layer internally (league/oauth2-server). An OIDC-based GIP replaces it rather than sitting on top of it.
- TWB already has a SAML `ReturnTo` hook. OIDC is simpler to implement than SAML and is the modern standard for new work; SAML can still be added later for enterprise partners if needed.
- Keycloak, Auth0, and Okta all speak OIDC out of the box if GIP is being built on one of those.

If GIP is a custom build, implement OIDC. Do not invent a bespoke protocol.

### 2. TWB's post-login onboarding gates are platform-specific and must stay that way

TWB has a strict multi-step onboarding sequence that GIP cannot own:

| Gate | What it checks | Where it lives |
|---|---|---|
| Email verification | `isUserVerified()` | TWB DB, set by clicking verification email link |
| Profile completion | `profile_completed == 2` | TWB DB; blocks most navigation until name, native language, qualified pairs are supplied |
| Code of Conduct acceptance | `terms_accepted` (1 = pending, 3 = full) | TWB DB; new Google/GIP users start at 1 and are routed to `googleregister` before access |
| Language pair qualifications | `UserQualifiedPairs` | TWB DB; required before claiming tasks |

GIP can only confirm that a user is who they say they are. It cannot satisfy these gates. After GIP authenticates a user, TWB must still run its own onboarding check and gate the user accordingly. This means the "invisible API pull" option (option B in the proposal) needs to be designed carefully — pulling profile data from GIP is fine for name/email, but TWB's qualification and consent records will always be TWB-owned.

**Practical consequence:** A user who is fully registered in GIP and Tarjimly will still hit TWB's profile completion flow on first TWB access. This is unavoidable and should be communicated as "you need to add your language qualifications to access translation tasks" — not as a bug.

### 3. The `getAuthCode` endpoint auto-creates TWB accounts — this needs a gate

In the current flow, the API's `getAuthCode` endpoint creates a new TWB account for any email it receives that doesn't exist yet (`apiRegister(email, md5(email), false)`). If GIP starts routing all users through this endpoint, every GIP user — including Tarjimly users who have never signed up for TWB — will silently get a stub TWB account.

This is probably not what you want. Before migration, add an explicit check: if the email arrives via GIP and no TWB account exists, route to a TWB registration/onboarding page rather than auto-creating a stub. The auto-create was designed for the Google Sign-In flow where account creation is an expected outcome; it is a side-effect risk in a wider GIP context.

### 4. "Adding user ID to the payload" needs a concrete data model

The proposal says each platform writes its user ID into the GIP user record. In OIDC terms, this means **custom claims** on the GIP user object. The shape would be something like:

```json
{
  "sub": "gip-uuid-1234",
  "email": "volunteer@example.com",
  "given_name": "Maria",
  "family_name": "Santos",
  "twb_user_id": 8821,
  "tarjimly_user_id": "tj-4499"
}
```

Questions to answer before building:
- **Who can write these claims?** Only the respective platform's backend service, via GIP's management API, after the platform's own registration is complete. Not the browser.
- **When are they written?** After TWB creates (or locates) its own user record, it calls GIP's API to set `twb_user_id`. This is a backend-to-backend call, not user-facing.
- **What if the claim isn't there yet?** On first login to TWB from GIP, `twb_user_id` won't exist in the token yet. TWB must handle a missing claim gracefully (create the user, then write the claim back).
- **Are these claims in the ID token or only in the userinfo endpoint?** For performance, `twb_user_id` should be in the ID token so TWB doesn't need an extra API call on every login.

### 5. Dual-account detection needs to be precise

The proposal assumes "same email = same person." This is almost always true but has edge cases:

- A user who registered in TWB with a work email and Tarjimly with a personal email — these are two different people in the system even though they are one physical person. GIP cannot merge them without explicit user action.
- A shared NGO email address used by multiple staff — merging these would be a security incident.

Recommendation: the password-reset + "shared login" prompt should trigger only when the **same email** exists in both TWB and GIP/Tarjimly. Do not attempt to match by name or any fuzzy field.

### 6. Session management across the boundary

TWB's current session is an AES-128-CBC encrypted cookie (`slim_session`) storing `user_id` and the OAuth access token. After GIP authenticates a user and redirects them back to TWB, TWB still needs to issue its own session cookie. The existing SAML `ReturnTo` redirect mechanism handles the browser hand-off, but TWB's `setSession($user_id)` and `setAccessToken($oauthResponse)` must still fire locally. This is not a blocker — it is just a reminder that GIP handles authentication; TWB handles its own session independently afterwards.

When TWB migrates to Next.js, replace the custom session cookie with NextAuth/Auth.js, which has first-class OIDC provider support and handles the GIP integration cleanly.

### 7. The internal TWB UI→API OAuth layer is a separate concern

TWB currently uses OAuth2 internally between its UI app and its API app (the confusing loop described in `login.md`). GIP sits above this — it authenticates the user at the UI layer. The internal UI→API OAuth layer is an implementation detail that GIP does not replace or interact with. During the Next.js migration this internal OAuth layer will be removed anyway (server components call services directly), which simplifies the picture significantly.

---

## Recommended Implementation Sequence

**Step 0 — Define GIP's OIDC contract before writing any code**

Agree on:
- The OIDC issuer URL and discovery document
- Custom claim names (`twb_user_id`, `tarjimly_user_id`)
- Which grants are supported (authorization_code with PKCE is the correct choice for web apps; device flow for mobile)
- The management API shape for writing custom claims back from each platform

**Step 1 — Add GIP as a login option in TWB (additive, no migration yet)**

Add a "Sign in with GIP" button alongside the existing Google and password options. New GIP-authenticated users go through TWB's normal onboarding. Existing users are unaffected. This lets you prove the integration works in production before any migration.

**Step 2 — Migrate Google-authenticated TWB users to GIP (lowest risk)**

Users currently logging in via Google are already using an external IdP. Re-pointing their sign-in from `accounts.google.com` (via Google Identity Services) to GIP (which itself may use Google as a backing IdP) is transparent to them. This is the largest single segment to migrate with the least disruption.

**Step 3 — Password-reset migration for email/password users**

On next password-login attempt: detect that the user has no GIP account, pause, show the "your login is moving to a shared account" screen, force a password reset that creates the GIP account, and on completion log them into TWB via GIP. Do not bulk-force-reset everyone simultaneously — that generates a support spike.

**Step 4 — Write `twb_user_id` back to GIP for all migrated users**

Once a user's TWB account is linked to a GIP identity, make the backend API call to set `twb_user_id` in GIP. From this point, TWB can identify a returning GIP user by the claim without a DB lookup by email.

**Step 5 — Deprecate direct TWB password login**

Once the migration percentage is high enough (monitor the `UserLogins` table for login method distribution), stop accepting direct email/password login and require GIP. Keep a staff bypass for operations access.

---

## Summary Table

| Proposal element | Assessment | Action needed |
|---|---|---|
| GIP as central IdP | Correct direction | Specify OIDC as the protocol |
| Password reset on migration | Only viable option for TWB's hashing scheme | Implement with clear UX messaging; do not bulk-reset |
| Google as alternative | Already works in TWB | Wire Google through GIP rather than directly |
| Each platform writes its user ID to GIP payload | Sound federated identity pattern | Define custom OIDC claim names and write-back API before building |
| TWB onboarding on first GIP access | Will still be required | Design onboarding screen for GIP-authenticated new TWB users; do not assume GIP profile is sufficient |
| Invisible API profile pull | Viable for name/email | Language qualifications, CoC acceptance, profile completion stay TWB-owned forever |
| `getAuthCode` auto-registration | Side-effect risk | Gate it: unknown GIP users should go to explicit onboarding, not silent auto-create |
| SAML `ReturnTo` hook | Still works as the redirect hand-off mechanism | Can remain in place; for new work use OIDC redirect rather than the SAML-style parameter |
