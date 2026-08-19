# TWB Platform / SOLAS-Match — Codebase Audit Report

**Date:** 2026-05-06  
**Repositories audited:**
- `SOLAS-Match/` — PHP web frontend (UI + REST API)
- `SOLAS-Match-Backend/` — C++/Qt background worker daemon

---

## Table of Contents

1. [Repository Overview](#1-repository-overview)
2. [Technology & Dependency Map](#2-technology--dependency-map)
   - 2.1 Frontend (SOLAS-Match)
   - 2.2 Backend daemon (SOLAS-Match-Backend)
   - 2.3 Database
   - 2.4 Deployment & infrastructure
3. [Functional Documentation](#3-functional-documentation)
   - 3.1 Platform summary
   - 3.2 User roles
   - 3.3 Task types & status
   - 3.4 End-user workflows
   - 3.5 Authentication & session
   - 3.6 Architecture diagram
   - 3.7 Async work model
   - 3.8 Email system
   - 3.9 External integrations
   - 3.10 Localisation
4. [URL & Route Map](#4-url--route-map)
5. [Migration Assessment — Next.js / React Native](#5-migration-assessment--nextjs--react-native)
   - 5.1 What can transfer cleanly
   - 5.2 What must be rewritten
   - 5.3 Risks & gotchas
   - 5.4 Suggested migration phasing
6. [Files Inspected](#6-files-inspected)

---

## 1. Repository Overview

Two repositories make up the platform:

- **`SOLAS-Match/`** — PHP web application. Contains both the browser-facing UI (Slim 4 + Smarty templates) and a separate internal REST API (Slim 4 + OAuth2). This is what users interact with in their browser.
- **`SOLAS-Match-Backend/`** — C++/Qt daemon. Runs as a background process: polls a database queue every 250 ms, performs deadline checks, sends emails, and dispatches notifications.

Both repos descend from the original SOLAS Match academic project (University of Limerick / Rosetta Foundation) and are now operated as the **TWB Platform** (also branded "Kató") by **Translators Without Borders / CLEAR Global**. The README shows version history through v20.x.

---

## 2. Technology & Dependency Map

### 2.1 Frontend (`SOLAS-Match/`)

**Language / runtime:** PHP 8 (migrated from PHP 5/7 in v13).

**HTTP framework:** Slim Framework 4.x (upgraded from Slim 2 in v10.0). Two separate Slim apps live in the same repo:
- `index.php` — UI app (Smarty templates, browser-facing)
- `api/dispatcher.php` — REST API app (JSON over HTTP)

#### Composer dependencies — UI (`ui/composer.json`)

| Package | Version | Purpose |
|---|---|---|
| `slim/slim` | 4.* | HTTP routing framework |
| `slim/psr7` | ^1.4 | PSR-7 request/response |
| `smarty/smarty` | ^4.1 | Primary view/template engine |
| `twig/twig` | ^3.3 | Secondary template engine (present but less prominent) |
| `lightopenid/lightopenid` | dev-master | Legacy OpenID authentication |

#### Composer dependencies — API (`api/composer.json`)

| Package | Version | Purpose |
|---|---|---|
| `slim/slim` | 4.* | HTTP routing framework |
| `slim/psr7` | ^1.4 | PSR-7 request/response |
| `league/oauth2-server` | dev-master (pinned commit `54ffa58e`) | OAuth2 password & auth-code grants |

> **Note:** The `league/oauth2-server` pin to a specific dev-master commit is a years-old snapshot of an upstream that has since had several breaking releases. This is a known technical debt item.

#### Front-end client code

- **~37 hand-rolled JavaScript files** in `ui/js/` (`Home5.js`, `ProjectCreate14.js`, `TaskView5.js`, `UserPrivateProfile6.js`, `pagination_task_stream.js`, `eligible.js`, `profile.js`, `project-view1.js`, etc.). Numbered suffixes appear to be manual cache-busting.
- **jQuery 1.9** + jQuery UI loaded directly from `ui/js/lib/jquery-1.9.0.js` (referenced from `templates/header.tpl`).
- **Bootstrap 3**-style CSS (`bootstrap.min1.css`, `style.2.css`, custom `solas3.css`).
- **Protobuf in browser:** `ProtoBuf.min.js`, `ByteBuffer.min.js`, `Long.min.js`, `ProtoClasses.js` — the browser parses the same protobuf models used server-side by the PHP API.
- **Quill 2.0.2** rich-text editor (CDN).
- **Select2** for enhanced dropdowns.
- **Font Awesome 4.7** (CDN).
- **Google Tag Manager** (`G-3Z3VNH71D6`).

#### Dart sub-project (`ui/dart/`)

A legacy Polymer / dart2js-compiled sub-application (`pubspec.yaml`). Contains `UserPrivateProfile.dart`, `ClaimedTasks.dart`, `ProjectCreate.dart`, `ProjectAlter.dart`. Polymer/Dart 1 is end-of-life. Effectively dead code in the repository; production runs the dart2js-compiled JS output. These four pages need to be reimplemented in any migration.

#### Templating

Smarty 4 with custom helpers (`ui/lib/TemplateHelper.php`). Approximately 120 `.tpl` templates under `ui/templating/templates/`. Templates use raw PHP static-method calls inside `{}` blocks — for example `{Localisation::getTranslation('…')}`, `{TaskTypeEnum::$enum_to_UI[$type_id]['type_text']}`, `{TemplateHelper::uiCleanseHTMLKeepMarkup($flash['error'])}`.

#### Vendored libraries

- `ui/google-api-php-client/` — vendored Google API PHP client (Google Drive, Google OAuth).
- `resources/TCPDF-main/` — PDF generation.
- `resources/bootstrap/` — Bootstrap static assets.

#### Other repo artifacts at root

`user 2026 Code of Conduct.pdf`, `user_TWB_Dashboard_Final.html`, `virtual-tour.html`, `pre-commit` (git hook), brand images, `phpinfo.php` (security smell — should not be present in production).

---

### 2.2 Backend Daemon (`SOLAS-Match-Backend/`)

**Language / runtime:** C++ on **Qt 5.x / 6.x** (originally Qt 4.8). Build system: qmake (`SOLASMatchWorkerDaemon.pro`, `Common.pro`).

#### Build modules (subdirs)

| Module | Content |
|---|---|
| `Common/` | Shared models, ConfigParser, MySQLHandler, DAOs, protobuf-compiled `.pb.cc/.pb.h` |
| `PluginHandler/` | `main.cpp`, PluginLoader, WorkerInterface (Qt plugin discovery and threading) |
| `CorePlugin/` | Three queue listeners: TaskQueueHandler, UserQueueHandler, ProjectQueueHandler; plus TaskJobs/ and UserJobs/ |
| `EmailPlugin/` | SMTP plumbing (`Smtp.cpp`, Qxt mail classes) + 30+ Generator classes in `Generators/` |
| `PluginScheduler/` | XML-driven cron equivalent (`schedule.xml`, TimedTask) |

#### External C++ libraries

| Library | Purpose |
|---|---|
| Qt (`QtCore`, `QtNetwork`, `QtSql` with `QMYSQL`, `QtXml`) | Core framework, DB, networking |
| `libprotobuf` | Shared protobuf model serialisation |
| `libctemplate` (Google ctemplate) | Email template rendering (mustache-like `{{…}}` syntax in `templates/emails/`) |
| Qxt (libqxt-dev, kept locally) | SMTP client (`qxtsmtp`, `qxtmailmessage`, `qxtmailattachment`, `qxthmac`, `qxtglobal`) |

> **Note on RabbitMQ:** Vagrant `provision.sh` installs rabbitmq-c and amqpcpp, and old README references AMQP. The current code (v14 milestone "RabbitMQ removed") uses **DB-queue tables** (`queue_requests`) instead. The provision script is a historical artifact.

#### Runtime configuration

- **Backend:** `SOLAS-Match-Backend/conf.template.ini` — database DSN, SMTP server, exchange name (`SOLAS_MATCH`), poll rate (250 ms), max_threads (20). Installed to `/etc/SOLAS-Match/conf.ini` in production.
- **Frontend/API:** `SOLAS-Match/Common/conf/conf.template.ini` — loaded from hard-coded path `/repo/SOLAS-Match/Common/conf/conf.ini`. Contains: Phrase/Memsource API URL (`https://cloud.memsource.com/web/api2/v1/`), Asana keys, Discourse URL, Neon CRM keys, badge keys, Google OAuth credentials, OpenGraph/Twitter card config, file upload paths, format converter endpoint, project image limits, Slack webhooks.

---

### 2.3 Database

**Engine:** MySQL 8 (migrated from MySQL 5.5 in v13), InnoDB, `utf8mb4_unicode_ci`.

**Schema:** `SOLAS-Match/db/schema.sql` — 16,599 lines.

**Scale:** ~100+ tables, **300+ stored procedures**. All data access in both PHP and C++ flows exclusively through stored procedures — there is essentially no inline SQL in application code. PHP calls `PDOWrapper::call($procName, $args)` and C++ calls `MySQLHandler::call(proc_name, args)`.

#### Table groups

**Identity & organisations**
`Users`, `Admins`, `Organisations`, `OrganisationMembers`, `OrganisationExtendedProfiles`, `BannedUsers`, `BannedOrganisations`, `BannedTypes`, `RegisteredUsers`, `GoogleUserDetails`, `TermsAcceptedUsers`, `password_reset_requests`, `WillBeDeletedUsers`

**Projects & tasks (core domain)**
`Projects`, `Tasks`, `TaskClaims`, `TaskTypes`, `TaskStatus`, `TaskFileVersions`, `TaskPrerequisites`, `TaskReviews`, `ProjectFiles`, `ProjectTags`, `Tags`, `Languages`, `Countries`, `TaskNotificationSent`, `TaskTranslatorBlacklist`, `TaskUnclaims`, `TaskViews`, `TaskChunks`, `TaskPaids`, `TaskInviteSentToUsers`, `RestrictedTasks`, `RequiredTaskQualificationLevels`, `TaskCompleteDates`, `possible_completes`, `task_urls`, `task_type_categorys`, `task_type_details`

**Archive**
`ArchivedProjects`, `ArchivedProjectsMetadata`, `ArchivedTasks`, `ArchivedTasksMetadata`

**User profile & qualifications**
`UserPersonalInformation`, `UserSecondaryLanguages`, `UserTaskStreamNotifications`, `UserQualifiedPairs`, `UserURLs`, `UserExpertises`, `UserHowheards`, `UserCertifications`, `communications_consents`, `user_paid_eligible_pairs`, `user_rate_pairs`, `user_task_limitations`, `linguist_payment_informations`, `enforce_native_languages`, `user_country_id_to_variant`

**Async queues & messaging**
`emails`, `queue_requests` (replaces RabbitMQ), `qxt_smtp_emails`, `queue_claim_tasks`, `queue_copy_task_original_files`, `queue_po_responses`

**Phrase TMS / Memsource (CAT tool integration)**
`MemsourceUsers`, `MemsourceClients`, `MemsourceProjects`, `MemsourceProjectLanguages`, `MemsourceSelfServiceProjects`, `MemsourceTasks`, `ProcessedMemsourceTaskUIDs`, `memsource_statuses`, `MatecatRecordedJobStatus`, `TaskTranslatedInMatecat`, `master_kato_tm_tasks`, `no_mt_for_orgs`

**Asana (project management)**
`AsanaProjects`, `AsanaTasks`, `asana_quality_tasks`, `asana_board_for_org`

**Discourse (community forum)**
`DiscourseID`

**Moodle (e-learning center)**
`moodle_datas`, `moodle_task_users`

**Finance (Sun Systems / Zahara / HubSpot / DocuSign)**
`hubspot_deals`, `zahara_purchase_orders`, `sync_po_events`, `sun_purchase_requisitions`, `poll_sun`, `po_cut_off_sun`, `sun_po_errors`, `invoices`, `sent_contracts`, `project_complete_dates`, `prozdata`

**Content / CMS**
`content_items`, `content_attachments`, `content_for_projects`, `org_images`, `post_login_messages`

**Quality**
`quality_requests`, `UserTaskScores`, `UserTaskScoresUpdatedTime`, `admin_comment`, `adjust_points`, `adjust_points_strategic`

**Tracking & notifications**
`TrackCodes`, `TrackedRegistrations`, `Referers`, `UserTrackedProjects`, `UserTrackedTasks`, `UserTrackedOrganisations`, `UserLogins`, `UserNotifications`, `seen_tutorials`

**OAuth2**
`oauth_clients`, `oauth_client_endpoints`, `oauth_sessions`, `oauth_session_*` (managed by `league/oauth2-server`)

**Operations & misc**
`Statistics`, `OrgRequests`, `OrgTranslatorBlacklist`, `ProjectRestrictions`, `TestingCenterProjects`, `OrgIDMatchingNeon`, `PrivateTMKeys`, `UserNeonAccount`, `special_registrations`, `org_TWB_contacts`, `user_org_defaults`, `Services`, `UserServices`, `selections`, `entitlements`, `entitlement_logs`, `user_instructions`, `UserRequest`

**Auxiliary SQL files:** `db/Reporting.sql`, `db/country_codes.sql`, `db/languages.sql` (seed data).

---

### 2.4 Deployment & Infrastructure

**Vagrant-based dev box (`SOLAS-Match/vagrant/`):**
- `Vagrantfile`: Ubuntu 14.04 (`ubuntu/trusty64`). Port 8080→80 forwarded; repo mounted at `/opt/match`.
- `provision.sh`: Provisions Apache 2 + mod_xsendfile + mod_rewrite; PHP 5 + mcrypt + curl + mysql; MySQL 5.5; Composer; `php-protocolbuffers` built from source; RabbitMQ-C + amqpcpp + Qt5. Checks out `SOLAS-Match-Backend` and builds with qmake.
- **This provision script is significantly out of date.** The current code targets PHP 8 / MySQL 8 / Qt 6 and has no RabbitMQ dependency. The Vagrantfile/provision.sh is a historical artifact, not a working dev environment.
- `vagrant/assets/001-match.conf` — Apache vhost: `DocumentRoot /var/www/html/match/`, `XSendFilePath /var/www/html/match/uploads/`, `SetEnv SOLAS_CONFIG /vagrant/assets/conf.ini`. **`X-Sendfile` is the file-download mechanism** — files are served via Apache's X-Sendfile header after PHP validates access, not served directly from PHP.

**Hard-coded production paths (will need attention in any migration):**
- `Common\Lib\Settings::get` reads from `/repo/SOLAS-Match/Common/conf/conf.ini`
- `index.php` requires `/repo/SOLAS-Match/ui/vendor/smarty/smarty/libs/Smarty.class.php`
- `api/dispatcher.php` requires `/repo/SOLAS-Match/api/vendor/league/oauth2-server/src/...`
- C++ daemon: template path `/etc/SOLAS-Match/templates/`, config `/etc/SOLAS-Match/conf.ini`, kill-switch file `/repo/SOLAS-Match-Backend/STOP_consumeFromQueue` (checked every 250 ms — placing this file pauses queue processing)

---

## 3. Functional Documentation

### 3.1 Platform Summary

The TWB Platform is a **translation crowdsourcing system** for the humanitarian sector. Non-profit organisations (NGOs) post translation and proofreading projects; volunteer linguists worldwide claim individual tasks, work inside the Phrase TMS (formerly Memsource) CAT tool, and submit completed files. The platform manages the full lifecycle: task routing, quality assurance, volunteer qualifications, gamification (badges, certificates), community engagement (Discourse forum, Moodle e-learning), and financial operations (invoicing via Sun Systems/Zahara, contract signing via DocuSign). A C++ background daemon handles deadline checking, email generation, and notification fan-out.

**For non-technical stakeholders:** Think of the platform as a matching engine and workflow manager. NGO partners upload their documents and the system automatically breaks them into translation tasks that are matched to qualified volunteer linguists by language pair, expertise, and availability. The actual translation work happens in an integrated professional translation tool (Phrase/Memsource). The platform tracks who is doing what, enforces deadlines, sends reminder emails, awards volunteer recognition, and connects the translation activity to the organisation's financial and project management systems.

---

### 3.2 User Roles

Roles are stored as a bitmask in the `Admins.roles` column. Each role is a power of two, and users can hold multiple roles simultaneously.

| Bit value | Role constant | Description |
|---|---|---|
| 128 | `FINANCE` | TWB finance team — sees finance reports, marks invoices paid/bounced/revoked |
| 64 | `SITE_ADMIN` | TWB full platform administrator |
| 32 | `PROJECT_OFFICER` | TWB staff project manager |
| 16 | `COMMUNITY_OFFICER` | TWB community manager |
| 8 | `NGO_ADMIN` | NGO partner top-level administrator |
| 4 | `NGO_PROJECT_OFFICER` | NGO partner project owner |
| 2 | `NGO_LINGUIST` | NGO partner linguist (private/restricted volunteer) |
| 1 | `LINGUIST` | Standard volunteer linguist (the majority of users) |

A special constant `ORG_EXCEPTIONS = [773]` exists in `index.php` — one specific organisation receives bespoke business-logic handling in certain workflows.

---

### 3.3 Task Types & Status

**Task types** are defined in the `task_type_details` table and loaded into `TaskTypeEnum` at boot via `TaskTypeEnum::init()`. Core types:

| ID | Type | Description |
|---|---|---|
| 1 | SEGMENTATION | Splitting source document into translation-ready segments |
| 2 | TRANSLATION | Translating segmented content |
| 3 | PROOFREADING | Reviewing a completed translation |
| 4 | DESEGMENTATION | Reassembling segments into a final document |
| 5 | QUALITY | Quality evaluation |
| 6 | APPROVAL | Final approval step |
| 38 | SPOT_QUALITY_INSPECTION | Spot-check quality review |
| 39 | QUALITY_EVALUATION | Structured quality evaluation |

Plus 24+ additional "shell" task types covering DTP, voice recording, subtitling, machine translation post-editing (MTPE), plain language, and other specialised services.

**Task status codes** (`TaskStatusEnum`):

| Code | Status |
|---|---|
| 1 | WAITING_FOR_PREREQUISITES |
| 2 | PENDING_CLAIM |
| 3 | IN_PROGRESS |
| 4 | COMPLETE |
| 10 | CLAIMED (PHP-layer concept; C++ uses only 1–4) |

---

### 3.4 End-User Workflows

#### A. Volunteer linguist — registration to first task

1. **Register** at `/register` (or `/register_track/{track_code}` for campaign-tracked registrations, or `/special_registration` for partner portals). Captures email, password, communications consent. Google OAuth available via `/auth/code` round-trip.
2. **Email verification**: user receives a link to `/user/{uuid}/verification`. Until verified, access is restricted.
3. **Profile completion gate** (`Middleware::authUserIsLoggedIn` blocks navigation while `profile_completed != 2`): user must supply personal information, native language, qualified language pairs, optionally upload certifications, and accept the Code of Conduct.
4. **Browse task stream** at `/task_stream` or `/paged/{page_no}/tt/{tt}/sl/{sl}/tl/{tl}` — paginated, filterable by task type, source language, and target language.
5. **Claim a task** at `/task/{task_id}/claim` — gated by: language-pair qualifications, blacklist checks, restricted-task rules, language-pair limits, and MT permissions.
6. **Translate** either in the Phrase TMS web UI (linked from the claim email) or by downloading the source file and uploading a translated file at `/task/{task_id}/uploaded`.
7. **Mark complete**; deadline checker enforces submission deadlines and can auto-revoke overdue tasks.

#### B. NGO partner — project lifecycle

1. NGO admin or project officer lands on `/org/{org_id}/home_ngo` or `/org/{org_id}/ngo_projects`.
2. **Create project** at `/org/{org_id}/createProject`: supplies title, summary, instructions, reference number, source file, project image, source and target language(s). Word count is computed from the uploaded file.
3. If configured, a **Phrase TMS project is created automatically** via `ProjectDao::create_memsource_project` (a ~1,900-line method — the most complex piece of code in the system). This project links all translation jobs in the professional CAT tool.
4. **Sub-tasks are generated automatically**: the platform creates the segmentation → translation → proofreading → desegmentation chain (or shell task workflow), with `TaskPrerequisites` linking each step to its predecessor.
5. NGO can configure services, set access restrictions, change project ownership, assign specific verified translators ("strict sourcing"), and set entitlements.
6. When complete, NGO **downloads the final file** via `/download-task-latest-version`. The project and its tasks are archived to `ArchivedProjects` and `ArchivedTasks`.

#### C. TWB staff workflows

Staff roles access a suite of management tools:

- **Site admin dashboard** (`/site-admin-dashboard`): platform-wide administration.
- **Community dashboard** (`/community_dashboard`): volunteer engagement tools.
- **Finance tools**: `paid_projects`, `all_deals_report`, `po_readyness_report`, `sun_po_errors`, `sow_report`, `sow_linguist_report`, `pr_report`, `po_report`, `set_invoice_paid`, `set_invoice_bounced`, `set_invoice_revoked`.
- **Memsource/Phrase project list** (`/list_memsource_projects`): view all CAT tool projects.
- **User management**: ban/unban users and organisations, revoke in-progress tasks, manage qualifications.
- **Quality evaluation**: `quality_requests`, `asana_quality_tasks`, score adjustment (`adjust_points`, `adjust_points_strategic`).
- **Invitations**: dedicated invite pages per staff role (`/invite_admins`, `/invite_site_admins`, etc.).
- **Certificates & reference letters**: generate and download volunteer recognition documents.
- **DocuSign contracts**: `/docusign_redirect_uri` and `/docusign_hook` for linguist contract signing.
- **Content management**: tile-based per-organisation content items (homepage widgets).
- **Analytics**: `/analytics` (embedded dashboard), `/metabase` (Metabase BI), `/metabase_ngo` (partner-facing).

---

### 3.5 Authentication & Session

**Session storage:** PHP's native session is neutered (`session.use_cookies=0`; all session handlers return defaults). Real session state is AES-128-CBC encrypted with HMAC-SHA1 into a `slim_session` cookie — key from `session.site_key` in config, 12-hour expiry, 4 KB cap. Implementation is in `Middleware::SessionCookie` (`ui/lib/Middleware.class.php:15-83`). The crypto is hand-rolled using 16-byte keys/IVs derived by truncating SHA-1 HMACs — functional but non-standard; a dedicated security review is recommended.

**Login flow:** User posts email/password to `/login` → `UserDao::apiLogin` → Slim API issues an OAuth2 password-grant token via `league/oauth2-server` → `UserSession::setAccessToken($token)` stores it in `$_SESSION` → all subsequent `APIHelper::call` requests include `Authorization: Bearer <token>` plus the session cookie.

**Password hashing:** `Authentication::hashPassword` computes `hash_hmac('sha512', password + nonce, site_key)`. Per-user nonce is a `mt_rand` integer stored in the DB. The site-wide HMAC key means a key compromise invalidates every password hash — this should be migrated to bcrypt or argon2id.

**CSRF:** `UserSession::getCSRFKey()` generates a random 10-character string stored in `$_SESSION['SESSION_CSRF_KEY']`. Each form submits it as `sesskey`; `UserSession::checkCSRFKey()` validates it server-side.

**Google OAuth:** `UserDao::requestAuthCode` / `loginWithAuthCode`. On first login, auto-creates the user account and sets `profile_completed=1` (pending Code of Conduct acceptance).

**Per-request middleware** (`Middleware::beforeDispatch`): refreshes session from token expiry, loads the current user, computes role flags (`site_admin`, `ngo_orgs`, `user_has_active_tasks`) and injects them into the Smarty template data for every page.

**Permission middleware methods** (17 helpers in `Middleware.class.php`): `authUserIsLoggedIn`, `authIsSiteAdmin`, `authIsSiteAdmin_or_PO`, `authIsSiteAdmin_or_COMMUNITY`, `authIsSiteAdmin_any_or_FINANCE`, `authIsSiteAdmin_any_or_org_admin_or_po_for_any_org`, `authenticateUserForTask`, `authUserForOrg`, `authUserForOrgTask`, `authUserForOrgProject`, `authUserForProjectImage`, `authUserForTaskDownload`, etc. Each queries `AdminDao` and checks role bitmask intersection.

---

### 3.6 System Architecture

```
Browser
  │  HTML + jQuery 1.9 + Quill + protobuf-in-JS + Dart/Polymer (legacy, compiled)
  │  HTTPS
  ▼
SOLAS-Match/index.php  ── Slim 4 UI app ── Smarty 4 templates
  │   Route Handlers: UserRouteHandler, TaskRouteHandler,
  │                   ProjectRouteHandler, OrgRouteHandler, AdminRouteHandler
  │   Middleware: session cookie, CSRF, per-route auth checks
  │   Localisation: XML files via Localisation.php
  │
  │   UI DAOs call API over internal HTTP via curl (APIHelper.class.php)
  │   (UI → API is a same-server HTTP round-trip with Bearer token)
  ▼
SOLAS-Match/api/dispatcher.php ── Slim 4 API app
  │   OAuth2 Bearer token validation (league/oauth2-server)
  │   Routes: v0/Users.php, v0/Tasks.php, v0/Projects.php, v0/Orgs.php,
  │           v0/Admins.php, v0/IO.php, v0/Tags.php, v0/Badges.php,
  │           v0/Static.php, v0/Countries.php, v0/Langs.php
  │
  │   API DAOs call PDOWrapper::call(stored_proc_name, args)
  ▼
MySQL 8  (utf8mb4_unicode_ci)
  ├── ~100 tables
  └── 300+ stored procedures  ← canonical home of all business logic
  ▲
  │ same DB credentials, same tables
  │
SOLAS-Match-Backend  (C++ Qt 6 daemon — single binary PluginHandler)
  ├── PluginHandler/main.cpp         Qt plugin discovery, one thread per plugin
  ├── CorePlugin                     three queue pollers (250 ms interval)
  │     ├── TaskQueueHandler         handles: RUNDEADLINECHECK, RUNSTATISTICSUPDATE
  │     ├── UserQueueHandler         handles: RUNTASKSTREAM
  │     └── ProjectQueueHandler      handles: ~20 email notification types
  ├── PluginScheduler                XML cron (schedule.xml)
  │     ├── 07:05 daily   → DeadlineCheckRequest
  │     ├── 09:00 + 1 hr  → StatisticsUpdateRequest
  │     └── 08:10 + 1 hr  → UserTaskStreamNotificationRequest
  └── EmailPlugin                    ctemplate ({{...}}) → Qxt SMTP → qxt_smtp_emails table
        └── 30+ Generator classes ── templates/emails/*.tpl (53 templates)
```

---

### 3.7 Async Work Model (Post-RabbitMQ)

The PHP API enqueues async work into the `queue_requests` table by calling `UserDao::insert_queue_request($queueId, $type, $taskId, $userId, $orgId, $badgeId, $claimantId, $feedback)`. The three C++ queue handlers poll their respective queues every 250 ms, acquire a `QMutex::tryLock`, fetch one row, dispatch based on the integer `type` code, call the appropriate handler, then mark the row handled via `mark_queue_request_handled`.

Queue type codes are defined in both PHP (`api/dispatcher.php:63-82`) and C++ (`Common/Definitions.h:73-96`) and **must stay in sync**. A `STOP_consumeFromQueue` sentinel file (if placed on disk at the configured path) causes the daemon to pause queue consumption — an ops-level kill switch.

Email priorities on the `emails` table: `HIGH=3`, `MEDIUM=2`, `LOW=1`, `SENT=0`. The actual SMTP outbox is the `qxt_smtp_emails` table, written by `UserDao::queue_email`.

---

### 3.8 Email System

**53 email templates** in `SOLAS-Match-Backend/templates/emails/` using Google ctemplate's mustache-like `{{VARIABLE}}` and `{{#SECTION}}...{{/SECTION}}` syntax.

Each template has a corresponding C++ Generator class in `EmailPlugin/Generators/`. The generator queries the DB (via TaskDao, UserDao, ProjectDao), assembles a ctemplate dictionary, and calls the base `IEmailGenerator` to render the template and enqueue via `queue_email`.

**Notable templates and logic:**
- `password-reset.tpl` — plain text with `{{USERNAME}}`, `{{URL}}`, `{{SITE_NAME}}`.
- `user-task-claim.tpl` — includes conditional sections `{{#TRANSLATION}}`, `{{#REVISING}}`, `{{#REVISING_NO_MATECAT}}`, `{{#SEGMENTATION}}` with links to Kató TM (Phrase) and the Discourse community.
- `user-task-claim-memsource.tpl`, `user-task-claim-chunk.tpl`, `user-task-claim-verification.tpl` — variants selected by `UserTaskClaimEmailGenerator.cpp:108-120` based on whether the task has a Phrase/Memsource job, is chunked, or is a test/verification task.
- `user-task-stream.tpl` — fully-styled HTML newsletter with a `TASK_SECT` loop, CLEAR Global branding, deadline and word count per task, social media footer.

**`TaskStreamNotificationHandler`** (in `CorePlugin/UserJobs/`) contains notable production logic:
- For each pending user, fetches their top 25 matching tasks (`UserDao::getUserTopTasks`).
- Filters tasks with deadlines within the last `mail.task_stream_cutoff_months` (default: 3 months).
- Hard-caps at **8,000 emails per hour** total, using a circular rotation for fairness.
- Special-cases Microsoft/Hotmail domains (`@hotmail.com`, `@outlook.com`, `@msn.com`, `@live.com`, plus regional variants) with a separate `mail.max_microsoft_per_hour` cap and `defer_task_stream` logic for excess — a documented deliverability workaround for Microsoft's inbound filters.
- Contains an "Earthquake" emergency notification path (`get_users_list_for_earthquake`) capped at 1,000/hour.

Per-email send logging: `UserDao::log_email_sent(user_id, task_id, …, "tag")`.

---

### 3.9 External Integrations

| Integration | Direction | Purpose | Key code locations |
|---|---|---|---|
| **Phrase TMS / Memsource** (CAT tool — central to platform) | Bidirectional API + SSO | Creates/manages translation jobs; SSO for linguists; webhook triggers task completion | `UserDao::create_memsource_project` (~1,900 LOC), `claimTask`, `propagate_cancelled`, `update_phrase_field`, `set_dateDue_in_memsource`, `memsource_list_jobs`, `set_memsource_status`; `TaskDao::get_memsource_task`, `is_task_translated_in_memsource`, `get_matecat_url` |
| **Asana** | Outbound | Staff project management; quality task tracking | `AsanaProjects`, `AsanaTasks`, `asana_quality_tasks`, `asana_board_for_org` tables; `ProjectDao::follow_asana_tasks`; API key in conf |
| **Discourse** (community forum) | Outbound + linking | Per-project community discussion threads | `DiscourseID` table; `ProjectDao::discourse_parameterize`; URL embedded in claim emails |
| **Moodle** (TWB Learning Center) | Outbound DB | Tracks linguist learning progress | `moodle_datas`, `moodle_task_users`; `ProjectDao::moodle_db`; `UserDao::get_moodle_data_for_user`; `Common/lib/MoodleRest.php` |
| **Neon CRM** | Outbound | Organisation record matching | `OrgIDMatchingNeon`, `UserNeonAccount` tables |
| **HubSpot** | Outbound | Deal ID sync for business development | `hubspot_deals` table; `TaskDao::update_hubspot_deals` |
| **Sun Systems** (financial ERP) | Bidirectional | Purchase requisitions, PO polling, invoicing | `sun_purchase_requisitions`, `poll_sun`, `po_cut_off_sun`, `sun_po_errors`; `ProjectDao::poll_sun` (~190 LOC), `get_sun_access_token` |
| **Zahara** (procurement) | Bidirectional | Purchase order management | `zahara_purchase_orders`, `sync_po_events` |
| **DocuSign** | Webhook inbound | Linguist contract signing | `sent_contracts` table; routes `/docusign_redirect_uri`, `/docusign_hook`; `UserDao::update_sent_contract` |
| **Google OAuth + Drive** | Inbound auth, outbound API | User single sign-on; file access | `GoogleUserDetails` table; `UserDao::set_google_user_details`; vendored `google-api-php-client/` |
| **Slack** | Outbound notifications | Internal staff alerts | Webhooks in `conf.template.ini` |
| **Gravatar** | Outbound | User avatar images | `https://www.gravatar.com/avatar/{md5(email)}` in `header.tpl:292` |
| **Google Analytics** | Outbound | Site usage tracking | `gtag G-3Z3VNH71D6` in `header.tpl:48-54` |
| **Metabase** | Embedded | Business intelligence dashboards | `/metabase`, `/metabase_ngo` routes |

---

### 3.10 Localisation

XML-based, one file per locale: `ui/localisation/strings_<lang>.xml`. `Localisation::getTranslation('key_name')` resolves the key via XPath with fallback to `strings_en.xml`. Results are cached via `CacheHelper` for one hour. User locale is stored in the session via `UserSession::setUserLanguage`.

---

## 4. URL & Route Map

### 4.1 UI Routes (Slim app `index.php`)

**Public / authentication**
`home`, `home_ngo`, `task_stream`, `pagination`, `register`, `register_track`, `special_registration`, `login`, `logout`, `password-reset`, `googleregister`, `change_email`, `email_verification`

**User profile & settings**
`user-public-profile`, `user-private-profile`, `user_rate_pairs`, `user-admin-profile`, `user-uploads`, `user-task-stream-notification-edit`, `user-code-of-conduct`, `add_tracking_code`, `users_review`, `users_new`

**Tasks** (from `TaskRouteHandler`)
`task`, `task-view`, `task-uploaded`, `task-alter`, `task-claim-page`, `task_complete`, `task-org-feedback`, `task-user-feedback`, `task-review`, `claimed-tasks`, `claimed-tasks-paged`, `archived-tasks`, `recent-tasks`, `recent-tasks-paged`, `archive-task`, `download-task`, `download-task-latest-version`, `download-task-version`, `download-task-external`, `task-search_translators` (4 variants), `task-invites_sent`

**Projects** (from `ProjectRouteHandler`)
`project-view`, `project-create`, `project-create-empty`, `project-add-shell-tasks`, `project-created`, `project-alter`, `archive-project`, `change_owner`, `download-project-file`, `download-project-image`, `project_get_wordcount`, `memsource_hook`, `project_cron_1_minute`, `task_cron_1_minute`, `cron_every_hour`

**Organisations** (from `OrgRouteHandler`)
`create-org`, `org-dashboard`, `org-projects`, `metabase_ngo`, `org-private-profile`, `org-public-profile`, `partner_deals`, `org-manage-badge`, `org-create-badge`, `org-edit-badge`, `org-search`, `org-task-complete`, `org-task-review`, `org-task-reviews`, `org_members`, `set_entitlement`

**Administration** (from `AdminRouteHandler`)
`site-admin-dashboard`, `list_memsource_projects`, `community_dashboard`, `analytics`, `metabase`, `deal_id_report`, `paid_projects`, `all_deals_report`, `po_readyness_report`, `sun_po_errors`, `sow_report`, `sow_linguist_report`, `pr_report`, `po_report`, `set_invoice_paid`, `set_invoice_bounced`, `set_invoice_revoked`, `set_piem_text`, plus additional admin tools

**Static & content**
`faq`, `privacy`, `terms`, `videos`, `statistics`, `maintenance`, `content_items`, `content_display`, `content_list`, `content_item_org`

**Tags & badges**
`tag-list`, `badge-list`

### 4.2 API Endpoints (`api/v0/*.php`, approximately 100 routes)

| File | Content |
|---|---|
| `Users.php` | ~50 routes: CRUD, auth, login/register, OAuth code exchange, badges, blacklist, tracked tasks/projects, claimed-task pagination, password reset, banned-comment |
| `Tasks.php` | orgFeedback, tags, version info, claim status, archive, recordView, proofreadTask |
| `Projects.php` | CRUD, task listing, tags, file upload, image approval, archive |
| `Orgs.php` | CRUD + extended profile, search by name; on create triggers org-created notification |
| `Admins.php` | Ban/unban users/orgs, revoke tasks, list banned (all gated by `authenticateSiteAdmin`) |
| `IO.php` | File upload/download using Apache X-Sendfile; MIME-type detection map for Office documents |
| `Tags.php` | Tag CRUD |
| `Badges.php` | Badge CRUD |
| `Static.php` | Static lookup data |
| `Countries.php` | Country list |
| `Langs.php` | Language list |

---

## 5. Migration Assessment — Next.js / React Native

### 5.1 What Can Transfer Cleanly

| Asset | Reusable | Notes |
|---|---|---|
| **Database schema** (`db/schema.sql`) | Yes | Tables, columns, and foreign keys are sound and can be kept as-is, or ported to Postgres. Stored procedures are the bigger challenge — see §5.3. |
| **Protobuf model definitions** | Yes | PHP models in `Common/protobufs/models/*.php` are generated from `.proto` files. Regenerate as TypeScript types, or generate Zod schemas directly. |
| **Email templates** (53 files) | Mostly | `templates/emails/*.tpl` are mustache-syntax ctemplate files. Port to Handlebars, MJML, or React-Email components. Per-template conditional logic (chunk vs. Memsource vs. verification variants) lives in the C++ generator code and must be reimplemented alongside the templates. |
| **Localisation strings** | Yes | XML → JSON conversion is straightforward. Fallback-to-English logic is trivial. Use i18next or similar. |
| **Business rules in stored procedures** | Conceptually | The stored procs are the authoritative specification for claim eligibility, prerequisite cascades, blacklisting, qualification matching, and more. Read them as the spec; rewrite as TypeScript service functions or as Postgres functions initially. |
| **Role bitmask, task type/status enums** | Yes | Direct constant copy into TypeScript. |
| **External integration credentials & contracts** | Yes | All API URLs, scopes, and auth flows are documented in `conf.template.ini` and the DAO code. |
| **Static assets** | Yes | Images, CSS, brand assets in `ui/img/`, `resources/`. |

---

### 5.2 What Must Be Rewritten

| Component | Why | Migration target |
|---|---|---|
| **Smarty 4 templates** (~120 `.tpl` files) | Server-rendered HTML with PHP method calls embedded inside `{}`. Not portable to any JS framework. | React components in Next.js pages or app router. Templates are predominantly forms and paginated lists — direct conceptual mapping. |
| **Slim 4 UI route handlers** (4 handler classes, ~7,600 LOC total) | PHP handlers with a global `$template_data` bag, flash messages via custom middleware, server-side redirects. The "named routes" pattern (`urlFor`) is the only clean abstraction. | Next.js file-based routing (or `app/` directory). Flash messages → toast notifications via state library. `urlFor` → typed Next.js routes. |
| **Slim 4 REST API** (~11 endpoint files, ~2,000 LOC) | OAuth2 password grant via `league/oauth2-server` dev-master (stale/deprecated pin). | Next.js Route Handlers, or NestJS/Fastify. Replace auth with NextAuth / Auth.js, or move to standard OIDC. |
| **API DAOs and UI DAOs** (PHP) | Large classes wrapping stored proc calls plus heavy integration glue. `UserDao` is 2,967 LOC; `ProjectDao` is 2,334 LOC; `TaskDao` is 1,207 LOC. Data access and integration logic are not separated. | Decompose into TypeScript service modules: `services/auth/`, `services/tasks/`, `services/integrations/phrase/`, `services/integrations/asana/`, etc. Keep stored proc calls behind a thin DB layer for v1, then migrate procs to application code incrementally. |
| **Hand-rolled session crypto** (`Middleware::SessionCookie`) | AES-128-CBC + HMAC-SHA1, 4 KB cookie cap, custom binary format. | Replace with Auth.js (encrypted JWT) or iron-session. Same UX, modern crypto, battle-tested implementation. |
| **Custom CSRF** (`UserSession::getCSRFKey`) | Manual 10-character token injected into every form, validated manually. | Built-in CSRF protection via Next.js Server Actions, or a standard `csrf` middleware. |
| **Password hashing** (`hash_hmac sha512` + per-user nonce + global site key) | A site-key compromise invalidates every password in the system. | bcrypt or argon2id. Migrate on login: re-hash with argon2 when the user supplies a correct password that still uses the old scheme. |
| **Internal curl API client** (`APIHelper.class.php`) | UI makes HTTP-to-self via curl, parsing cookies and OAuth tokens at every request. | In Next.js, server components call service functions directly — no internal HTTP round-trip needed. Mobile (React Native) calls the API directly using a stored bearer token. |
| **Hand-rolled JavaScript** (37 files, jQuery 1.9 era) | Manual DOM manipulation, version-number-suffixed filenames for cache busting, protobuf-in-browser. | React components per page. TanStack Query for data fetching and server state. Zustand or similar for client state. |
| **Quill rich-text editor** | Currently loaded from CDN. | TipTap or Lexical in React. |
| **Dart sub-app** (`ui/dart/`) | Polymer / Dart 1, end-of-life since ~2017. Production runs the dart2js-compiled output. | Discard entirely. Re-implement the four affected pages (`UserPrivateProfile`, `ClaimedTasks`, `ProjectCreate`, `ProjectAlter`) as native React components. Behaviour can be inferred from the corresponding `.tpl` templates. |
| **C++ Qt backend daemon** (entire `SOLAS-Match-Backend/`) | Qt 6 + ctemplate + Qxt SMTP + custom Qt plugin loader, polling DB every 250 ms. High maintenance burden; incompatible with a JS-shop stack. | **Replace wholesale** with a Node.js worker. The pattern is straightforward: a job queue worker (BullMQ + Redis, Inngest, Trigger.dev, or AWS SQS + Lambda) consuming from `queue_requests`. Scheduled tasks from `schedule.xml` → cron jobs (Vercel Cron, GitHub Actions, or node-cron). Email generators → React-Email + Resend / SES. The 53 `.tpl` files map 1-to-1 to React-Email components. All individual job handlers (DeadlineChecker, TaskStreamNotificationHandler, OrgCreatedNotifications, TaskRevokedNotificationHandler, SendTaskUploadNotifications) are well-scoped, mostly under 200 LOC each, and straightforward to port. |
| **Apache X-Sendfile file downloads** | Apache-specific header; not portable. | Stream from S3-compatible storage using signed URLs, or a Next.js Route Handler that streams the file after authorisation check. The existing `proj-{id}/task-{id}/v-{version}/{filename}` path structure translates cleanly to an S3 key prefix. |
| **Hand-rolled HTTP clients** (MoodleRest.php, Asana cURL, Phrase cURL, HubSpot cURL) | Bespoke cURL wrappers with no retry or error-normalisation logic. | Use official SDKs where available (Phrase TMS has an official TS SDK). Use `fetch` with proper error handling and retry for others. |

---

### 5.3 Risks & Gotchas

**1. Stored procedures are the canonical specification.**
With 300+ stored procedures encoding claim eligibility, prerequisite cascades, blacklist logic, qualification matching, native-language enforcement, restricted tasks, paid eligibility, language-pair limits, MT permissions, and organisation entitlements, the 16,599-line `schema.sql` is effectively the application's brain. Every procedure must be read and understood before the logic can be rewritten in TypeScript. Building a golden test suite from production traffic before touching the procs is essential. This is the single highest-risk item in the migration.

**2. Phrase TMS coupling is deep and central.**
`UserDao::create_memsource_project` is approximately 1,900 lines. `UserDao::claimTask`, `propagate_cancelled`, `update_phrase_field`, `set_memsource_status`, `is_task_translated_in_memsource`, `get_matecat_url`, `get_memsource_task` together add several thousand more lines. The task lifecycle on the TWB Platform *is* the task lifecycle on Phrase. Webhooks from Phrase (the `memsource_hook` route) drive task completion. Plan a dedicated integration service tier with extensive contract tests against a Phrase sandbox environment.

**3. Dual enum/constant duplication between PHP and C++.**
Integer codes for queue message types (e.g., `EmailVerification=13`, `PasswordResetEmail=5`) are defined identically in `api/dispatcher.php:63-82` and `Common/Definitions.h:73-96`. Rows in `queue_requests` reference these integers. Any migration must preserve these values (or perform a one-shot data migration) until all producers and consumers are cut over simultaneously.

**4. Hard-coded absolute paths throughout the codebase.**
`/repo/SOLAS-Match/Common/conf/conf.ini`, `/etc/SOLAS-Match/templates/`, `/repo/SOLAS-Match-Backend/STOP_consumeFromQueue`, and paths inside `index.php` and `api/dispatcher.php` require specific filesystem layouts. Sweep the entire codebase for these before containerisation or cloud deployment.

**5. Session migration across the cutover.**
The custom `slim_session` cookie format is incompatible with all standard auth libraries. Strategy options: (a) deploy the new system on a new subdomain and force re-login; (b) implement a bridge endpoint that reads the legacy cookie and re-issues a modern JWT; (c) accept a forced logout for all users at cutover.

**6. Microsoft/Hotmail email deliverability throttling.**
`TaskStreamNotificationHandler.cpp` implements a separate per-hour cap for Microsoft-domain addresses (`@hotmail.com`, `@outlook.com`, `@msn.com`, `@live.com`, and regional variants) with deferred sending for excess messages. This reflects real production experience with Microsoft's inbound rate limiting. This logic must be preserved when replacing the daemon — domain-based throttling should be a first-class feature in the new email sender.

**7. Protobuf-in-browser decision.**
The current system uses protobuf for browser–server communication (unusual). For the mobile (React Native) scope, decide early whether to share protobuf models and use gRPC, or move to plain JSON + Zod schemas for both web and mobile. Most teams choose the latter; the protobuf schemas are useful as a model specification regardless.

**8. Vagrant dev environment is misleading.**
`provision.sh` targets PHP 5 / MySQL 5.5 / Ubuntu 14.04 / Qt 5 / RabbitMQ. The production codebase requires PHP 8 / MySQL 8 / Qt 6 / no RabbitMQ. Do not use the Vagrant scripts as a basis for new dev environment setup. The actual production environment spec must be obtained from the operations team.

**9. OAuth2 dependency is stale.**
`league/oauth2-server` is pinned to a specific dev-master commit from several years ago. The migration is the right moment to adopt a modern, standards-compliant auth library (Auth.js, Keycloak, Auth0) rather than forward-porting the existing pin.

**10. No automated tests.**
No `phpunit.xml`, no CI configuration, and no test suite in `SOLAS-Match/test/` (it is a tools directory, not a test suite). The migration must build a test corpus from scratch. Start by writing integration tests against the live API (the REST contract is the only externally observable specification), then use those as a regression harness during the port.

**11. `phpinfo.php` at document root.**
This file exposes server configuration details. It must be removed before any public-facing deployment.

**12. Hand-rolled HTML sanitisation in templates.**
`TemplateHelper::uiCleanseHTMLKeepMarkup` (at `TemplateHelper.php:434-466`) uses a sentinel-substitution scheme (`@BRAxyz@`, `@KETxyz@`) to preserve certain markup while escaping the rest — it is not a proper HTML sanitiser. Several templates render user-supplied flash message content through this function. Replace with DOMPurify (client-side) or sanitize-html / zod-html (server-side) in the new stack.

---

### 5.4 Suggested Migration Phasing

**Phase 0 — Instrument and test (prerequisite)**
Add structured logging to the PHP API and to the C++ daemon. Capture every `queue_requests` dispatch. Build a contract test suite by replaying representative production traffic against a staging environment. This is the safety net for every subsequent phase.

**Phase 1 — Replace the C++ daemon (lower risk, high payoff)**
Rewrite `SOLAS-Match-Backend` as a Node.js worker service (BullMQ on the same `queue_requests` table, or a Redis-bridged equivalent). Port email templates to React-Email + Resend or AWS SES. Port DeadlineChecker, TaskStreamNotificationHandler, OrgCreatedNotifications, TaskRevokedNotificationHandler, SendTaskUploadNotifications, and the remaining job handlers. The PHP frontend continues writing to `queue_requests` unchanged; the new worker reads it. Cut over one queue handler at a time with a feature flag. This phase removes the C++ build dependency and makes the email system maintainable by a JS team.

**Phase 2 — Next.js frontend against the existing Slim API**
Build the new Next.js frontend talking to the unchanged Slim REST API. Migrate pages incrementally (task stream, task view, profile, project creation first — the high-traffic paths). Keep the Slim API as the stable contract surface. Switch domains or use a reverse proxy to route traffic gradually.

**Phase 3 — Replace the REST API**
Introduce Next.js Route Handlers (or a NestJS/Fastify service) that call the MySQL stored procedures directly, bypassing the Slim API layer. Retire the PHP API once the new layer has parity. This also eliminates the internal curl round-trip.

**Phase 4 — Migrate stored procedures to application code**
Move business logic out of MySQL stored procedures and into TypeScript service functions. This is the largest and riskiest phase — tackle in domain-by-domain slices (auth, task claim, project creation, qualifications, finance). Each slice needs a before/after contract test.

**Phase 5 — React Native mobile app**
Consumes the same API produced in Phase 3, with a thin BFF (Backend for Frontend) layer if mobile needs differ materially from web. The volunteer linguist task-claim workflow (browse task stream → claim → translate → complete) is the obvious first mobile MVP, as it covers the highest-volume user journey.

---

## 6. Files Inspected

The following files were read during this audit:

**Frontend root**
- `SOLAS-Match/README.md`
- `SOLAS-Match/index.php`
- `SOLAS-Match/api/composer.json`
- `SOLAS-Match/api/dispatcher.php`
- `SOLAS-Match/ui/composer.json`
- `SOLAS-Match/db/schema.sql`

**Configuration**
- `SOLAS-Match/Common/conf/conf.template.ini`
- `SOLAS-Match-Backend/conf.template.ini`
- `SOLAS-Match-Backend/schedule.xml`

**Common enums & libraries**
- `SOLAS-Match/Common/Enums/*.class.php` (10 files)
- `SOLAS-Match/Common/lib/ModelFactory.class.php`
- `SOLAS-Match/Common/lib/APIHelper.class.php`
- `SOLAS-Match/Common/lib/UserSession.class.php`
- `SOLAS-Match/Common/lib/Authentication.class.php`
- `SOLAS-Match/Common/lib/Settings.class.php`
- `SOLAS-Match/Common/lib/SolasMatchException.php`

**API DAOs and endpoints**
- `SOLAS-Match/api/v0/Users.php`, `Tasks.php`, `Projects.php`, `Orgs.php`, `Admins.php`, `IO.php`, `Tags.php`, `Badges.php`, `Static.php`, `Countries.php`, `Langs.php`
- `SOLAS-Match/api/DataAccessObjects/*.class.php`

**UI layer**
- `SOLAS-Match/ui/lib/Middleware.class.php`
- `SOLAS-Match/ui/lib/TemplateHelper.php`
- `SOLAS-Match/ui/lib/Localisation.php`
- `SOLAS-Match/ui/lib/Validator.php`
- `SOLAS-Match/ui/RouteHandlers/UserRouteHandler.class.php`
- `SOLAS-Match/ui/RouteHandlers/TaskRouteHandler.class.php`
- `SOLAS-Match/ui/RouteHandlers/ProjectRouteHandler.class.php`
- `SOLAS-Match/ui/RouteHandlers/OrgRouteHandler.class.php`
- `SOLAS-Match/ui/RouteHandlers/AdminRouteHandler.class.php`
- `SOLAS-Match/ui/DataAccessObjects/UserDao.class.php` (signatures; 2,967 LOC)
- `SOLAS-Match/ui/DataAccessObjects/ProjectDao.class.php` (signatures; 2,334 LOC)
- `SOLAS-Match/ui/DataAccessObjects/TaskDao.class.php` (signatures; 1,207 LOC)
- Selected Smarty templates: `header.tpl`, `index-home.tpl`, `project/project.create.tpl`, `task/task.view.tpl`, `user/user-private-profile.tpl`

**Backend daemon**
- `SOLAS-Match-Backend/README.md`
- `SOLAS-Match-Backend/SOLASMatchWorkerDaemon.pro`
- `SOLAS-Match-Backend/Common/Common.pro`
- `SOLAS-Match-Backend/Common/Definitions.h`
- `SOLAS-Match-Backend/Common/ConfigParser.cpp`
- `SOLAS-Match-Backend/Common/MySQLHandler.cpp`
- `SOLAS-Match-Backend/PluginHandler/main.cpp`
- `SOLAS-Match-Backend/PluginScheduler/PluginScheduler.cpp`
- `SOLAS-Match-Backend/CorePlugin/CorePlugin.cpp`
- `SOLAS-Match-Backend/CorePlugin/TaskQueueHandler.cpp`
- `SOLAS-Match-Backend/CorePlugin/UserQueueHandler.cpp`
- `SOLAS-Match-Backend/CorePlugin/ProjectQueueHandler.cpp`
- `SOLAS-Match-Backend/CorePlugin/TaskJobs/DeadlineChecker.cpp`
- `SOLAS-Match-Backend/CorePlugin/UserJobs/TaskStreamNotificationHandler.cpp`
- `SOLAS-Match-Backend/EmailPlugin/Email.cpp`
- `SOLAS-Match-Backend/EmailPlugin/IEmailGenerator.cpp`
- `SOLAS-Match-Backend/EmailPlugin/EmailDefinitions.h`
- `SOLAS-Match-Backend/EmailPlugin/Generators/PasswordResetEmailGenerator.cpp`
- `SOLAS-Match-Backend/EmailPlugin/Generators/UserTaskClaimEmailGenerator.cpp`
- `SOLAS-Match-Backend/templates/emails/password-reset.tpl`
- `SOLAS-Match-Backend/templates/emails/user-task-claim.tpl`
- `SOLAS-Match-Backend/templates/emails/user-task-stream.tpl`

**Deployment**
- `SOLAS-Match/vagrant/Vagrantfile`
- `SOLAS-Match/vagrant/provision.sh`
- `SOLAS-Match/vagrant/assets/001-match.conf`
- `SOLAS-Match/ui/dart/pubspec.yaml`

**Files not read in full** (available for deeper investigation if needed):
Full stored procedure bodies in `schema.sql`; individual JS files in `ui/js/`; the remaining 50 email templates; all 30 individual C++ Generator classes; the ~115 Smarty templates not specifically reviewed; full `UserRouteHandler` body; `Common/protobufs/models/*.php` (regular generated code); `MoodleRest.php`; `MemsourceTimezone.class.php`; `CacheHelper.class.php`; `Serializer*.class.php`. None of these change the migration assessment, but each represents a known unit of work to size when scoping individual migration phases.
