# TWB Platform — Developer Context

This workspace contains two repos for the TWB (Translators Without Borders) volunteer translation platform, also known as SOLAS-Match. The goal is to migrate the current PHP + C++ stack to **Next.js + React Native**. Detailed audit and analysis documents are on disk:

- `audit-report.md` — full architectural audit (2026-05-06)
- `login.md` — deep-dive on the login flow (all four paths: password, Google, OAuth callback, SAML)
- `login-unification.md` — advisory on unifying login with Tarjimly via a Global Identity Platform (GIP)

---

## Repo Structure

```
twb-audit/
  SOLAS-Match/          ← PHP 8 / Slim 4 frontend + REST API
  SOLAS-Match-Backend/  ← C++ Qt 6 daemon (email + queue worker)
```

### SOLAS-Match (PHP)

```
index.php                          ← Entry point; defines role constants
api/
  dispatcher.php                   ← Slim route dispatcher for /api/v0/*
  v0/                              ← API endpoint classes (Users.php, Tasks.php, Projects.php, Orgs.php, …)
  DataAccessObjects/               ← API-layer DAOs (call MySQL stored procs)
ui/
  RouteHandlers/                   ← Page controller classes (UserRouteHandler, TaskRouteHandler, …)
  DataAccessObjects/               ← UI-layer DAOs (call /api/v0/* over localhost HTTP)
  templating/
    templates/                     ← Smarty .tpl templates (see Design section)
    templates/user/                ← User page templates
    templates/task/                ← Task page templates
    templates/project/             ← Project page templates
    templates/admin/               ← Admin dashboard templates
    templates/org/                 ← Org page templates
  css/style.css                    ← Legacy Bootstrap 4 CSS (old pages)
  css/custom.css                   ← Compiled output from scss/custom.scss (active pages)
  js/                              ← Per-page jQuery scripts
  lib/
    Middleware.class.php           ← Slim middleware; enforces role-based access
Common/
  Enums/                           ← Shared PHP enums (TaskTypeEnum, TaskStatusEnum, …)
  lib/
    UserSession.class.php          ← Session helpers
  conf/                            ← Config files (DB credentials, API keys)
  protobufs/                       ← Protobuf definitions (user, task, project objects)
scss/
  custom.scss                      ← Bootstrap 5 SCSS source; compile → ui/css/custom.css
db/                                ← ~100 table schemas + 300+ stored procedures
```

### SOLAS-Match-Backend (C++)

```
CorePlugin/          ← Base task dispatcher
EmailPlugin/         ← 53 email templates via Google ctemplate
PluginScheduler/     ← Polls DB queue every 250 ms
templates/           ← Email HTML/text templates
```

---

## Active Design System

**New pages use `new_header.tpl`.** Legacy pages use `header.tpl` (Bootstrap 3, jQuery nav). Do not add new features to legacy templates.

### Stack (active pages)
- **Bootstrap 5.3** (compiled from `scss/custom.scss` → `ui/css/custom.css`)
- **Font Awesome 6.5** (icons via `fa-solid`, `fa-regular`, etc.)
- **Inter** font (Google Fonts)
- **jQuery 1.9** + **jQuery UI** (still used for datepickers, AJAX)
- **Day.js** (date handling, with UTC plugin)
- **Quill 2** (rich text editor where needed)
- **Tempus Dominus 6** (datetime picker)

### Page shell
Every new page starts with `{include file="new_header.tpl"}` and wraps content in:
```smarty
<div class="container-xxl px-4 px-sm-5 px-lg-5 pb-5 pt-4">
  <div class="row g-4">
    ...
  </div>
</div>
```
The `<main class="flex-grow-1">` wrapper is opened in `new_header.tpl` and should be closed with `</main>` before the footer. No separate footer include is needed on new pages (footer is inline in `new_header.tpl`).

### Colour tokens (SCSS variables → Bootstrap utility classes)

