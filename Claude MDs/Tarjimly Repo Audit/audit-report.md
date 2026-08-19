# Tarjimly Web — Codebase Audit Report

**Date:** 2026-08-19
**Repository audited:** `tarjimly-web` (commit `ba0929e`, branch `claude/twb-tarjimly-merge-analysis-eh7jlw`)

> **Scope note:** Tarjimly's actual backend (`tarjim.ly` / `us.tarjim.ly`) — where the database, business logic, and auth server live — is **not** in this repository and was not available to audit. Everything below describes the Next.js client: what it renders, how it talks to the backend, and what the backend's data contracts appear to be, inferred from TypeScript interfaces in the client code. Anywhere this report describes "the data model," read it as "the client's assumed shape of the data," not a verified schema.

---

## Table of Contents

1. [Repository Overview](#1-repository-overview)
2. [Technology & Dependency Map](#2-technology--dependency-map)
3. [Functional Documentation](#3-functional-documentation)
4. [Route Map](#4-route-map)
5. [Data Model (Inferred)](#5-data-model-inferred)
6. [Authentication & Tenancy](#6-authentication--tenancy)
7. [External Integrations](#7-external-integrations)
8. [Merge-Relevant Risks & Gotchas](#8-merge-relevant-risks--gotchas)
9. [Files Inspected](#9-files-inspected)

---

## 1. Repository Overview

`tarjimly-web` is the web frontend for **Tarjimly**, an interpretation/translation platform serving two populations: displaced people or aid-seekers needing language help ("aidworker" role — despite the name, this is the person requesting help, not TWB-style NGO staff) and volunteer/professional translators. It also serves **enterprise** customers (white-labeled tenants) on separate domains.

Unlike TWB Platform / SOLAS-Match (a self-contained PHP+C++ monolith with its own MySQL database), `tarjimly-web` is a **Next.js 15 App Router client** with:

- No database, no ORM, no server-rendered business logic beyond one file-upload route.
- Almost all `/api/*` traffic transparently proxied (via `middleware.ts`) to an external Node/Express backend at `tarjim.ly` (regular) or `us.tarjim.ly` (enterprise).
- A side-channel to **Firebase** (Auth custom tokens, Firestore, Storage) for real-time state and file uploads, independent of the backend's own session mechanism.
- Shared API surface with a native mobile app (`/api/mobile/v2/*` naming makes this explicit).

Deployment is via **Vercel's GitHub integration**: `master` → production (`app.tarjimly.org`, `enterprise.tarjimly.org`), `staging` → staging, feature branches → preview deployments. No manual deploy step; environment variables live in the Vercel dashboard, not in the repo.

---

## 2. Technology & Dependency Map

### Framework & language
- **Next.js 15.5.18** (App Router, `src/app/`), **React 18.3.1**, **TypeScript 5**, **Tailwind CSS 3.4**
- **ESLint 9** (`eslint-config-next`)

### UI
- **shadcn/ui** convention (`components.json`) over **Radix UI** primitives (`@radix-ui/react-*`: dialog, dropdown, select, tabs, toast, tooltip, avatar, checkbox, popover, progress, radio-group, switch, toggle, separator, label)
- `class-variance-authority`, `clsx`, `tailwind-merge`, `tailwindcss-animate` — standard shadcn styling stack
- `lucide-react` icons, `cmdk` command palette, `lottie-react` for animations, `react-phone-number-input` for phone fields

### Data & forms
- **SWR 2.3** — the only data-fetching library; no React Query, no Redux/Zustand
- **react-hook-form 7.54** + **zod 3.24** + **@hookform/resolvers** for validated forms
- **js-cookie** for reading/writing the app's own (non-HttpOnly) cookies

### Real-time / calling
- **LiveKit** (`livekit-client` 2.10, `@livekit/components-react` 2.9, `@livekit/components-styles`) — current SFU-based audio/video provider
- **`@livekit/krisp-noise-filter`** — AI noise suppression for calls
- Legacy **Dyte** provider still present in the `FirebaseSessionEntity.callProvider` type union (`"dyte" | "livekit"`) — status of full retirement unconfirmed from this repo alone

### Firebase
- **`firebase` 11.9** (client SDK): `firebase/app`, `firebase/firestore`, `firebase/auth` — used for real-time session-state documents and custom-token sign-in
- **`firebase-admin` 13.4** (server SDK, used only in `src/firebase/firebaseAdmin.ts` and the `/api/upload` route): Storage uploads

### Analytics / observability
- **PostHog** (`posthog-js`) — product analytics, wraps the whole app in a `PostHogProvider`
- **Statsig** (`@statsig/react-bindings`, `@statsig/web-analytics`) — feature flags + autocapture analytics, scoped by `tarjimly_id`
- **LogRocket** (`logrocket`) — session replay, identified by `tarjimlyID`/email/role

### Other notable deps
- `uuid`, `date-fns`, `zod` — utility
- No test runner (`jest`, `vitest`, `playwright`, etc.) is configured
- No `.env.example` — env vars are documented only in `README.md`
- **Both `package-lock.json` and `yarn.lock` are committed** (added together in the single initial commit) — the canonical package manager is ambiguous from the repo alone

### Local infra
- `docker-compose.yml` defines only an `nginx` reverse-proxy service (`proxy/conf.d/tarjimly-web.conf`) for local HTTPS termination — there is no app or database service in the compose file, consistent with the app having no local backend.

---

## 3. Functional Documentation

### 3.1 Platform Summary

Tarjimly connects people needing language assistance ("aidworkers" in the codebase's naming — this is the *help-seeker*, not TWB-style NGO staff) with volunteer or professional translators/interpreters, in two modes:

1. **Real-time interpretation** — an on-demand phone/video call with a live interpreter (LiveKit-based), for situations needing immediate spoken communication.
2. **Asynchronous translation jobs** — text or document translation, optionally AI-assisted first-pass ("Spark"), then reviewed/completed by a human translator.

**Enterprise** customers get the same product on a white-labeled domain, with an org-scoped dashboard for creating and tracking translation jobs and interpretation requests, plus billing/subscription management for their staff.

### 3.2 User Roles

Roles are far simpler than TWB's 8-role bitmask — there are two base roles plus a tenancy dimension:

| Role (`TarjimlyRole`) | Who | Primary routes |
|---|---|---|
| `aidworker` | Person requesting translation/interpretation (individual or enterprise staff) | `/user/jobs`, `/user/billing`, `/user/spark`, `/call_request` |
| `translator` | Volunteer or professional translator/interpreter | `/translator/jobs` |

Orthogonal to role: **organization type** (`OrganizationTypeEnum`: `non_profit` | `enterprise`), which determines whether a user is routed to the regular app or the enterprise app (`/enterprise/*`), and **org role** within an organization (`OrganizationMember.orgRole`: `user` | `admin`), which gates billing/member-management pages.

### 3.3 Session (Real-Time Interpretation) Status

`SessionRequestStatus`: `open` → `matched` → (call happens) → closed, or `cancelled` / `expired` / `rejected` along the way. A matched request becomes an `ActiveSessionEntity` with `participants[]` and drives the `/active_call` and `/meeting/[sessionId]` UI. Real-time fields (`isActive`, `participants`, `callProvider`) live in a Firestore document, not the REST API — polled/subscribed directly from the client.

### 3.4 Translation Job Status

Async jobs (`TranslationJob`) move through an `AidworkerTranslationJobStatus` enum (defined in `translation-job-row.tsx`, not fully enumerated in this pass) from creation → optional AI first-pass (`initialAITranslation`) → human `Submission` → completion, with structured `feedback.issues[]` for revision requests. Jobs can carry `additionalData` for NDA and Certificate-of-Translation (CoT) documents, both regular- and template-variant URLs.

### 3.5 End-User Workflows

#### A. Aidworker — request a live interpreter

1. Log in at `/login`. Redirect target depends on role and domain (see §6).
2. Land on `/user/jobs` (or `/call_request` via the floating "request translator" button present on all sidebar screens).
3. Fill in session preferences: medium (`tarjimly-call` vs `tarjimly-video`), translator type (`volunteer` vs `professional`), topic(s), and filters (documents, phone interpret, female-only, certified, urgent, "my organization" only, no-record, longer session, NDA required, certificate-of-translation required, or a specific translator by UID — "BYOT").
4. On submit, an open `SessionRequestEntity` is created and polled every 10s (`useOpenSessionRequestsForSelf`) until a translator accepts (`matched`) or it expires/is cancelled.
5. Once matched, `useActiveSessionCheck()` (running globally in the regular layout) redirects to `/active_call` automatically from anywhere in the app except the login/verify-email pages.
6. In `/active_call`, the client fetches a LiveKit token (`joinActiveSessionMeetingRequest`) and joins the call. A third-party participant can be added by phone number (`AddParticipantToMeetingRequestBody`, LiveKit-only).
7. After the call, a session-rating modal (`SessionRatingsModal`, rendered globally) prompts for feedback (`CreateSessionRatingRequestBody`).

#### B. Translator — respond to interpretation requests / async jobs

1. Log in, land on `/translator/jobs`.
2. Async jobs: view assigned/available `TranslationJob`s, open `/translator/jobs/[id]` to submit a translation, see status badges and completion modals.
3. Real-time requests: presumably surfaced via a similar open-request polling mechanism (this pass did not trace the translator-side session-accept UI in full detail).

#### C. Aidworker — async translation job with AI assist ("Spark")

1. From `/user/jobs`, create a translation job (text or document) via a multi-step form (`multi-step-translation-form.tsx`), specifying source/target language and topics.
2. `/user/spark` offers direct AI-assisted translation (`translateWithAI`, hits `POST /api/v3/aidworkers/me/translate`) with a human-review modal step before finalizing — a lighter-weight path than the full job/submission workflow.
3. Job detail page (`/user/jobs/[id]`) shows the AI first-pass, then the human `Submission` once a translator completes it, with structured feedback if revisions were requested.

#### D. Enterprise admin — org-scoped jobs and interpretation

1. Log in at `/enterprise/login` (on any non-regular-host domain; `/login` transparently rewrites to this on enterprise hosts).
2. Land on `/enterprise/language-interpretation` — a dashboard for requesting/tracking live interpretation on behalf of the org.
3. `/enterprise/jobs` — org-scoped async translation job list, each with the same AI-first-pass + human-submission + structured-feedback pattern as the regular flow (`enterprise/jobs/[id]/components/translation-feedback.tsx`).
4. Org billing/subscriptions are managed under `/user/billing/[orgID]` and `/user/manage/[orgID]` (add/remove members, assign subscriptions) even though these routes live under the `/user/` tree rather than `/enterprise/` — a naming inconsistency worth flagging for anyone extending the enterprise surface.

### 3.6 System Architecture

```
Browser
  │  Next.js 15 App Router (client + server components), Tailwind, shadcn/Radix
  │  HTTPS
  ▼
tarjimly-web (Vercel)
  │
  ├── middleware.ts
  │     ├── maintenance-mode short-circuit (env flag)
  │     ├── rewrites /api/* (except /api/upload) → https://tarjim.ly or https://us.tarjim.ly
  │     │     depending on Host header (REGULAR_HOSTS allowlist)
  │     ├── redirects based on connect.sid / user_type / regular_user_type cookies
  │     └── rewrites /login ⇄ /enterprise/login depending on host
  │
  ├── /api/upload (the one real local route)  ──────────────▶  Firebase Storage (via firebase-admin)
  │
  ├── Client-side Firebase SDK  ──────────────▶  Firestore (real-time session/call docs)
  │                              ──────────────▶  Firebase Auth (custom-token sign-in only)
  │
  └── LiveKit client  ──────────────▶  LiveKit SFU (audio/video call transport)

External backend (NOT in this repo)
  tarjim.ly / us.tarjim.ly  — Node/Express (per `connect.sid` cookie convention)
  │   /api/mobile/v2/*  — legacy, shared with native mobile app
  │   /api/v3/*         — newer, web/enterprise-first
  │   Owns: users, organizations, subscriptions (Stripe), session-request lifecycle,
  │         translation-job lifecycle, AI translation proxy, LiveKit token issuance
  └── MySQL/Postgres/Mongo/etc. — unknown, not observable from this repo
```

### 3.7 Async / Real-Time Model

There is no message queue or websocket layer owned by this repo. Two mechanisms substitute:

1. **SWR polling** (typically `refreshInterval: 10000`) against REST endpoints for anything that changes moderately slowly: open session requests, active session existence, job status.
2. **Firestore subscriptions** for anything that needs to feel instantaneous: in-call participant state (`FirebaseSessionEntity`), enabling/disabling video mid-call, chat-enabled participant lists.

This is architecturally the rough equivalent of TWB's `queue_requests` DB-polling model, except the "queue" here is either short-interval REST polling or a managed real-time database (Firestore), and none of it is owned by the frontend repo.

---

## 4. Route Map

### 4.1 Public / auth (unauthenticated-accessible)
`/login`, `/forgot-password`, `/reset-password`, `/enterprise/login`, `/enterprise/reset-password`, `/verify-email` (authenticated but pre-verification), `/maintenance`, `/blacklisted`

### 4.2 Regular — aidworker
`/user/jobs`, `/user/jobs/[id]`, `/user/spark`, `/user/billing`, `/user/billing/[orgID]`, `/user/subscription/[orgID]`, `/user/manage/[orgID]`, `/user/manage/[orgID]/add-user`

### 4.3 Regular — translator
`/translator/jobs`, `/translator/jobs/[id]`

### 4.4 Regular — shared (either role)
`/call_request`, `/active_call`, `/meeting`, `/meeting/[sessionId]`

### 4.5 Enterprise
`/enterprise/language-interpretation`, `/enterprise/language-interpretation/call`, `/enterprise/jobs`, `/enterprise/jobs/[id]`

### 4.6 Local API routes (this repo)
`/api/upload` — the only endpoint not proxied to the external backend; writes directly to Firebase Storage via `firebase-admin`.

### 4.7 Proxied API surface (external backend, as referenced by client code)

**`/api/mobile/v2/*`** (legacy, shared with native mobile app):
`auth/logout`, `auth/reset-password`, `auth/new-password`, `users/metadata`, `users/profilePicture`, `users/resend-verification-email`

**`/api/v3/*`** (newer, web/enterprise-first):
`auth/login`, `auth/enterprise-login`, `auth/test-cookie`, `auth/firebase`, `users/me/app-version`, `organizations`, `organizations/:id`, `organizations/:id/memberships`, `organizations/:id/billing/subscriptions`, `aidworkers/me/translate`, `public/join-meeting`, `public/sessions/:id/join-meeting`

Note the split is not clean by version — some concerns (user metadata, password reset) only exist on `/v2`; others (login, org management, AI translate) only exist on `/v3`. Treat both as live, not `/v2` as deprecated.

---

## 5. Data Model (Inferred)

**Caveat repeated for emphasis:** none of this is a verified schema. It is reconstructed from TypeScript interfaces in `requests.ts`/`requests.tsx`/`types.ts` files across the client. Field names, nullability, and relationships reflect what the client expects the API to return, not a database definition.

### Identity
- **User** — `uid` (or `tarjimly_id`), `email`, `emailVerified`, `role` (`translator`|`aidworker`), `gender`, `phoneNumber`, `firstName`/`lastName`, `premiumExpiryDate`, `sponsoredExpiryDate`, `customerId` (Stripe), `refCode`
- **Organization** — `id`, `customerId` (Stripe), `name`, `address`, `email`, `type` (`non_profit`|`enterprise`), `subscriptions[]`
- **OrganizationMember** — `uid`, `organizationId`, `orgRole` (`user`|`admin`), `subscriptions[]` (product names), embeds a `User`-shaped `Member`

### Billing (Stripe, all writes happen backend-side)
- **StripeSubscription** — `id`, `customer`, `status` (`active`|`cancelled`|`pending`|`trial`|`expired`|`overdue`), `product` (`StripeProductNames`: `premium_monthly`, `premium_yearly`, `byot_monthly`, `sponsored_monthly`, `hotline`), `quantity`, `invoiceUrl`, `uids[]` (which members the subscription covers — a subscription can cover multiple org members)
- **StripePromotionCode** — `id`, `code`, `discount` (percentage or amount), `restrictions` (min amount, new-subscription-only)

### Real-time interpretation
- **SessionRequestEntity** — `id`, `uid` (requester), `languages[]`, `status` (`open`|`matched`|`cancelled`|`expired`|`rejected`), `acceptedBy`, `preferences` (medium, translator type, high-priority, no-record), `filters` (specific translator UID for BYOT, skills, gender, NDA/CoT required), `info` (topics, free-text, duration estimate), `preSessionForm` (dynamic field/value pairs), `scheduledAt`/`closedAt`
- **ActiveSessionEntity** — `id`, `requestId`, `languages[]`, `participants: SessionParticipantEntity[]`
- **FirebaseSessionEntity** (Firestore, not REST) — `isActive`, `enableVideo`, `participants[]`, `callProvider` (`dyte`|`livekit`), `participantsChatEnabled[]`
- **RatingStatus** / **GetRatingStatusResponse** — post-call rating prompt data (other participant's name/photo, session date)

### Async translation jobs
- **TranslationJob** — `id`, `uid`, `title`, `text` or `documentUrl`, `type` (`TranslationJobType`), `sourceLanguageId`/`targetLanguageId`, `status` (`AidworkerTranslationJobStatus`), `topics[]`, `initialAITranslation`, `metadata` (text-size description, comment), `additionalData` (NDA/CoT template URLs, other supporting documents), `submission` (0 or 1 `Submission`)
- **Submission** — `id`, `translation`, `status`, `metadata.feedback.issues[]` (structured revision requests), `additionalData` (NDA/CoT *filled* document URLs), `translatorUid`, `translationJobId`
- **LanguageEntry** — `id`, `language`, `native_name`, `code` (ISO); languages with `id >= 110` have `code: null` in the backend and the client falls back to a hardcoded `FirstPassSupportedLanguages` name→ISO map

---

## 6. Authentication & Tenancy

See `login.md` for the full flow-by-flow breakdown. Summary:

- **Session token**: cookie named `connect.sid` (Express-session convention — confirms the backend is Node/Express). Returned as a raw string in the JSON login response body (not a `Set-Cookie` header), parsed and written client-side via `document.cookie` — **not `HttpOnly`**.
- **Role/tenancy cookies**: `user_type` (`enterprise`|`regular`) and `regular_user_type` (`translator`|`aidworker`) — client-set, used only for UI routing, not a security boundary.
- **Firebase identity**: obtained via `signInWithCustomToken` immediately after backend login, using a `firebase_token` also returned in the login response. Powers Firestore access for real-time session data. Entirely separate lifecycle from `connect.sid` — a user can be backend-authenticated without a valid Firebase session or vice versa if either step fails silently (the code does catch and log Firebase setup failures without blocking login).
- **Tenancy**: decided by HTTP `Host` header against a hardcoded `REGULAR_HOSTS` allowlist in `auth-utils.ts`. Not in that set → treated as an enterprise white-label domain, proxied to a **different backend host** (`us.tarjim.ly` vs `tarjim.ly`).
- **Gates enforced client-side** (in `ProtectedRoute`, via `useEffect` + redirect, not in `middleware.ts`): authentication, email verification (`is_email_verified`), and blacklist status (polled via SWR). `middleware.ts` only checks for the *presence* of the auth cookie, not its validity or role.

---

## 7. External Integrations

| Integration | Direction | Purpose | Key code locations |
|---|---|---|---|
| **Backend API** (`tarjim.ly` / `us.tarjim.ly`) | Bidirectional, proxied | Owns all users, orgs, sessions, jobs, Stripe billing, LiveKit token issuance | `middleware.ts` rewrite; every `requests.ts`/`requests.tsx` |
| **Firebase Auth** | Inbound (custom token) | Real-time-feature identity, separate from backend session | `AuthContext.tsx` (`signInWithCustomToken`) |
| **Firestore** | Bidirectional, real-time | Live call/session state | `lib/firebase.ts`, `FirebaseSessionEntity` consumers |
| **Firebase Storage** | Outbound (server-side only) | Document uploads (translation source docs, NDAs) | `firebase/firebaseAdmin.ts`, `/api/upload` |
| **LiveKit** | Bidirectional | Audio/video call transport (current default) | `livekit-client`, `active_call/`, `meeting/` |
| **Dyte** | Unknown (legacy) | Prior call transport, still in the `callProvider` type union | `active_call/types.ts` |
| **Stripe** | Backend-mediated only | Subscriptions/billing | Referenced via `customerId`, `invoiceUrl`, `StripeProductNames` — no direct Stripe SDK calls in this repo |
| **Statsig** | Outbound | Feature flags + analytics autocapture | `src/statsig.tsx` |
| **PostHog** | Outbound | Product analytics | `src/posthog/` |
| **LogRocket** | Outbound | Session replay | `src/logrocket/` |
| **Vercel** | Hosting/CI | Build, deploy, env vars | `README.md` |

---

## 8. Merge-Relevant Risks & Gotchas

1. **The real data model, auth server, and business logic live outside this repo.** Any plan to merge Tarjimly and TWB Platform needs access to the `tarjim.ly`/`us.tarjim.ly` backend codebase — this audit can only describe the contract this client expects, not the authoritative schema.
2. **Non-`HttpOnly` session cookie.** `connect.sid` is set by client JS from a value in the response body. Any XSS vulnerability in this app can read and exfiltrate the session token directly — this is a more exposed pattern than a standard `Set-Cookie: HttpOnly` session, and more exposed than TWB's own (already-flagged) hand-rolled cookie crypto, since TWB's cookie is at least presumably server-set.
3. **Two overlapping API surfaces** (`/api/mobile/v2/*`, shared with the mobile app, and `/api/v3/*`, web/enterprise-first) with inconsistent coverage — e.g., logout and password-reset submission exist only on `/v2`; login and org management only on `/v3`. A unified/merged backend should not assume either is fully deprecated without checking both.
4. **Tenancy is a hardcoded host allowlist**, not data-driven. A merge that introduces new domains (e.g. a combined TWB+Tarjimly host) must update `REGULAR_HOSTS` in `auth-utils.ts` or the new domain will be silently treated as an unrecognized enterprise tenant, routed to the wrong backend.
5. **Two session/identity systems run in parallel** (backend `connect.sid` cookie + Firebase custom-token auth), each independently fallible. A merged Global Identity Platform (per TWB's `login-unification.md` proposal) needs to account for *both* — Firebase identity is currently a Tarjimly-specific side channel for real-time features, not part of the primary auth contract.
6. **Client-side authorization gates only.** Email verification and blacklist enforcement happen in React `useEffect` redirects, not in `middleware.ts` or (as far as this repo shows) revalidated on every backend call. Confirm the backend independently enforces these — do not assume the frontend gate is the actual security boundary in a merged system.
7. **Uploaded files are made public unconditionally** (`fileRef.makePublic()` in `firebaseAdmin.ts`), relying solely on a UUID-prefixed filename for obscurity. This includes NDA and translation-source documents. Worth remediating regardless of the merge.
8. **Legacy `dyte` call provider** still typed alongside `livekit` — confirm retirement status before assuming a merged calling experience can standardize on LiveKit alone.
9. **No automated tests, no `.env.example`.** Any merge work that touches auth, middleware, or the API proxy has no regression safety net in this repo today.
10. **Ambiguous package manager** (`package-lock.json` + `yarn.lock` both committed) — resolve before adding this repo to a shared monorepo or CI pipeline alongside TWB's Composer/PHP tooling.

---

## 9. Files Inspected

**Root / config**
- `README.md`, `package.json`, `next.config.mjs`, `docker-compose.yml`, `proxy/conf.d/tarjimly-web.conf`, `tailwind.config.ts`, `components.json`

**Routing & middleware**
- `src/middleware.ts`
- `src/config.ts`

**Auth**
- `src/lib/auth/auth-utils.ts`, `AuthContext.tsx`, `useAuth.ts`, `ProtectedRoute.tsx`
- `src/lib/cookies.ts`
- `src/lib/firebase.ts`, `src/firebase/firebaseAdmin.ts`
- `src/app/login/requests.tsx`, `src/app/login/page.tsx`
- `src/app/enterprise/login/page.tsx`, `src/app/enterprise/reset-password/page.tsx`
- `src/app/verify-email/page.tsx`, `src/app/blacklisted/page.tsx`

**Shared model / hooks layer**
- `src/app/requests.ts` (698 lines — the closest thing to a shared model file)
- `src/app/hooks.ts`
- `src/app/constants.ts`

**Real-time interpretation**
- `src/app/call_request/types.ts`, `requests.tsx`, `hooks.ts`, `page.tsx`
- `src/app/active_call/types.ts`, `requests.tsx`, `hooks.ts`
- `src/app/meeting/requests.ts`, `MeetingRoom.tsx`, `JoinMeetingUI.tsx`, `[sessionId]/page.tsx`

**Async translation jobs**
- `src/app/user/jobs/requests.tsx` (partial), `hooks.ts`
- `src/app/user/spark/requests.ts`, `page.tsx`
- `src/app/translator/jobs/requests.tsx`, `hooks.ts`
- `src/app/enterprise/jobs/requests.tsx`, `hooks.ts`

**Layout & providers**
- `src/app/layout.tsx`, `client-layout.tsx`
- `src/app/components/regular-layout.tsx`, `enterprise-layout.tsx`
- `src/statsig.tsx`

**Directory structure only (not fully read)**
- `src/app/user/billing/`, `user/manage/`, `user/subscription/`, `enterprise/components/`, `translator/components/`, `src/components/ui/`, `src/hooks/`

**Not read at all (flagged for follow-up if needed)**
- The external backend (`tarjim.ly` / `us.tarjim.ly`) — not in this repository
- Full contents of every component file under `components/`, `enterprise/components/`, `translator/components/`, `user/components/`
- `src/hooks/` (top-level, distinct from `src/app/hooks.ts`)
