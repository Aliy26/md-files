CosMos AI V2 — System Scope and Complexity Analysis
Analysis date: 2026-10-14 repository snapshot
Repository: cosmosai_V2-rebuild
Basis: Current FastAPI/Next.js implementation, ORM models, Alembic migrations, tests, deployment configuration, help content, and runtime scripts.
Purpose: Evaluate the platform's functional breadth, architectural intricacy, operational burden, and likely maintenance effort.
1. Executive assessment
CosMos AI V2 is the internal operations platform used to manage KCCI workforce-development programs and their advertising lifecycle. Its scope combines:
tenant and user administration;
business, company, course, cohort, agency, and campaign master data;
campaign workflow, creative review, attachments, performance entry, and UTM tracking;
operational calendars and weekly staff reporting;
PDF business-document generation, configurable/manual document chains, and season-level PDF assembly;
a durable notification outbox with two outbound-mail transports;
an AI email drafting assistant;
analytics, ranking, budgets, report exports, and reference-library functions;
issuing-information, ad-product, document-layout, and mail-management consoles;
platform feedback, error tracking, backups, Slack monitoring, Wiki export, and super-admin developer tools.
The application is a modular monolith. Its complexity comes from six user roles, organization/business/company/vendor scopes, many-to-many participation relationships, role-specific read/write rules, document workflows, scheduled work, and external integrations.
Headline size indicators
Indicator	Current measured value
FastAPI route handlers	239
Backend API modules containing handlers	43
Next.js pages	54
ORM/application tables at current code head	51
Physical MySQL tables after migration 0065	52, including alembic_version
Physical tables in the inspected local DB at revision 0064	50: 49 application tables plus alembic_version
Documented production database revision	0065
Alembic migration files	66
Service-layer Python files	88
Repository-layer Python files	37
Backend test modules	162 test_*.py files
Playwright E2E specifications	25
Supported application roles	6
Externally triggered cron endpoints	7
Notification templates / governed events	15 rendered templates / 16 events including preference-only document_issued
User-facing languages	Korean and English
System profile: a live, stateful business-operations platform. Maintenance requires knowledge of the operating domain as well as the FastAPI, Next.js, MySQL, deployment, security, document, and mail subsystems.
2. What the platform is for
The product supports organizations that run workforce-development businesses or programs (academies, called 사업 in the UI) and coordinate training companies, courses, cohorts, advertising agencies, campaigns, creatives, reports, schedules, and administrative work.
The core operational chain is:
Organization
  ├─ Business/Academy participation
  │    ├─ Companies
  │    │    └─ Courses
  │    │         └─ Seasons/Cohorts
  │    │              ├─ Training rounds
  │    │              └─ Campaigns
  │    │                   ├─ Agency
  │    │                   ├─ Creatives and approvals
  │    │                   ├─ Attachments
  │    │                   ├─ Performance
  │    │                   ├─ UTM/tracking
  │    │                   └─ Generated and closing documents
  │    ├─ Schedules
  │    ├─ Weekly work reports
  │    ├─ Budgets
  │    └─ Work groups
  └─ Users, settings, notifications, feedback, and audit records
