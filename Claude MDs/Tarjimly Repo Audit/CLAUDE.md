# Tarjimly Web — Developer Context

This workspace contains the `tarjimly-web` repo: the Next.js frontend for **Tarjimly**, an on-demand interpretation and translation platform. Unlike the TWB Platform audit (a monolith with its own DB and business logic), **this repo is a thin client** — nearly all business logic, auth, and data live in an external backend that is not in this repository. Detailed audit and analysis documents are on disk:

- `audit-report.md` — full architecture audit (2026-08-19)
- `login.md` — deep-dive on the login flow (regular, enterprise, password reset, email verification, blacklist)

---

## The One Thing to Understand First

`tarjimly-web` does not own its data model or its auth. It is a Next.js 15 (App Router) presentation layer that:

1. Proxies almost every `/api/*` call straight through to an external Node/Express backend (`tarjim.ly` for regular users, `us.tarjim.ly` for enterprise), via a rewrite in `src/middleware.ts`.
2. Uses Firebase (Auth custom tokens + Firestore) as a **side channel** for real-time state — active call/session status — separate from the backend's own session cookie.
3. Has no database, no ORM, no schema file, and no server-side business logic of its own beyond one local route (`/api/upload`, which writes to Firebase Storage).

Any "data model" documented here is **inferred from TypeScript interfaces in the client code**, not verified against a real schema — the source of truth is the external backend repo, which is out of scope for this audit.

---

## Repo Structure

```
tarjimly-web/
  src/
    app/
      login/, forgot-password/, reset-password/, verify-email/     ← regular auth pages
      enterprise/login/, enterprise/reset-password/                 ← enterprise auth pages
      blacklisted/, maintenance/                                    ← gate pages
      call_request/, active_call/, meeting/[sessionId]/             ← real-time interpretation (LiveKit)
      translator/jobs/                                              ← translator's async job queue
      user/jobs/, user/spark/                                       ← aidworker's async jobs + AI-assist ("Spark")
      user/billing/, user/subscription/, user/manage/               ← Stripe billing & org management
      enterprise/jobs/, enterprise/language-interpretation/         ← enterprise-tenant equivalents
      api/upload/                                                   ← the ONLY real server route in this repo
      requests.ts                                                   ← shared types + fetch wrappers (the closest thing to a "model layer")
      hooks.ts                                                      ← shared SWR hooks
      client-layout.tsx                                             ← picks Regular vs Enterprise shell by host
      middleware.ts                                                 ← API proxy + auth/tenancy routing (see below)
    components/ui/            ← shadcn/Radix component library
    lib/auth/                 ← AuthContext, ProtectedRoute, auth-utils, cookies
    lib/firebase.ts           ← client Firebase (Firestore + Auth)
    firebase/firebaseAdmin.ts ← server-side Firebase Admin (Storage uploads only)
    posthog/, logrocket/, statsig.tsx  ← analytics/observability providers
  proxy/conf.d/               ← nginx config (self-hosted reverse proxy option; prod uses Vercel)
  docker-compose.yml          ← local nginx proxy only, no app/db services
```

There is **no** `SOLAS-Match`-style backend counterpart in this repo. The backend (`tarjim.ly` / `us.tarjim.ly`) is a separate, unaudited codebase.

---

## Stack

- **Next.js 15.5** (App Router), **React 18.3**, **TypeScript**, **Tailwind 3**
- **shadcn/ui** on **Radix UI** primitives (`components.json`, `class-variance-authority`, `tailwind-merge`) — this is the closest analogue to TWB's Bootstrap 5 design system
- **SWR** for all data fetching/caching (no React Query, no Redux)
- **react-hook-form + zod + @hookform/resolvers** for forms
- **Firebase** (`firebase` client SDK + `firebase-admin`): Auth (custom-token sign-in only, no email/password via Firebase), Firestore (real-time session/call state), Storage (document uploads)
- **LiveKit** (`livekit-client`, `@livekit/components-react`, Krisp noise filter) — current audio/video call provider
- **Statsig** — feature flags (`ff_sponsored_plan_web`, `ff_hotline_plan_web`) + product analytics autocapture
- **PostHog** and **LogRocket** — additional analytics / session replay
- **Vercel** — hosting, CI/CD, env var management (branch-based: `master`→prod, `staging`→staging, feature branches→preview)

No server framework, no ORM, no test runner is configured in this repo.

---

## Two Parallel Product Domains

Tarjimly does two structurally different things, and the codebase keeps them mostly separate:

### 1. Real-time interpretation ("Sessions")
`call_request/` → `active_call/` → `meeting/[sessionId]/`. An aidworker requests a session (`SessionRequestEntity`, status `open`|`matched`|`cancelled`|`expired`|`rejected`), a translator/interpreter accepts it, and both join a call via LiveKit (legacy `dyte` provider still referenced in types but being phased out). Real-time state (`isActive`, `participants`, `callProvider`) lives in **Firestore**, polled/subscribed client-side; the request/accept lifecycle lives in the external backend via SWR-polled REST endpoints (10s refresh interval, no websockets).

### 2. Async translation jobs
`user/jobs/` (aidworker) and `translator/jobs/` (translator) and `enterprise/jobs/` (enterprise tenant). A `TranslationJob` (text or document) gets an `initialAITranslation` (via the "Spark" AI feature, `POST /api/v3/aidworkers/me/translate`), then a human `Submission` with structured `feedback`/`issues`. Supports NDA and Certificate-of-Translation (CoT) document attachments.