| Token | Hex | Usage |
|---|---|---|
| `$primary` | `#f89406` | Orange — primary CTA buttons, links, active states |
| `$secondary` | `#133978` | Navy — navbar bg, headings, profile name |
| `$body-color` | `#1c1d36` | Near-black body text |
| `primaryDark` | `#e8991c` | Darker orange for hover / task title emphasis |
| `primaryBorder` | `#ba7b1a` | Border on primary buttons |
| `grayish` | `#364f67` | Steel blue-grey — secondary action buttons |
| `greenish` | `#1b8912` | Success / complete states |
| `quartenary` | `#36a6e7` | Light blue — informational accents |
| `quinary` | `#7b61ff` | Purple — chunk/part indicators |
| `yellowish` | `#fef4e2` | Pale yellow — warning background tints |
| `light` | `#f8f8f8` | Off-white card/section backgrounds |

Available as Bootstrap utility classes: `text-primary`, `bg-secondary`, `text-primaryDark`, `bg-yellowish`, `text-quinary`, etc.

### Button classes (defined in custom.scss)

| Class | Description |
|---|---|
| `btnPrimary` | Orange filled, white text — primary action |
| `btnSuccess` | Green filled, white text — complete/submit |
| `btngray-sm` | Small steel-grey button |
| `btngray-lg` | Large steel-grey button |
| `.btn.btn-primary` | Bootstrap primary (also orange via token override) |
| `.btn.btn-secondary` | Navy; hover turns orange |

### Typography scale

| Class | Size |
|---|---|
| `fs-1` / `h1` | `font-size-base * 2.5` |
| `fs-2` / `h2` | `font-size-base * 2` |
| `fs-3` / `h3` | `20px` |
| `fs-4` / `h4` | `16px` |
| `fs-5` / `h5` | `14px` |
| `fs-6` / `h6` | `12px` |
| `fs-7` | `11px` |

Spacer base is `8px`. Bootstrap spacers (`m-1` … `m-6`) map to `4px, 8px, 16px, 24px, 32px, 40px`.

### Cards and sections
Standard card pattern:
```html
<div class="card bg-light-mariam custom-card p-4 card-border-top-accent">
  <div class="d-flex justify-content-between align-items-center mb-4 border-bottom pb-3">
    <h2 class="fs-3 fw-bold text-dark-mariam mb-0">Section title</h2>
  </div>
  ...
</div>
```

Section backgrounds: `bg-light-subtle` for page sections, `bg-light` (`#f8f8f8`) for cards.

### Flash messages
Use the standard flash pattern (already in `home_mariam.tpl` and most page templates):
```smarty
{if isset($flash['error'])}
  <div class="alert alert-danger alert-dismissible fade show mt-4">
    <p><strong>Warning! </strong>{TemplateHelper::uiCleanseHTMLKeepMarkup($flash['error'])}</p>
    <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
  </div>
{/if}
```
Four types: `error` (danger), `info`, `success` (with `success.svg` icon), `warning`.

### Icons
Font Awesome 6 solid (`fa-solid fa-*`). Common: `fa-chevron-right` (breadcrumbs), `fa-circle-half` (in-progress), `fa-exclamation-circle-fill` (overdue), edit uses `edit.svg` from `ui/img/`.

### Dark mode
`new_header.tpl` includes a light/dark toggle (light.svg / night.svg). Use `data-bs-theme` attributes and `[data-bs-theme="dark"]` selectors in SCSS when needed. The `bg-body-tertiary` and `bg-secondary` navbar classes adapt automatically.

### Navigation
Navbar items added in `new_header.tpl`. Role-gated with Smarty conditionals on `$site_admin`, `$user`, `$ngo_orgs`. Dropdown menu opens on hover (desktop, via `.nav-item.dropdown:hover`) and click (mobile via Bootstrap collapse).

---

## Smarty Template Conventions

- Always escape output: `{TemplateHelper::uiCleanseHTML($var)}` for plain text, `{TemplateHelper::uiCleanseHTMLKeepMarkup($var)}` where markup is allowed.
- URLs: `{urlFor name="route-name" options="param.value"}` — never hardcode paths.
- Translations: `{Localisation::getTranslation('key')}` — don't use raw strings for UI labels.
- Config values: `{Settings::get('section.key')}`.
- User avatar: `https://www.gravatar.com/avatar/{md5(strtolower(trim($user->getEmail())))}?s=20`.
- Assign intermediate values with `{assign var="foo" value=$bar}`.
- Breadcrumb pattern: inline `<a>` + `<i class="fa-solid fa-chevron-right mx-1">` chain.

