# Smart Food Compliance & Quality Engine — backend foundation

Phase 2 ("Foundation") of the MVP roadmap plus phase 3 (HACCP records): multi-tenant lot
traceability, CCP and sanitation records with electronic signatures, signed amendments,
corrective actions, alerts by e-mail and WhatsApp, Arabic/English PDF reports, an Arabic web
app (PWA) with worker languages, account lockout, record numbering and document control,
supplier approval, training and competence, immutable records and a tamper-evident audit
trail. User-facing text is Modern Standard Arabic; worker screens also come in Hindi, Urdu and
Bengali.

## What is in this version

| Area | What it does |
| --- | --- |
| Tenants & users | One database for all factories, isolated by PostgreSQL row-level security. JWT login, five roles. Admins rename users, change roles and deactivate accounts (not their own role or account); old signatures keep the name and role at signing. |
| Lockout | `CE_MAX_FAILED_PASSWORDS` (default 5) wrong passwords in a row, at login, when signing or when changing the password, lock the account: no login, no API, no signing until an admin unlocks it or resets the password. Every wrong password and lock is audited. |
| Master data | Items (raw, packaging, WIP, finished), suppliers, customers. |
| Supplier approval | Each supplier has a status (pending, approved, conditional, suspended, rejected) set only by a signed decision (`SAP-…`), a risk level that sets the re-evaluation interval (24/12/6 months), documents and certificates with expiry dates and an optional scanned file (withdrawn, never edited or deleted), and signed performance evaluations (`SEV-…`, quality 40 %, delivery 25 %, documents 20 %, responsiveness 15 %) with a recommendation. Receiving is blocked unless the supplier is approved or conditional and no required document has expired. |
| Training & competence | Courses (validity in months, required roles, control points they qualify for), signed training records per attendee (`TRN-…`, passed / failed / attended), a competence matrix (valid, expiring, expired, failed, missing) and per-user competence. With `enforce_training` on, a CCP reading is refused from someone without valid training for that control point. |
| Settings in the app | SMTP and WhatsApp credentials per tenant, entered on the notifications page and preferred over `.env`; the password and token are stored encrypted with a key derived from `CE_JWT_SECRET` and never returned. Quality settings: training enforcement and the expiry warning window (default 30 days). |
| Lots | Receive → quarantine → release/block → consume in production → ship. Balances come from a signed ledger. |
| Rules | No receiving from unapproved suppliers or of expired material; only released, unexpired lots can be used; no over-consumption; only finished goods ship. |
| Traceability | Backward (finished lot → ingredients → supplier lots) and forward (raw lot → products → customers) from one recursive query. |
| FEFO | Lists usable lots of an item, earliest expiry first. |
| Immutability | Database triggers reject UPDATE/DELETE on ledger tables; on lots only `status` may change. |
| Audit trail | Every write is logged in a SHA-256 hash chain per tenant. `GET /audit/verify` detects any edit. |
| CCP monitoring | Control points from the HACCP plan with critical limits. Each reading copies the limits in force, is flagged in or out of limits, and cannot be saved out of limits without a corrective action. |
| Sanitation | Cleaning, sanitizing, deep cleaning, cleaning after use and pre-op inspections per area, with chemical and ppm. A failed check needs a corrective action. |
| E-signatures | The author re-enters their password to sign an entry ("performed"); a supervisor or QM signs it "verified". The author cannot verify their own entry. Each signature stores name, role, time, meaning and a SHA-256 fingerprint of the record, so any later change shows `intact: false`. |