Billing (`user/billing/`, `user/subscription/`) is Stripe-backed (`StripeProductNames`: `premium_monthly`, `premium_yearly`, `byot_monthly`, `sponsored_monthly`, `hotline`) but all Stripe calls happen server-side in the external backend — this repo only reads subscription status.

---

## Tenancy: Regular vs Enterprise

There is no separate enterprise app — the **same Next.js deployment** serves both, branching on HTTP `Host` header:

- `src/lib/auth/auth-utils.ts` — `REGULAR_HOSTS` is a hardcoded allowlist (`app.tarjimly.org`, `app.tarjim.ly`, `localhost`, staging variants). Anything **not** in that set is treated as an enterprise white-label domain.
- `client-layout.tsx` picks `RegularLayout` or `EnterpriseLayout` based on this check.
- `middleware.ts` proxies `/api/*` to `tarjim.ly` (regular) or `us.tarjim.ly` (enterprise) accordingly — **different backend hosts entirely**, not just different paths.

**Any new domain added for a merged platform must be added to `REGULAR_HOSTS`, or it will silently be routed as an unrecognized enterprise host.**

---

## Auth Model (see `login.md` for the full flow)

- Session token cookie is literally named `connect.sid` (Express-session convention) — confirms the backend is Node/Express, not the login-page's own concern.
- The cookie is **not** set via a standard `Set-Cookie` header from a cross-origin call — the backend returns it as a string in the JSON response body, and client JS (`src/lib/cookies.ts`) parses it and writes it via `document.cookie`. This means the session cookie is **not `HttpOnly`** and is readable by any JS running on the page.
- Two more cookies, `user_type` and `regular_user_type`, are also client-set and used **only** for UI routing (which login page/redirect to show) — they are not a security boundary and must never be trusted as one in a merged system.
- A Firebase custom-token sign-in happens immediately after backend login, giving the client a Firebase identity (`signInWithCustomToken`) used for Firestore security rules on real-time session data — this is entirely separate from the `connect.sid` session.
- Two API path prefixes exist with overlapping responsibilities: `/api/mobile/v2/*` (older, shared with the native mobile app — e.g. `auth/logout`, `auth/reset-password`) and `/api/v3/*` (newer, web/enterprise-first — e.g. `auth/login`, `auth/enterprise-login`). `getUserMetadata` and password-reset logic exist on both; a merge should pick one as canonical.

---

## Conventions

- **Types + fetch wrappers live together** in a `requests.ts` (or `requests.tsx` if it needs JSX-adjacent types) per route folder — e.g. `src/app/login/requests.tsx`, `src/app/call_request/requests.tsx`. This is the de facto "DAO layer."
- **Data fetching is SWR-only**: `hooks.ts` per route folder wraps `requests.ts` calls in `useSWR`. Polling intervals (commonly 10000ms) substitute for websockets on session/job status.
- **Route-gating is client-side**, not server-side: `ProtectedRoute` (in `lib/auth/ProtectedRoute.tsx`) checks auth, email verification, and blacklist status in `useEffect` hooks and redirects — there is no server-side session check beyond the coarse cookie-presence check in `middleware.ts`.
- **Feature flags** via Statsig gate whole product surfaces (e.g. `ff_sponsored_plan_web`) — check `useSubscriptionFeatureFlags()` before assuming a plan/feature is universally available.
- **Component library**: shadcn primitives live in `src/components/ui/`; feature components live next to their route under `<route>/components/`.

---

## Known Landmines

- **Non-`HttpOnly` session cookie.** `connect.sid` is set by client JS from a value in the response body, not a proper `Set-Cookie` header from the API. Any XSS on this app can read and exfiltrate the session token. This is a materially different (and arguably worse) risk profile than TWB's own hand-rolled-but-server-set session cookie.
- **`user_type` / `regular_user_type` are not trust boundaries.** They're client-writable and only drive which page renders — real authorization must happen (and is assumed to happen) on every backend API call, independent of these cookies.
- **The real data model is off-repo.** Every interface in `requests.ts` files is a best-effort client-side guess at the external backend's shape. Treat this repo's "data model" as documentation of the *contract*, not the *schema* — verifying it requires the backend repo.
- **Two overlapping API surfaces** (`/api/mobile/v2/*` and `/api/v3/*`) with duplicate endpoints for the same concern (e.g. user metadata). Don't assume `/v3` fully supersedes `/v2` — some flows (logout, password reset submission) currently only exist on `/v2`.
- **Tenancy is a hardcoded host allowlist**, not a config value or DB lookup (`REGULAR_HOSTS` in `auth-utils.ts`). Adding a merged-platform domain requires a code change here.
- **Legacy call provider `dyte`** still appears in `FirebaseSessionEntity.callProvider` alongside `livekit`. Confirm it's fully retired before designing a merged calling experience around LiveKit only.
- **Uploaded files are made public unconditionally.** `uploadFileToFirebaseAdmin` in `firebase/firebaseAdmin.ts` calls `fileRef.makePublic()` on every upload (including NDAs and translation source documents), relying only on an unguessable UUID in the path for privacy. No signed URLs, no expiry, no access list.
- **Both `package-lock.json` and `yarn.lock` are committed** — pick one canonical package manager before onboarding new contributors to avoid dependency drift.
- **No `.env.example`, no test suite.** Environment variables are documented only in `README.md` and live in the Vercel dashboard; there's nothing in-repo to diff a new environment against.