---

## PHP / Backend Conventions

### Role bitmask (defined in `index.php`)

| Constant | Value | Who |
|---|---|---|
| `LINGUIST` | 1 | Volunteer translator |
| `NGO_LINGUIST` | 2 | NGO-internal linguist |
| `NGO_PROJECT_OFFICER` | 4 | NGO project manager |
| `NGO_ADMIN` | 8 | NGO organisation admin |
| `COMMUNITY_OFFICER` | 16 | TWB internal staff |
| `PROJECT_OFFICER` | 32 | TWB project manager |
| `SITE_ADMIN` | 64 | Platform admin |
| `FINANCE` | 128 | Finance role |

Check roles with bitwise AND: `$roles & (SITE_ADMIN | PROJECT_OFFICER)`. Available in templates as `$roles`, `$site_admin`, `$ngo_orgs`.

### DAO pattern
- `ui/DataAccessObjects/XxxDao.class.php` — UI DAOs make HTTP calls to `/api/v0/*`
- `api/DataAccessObjects/XxxDao.class.php` — API DAOs call MySQL stored procedures
- Key large files: `UserDao` (2,967 LOC), `ProjectDao` (2,334 LOC — includes ~1,900 LOC Phrase TMS integration)

### Task types (TaskTypeEnum)

| Constant | Value | Meaning |
|---|---|---|
| SEGMENTATION | 1 | Split source into segments |
| TRANSLATION | 2 | Translate segments |
| PROOFREADING | 3 | Review translation |
| DESEGMENTATION | 4 | Merge segments back |
| QUALITY | 5 | Quality check |
| APPROVAL | 6 | Final approval |
| SPOT_QUALITY_INSPECTION | 38 | Random QA sample |
| QUALITY_EVALUATION | 39 | Full QA evaluation |

Task colours and UI labels are stored in DB (`task_type_details` table), loaded at runtime into `TaskTypeEnum::$enum_to_UI`.

### Task statuses (TaskStatusEnum)
`WAITING_FOR_PREREQUISITES=1`, `PENDING_CLAIM=2`, `CLAIMED=10`, `IN_PROGRESS=3`, `COMPLETE=4`

---

## Migration Plan (5 phases)

1. **C++ daemon → Node.js worker** (BullMQ + React-Email) — replace `SOLAS-Match-Backend`
2. **New Next.js frontend** consuming existing Slim API — incremental page migration
3. **Next.js Route Handlers** calling stored procs directly — replace Slim REST layer
4. **Stored procs → TypeScript service functions** — domain by domain
5. **React Native mobile app** — consuming Phase 3 API

**Current phase:** Phase 2 work (design updates to existing PHP/Smarty templates while planning Next.js migration).

---

## Known Landmines

- **`getAuthCode` auto-registers users** silently (`md5(email)` dummy password). Any new auth integration must gate this.
- **`X-Custom-Token` response header** — non-standard; carries base64 protobuf OAuth token between UI and API. Must be replaced in any new API layer.
- **Duplicate post-login redirect block** at lines 1066–1094 and 1172–1203 of `UserRouteHandler.class.php` — keep in sync until consolidated.
- **`phpinfo.php` at document root** — active security exposure.
- **Password hashing is `hash_hmac('sha512', …, $site_key)`** — site-wide HMAC key; passwords cannot be migrated to a new system without a reset flow.
- **Stored procedures are the highest migration risk** — ~300 procs hold all business logic; no ORM.
- **Phrase TMS (Memsource) coupling** — ~5,000 LOC across DAOs; deep integration for project/task file handling.
- **Bootstrap 3 / Bootstrap 5 coexistence** — old templates use Bootstrap 3 classes (`navbar-inner`, `span4`, `row-fluid`); new templates use Bootstrap 5. Do not mix.
- **Legacy `header.tpl`** loads Bootstrap 4 + jQuery 1.9; `new_header.tpl` loads Bootstrap 5 + jQuery 1.9. Always use `new_header.tpl` for new or updated pages.