Companies are organization-level entities that can participate in multiple businesses through company_academies. Agencies have a similar participation relationship through agency_academies. Access and workflow rules therefore traverse relationship tables rather than a single hierarchy.
3. Architecture
3.1 Runtime topology
The repository is a two-application monorepo:
Web application: Next.js 16, React 19, TypeScript, Tailwind CSS 4, next-intl.
API application: FastAPI, Python 3.11+, Pydantic 2, SQLAlchemy 2, Alembic.
Database: MySQL 8 only.
In normal operation:
Browser
  → Next.js web service
  → same-origin /api/* rewrite
  → FastAPI service
  → MySQL and external integrations
The browser does not need a public API URL. Next.js proxies /api/* to the internal API service, keeping the session cookie first-party and avoiding most browser CORS complexity.
Core local/production containers are:
db;
api;
web;
monitor in production, observing API/web logs and API readiness and posting to Slack;
optional local Mailpit overlay for safe mail testing.
The runtime is a modular monolith with one API process boundary and one relational database. Cross-domain behavior is implemented within the API service.
3.2 Backend layering
The backend uses a conventional layered design:
API router → service/use-case layer → repository layer → SQLAlchemy model → MySQL
Measured layer sizes:
Layer	Size	Responsibility
api/app/api	45 Python files	HTTP contracts, dependencies, error translation
api/app/services	88 Python files	Business rules and workflows
api/app/repositories	37 Python files	Scoped data access and query composition
api/app/schemas	39 Python files	Pydantic request/response models
api/app/models	45 Python files	ORM persistence models
Business rules are primarily implemented in services rather than route handlers. Authorization also occurs in services, so feature analysis follows the full route → service → repository path.
3.3 Frontend architecture
The frontend is a server-built Next.js application with client-heavy operational pages. It includes:
54 page routes plus one short-link route handler;
a shared authenticated app shell;
role-filtered navigation;
URL-persisted filters;
responsive mobile/desktop navigation;
Korean/English translations;
charting, Excel generation, PDF/browser capture, QR codes, and calendar interfaces.
The app shell calls /auth/me, redirects anonymous sessions, and filters menus through canSeeNav. The API remains the security authority: hiding a menu item is a user-experience measure, not an access-control boundary.
3.4 Deployment and operations
V2 is live in production for KCCI. The current production path is manual: build the API, web, and monitor images for linux/amd64 on a workstation, push immutable short-SHA tags to NCP Container Registry, update IMAGE_TAG, run Alembic before container replacement, and deploy with Docker Compose behind nginx. GitHub Actions remains manual/dormant. An infrastructure-led move to IDC/Coolify is planned, including a staging environment at cosmos-stg.jcg.zone; until that cutover occurs, the manual NCP path remains authoritative.
Operational safeguards include:
explicit Alembic migration step before serving a release;
startup database-revision checking;
configurable SQLAlchemy pool capacity/timeouts/recycling, capped sync-worker concurrency, and /health/ready pool-exhaustion signaling;
UTC database sessions with KST-specific business-date handling;
structured JSON logs and stable event names;
daily compressed MySQL backups to NCP object storage;
an always-present production monitor sidecar plus daily Slack error digest;
rate limiting, security headers, centralized error capture, and access logging;
local pre-push quality gates and manual CI.
Operational constraints:
CI is manual-only; local hooks carry much of the merge-gate responsibility.
Rate-limit state is in-process and will multiply with replicas.
Uploaded object storage has no repository-documented secondary backup.
Host cron freshness and backup-failure delivery still depend on external operational discipline.
There is no Redis/Celery-style queue or worker tier.
4. Functional scope
4.1 Tenant and master-data management
The platform manages:
organizations;
businesses/academies;
organization-level companies and their business participation;
courses and provider companies;
seasons/cohorts and dates;
1:N administrative training rounds below a season, including planned, started, dropout, completed, and employed counts, coding-test code, employment center, report status, ordering, filtering, duplication, and per-round training-period overrides;
season bulk import from read-only Google Sheets, season cloning, channel enrollment, and operations-status views;
advertising agencies and their business participation;
work groups spanning several businesses;
users, invitations, roles, status, and assignments;
system and organization settings.
Training operations are integrated into the season list rather than exposed as a separate page. The view aggregates round data into dashboard totals and company charts, groups rows by company → course → season → round, and supports inline edits, creation, duplication, and deletion. Administrative managers use the list-only workflow; other authorized roles can switch between card and list views.
It also includes a one-time V1-to-V2 migration script. Some V2 modules—schedules, notifications, email-assistant records, and newer operational tables—start empty at cutover because there is no direct V1 source.
The current Korean product vocabulary has also changed over time: the UI now distinguishes 사업, 기업, 기수, 업체, 매체, and 광고상품, while many stable database/API identifiers still use legacy English names such as academy, agency, utm_source, and utm_medium. This translation layer is itself a maintenance cost: changing a label is not the same as changing the persisted domain model.
4.2 Campaign and creative workflow
Campaigns form a major domain subsystem. The platform supports:
campaign creation and editing;
status lifecycle/history;
agency assignment;
campaign attachment management;
creative upload, assignment, status history, review, approval, and change requests;
a review queue surfaced in the UI;
start/end date and time transitions;
preflight warnings;
performance capture and campaign ranking;
UTM source/medium configuration and tracked short links.
Campaign status changes can trigger notifications and document generation. These are not isolated CRUD screens; they form a stateful workflow with dependencies among campaigns, agencies, creatives, dates, documents, and approvals.
4.3 Performance, analytics, and reports
The product contains:
performance dashboards;
campaign and channel analysis, including a dedicated platform breakdown page and browser-side PDF export;
campaign ranking;
per-campaign performance entry;
academy budget reporting;
company efficiency and comparison reports;
reference-library Excel generation;
promotional/publication/expenditure-related document outputs;
season upload/status reports.
This is reporting over manually managed campaign data; the README explicitly excludes automatic ad-platform synchronization and ClickHouse.
Export-format downloads
The Reference Library has a server-side export suite rather than relying only on browser printing:
Overall performance .xlsx: one workbook across the selected period/business, with a 성과 데이터 sheet containing season rows and their campaign data plus a 채널별 집행 online/offline channel rollup.
Company-efficiency .xlsx: one row per provider company, covering course/season counts, achievement, advertising and creative costs, enrollment, CPA, and budget utilization. The on-screen preview and workbook use the same computed rows. A provider_readonly user is automatically restricted to its own company.
By-course .zip: one .xlsx workbook per course, with all matching seasons for that course. Duplicate/truncated filenames are made unique rather than overwriting an earlier workbook.
Empty-data behavior: overall/company exports can intentionally produce header-only workbooks after client confirmation; an empty by-course archive is refused because it would contain no files.
Download contracts: the API supplies UTF-8 filenames through Content-Disposition and marks generated downloads Cache-Control: no-store.
These exports are generated with openpyxl and zipfile and are separate from the PDF approval/document assembly described below.
Season and campaign PDF assembly
The Reference Library includes a pypdf/ZIP assembly subsystem for campaign documents grouped by season:
Promotional approval (홍보 품의서), one PDF per season
The operator selects and orders campaigns in a season.
The server collects each campaign's latest quotation and matching service-description PDF.
It merges all quotations in selected campaign order, followed by all service descriptions, into one season-level PDF.
The source may be an uploaded campaign attachment or the latest auto-generated document; an uploaded attachment takes precedence.
Expenditure approval (지출 품의서), ZIP containing one merged PDF per campaign
The operator selects campaigns after a readiness check identifies missing documents.
Each campaign PDF is merged in a strict business order: transaction statement → tax invoice → publication report → result report for online campaigns → master approval → season sub-approval/or additional sub-approval → quotation → service description.
The API resolves campaign documents, season-level approval attachments, and academy/common documents into the bundle.
Offline campaigns correctly omit the result-report requirement.
The ZIP uses collision-safe filenames so campaigns with equivalent agency/platform names do not overwrite each other.
Publication/placement reports (게재보고서), one PDF per season
The operator can select and reorder campaigns.
The server merges each campaign's latest publication report into one season-level PDF in that order.
Campaigns with missing, corrupt, or unreadable reports are skipped and returned in X-Skipped-Campaigns, allowing the completed download and a warning to coexist.
All three flows have preflight/readiness APIs and selection dialogs in the Reference Library UI. PDF loading attempts owner-password-compatible decryption, skips files that cannot be read or appended, and fails if no usable pages remain. The implementation composes documents across campaign, season, academy, common-document, and generated-document storage scopes while preserving order and tenant authorization.
4.4 Files and document automation
File features use NCP's S3-compatible object storage:
season attachments;
campaign attachments;
creative assets;
common documents;
feedback screenshots;
generated PDFs.
The business-document subsystem uses Jinja2/HTML/CSS and WeasyPrint, plus pypdf/openpyxl where needed. It handles five document concepts:
quotation;
transaction statement;
tax invoice;
placement/publication report;
result report.
Document behavior is deliberately conservative:
quotation auto-issue/reissue exists, but is off by default per organization;
schedule, amount, or channel changes create a new quotation version only when that organization enables quotation auto-issue;
placement reports are manual-only and built from operator-uploaded live-placement screenshots; they are not generated merely because a campaign starts or a creative changes;
attempt a result report when a campaign ends;
result report → statement → tax invoice can run automatically, but the organization-controlled chain defaults to manual, one confirmed issue step at a time;
generated documents do not automatically email everyone: automatic document mail is recipient opt-in, while an operator's explicit Send action goes to the resolved campaign recipients;
organizations can choose which uploaded or generated document types count toward closing-document completion.
Triggers are intentionally non-blocking. A document/mail failure should not roll back the campaign action that caused it. Generated documents and external-send attempts leave database records for replay and reconciliation.
Some document paths are feature- or template-gated. Therefore the architecture is implemented more broadly than every document type is necessarily production-ready.
4.5 Issuing-management subsystem
The platform now has a separate document-governance surface, visible to super_admin and agency_manager:
Issuing information: supplier corporate identity, registration details, representative, address, bank information, contact, quotation notes, seal, and company logo.
Ad-product catalog: maps (utm_source, utm_medium) combinations and line numbers to the product names printed on quotations/statements. Deletion is soft and historical performance references can block destructive renaming/deletion.
Document layouts: stores per-company quotation/statement column structures, labels, ordering, assignments, and read-only PDF previews before issue.
Official trade name: a campaign can carry the contractual line-item name intended for tax invoices, separate from the internal campaign label.
Issue ledger: generated documents retain versions, issuer identity, manual/automatic provenance, send state, recipients, and soft-deletion rules.
Layout storage, assignment, editing, and preview are implemented, while normal document generation still uses fixed renderer templates. Applying assigned layouts to issued documents remains a separate integration step. Tax-invoice PDFs can be generated, but national e-issuance through Popbill is not yet connected to a user/API workflow.
5. Authentication, RBAC, and user types
5.1 Authentication model
Authentication uses a signed JWT in an httpOnly cookie. A Bearer token is also accepted for service/testing clients. Important controls include:
default-deny authentication;
JWT signature and claim validation;
per-user token_version checked against the database on every authenticated request;
global session revocation on logout/password-sensitive changes;
sliding renewal after half the token lifetime;
secure cookies outside local development;
password reset and single-use invitation flows;
optional Google OAuth for existing accounts only.
The configured session TTL is five days, but active use renews it. token_version is the main mechanism for ending existing sessions.
5.2 Canonical roles
The backend Role enum defines six roles:
Role	Identity and scope	Main responsibilities
super_admin	Platform-wide, cross-tenant	Organization/system administration, all data, error logs, AI quality metrics, AIW and Git tools
org_admin	One organization	Full organization operations, users/invitations, settings, budgets, reporting, staff management
academy_admin	Organization plus assigned businesses	Operates assigned businesses, companies/courses/seasons/campaigns, reads peer users, invites company representatives
admin_manager	Exactly one assigned business plus assigned company duties	Restricted operations role for seasons/training rounds, schedules, weekly work, peer list, and email assistant; writes remain company-bounded
agency_manager	One or more assigned vendor/agency records	Vendor campaign/creative work plus issuing-information, ad-product, and document-layout management, with optional read-only company/course/season context
provider_readonly	One assigned company	Company representative with company-scoped read access and limited reporting/assistant functions
User account status is separate from role: invited, active, or disabled.
5.3 Scope dimensions
Authorization is more complex than a role matrix because data visibility can depend on:
organization_id;
assigned academy_ids;
singular company identity (company_id) for a provider;
per-business company-duty assignments (company_ids) for staff;
assigned agency_ids;
organization feature switches;
whether an operation is read or write;
resource ownership, creator identity, or manager identity.
scoped_select() is the central default filter:
super admin: no tenant filter;
organization admin: organization filter;
academy admin: organization plus assigned academies;
administrative manager: assigned academy and usually assigned companies;
agency manager: assigned agencies;
provider: assigned company;
unknown/missing required scope: no rows.
Cross-tenant queries require explicit unscoped() usage.
5.4 Administrative duty is a second axis
DUTY_ROLES includes organization admins, academy admins, and administrative managers. A company duty determines whose row appears in weekly work and calendar workflows.
For org_admin and academy_admin, duty does not reduce their broader read scope. For admin_manager, company assignments can also be an authorization boundary. An organization switch can widen that role's reads to its whole business while keeping writes company-bounded.
The staffing model treats role, data scope, and assigned duty as related but separate authorization inputs.
5.5 User-management rules
Notable rules include:
only super admins create/manage organization-admin authority;
organization admins can manage most users in their own organization but not peer organization admins;
academy admins can read a narrowed user list and invite only company representatives;
administrative managers receive exactly one business assignment;
agency managers receive one or more agency assignments;
provider accounts receive one company identity;
duty-capable users may manage their own company-duty assignments;
changing away from a scope-bearing role clears stale role-specific assignments;
users cannot modify or delete their own administrative account through the management API;
invitations expire, are single-use, and record delivery attempts.
5.6 RBAC strengths
One canonical Python role enum feeds the database enum.
Data access is scoped by default and fails closed.
Scope-bearing junctions are re-read per request, so academy/agency/company-assignment revocations apply immediately.
Many out-of-scope object lookups return 404 to avoid confirming another tenant's object.
A route-authentication sweep test checks that every API route is either protected or explicitly public.
Cron and AIW bearer endpoints refuse operation when their secret is absent.
The API, not the frontend, owns the real authorization decision.
5.7 RBAC risks and maintenance concerns
Some JWT identity claims can become stale. Academy, agency, and duty scopes are refreshed from the database, but role, organization, provider company, and email remain token claims until re-login/token replacement; sliding renewal re-signs the existing claims. Managed status/role/email updates bump token_version, but authenticated requests do not independently recheck account status. Direct SQL or any authority-changing path that omits the version bump can therefore leave old authority alive temporarily.
Role strings are duplicated. The frontend, invitations, user-management validation, and page-level checks maintain separate string lists. A new role or renamed role requires coordinated edits.
Frontend URL guards are thin. There is no general Next.js route middleware for role access. An authenticated user can open many hidden URLs and then encounter API refusal. This is not a backend data leak if APIs remain correct, but it creates confusing UX and makes the API the only defensive layer.
Authorization is distributed. Many routes only require authentication and defer role/write decisions to service code. The route sweep proves authentication, not that every new service method applies correct scope.
No physical foreign keys. Relationship integrity and tenant consistency are largely enforced in application services and migrations. This increases the consequence of direct database edits and incomplete validation.
6. Weekly report subsystem
Weekly reports are fully persisted operational records, not generated summary emails.
6.1 Data model
One row represents one user's work for one week and sheet:
organization_id;
academy_id;
user_id;
Monday-normalized week_start;
sheet;
content (what happened this week);
plan (the plan written in advance);
issues.
The unique key is (user_id, week_start, sheet).
6.2 Behavior
Users read the relevant shared grid but write only their own row.
This week's actual work and next week's plan are stored as separate facts.
A new week initially displays its previous plan until actual content is entered.
Work can be scoped by one business or by a configured work group spanning businesses.
The normal staff sheet is shared by duty roles.
A separate org_admin sheet is visible to organization admins and super admins; only an organization admin writes its own row there.
The UI supports autosave and week navigation.
Version history is intentionally not implemented.
There is no AI weekly-report summarization.
6.3 Weekly reminder mail
The weekly reminder job is a notification workflow, not the report itself:
first working day: announce to the entire roster;
second working day: follow up only with users whose current-week content is still empty;
public holidays shift both reminder days;
each person/week/reminder-kind has an idempotency key;
recipients are linked back to /weekly-reports;
organization notification switches can suppress the send while retaining an audit row.
7. Scheduling and calendar subsystem
The unified calendar combines derived business dates with editable staff records.
7.1 Event types
Type	Persistence	Behavior
Recruitment	Derived from a season	Read-only in calendar
Training	Derived from a season	Read-only in calendar
Project evaluation	Stored in schedules	Create/update/delete; may initially have only planned year/month
Business trip	Stored in schedules	Create/update/delete; can cover several destination companies
Derived events avoid duplicate dates that could drift away from their season source.
Stored schedules contain manager, company/course/cohort context, status, dates, evaluation fields, trip details, companion, and memo. schedule_companies supports multi-company trips and per-destination course/cohort context.
Reads are business/work-group scoped. A business can see its full calendar, while “my schedule” narrows to the caller's responsibility. Trips generally belong to the manager who owns them; organization/super admins can amend more broadly.
7.2 Schedule reminder mail
The daily reminder job:
runs by external cron at 09:00 KST;
Monday–Thursday covers the next day;
Friday covers Saturday, Sunday, and Monday;
weekends are a successful no-op;
groups all upcoming entries into one digest per manager;
addresses the schedule's manager_id, not merely the record creator;
skips deleted/disabled/unreachable accounts;
deduplicates by manager and run date.
8. AI email assistant
The email assistant is an implemented but deliberately bounded AI feature.
8.1 What it does
creates a subject and body from an operator's request;
supports reply mode using pasted received-email text;
applies a per-user standing writing-style instruction;
supports reusable email “types”/starters;
lets AI refine a starter;
stores selected drafts in a personal drawer;
records model, prompt ID/version, generation latency, and generated-vs-saved text;
gives super admin aggregate quality metrics without exposing draft text across users.
8.2 Privacy and scoping
Drafts and writing styles are per user.
Staff share an organization-level starter library.
Agency/provider accounts use personal starter pools.
Another user's draft is returned as not found rather than exposing ownership.
The LLM call records the actor ID for usage/cost logging.
8.3 LLM implementation
The provider-neutral LLM layer supports:
Anthropic as the production provider;
Gemini as a development-only provider;
none as a cleanly disabled generator state;
versioned prompts;
JSON envelope extraction and malformed-response recovery;
bounded retries for transient upstream statuses;
separate timeout, quota, unavailable, and bad-response error codes;
per-call token and duration logging.
Prompts explicitly:
forbid inventing amounts, dates, names, or decisions;
use visible placeholders for missing facts;
treat pasted inbound mail as untrusted quoted material;
prevent instructions in pasted mail from overriding the reply task;
require plain-text Korean business-mail output;
preserve a controlled JSON response shape.
8.4 Important boundaries
The email assistant:
does not send mail;
does not ingest an inbox through IMAP/webhooks;
does not automatically read tenant data at generation time;
does not replace the notification/document mailer.
“Reply mode” means the user manually pastes a received email. A platform-data panel can insert visible, scoped facts into the editable prompt text, but the model remains data-blind beyond what the user sends.
9. Notifications and mail integration
The platform has two separate mail concepts:
the AI email assistant, which drafts but never sends;
the operational mail system, which sends notifications, credentials, and documents.
9.1 Durable notification outbox
Every operational notification is first recorded in notifications with:
event;
recipient email/type/ID;
organization;
rendered subject;
fact payload;
status (pending, sent, failed, skipped);
transport;
error;
idempotency key;
creator and sent time.
This provides:
an audit answer to “was it sent?”;
retryable facts after transient failure;
duplicate suppression;
explicit logging of deliberately suppressed sends;
separation between user actions and mail-server reliability.
Explicit “send now” actions report delivery results. Event-triggered mail is best-effort: a mail failure does not roll back a successful campaign or document action.
9.2 Notification preferences
The outbox applies preference decisions centrally:
organization-wide disabled-event settings;
per-user preferences for addressed events;
administrator-managed per-user exclusions for eligible organization events;
opt-in behavior for selected warnings.
Suppressed sends still receive a skipped audit row explaining whether the organization, recipient, or default policy stopped delivery.
9.3 Notification event catalog
Implemented templates cover:
advertising resources shared with an agency;
creative review requested;
creative approved;
creative changes requested;
campaign preflight warning to agency;
campaign preflight warning to operator;
campaign closing documents complete;
closing-document request to agency;
closing-document request copy to operator;
weekly-report reminder;
schedule reminder;
password reset requested/completed;
feedback submitted/answered.
document_issued is a sixteenth governed event without an outbox renderer: its PDF and delivery use the document-send service, while the shared preference system supplies the opt-in/exclusion decision.
The notification-log UI's filter list may lag newer backend events, so not every stored event is equally discoverable by dropdown.
9.4 SMTP lane
SMTP supports credential and rendered-message flows, including:
invitations;
password reset;
events that remain on rendered templates;
local Mailpit testing.
When SMTP is absent, local/event flows can use a logging transport. The row is marked with transport=log so it is not mistaken for a real delivery.
The daily internal error digest no longer uses this lane; it moved to Slack on 2026-10-01.
9.5 JCG external AI mailer lane
The JCG mailer integration posts structured facts to rnd.jcg.zone, where an external AI-managed template composes and sends the message. It supports:
bearer API authentication;
required caller-defined idempotency;
recipient/CC lists;
subject hints;
Korean language;
up to three supported attachments;
local attachment-size/type validation;
test mode;
asynchronous acknowledgement;
stable refusal/error codes.
Each call creates an email_send_logs row containing the request identity, recipients, status, test mode, issue facts, related campaign/document, actor, preview, and error details. This is separate from notifications, because it reconciles this system's request with the external mailer's own send ledger.
The safe code/configuration default remains JCG_MAILER_TEST_MODE=true, but the documented production environment completed its tenant cutover on 2026-10-07 and explicitly runs with JCG_MAILER_TEST_MODE=false. Operational notifications therefore go to real KCCI recipients. Credential/token mail remains on SMTP because externally AI-composed wording is inappropriate for exact authentication links.
9.6 Document mail
Generated PDF documents use the JCG mailer attachment path. The client enforces:
maximum 3 attachments;
maximum 5 MB per attachment;
maximum 6 MB total;
a controlled extension allowlist.
Document sends and notification sends therefore share the JCG API but have different business records and call paths. Automatic sends honor each account's document_issued opt-in and exclude the issuer contact when it has no user preference; a manual Send action is an explicit operator decision and targets the full resolved recipient set.
9.7 Mail-management console
The /mail-management console turns notification governance into an explicit product surface:
super_admin can select an organization; org_admin manages its own organization;
every backend mail event is listed with mandatory/switchable, opt-in/duty, default, and recipient-role metadata;
organization-wide enable/disable controls reuse the notification settings model;
administrators can exclude specific eligible users from selected administrative mail events;
send history is available from the notification ledger.
The console does not yet provide arbitrary message-body editing or a general test-send composer. That limitation matters: it governs delivery policy, not the external AI template itself.
10. Background and scheduled work
There is no in-process scheduler and no durable general-purpose job queue. Host cron or another trusted scheduler invokes seven synchronous HTTP endpoints with Authorization: Bearer $CRON_SECRET.
Endpoint	Purpose	Typical timing
/cron/campaign-status-transitions	Start/end campaigns, send preflight warnings, retry pending notifications, trigger reports	More than once daily
/cron/schedule-reminders	Manager schedule digest	09:00 KST daily
/cron/closing-document-requests	Ask agencies for outstanding post-campaign documents	09:00 KST daily
/cron/weekly-report-reminders	First/second working-day report reminders	09:00 KST daily
/cron/error-digest	Summarize recent application errors	09:00 KST
/cron/feedback-diagnostics-purge	Clear retained diagnostic bundles after the configured retention period	Host cron
/cron/feedback-bridge-sync	Retry and reconcile feedback handoffs with the development bridge	Host cron
All endpoints fail closed if CRON_SECRET is unset and use POST only.
The database backup is a separate host-cron command that launches a fresh API image to run mysqldump, gzip, upload, validate the completion marker, and prune old backups. It intentionally does not depend on the health of the running API container.
Complexity/operational consequences:
synchronous external mail calls can make cron requests long-running;
there is no queue-level retry schedule, concurrency control, or dead-letter queue;
the pending-notification retry is coupled to the campaign lifecycle sweep;
missing host cron can fail silently as stale campaigns or absent reminders;
repository documentation fully details older jobs but lags some newer endpoints.
11. Database scope
11.1 Exact count
Current code/migration target: 51 ORM/application tables. After Alembic migration 0065, MySQL contains 52 physical tables including alembic_version.
Inspected local Docker database: revision 0064, with 50 physical tables total: 49 application tables plus alembic_version. It is one migration behind code head and does not yet contain feedback_diagnostics or feedback_handoffs.
Documented production database: revision 0065, with 51 application tables and 52 physical tables including alembic_version.
Table-count references should state the Alembic revision and whether alembic_version is included.
An older deployment restore note references 42 tables and 9,516 rows. That is a historical V1/migration snapshot, not the current V2 schema count.
11.2 Table inventory
Tenant and business structure (13 at code head)
organizations
academies
companies
company_academies
courses
seasons
training_rounds
agencies
agency_academies
work_groups
work_group_academies
business_entities
ad_products
Users, access, and credentials (8)
users
user_academy_assignments
user_company_assignments
user_agency_assignments
user_notification_preferences
invitation_tokens
password_reset_tokens
system_settings
Campaigns, creatives, files, performance, and UTM (15)
campaigns
campaign_statuses
campaign_status_history
campaign_creatives
campaign_creative_assignments
campaign_creative_status_history
campaign_performance
attachments
academy_budgets
utm_codes
system_utm_sources
system_utm_mediums
organization_utm_sources
organization_utm_mediums
common_documents
Work management (4)
schedules
schedule_companies
weekly_reports
public_holidays
AI, communication, documents, and operations (11 at code head)
email_drafts
email_starters
email_styles
notifications
email_send_logs
generated_documents
feedback
feedback_diagnostics
feedback_handoffs
error_logs
aiw_work
11.3 Data-integrity model
The schema generally does not use physical foreign-key constraints, following compatibility choices inherited from V1. Relationships and cross-tenant validity are enforced in repositories/services.
Advantages:
easier legacy migration and consolidation;
fewer database-level obstacles during V1-to-V2 reshaping.
Costs:
direct SQL can create orphaned/cross-tenant rows;
correctness depends on every write path using the proper service;
destructive migrations and repairs need deeper application knowledge;
database tooling cannot infer the full relationship graph.
One minor ORM-registration issue exists: PublicHoliday has a model and migration but is not imported in models/__init__.py, so future Alembic autogeneration may not see it unless another import registers it.
12. External integrations
Integration	Platform use	Failure/degraded behavior
MySQL 8	Primary datastore	API health/revision failure
NCP Object Storage	Attachments, creatives, generated docs, screenshots, DB backups	File features refuse clearly when unconfigured
Google OAuth	Existing-user sign-in	Disabled when credentials absent
Google Sheets service account	Read-only season bulk import	Import path refused when absent
Anthropic	Production email-draft generation	Generator unavailable; saved drafts still work
Gemini	Development email-draft provider	Application refuses this provider in production
Gmail/configurable SMTP	Credential and rendered operational mail	Logging/dev fallback or explicit skipped delivery
JCG external mailer	Production AI-composed operational notifications and attached document delivery	Errors logged; code default is test-safe, deployed production override is live
GitHub API	Super-admin read-only Git/PR/Actions view	Feature returns “not configured” or upstream error
Slack incoming webhook	Real-time API/web errors, crashes/readiness transitions, and daily error digest	Digest skips and monitor idles when unset
TAP Wiki Manage	Read-only localized document tree/content feed	IP allowlist is fail-closed; optional bearer token; local unproxied exception
JCG Popbill relay	Safety-hardened client scaffold for tax-invoice e-issuance	Sandbox default, production-host guard, no auto-retry; not yet connected to routes/live issuance
AI work bridge	Receives development handoffs from Feedback and reports progress back	Bearer/HMAC protected; closed when configuration is absent
WeasyPrint/Jinja2/openpyxl/pypdf	Server-side document/report generation	Individual generation fails without undoing origin action
Explicitly excluded:
ClickHouse;
automatic ad-platform synchronization;
native mobile app;
desktop/Electron client;
user-facing Slack bot;
Telegram monitoring, which was retired rather than merely disabled;
real inbound-email ingestion.
13. Platform/admin and developer features
13.1 Feedback and error handling
A Quick Feedback control is available throughout the authenticated app. All roles can submit and view their own feedback; super admin performs triage and responses.
Feedback V2 records page context, recent API calls, client errors, breadcrumbs, failed-request details, application version, and an optional screenshot. Sensitive fields are masked before storage. Diagnostics are stored separately from the user's report and are visible only to super admin.
When an LLM is configured, triage runs after submission and records a summary, severity, reproduction steps, suspected areas, and possible duplicates. A model failure does not affect the submitted report.
Super admin can hand a feedback ticket to the AI work bridge. The bridge receives a signed request, reports linked/in-progress/review/done/dropped states back, and is reconciled by scheduled retry. Diagnostic evidence is purged after the configured retention period, 90 days by default; the report, response, and triage remain.
Central exception handlers record application errors in error_logs. Super admin can inspect them. A scheduled digest posts grouped 24-hour results to a private Slack channel even on a quiet day, while the production monitor sidecar watches API and web logs, hard crashes, and readiness transitions from outside the application process.
13.2 AIW ledger
AIW is an append-only AI-work ledger:
bearer-token ingest for tooling;
idempotent work-unit creation;
duration, token, cost, model, phase, agent, commit, and document metadata;
correction chains through ref_id;
super-admin-only read UI;
no browser update/delete path.
It does not implement the legacy AIW report-email workflow described in older TAP documents.
13.3 Git Manage
Git Manage is a super-admin-only read surface over the GitHub API:
repository/branch overview;
commit history;
pull requests;
GitHub Actions runs.
The browser cannot trigger deployment. That is explicitly deferred to a later, separately reviewed phase.
13.4 Help and sitemap
Help is read-only localized Markdown served by the API. There is no database-backed help editor. Ten guide topics exist in both Korean and English, which is smaller than the roughly 35 sidebar destinations. The sitemap uses the same frontend navigation definition and displays inaccessible areas as locked.
14. Test and quality posture
Backend test suite:
162 backend test modules;
coverage of RBAC scopes, campaign lifecycle, creative workflows, schedules, weekly reports, training operations, feedback diagnostics/handoffs, notifications, documents, backup, OAuth, and security headers.
Frontend assurance consists of:
ESLint and TypeScript checks;
production Next.js build;
25 Playwright specifications for selected high-value journeys;
no separate frontend unit-test harness.
Current limitations:
manual-only CI makes enforcement dependent on local hooks and discipline;
E2E coverage is much smaller than the page count;
many business rules are encoded in code comments and tests rather than one formal domain specification.
15. Implemented, partial, and absent features
Implemented
multi-tenant organization/business/company/agency model;
six-role RBAC and invitation lifecycle;
courses, seasons, campaigns, creatives, approvals, files, and performance;
training-round operations with dashboard aggregation, inline editing, duplication, deletion, effective-period overrides, and role-specific list behavior;
UTM generation and public tracked redirects;
schedule calendar and reminders;
weekly reports and reminders;
AI email compose/reply/refine and personal draft storage;
notification outbox, preferences, idempotency, and transport audit;
SMTP, production JCG external mailer, and centralized mail-management controls;
multi-format Excel/ZIP exports, season-level PDF merging, per-campaign document bundles, and automated document triggers;
issuing information, ad-product catalog, document-layout storage/assignment/preview, and generated-document ledger;
Feedback V2 diagnostics, AI triage, development handoff, error logs, Slack digest/monitor, pool readiness, and backup script;
read-only TAP Wiki feed;
super-admin AIW and read-only Git tools;
Korean/English UI and help.
Partial or pilot-stage
document layouts: catalog, assignment, editor, and preview are implemented, but assigned layouts are not yet applied by normal issuance;
document chain: manual/auto governance exists, but some report/template/data paths remain gated;
Popbill tax e-issuance: guarded relay client exists; routes, UI, production scope, and live calls do not;
platform-data use in email assistant: facts are manually inserted into prompt text, not fetched invisibly by the LLM;
Git Manage: read-only, no deploy trigger;
help coverage: fewer guides than major screens;
GitHub Actions deployment: retained but dormant; production releases are manual.
IDC/Coolify migration and staging: planned by the infrastructure team but not yet the active production path.
Not implemented
inbox/IMAP/webhook email ingestion;
sending from the email-assistant screen;
AI generation or summarization of weekly reports;
ad-platform auto-sync;
ClickHouse analytics;
Redis/Celery queue and workers;
native mobile or desktop clients;
UI-triggered Git deployment;
object-storage replication/backup managed by this repository;
per-device session revocation.
16. Main complexity drivers
16.1 RBAC is multidimensional
Six roles alone are manageable. The complexity comes from combining role with organization, business, company, agency, duty assignments, ownership, read/write asymmetry, feature switches, and special sheets. Every new query needs an explicit answer for each relevant scope.
16.2 Business entities are not a simple tree
Organization-level companies and agencies participate in multiple businesses. Schedules can be filed under one business while referring to another business's company in the same organization. Work groups span businesses. These are valid domain requirements but create sophisticated joins and validation.
16.3 Campaign lifecycle has many side effects
A campaign can affect creative status, approval queues, performance, UTM, notifications, generated documents, document versions, closing-document requests, and timed transitions. Failure isolation and idempotency are therefore essential.
16.4 Communication is a subsystem
The product has personal AI drafts, shared/personal templates, notification preferences, an append-only outbox, SMTP, an external AI mailer, external-send audit, attachments, retries, idempotency, and multiple scheduled reminder policies. “Email” is not one integration; it is several related systems with different trust and delivery contracts.
16.5 Scheduling contains business calendars
Calendar dates, KST wall-clock behavior, weekend handling, public holidays, Monday/Tuesday report rules, Friday-to-Monday schedule reminders, and missed-run behavior are domain logic that needs tests and operational scheduling.
16.6 Legacy migration and cutover remain relevant
The platform preserves V1 object keys, supports a mirror-bucket transition, consolidates old company/agency grain, and has historical migration/restore procedures. Until V1 is fully retired, changes must consider both V2 correctness and cutover compatibility.
16.7 Operational controls and dependencies
The application includes tests, backup validation, structured logging, fail-closed configuration, pool-aware readiness, and out-of-process Slack monitoring. CI, deployment, and cron are manually or host managed; rate limits are process-local; object storage has no repository-managed replication; and mail calls are synchronous.
16.8 Configuration and document governance are now first-class domains
Organization switches now alter notification events, user exclusions, creative approval, closing-document sets, quotation auto-issue, document-chain mode, document-ledger registration, context-page visibility, shared service descriptions, and mail controls. Issuing identity, ad-product mappings, layout assignments, manual/automatic provenance, screenshots, and opt-in delivery further affect what a document means and who receives it. This is useful configurability, but it increases the number of valid states and regression combinations well beyond the raw table increase.
16.9 Working tree and deployed schema can diverge
Code head and the documented production database are at migration 0065, while the inspected local database is at 0064. Startup revision checks warn about this state, but migration sequencing and environment verification remain part of every release. Table-count claims must name the Alembic revision and whether alembic_version is included.
17. Risks and recommended evaluation points
These are not all bugs; they are areas that increase ownership cost or deserve confirmation before production evaluation.
Keep live deployment evidence current. README and DEPLOYMENT now agree that KCCI production is live; periodically verify the running image SHA, Alembic revision, monitor, cron entries, and mailer mode against those documents.
Ensure role/org/provider-company changes revoke sessions. Verify all authority-changing user updates bump token_version.
Generate a shared role contract. Reduce duplicated backend/frontend role strings and ad hoc page checks.
Add frontend route-level access handling. Keep API authority, but avoid rendering inaccessible screens before an API refusal.
Strengthen authorization regression checks. Authentication sweep tests cannot prove that every service query is correctly scoped.
Document the scope model. Especially distinguish provider company identity from staff company-duty assignments.
Add database integrity monitoring. With no FKs, periodic orphan/cross-tenant consistency checks would detect bad direct writes.
Keep ORM registration and schema-count checks automated. Prevent Alembic metadata drift, including models that may be omitted from the central registry.
Separate background work when volume grows. A durable queue/worker would isolate slow external mail, retries, concurrency, and dead letters.
Monitor cron freshness. Alert on last successful execution, not only endpoint errors.
Finish notification-log and mail-console parity. Ensure every new event remains filterable, understandable, and governed consistently.
Close backup gaps. Replicate object storage and deliver backup failures to a monitored human channel.
Restore automated CI enforcement so the existing suite runs consistently.
Protect the completed JCG mailer cutover. Monitor real-recipient runs, keep credential mail on SMTP, and make production/test mode visible in operational checks.
Track incomplete document integrations explicitly. Separate layout preview from live layout application, PDF tax documents from Popbill e-issuance, and schema/API work from a migrated deployment.
Monitor Feedback V2 retention and bridge delivery. Verify purge freshness, failed handoffs, token/HMAC configuration, and access to diagnostic bundles.
Add cron/backup success telemetry so a missing host job is reported directly.
18. Complexity conclusion
CosMos AI V2 is a business-operations platform covering program structure, advertising execution, staff work, documents, communication, reporting, and operational support.
At current code head it has 51 application tables, 239 API handlers, 54 frontend pages, 66 migrations, 162 backend test modules, and 25 browser E2E specifications. Seven scheduled endpoints support lifecycle transitions, reminders, error reporting, feedback retention, and development-bridge synchronization. The main sources of complexity are:
role and scope determine which tenant/business/company/agency data can be read or changed;
campaigns drive creative, approval, performance, notification, and document workflows;
weekly work and scheduling encode staffing, holiday, and time-zone policies;
communications span AI drafting, SMTP, external AI composition, attachments, preferences, audit, and retries;
production operation depends on exact migration sequencing, object storage, seven cron endpoints, backups, external APIs, readiness monitoring, and Slack alerts.
The runtime topology is compact, but the domain and configuration surface is broad. Role scopes, organization switches, campaign state, training rounds, document provenance, mail preferences, diagnostics, and external delivery state create many valid combinations. Maintenance therefore requires coordination across product operations, FastAPI, Next.js, MySQL, deployment, security, JCG mail, document generation/layouts, the feedback bridge, and future Popbill issuance.
19. Primary evidence paths
README.md
DEPLOYMENT.md
docker-compose.yml
docker-compose.prod.yml
api/pyproject.toml
api/app/main.py
api/app/api/router.py
api/app/api/cron.py
api/app/auth/deps.py
api/app/auth/scope.py
api/app/core/config.py
api/app/core/email.py
api/app/core/jcg_mailer.py
api/app/core/popbill_client.py
api/app/core/slack.py
api/app/models/
api/app/migrations/versions/
api/app/services/email_draft_service.py
api/app/services/llm/
api/app/services/notifications/
api/app/services/weekly_report_service.py
api/app/services/weekly_report_reminders.py
api/app/services/schedule_service.py
api/app/services/schedule_reminder_notifications.py
api/app/services/documents/
api/app/services/organization_settings_service.py
api/app/services/training_round_service.py
api/app/services/feedback_diagnostics_service.py
api/app/services/feedback_handoff_service.py
api/app/api/wiki_api.py
api/app/api/feedback.py
api/app/services/user_management_service.py
api/app/services/invitation_service.py
api/app/services/aiw_service.py
api/app/services/git_service.py
api/tests/
web/app/
web/components/app-shell.tsx
web/next.config.mjs
web/e2e/
Untracked root/TAP Markdown files were treated as planning or legacy reference material unless their claims matched current V2 code.