| Amendments | Mistakes are corrected with a signed amendment (fields, old and new value, reason), never by editing. Records are shown with amendments applied; each amendment needs its own verification by a second person. |
| Corrective actions | Every deviation (CCP out of limits, failed sanitation) opens a corrective action with the immediate action. A QM/admin closes it once with root cause, preventive action and product disposition, signed "approved". |
| Alerts | Raised for CCP deviations, failed sanitation, missed checks (no reading within a control point's `check_interval_minutes`; scanned every `CE_ALERT_SCAN_SECONDS`), supplier documents expiring or expired, supplier re-evaluation due, and training expiring or expired. A supervisor acknowledges each alert once. |
| Notifications | Each alert is queued for the active recipients (e-mail or WhatsApp, "all" or "critical only") and sent by a background task every `CE_NOTIFY_SECONDS`, outside the request. Failed sends are retried after 1, 5, 15 and 60 minutes, then marked failed. Test messages and a delivery log in the app. |
| Web app | Arabic, right-to-left, phone-friendly app at `/app/` for every role, with a language switch: worker screens in Hindi and Bengali (left to right) and Urdu (`web/i18n.js`; factory data and management pages stay in Arabic): record CCP and sanitation checks, review queue, acknowledge alerts, close corrective actions, amendments, reports, HACCP plan, users, notifications, audit check. Plain HTML/JS served by the API (no build step); installable as a PWA over HTTPS or on localhost. |
| Record numbers | Every CCP reading, sanitation check, corrective action and amendment gets a number per tenant, series and year (`CCP-2026-000123`, `SAN-…`, `CA-…`, `AMD-…`), taken from `record_counters` inside the inserting transaction, so numbers never repeat or leave gaps. Numbers are not part of the signed fingerprint. |
| Document control | Each report is a controlled form (code, revision, effective date, approver, reference, retention). Defaults follow common HACCP practice (`F-HACCP-01`, …); no regulator prescribes form codes. Changing a form's content needs a new revision and a reason, and is audited. Every page header prints form code, revision, effective date and report number. |
| Issued reports | Every PDF gets a number (`RPT-2026-000001`) and is registered with its SHA-256, so a printed or e-mailed copy can be checked later (`POST /issued-reports/check`). |
| Dashboard | Home page per user: live status of every control point (on time, due soon, overdue, deviation, from `check_interval_minutes`), open alerts, corrective actions, review queue, 7-day compliance score and the latest activity. A TV board (`#/tv`) shows the same live status in large type and refreshes every 30 s. |
| Analytics | `#/analytics`: period presets or custom range, compared with the period before. Compliance score (average of in-limit readings, passed sanitation, verified records, on-time checks, corrective actions closed within `CE_CAPA_TARGET_DAYS`), daily trends, a control chart per CCP (critical limits, mean ±3σ, Cpk from 10 readings), deviations per CCP, problems by weekday and hour, sanitation by area, corrective-action aging, activity per user (supervisor and above). Every number and chart opens the matching records; every chart has a table view; CSV export. Figures use amended values, like the reports. |
| Reports | PDF in Arabic (right to left) or English: CCP monitoring log, sanitation log, corrective action register, traceability / mock recall, approved supplier list, training matrix, training log. Plus a JSON HACCP summary. Fonts are bundled (Noto, SIL OFL). |

Not yet included: rules engine, AI assistant, Odoo connector, offline data entry. These are
the next phases.

## Project layout

```
src/compliance_engine/
  core/        settings, security (JWT, bcrypt), roles & permissions, domain errors
  db/          engine, sessions, tenant binding for row-level security
  models/      SQLAlchemy tables and enums
  schemas/     Pydantic request (Command) and response (View) models
  services/    business logic: identity, master data, inventory, traceability, audit,
               HACCP records and amendments, e-signatures, corrective actions, alerts, reports,
               record numbering, document control, wording of system-written texts
  reports/     PDF layout (fpdf2 + HarfBuzz shaping), report wording in English and Arabic, fonts
  notifications/  e-mail (SMTP) and WhatsApp (Meta Cloud API) senders
  web/         the Arabic web app (index.html, app.js, styles.css, service worker, icons)
  api/         FastAPI routers and dependencies (auth, permissions, one transaction per request)
  main.py      app factory and error mapping
  cli.py       create-tenant and reset-password commands
migrations/    Alembic: 0001 tables, 0002 row-level security + append-only triggers,
               0003 CCP, sanitation and signature tables,
               0004 amendments, corrective actions, alerts, missed-check intervals,
               0005 account lockout, notification recipients and deliveries,
               0006 record numbers, controlled forms, issued-report register
tests/         end-to-end tests against real PostgreSQL
```

Dependencies point one way: `api → services → models/db → core`. Services never import
FastAPI, so the same logic can serve the API, the CLI, background jobs and an ERP sync.

## Run locally

```bash
cp .env.example .env            # set CE_JWT_SECRET
docker compose up -d db
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
alembic upgrade head
python -m compliance_engine.cli create-tenant --slug demo --name "Demo Foods" \
    --admin-email qa@demo-foods.com --admin-name "QA Admin"
uvicorn compliance_engine.main:app --reload
```

Open http://localhost:8000/app/ for the web app, or http://localhost:8000/docs for the
interactive API. Log in with
`POST /api/v1/auth/token` (`tenant`, `email`, `password`) and send the token as
`Authorization: Bearer <token>`.

Or run everything in containers (works the same in PowerShell on Windows):

```bash
docker compose up --build -d
docker compose exec api python -m compliance_engine.cli create-tenant --slug demo --name "Demo Foods" --admin-email qa@demo-foods.com --admin-name "QA Admin"
```

A `.env` file is optional for the containers; without one the API uses a local-dev JWT
secret. If port 5432 is already taken by a local PostgreSQL, set `CE_DB_PORT=5433` in `.env`.
If the database was first created by an older or failed run, reset it with
`docker compose down -v` (this deletes the local data).

## Checks

```bash
pytest          # needs the compliance_test database from docker/init-db.sql
ruff check . && ruff format --check .
mypy src
```

The app must connect as a **non-superuser** role. PostgreSQL superusers bypass row-level
security, so tenant isolation would silently stop working.

## Main endpoints (`/api/v1`)

| Method | Path | Permission |
| --- | --- | --- |
| POST | `/auth/token` | public |
| POST, GET | `/users` | admin |
| PATCH | `/users/{id}` (`full_name`, `role`, `is_active`) | admin |
| POST | `/users/{id}/reset-password`, `/users/{id}/unlock` | admin |
| GET | `/auth/me` · POST `/auth/change-password` | any signed-in user |
| POST, GET | `/items`, `/suppliers`, `/customers` | write: QM/admin · read: all |
| POST | `/lots/receipts` | operator and up |
| GET | `/lots`, `/lots/{id}`, `/items/{id}/fefo` | all |
| POST | `/lots/{id}/status` | QM/admin |
| POST | `/production-runs`, `/shipments` | operator and up |
| GET | `/trace/lots/{id}?direction=backward\|forward` | all |
| GET | `/audit/events`, `/audit/verify` | QM, auditor, admin |
| POST | `/control-points` | QM/admin |
| GET | `/control-points`, `/ccp-readings`, `/sanitation-records` (+ `/{id}`) | all |
| POST | `/ccp-readings`, `/sanitation-records` (signed with `password`) | operator and up (not auditor) |
| POST | `/ccp-readings/{id}/verify`, `/sanitation-records/{id}/verify` | supervisor, QM, admin |
| PATCH | `/control-points/{id}` (limits, `check_interval_minutes`, with a `reason`) | QM/admin |
| POST | `/ccp-readings/{id}/amendments`, `/sanitation-records/{id}/amendments` (signed) | operator and up |
| GET | `/amendments` · POST `/amendments/{id}/verify` | all · supervisor, QM, admin |
| GET | `/corrective-actions` (+ `/{id}`) | all |
| POST | `/corrective-actions/{id}/close` (signed "approved") | QM/admin |
| GET | `/alerts`, `/alerts/summary` | all |
| POST | `/alerts/{id}/acknowledge`, `/alerts/scan-missed-checks` | supervisor, QM, admin |
| GET | `/reports/haccp-summary` (JSON), `/reports/ccp-log`, `/reports/sanitation-log`, `/reports/corrective-actions` (PDF, `lang=ar\|en`, `date_from`, `date_to`) | all |
| GET | `/reports/trace/{lot_id}` (PDF mock recall) | all |
| GET, POST | `/notification-recipients` · PATCH `/notification-recipients/{id}` · POST `/{id}/test` | QM/admin |
| GET | `/notification-deliveries`, `/notifications/channels` · POST `/notifications/dispatch`, `/notifications/retry-failed` | QM/admin |
| GET | `/document-forms` · PATCH `/document-forms/{key}` (new `revision` + `reason` for content changes) | all · QM/admin |
| GET | `/issued-reports` (+ `/{serial}`) · POST `/issued-reports/check` (PDF as the request body) | all |
| GET | `/suppliers/overview`, `/suppliers/{id}`, `/supplier-documents/{id}/file` | all |
| PATCH, POST | `/suppliers/{id}` · `/suppliers/{id}/decisions`, `/suppliers/{id}/evaluations` (signed) · `/suppliers/{id}/documents` · `/supplier-documents/{id}/withdraw` | QM/admin |
| GET | `/training/courses`, `/training/records`, `/training/matrix`, `/training/my-competence` | all |
| POST, PATCH | `/training/courses` (+ `/{id}`) · `/training/records` (signed) | QM/admin |
| GET, PUT | `/settings/notifications` · `/settings/notifications/email`, `/settings/notifications/whatsapp` | QM/admin |
| GET, PATCH | `/settings/quality` | all · QM/admin |
| GET | `/people` (active users with e-mail and phone, for recipients and attendees) | QM/admin |
| GET | `/reports/supplier-list`, `/reports/training-matrix`, `/reports/training-log` (PDF) | all |
| GET | `/analytics?date_from&date_to&control_point_id` (KPIs, trends, control chart, breakdowns, previous period) · `/analytics/live` (control points now, latest activity) | all |

List filters: `deviations_only` / `failures_only`, `unverified_only` (the review queue),
`control_point_id`, `area`, `measured_from`/`measured_to`, `performed_from`/`performed_to`,
`limit`. Filters always apply to the amended values.

## Alert notifications

Enter the e-mail and WhatsApp settings on the app's notifications page (QM/admin), or set
these in `.env` (then `docker compose up -d`). Settings saved in the app win over `.env`.
Unset channels are reported in the app and their deliveries fail with "not configured".

E-mail (any SMTP server; for Gmail use an App Password):

```
CE_SMTP_HOST=smtp.gmail.com
CE_SMTP_PORT=587
CE_SMTP_SECURITY=starttls        # or ssl (port 465), or none
CE_SMTP_USERNAME=qa.alerts@gmail.com
CE_SMTP_PASSWORD=app-password
CE_SMTP_FROM=qa.alerts@gmail.com
```

WhatsApp (Meta WhatsApp Business Cloud API):

```
CE_WHATSAPP_TOKEN=...                 # permanent token of a system user
CE_WHATSAPP_PHONE_NUMBER_ID=...       # from the WhatsApp > API setup page
CE_WHATSAPP_TEMPLATE=quality_alert    # approved template with one body variable {{1}}
CE_WHATSAPP_TEMPLATE_LANGUAGE=ar
```

Without a template, plain text is sent, which WhatsApp only delivers to people who messaged
the business number in the last 24 hours. `CE_PUBLIC_URL` (optional) adds a link to the app.
