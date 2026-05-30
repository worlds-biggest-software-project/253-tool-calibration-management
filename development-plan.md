# Tool & Calibration Management — Phased Development Plan

> Project: 253-tool-calibration-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.
>
> Research basis: research.md, features.md, standards.md, README.md, and data-model-suggestion-1..4.md (all present). The data design adopts **Suggestion 3 (Hybrid Relational + JSONB)** as the core schema for MVP velocity and instrument-type flexibility, incorporating the GUM uncertainty-budget and audit-trail rigour of Suggestion 1, and the purpose-built drift-history feature store from Suggestion 2 for the AI phase.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | The differentiating features (drift-rate ML interval prediction, GUM uncertainty propagation, MSA/gauge R&R statistics, NL audit assistant) are numerically and ML-heavy. Python has `numpy`/`scipy`/`scikit-learn` and the most mature LLM/MCP tooling, which dominate the post-MVP roadmap. |
| API framework | FastAPI | Native Pydantic v2 models, automatic **OpenAPI 3.1** generation (standards.md identifies OpenAPI 3.1 as the target and notes no open calibration API standard exists — this project should publish one), async support for webhooks and long AI calls, and first-class JSON Schema (Draft 2020-12). |
| ASGI server | Uvicorn + Gunicorn | Standard production combination for FastAPI; Gunicorn manages Uvicorn workers. |
| ORM / DB toolkit | SQLAlchemy 2.0 (async) + Alembic | Mature async ORM with JSONB support; Alembic gives auditable, repeatable schema migrations required for a regulated-data system. |
| Database | PostgreSQL 16 | The chosen hybrid model needs JSONB + GIN indexes, partial indexes, row-level security (multi-tenant), recursive CTEs (traceability chains), and partitioned audit tables. SQLite is insufficient for RLS and JSONB GIN. A single Docker-Compose Postgres covers self-hosted; managed Postgres covers SaaS. |
| Data validation | Pydantic v2 + `jsonschema` | Pydantic validates API request/response and the relational columns; `jsonschema` (Draft 2020-12) validates the JSONB `attributes`/`measurement_data` payloads against per-instrument-type templates, enforcing the discipline the hybrid model requires. |
| Task queue | Celery + Redis | Async workloads exist: due-date notification sweeps, certificate PDF rendering, webhook delivery with retry, ML inference, and event projection. Celery beat handles the daily scheduling sweep. Redis doubles as broker and cache. |
| Auth | OAuth 2.0 / OIDC (Authlib) + local password (Argon2) | standards.md mandates OAuth 2.0 / OIDC for enterprise SSO (Entra ID, Okta, Google). Local Argon2-hashed accounts cover self-hosted shops without an IdP. JWT access tokens, refresh tokens in DB. |
| Authorization | Casbin (RBAC with domains) | Multi-tenant, multi-site role-based access control (FDA Part 11, ISO 17025 require role separation, e.g. performer ≠ approver). Casbin domains map to tenants; policies are data-driven, not hard-coded. |
| PDF / certificate generation | WeasyPrint + Jinja2 | ISO 17025 certificates are HTML templates rendered to PDF/A. Jinja2 templates are customisable per tenant (a table-stakes feature); WeasyPrint produces print-quality, archival PDF/A-3. |
| Numerical / stats | numpy, scipy | GUM uncertainty combination (Welch–Satterthwaite), MSA ANOVA / X̄-R gauge R&R, drift regression. |
| ML | scikit-learn (MVP+1), with feature store in Postgres | Drift-rate interval prediction and OOT anomaly detection start as classical models (linear/robust regression, IsolationForest) trained on `drift_observation` rows — explainable, auditable, and adequate before deep learning. |
| LLM integration | Anthropic SDK + MCP server (`mcp` Python SDK) | NL audit assistant and certificate drafting use Claude. standards.md explicitly calls out MCP exposure of calibration data as an unaddressed differentiator; we ship an MCP server. |
| Frontend | Next.js 15 (App Router) + TypeScript + shadcn/ui | Dashboard, equipment registry, calibration entry forms, multi-site visibility, audit views. Server Components for fast tables; client components for measurement-entry forms. Mobile-responsive PWA covers the offline field-capture requirement in v1.1 without a separate native app initially. |
| Barcode / QR | `python-barcode` + `qrcode` (server labels); `@zxing/browser` (web scanner) | GS1-128/QR/DataMatrix label generation server-side; ZXing in the browser/PWA for camera-based scanning aligned with GS1 (standards.md EPCIS 2.0). |
| Object storage | S3-compatible (MinIO self-hosted / S3 SaaS) via `boto3` | Equipment documents, certificate PDFs, photos. MinIO ships in docker-compose for self-hosted parity. |
| Testing | pytest, pytest-asyncio, httpx, testcontainers, Playwright | Unit + async API tests; testcontainers spins real Postgres/Redis for integration tests; Playwright for frontend E2E. |
| Code quality | ruff (lint+format), mypy (strict), pre-commit | Single fast linter/formatter; strict typing on a compliance-critical codebase. |
| Package manager | uv (Python), pnpm (frontend) | uv for fast, reproducible Python installs; pnpm for the Next.js workspace. |
| Containerisation | Docker + docker-compose | Self-hosted is a primary deployment mode (open-source positioning, gap in market). Compose bundles API, worker, beat, Postgres, Redis, MinIO, frontend. |
| CI | GitHub Actions | Lint, type-check, test matrix, Docker build, OpenAPI spec diff check. |
| Migrations of OpenAPI consumers | Spectral + committed `openapi.json` | The generated spec is committed and linted; CI fails if it drifts from the running app, since third-party integrators depend on it. |

### Project Structure

```
tool-calibration-management/
├── pyproject.toml                 # uv project, ruff/mypy/pytest config
├── README.md
├── Dockerfile                     # multi-stage: api + worker share image
├── docker-compose.yml             # api, worker, beat, postgres, redis, minio, web
├── docker-compose.override.yml    # dev hot-reload
├── alembic.ini
├── openapi.json                   # committed generated spec (CI-checked)
├── .github/workflows/ci.yml
├── migrations/                    # Alembic versions
│   └── versions/
├── seeds/                         # JSON seed: system roles, instrument-type templates
│   └── instrument_templates/      # JSON Schema per instrument type
├── src/
│   └── tcm/
│       ├── __init__.py
│       ├── main.py                # FastAPI app factory, router mount, OpenAPI export
│       ├── config.py              # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── base.py            # async engine, session, declarative base
│       │   ├── models/            # SQLAlchemy ORM models (one file per domain group)
│       │   │   ├── infra.py       # tenant, site, app_user, role, user_role, api_key
│       │   │   ├── equipment.py   # equipment, category, manufacturer, document, checkout
│       │   │   ├── schedule.py    # procedure, schedule
│       │   │   ├── calibration.py # record, reference_standard, measurement_point
│       │   │   ├── uncertainty.py # uncertainty_budget, uncertainty_component
│       │   │   ├── oot.py         # oot_event, corrective_action
│       │   │   ├── msa.py         # msa_study, msa_measurement
│       │   │   ├── audit.py       # audit_log (partitioned)
│       │   │   ├── notify.py      # notification_rule, notification_log, webhook_endpoint
│       │   │   └── analytics.py   # drift_observation (AI feature store)
│       │   └── rls.py             # row-level-security session var helpers
│       ├── schemas/               # Pydantic request/response models, per domain
│       ├── jsonschema_templates.py# loads + validates instrument-type JSON Schemas
│       ├── repositories/          # data-access layer (one per aggregate)
│       ├── services/              # business logic
│       │   ├── equipment.py
│       │   ├── scheduling.py
│       │   ├── calibration.py
│       │   ├── uncertainty.py     # GUM engine
│       │   ├── msa.py             # gauge R&R engine
│       │   ├── oot.py
│       │   ├── certificate.py     # Jinja2 + WeasyPrint
│       │   ├── audit.py           # write-side audit hooks
│       │   ├── notifications.py
│       │   └── ai/
│       │       ├── interval_predictor.py
│       │       ├── oot_anomaly.py
│       │       └── audit_assistant.py
│       ├── api/
│       │   ├── deps.py            # auth, tenant, RBAC dependencies
│       │   └── routers/           # FastAPI routers, one per resource
│       ├── auth/                  # OAuth/OIDC, JWT, Argon2, Casbin enforcer
│       ├── tasks/                 # Celery tasks (notifications, certs, webhooks, ml)
│       ├── mcp/
│       │   └── server.py          # MCP server exposing calibration resources
│       └── labels/                # barcode/QR label generation
├── web/                           # Next.js 15 frontend (pnpm workspace)
│   ├── package.json
│   ├── app/
│   ├── components/
│   └── lib/api/                   # typed client generated from openapi.json
└── tests/
    ├── unit/
    ├── integration/               # testcontainers Postgres/Redis
    ├── e2e/                       # Playwright
    └── fixtures/                  # sample calibrations, instrument templates, payloads
```

Group is by concern (infra, equipment, calibration, uncertainty, …) so each phase adds files/columns rather than restructuring.

---

## Phase 1: Foundation — Project, Config, Multi-Tenant Core, Auth

### Purpose
Establish the runnable skeleton: FastAPI app, Postgres with Alembic migrations, Docker Compose, the multi-tenant identity core (tenant, site, user, role, RBAC), authentication (local + OIDC), and the append-only audit-log substrate that every later phase writes to. After this phase a user can authenticate, the API enforces tenant isolation and roles, and all writes are auditable — the non-negotiable compliance floor.

### Tasks

#### 1.1 — Project scaffold, config, and Docker Compose

**What**: Bootable FastAPI app with environment-driven config and a full local stack.

**Design**:
- `tcm/config.py` using `pydantic_settings.BaseSettings`:
```python
class Settings(BaseSettings):
    database_url: str          # postgresql+asyncpg://...
    redis_url: str = "redis://redis:6379/0"
    jwt_secret: str
    jwt_access_ttl_seconds: int = 900
    jwt_refresh_ttl_seconds: int = 1209600
    s3_endpoint: str | None = None
    s3_bucket: str = "tcm"
    oidc_providers: dict[str, OidcProvider] = {}   # keyed by slug
    environment: Literal["dev", "test", "prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="TCM_", env_nested_delimiter="__")
```
- `tcm/main.py` exposes `create_app() -> FastAPI`, mounts routers, adds CORS, exception handlers, and an `/healthz` endpoint returning `{"status":"ok","db":bool,"redis":bool}`.
- `docker-compose.yml` services: `api`, `worker`, `beat`, `postgres:16`, `redis:7`, `minio`, `web`. Healthchecks on postgres/redis; api `depends_on` healthy db.
- Add a `make` target / `scripts/export_openapi.py` that writes `openapi.json` from `create_app().openapi()`.

**Testing**:
- `Unit: Settings loads from env with TCM_ prefix → correct typed values; missing jwt_secret → ValidationError`.
- `Integration: GET /healthz against testcontainers Postgres+Redis → 200, db=true, redis=true`.
- `Integration: docker compose up; curl /healthz → 200` (smoke, marked slow).

#### 1.2 — Database base, RLS scaffolding, audit-log table

**What**: Async SQLAlchemy engine/session, declarative base, row-level-security helpers, and the partitioned `audit_log` table.

**Design**:
- `db/base.py`: `create_async_engine`, `async_sessionmaker`, `Base(DeclarativeBase)`, a `get_session` FastAPI dependency that sets the Postgres session variable `app.current_tenant` and `app.current_user` (consumed by RLS policies).
- `audit_log` (partitioned monthly by `performed_at`), per Suggestion 1:
```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  table_name VARCHAR(100) NOT NULL,
  record_id UUID NOT NULL,
  action VARCHAR(20) NOT NULL,            -- INSERT | UPDATE | DELETE
  changed_fields JSONB,                   -- {"field":{"old":..,"new":..}}
  performed_by UUID,
  performed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  signature_meaning VARCHAR(100),         -- created | reviewed | approved | modified
  signature_method VARCHAR(50),           -- password | mfa | oidc
  ip_address INET,
  user_agent VARCHAR(500)
) PARTITION BY RANGE (performed_at);
```
- `services/audit.py` exposes `record_audit(session, table, record_id, action, before, after, *, ctx)` computing `changed_fields` as a field-level diff. A SQLAlchemy `after_flush` event hook auto-audits all mapped tables flagged `__audited__ = True`.
- Audit rows are insert-only; no UPDATE/DELETE grant on `audit_log` (enforced by a DB role), satisfying FDA 21 CFR Part 11 §11.10(e).

**Testing**:
- `Unit: record_audit diff of {"a":1,"b":2} → {"a":1,"b":3} yields changed_fields {"b":{"old":2,"new":3}}`.
- `Integration: update an audited row → exactly one audit_log INSERT with correct diff and performed_by`.
- `Integration: attempt UPDATE on audit_log as app role → permission denied`.
- `Integration (RLS): session set to tenant A cannot SELECT tenant B rows`.

#### 1.3 — Identity core: tenant, site, user, role, RBAC

**What**: Migrations and models for `tenant`, `site`, `app_user`, `role`, `user_role`, `api_key`, plus Casbin RBAC.

**Design**:
- Tables from Suggestion 1 (`tenant`, `site`, `app_user`, `role`, `user_role`) plus:
```sql
CREATE TABLE api_key (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenant(id),
  name VARCHAR(255) NOT NULL,
  key_hash VARCHAR(255) NOT NULL,         -- Argon2 of the token; prefix stored plain
  key_prefix VARCHAR(12) NOT NULL,
  scopes JSONB NOT NULL DEFAULT '[]',
  created_by UUID REFERENCES app_user(id),
  last_used_at TIMESTAMPTZ,
  revoked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- System roles seeded (`seeds/roles.json`): `quality_manager`, `calibration_technician`, `reviewer`, `auditor`(read-only), `site_admin`, `tenant_admin`. Permissions are `resource:action` strings (`equipment:write`, `calibration:approve`, `calibration:perform`, `audit:read`, …).
- Casbin model: RBAC with domains; domain = `tenant_id`, optionally scoped per `site_id`. `auth/rbac.py` exposes `enforce(user, tenant, action, *, site=None)`.
- ISO 17025 separation rule encoded as policy: a user with `calibration:perform` on a record cannot also perform `calibration:approve` on the same record (enforced in service layer, see 4.4).

**Testing**:
- `Unit: technician role → enforce(equipment:write)=True, enforce(calibration:approve)=False`.
- `Unit: auditor role → all *:read True, all *:write False`.
- `Integration: create tenant + admin user → admin can create site and assign roles`.
- `Integration: API key auth with scope equipment:read → 200 on GET equipment, 403 on POST`.

#### 1.4 — Authentication: local + OIDC, JWT issuance

**What**: Login endpoints for local password and OIDC SSO, JWT access/refresh tokens, FastAPI auth dependencies.

**Design**:
- Endpoints:
  - `POST /auth/login` `{email,password}` → `{access_token, refresh_token, token_type:"bearer"}`. Argon2 verify.
  - `POST /auth/refresh` `{refresh_token}` → new access token (refresh tokens stored hashed in `refresh_token` table, rotation on use).
  - `GET /auth/oidc/{provider}/start` → 302 to IdP authorize URL (Authlib).
  - `GET /auth/oidc/{provider}/callback` → exchange code, match/JIT-provision `app_user` by email within the provider's tenant mapping, issue JWTs.
- JWT claims: `sub`(user id), `tid`(tenant id), `roles`, `sites`, `exp`, `iat`. RS256 in prod, HS256 acceptable for self-hosted single-instance.
- `api/deps.py`: `current_user()`, `require(action, site=None)` dependency factory using Casbin; sets RLS session vars from `tid`/`sub`.

**Testing**:
- `Unit: Argon2 hash/verify round-trip; wrong password → False`.
- `Integration: login valid → 200 + tokens; invalid password → 401, audit_log records failed-login`.
- `Integration (mocked IdP): OIDC callback with valid code → JIT user created, JWT issued`.
- `Integration: expired access token → 401; valid refresh → new access token; reused (rotated) refresh → 401`.

---

## Phase 2: Equipment Inventory & Identification

### Purpose
Deliver the table-stakes asset registry: equipment CRUD with instrument-type-specific JSONB attributes validated against JSON Schema templates, categories/manufacturers, document attachments to object storage, barcode/QR identification, and check-in/check-out. This is the entity every calibration record hangs off; after this phase a shop can catalogue its entire toolroom.

### Tasks

#### 2.1 — Instrument-type templates and JSONB validation

**What**: JSON Schema templates per instrument type that validate the equipment `attributes` JSONB and seed default calibration point structures.

**Design**:
- `seeds/instrument_templates/*.json` — one JSON Schema (Draft 2020-12) per type (`pressure_gauge`, `torque_wrench`, `multimeter`, `thermocouple`, `caliper`, `generic`). Example `pressure_gauge.json` requires `measurement_range`, `unit`, `accuracy_class`; optionally `default_points: [{nominal, unit, tolerance_lower, tolerance_upper}]`.
- `instrument_type` table:
```sql
CREATE TABLE instrument_type (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenant(id),
  code VARCHAR(50) NOT NULL,
  name VARCHAR(255) NOT NULL,
  attributes_schema JSONB NOT NULL,       -- JSON Schema
  default_points JSONB,                    -- seed measurement points
  is_system BOOLEAN NOT NULL DEFAULT false,
  UNIQUE(tenant_id, code)
);
```
- `jsonschema_templates.py`: `validate_attributes(type_code, attributes) -> None | raises AttributesValidationError(field, message)`.

**Testing**:
- `Unit: pressure_gauge attributes missing measurement_range → AttributesValidationError naming the field`.
- `Unit: valid multimeter attributes → passes`.
- `Fixture: every seeded template is itself a valid Draft 2020-12 schema`.

#### 2.2 — Equipment CRUD (hybrid relational + JSONB)

**What**: Equipment table and endpoints with relational lifecycle columns plus validated JSONB `attributes` and `custom_fields`.

**Design**:
- `equipment` table = Suggestion 3 hybrid: relational columns from Suggestion 1 (`asset_tag`, `serial_number`, `model_number`, `barcode`, `rfid_tag`, `equipment_type`, `status`, `site_id`, `custodian_id`, lifecycle dates, cost) **plus** `instrument_type_id UUID`, `attributes JSONB`, `custom_fields JSONB`. GIN index on `attributes` and `custom_fields`.
- Status enum: `in_service | due_calibration | overdue | out_of_tolerance | in_calibration | quarantined | retired | lost`.
- Endpoints (all tenant + RBAC scoped, all audited):
  - `POST /equipment` — validates `attributes` against the instrument type's schema (2.1) before insert.
  - `GET /equipment` — filter by `status`, `site_id`, `category_id`, `q` (asset_tag/serial/barcode), JSONB attribute filters (`?attr.unit=PSI`); cursor pagination.
  - `GET /equipment/{id}`, `PATCH /equipment/{id}`, `POST /equipment/{id}/retire`.
- `manufacturer`, `equipment_category` (hierarchical), `equipment_document` tables from Suggestion 1.

**Testing**:
- `Unit: status transition retire → status=retired, retirement_date set`.
- `Integration: POST equipment with invalid attributes → 422 with field name; valid → 201`.
- `Integration: GET /equipment?attr.unit=PSI returns only matching JSONB rows (GIN-indexed)`.
- `Integration: duplicate asset_tag within tenant → 409; same tag in other tenant → allowed`.

#### 2.3 — Document attachments to object storage

**What**: Upload/download equipment documents (manuals, datasheets, photos, SOPs) to S3/MinIO.

**Design**:
- `POST /equipment/{id}/documents` (multipart) → stores object at `s3://{bucket}/{tenant}/equipment/{id}/{uuid}` via `boto3`, inserts `equipment_document` row.
- `GET /equipment/{id}/documents/{doc_id}` → presigned URL (5-min TTL).
- Max size and MIME allowlist configurable.

**Testing**:
- `Integration (MinIO container): upload PDF → object exists, row created, presigned URL downloads identical bytes`.
- `Integration: disallowed MIME (e.g. .exe) → 415`.

#### 2.4 — Barcode/QR labels and check-in/check-out

**What**: Generate scannable asset labels and track tool issue/return.

**Design**:
- `GET /equipment/{id}/label?format=qr|datamatrix|gs1-128` → PNG/SVG. QR encodes a deep link `https://{host}/e/{barcode}`; GS1-128 follows GS1 AI structure (standards.md EPCIS 2.0).
- `equipment_checkout` table (Suggestion 1). Endpoints `POST /equipment/{id}/checkout` `{expected_return,purpose}`, `POST /equipment/{id}/checkin` `{condition_on_return,return_notes}`. Checkout sets a derived `is_checked_out` view flag; checkin records condition.
- `GET /scan/{barcode}` resolves a scanned code to an equipment summary (used by the web scanner).

**Testing**:
- `Unit: GS1-128 encoder produces valid AI(01) GTIN payload for given barcode`.
- `Integration: checkout then checkin → two audit rows, checkout closed with return condition`.
- `Integration: checkout already-checked-out item → 409`.
- `Integration: GET /scan/{barcode} unknown code → 404`.

---

## Phase 3: Calibration Scheduling & Due-Date Engine

### Purpose
Add configurable calibration intervals, due-date computation, status roll-forward (in_service → due → overdue), and the daily sweep that drives notifications. This converts a static inventory into a live compliance system that tells users what to calibrate and when (ISO 9001 7.1.5).

### Tasks

#### 3.1 — Procedures and schedules

**What**: Calibration procedures (versioned) and per-equipment schedules with interval source.

**Design**:
- `calibration_procedure` and `calibration_schedule` tables from Suggestion 1. `calibration_schedule.interval_source ∈ {fixed, manufacturer_recommended, ncsl_adjusted, ai_predicted}`; `next_due_date`, `notification_days_before`.
- Procedure carries `measurement_points_template JSONB` (default set points) reused when starting a calibration.
- Endpoints: CRUD `/procedures`, `/equipment/{id}/schedule`. Creating/updating a schedule recomputes `next_due_date = last_calibration_date + interval_days` (or registration date if never calibrated).

**Testing**:
- `Unit: next_due_date = last_cal + interval_days`.
- `Integration: procedure versioning — editing an active procedure creates v2, v1 superseded, historical records keep v1 link`.

#### 3.2 — Status roll-forward sweep (Celery beat)

**What**: Daily job that transitions equipment status based on due dates and emits due/overdue events.

**Design**:
- Celery beat task `sweep_due_dates` (daily 02:00 per-site timezone):
  1. For active schedules where `next_due_date - today <= notification_days_before` and equipment `in_service` → set `due_calibration`.
  2. Where `next_due_date < today` → set `overdue`.
  3. Enqueue `notification` events (Phase 6 consumes; Phase 3 logs intent).
- Idempotent: re-running the same day produces no duplicate status changes or notifications (dedupe key `equipment_id + due_date + rule`).

**Testing**:
- `Unit: schedule due in 10 days, threshold 30 → status due_calibration`.
- `Unit: schedule due yesterday → overdue`.
- `Integration: run sweep twice in one day → notifications enqueued once (idempotent)`.
- `Integration: per-site timezone respected (site in UTC+8 rolls over at its local midnight)`.

#### 3.3 — Schedule export (iCalendar)

**What**: Export due dates as an iCalendar feed for Outlook/Google.

**Design**:
- `GET /calendar/{site_id}.ics` (token-authenticated feed URL) emits `VEVENT` per upcoming due date with `RRULE` derived from `interval_days` (RFC 5545). Read-only.

**Testing**:
- `Unit: interval 365 days → RRULE FREQ=YEARLY`.
- `Integration: feed parses with the `icalendar` library and contains one VEVENT per active schedule`.

---

## Phase 4: Calibration Execution, Measurement Capture & Review Workflow

### Purpose
The heart of the product: capture a calibration — multi-point as-found/as-left measurements with tolerances and uncertainty, environmental conditions, reference-standard traceability, automatic pass/fail with ILAC G8 decision rules — and run it through a performer → reviewer → approver lifecycle that enforces ISO 17025 independence and FDA Part 11 e-signatures.

### Tasks

#### 4.1 — Calibration record and measurement points

**What**: Create a calibration record from a schedule/procedure and capture measurement points.

**Design**:
- `calibration_record`, `calibration_reference_standard`, `measurement_point` tables from Suggestion 1 (relational; measurement points are first-class rows for cross-equipment SQL, per Suggestion 1 decision #3). Add `calibration_record.measurement_data JSONB` for instrument-specific extras (hybrid).
- Measurement value/unit/uncertainty stored per the D-SI atomic structure (value + unit + uncertainty), matching Suggestion 2's payload shape for future DCC export.
- Endpoints:
  - `POST /calibrations` `{equipment_id, procedure_id?, calibration_type}` → record in `in_progress`, seeded with procedure default points.
  - `POST /calibrations/{id}/measurements` (bulk) → array of points (`nominal_value`, `unit`, `tolerance_lower/upper`, `as_found_value`, `as_left_value`, `direction`).
  - `POST /calibrations/{id}/environment` → ambient temp/humidity/pressure.
  - `POST /calibrations/{id}/reference-standards` → link reference equipment + its cert number/accreditation body.

**Testing**:
- `Unit: starting from a procedure copies its default points`.
- `Integration: POST measurements for a record in approved state → 409 (locked)`.

#### 4.2 — Tolerance evaluation and ILAC G8 decision rules

**What**: Compute per-point conformity and overall record result using configurable decision rules with guard banding.

**Design**:
- `services/calibration.py::evaluate_point(point, decision_rule, guard_band_factor)`:
  - `as_found_deviation = as_found_value - nominal_value`; same for as_left.
  - `simple_acceptance`: in-tolerance iff `tolerance_lower <= deviation <= tolerance_upper`.
  - `guard_banded_95`: acceptance limits tightened by `guard_band_factor * expanded_uncertainty` (ILAC G8 / ISO 14253-1).
  - sets `point_result ∈ {in_tolerance, out_of_tolerance, adjusted}` and `conformity_statement ∈ {conforming, non_conforming, not_stated}`.
- Overall `result`: `fail` if any as_found OOT; `pass_with_adjustment` if any as_found OOT but all as_left in tolerance; else `pass`.
- Any as-found OOT auto-creates an `oot_event` (Phase 5 manages disposition).

**Testing**:
- `Unit: deviation 0.3 within ±0.5 → in_tolerance, conforming`.
- `Unit: guard_banded_95 with U=0.2, k=2 reduces acceptance band → borderline point becomes not_stated`.
- `Unit: as_found OOT, as_left in-tol → record result pass_with_adjustment + oot_event created`.

#### 4.3 — Certificate generation (ISO 17025 PDF/A)

**What**: Render an ISO/IEC 17025-compliant calibration certificate to PDF/A from record data.

**Design**:
- `services/certificate.py`: Jinja2 HTML template → WeasyPrint → PDF/A-3, stored to object storage; `calibration_record.certificate_number` assigned on first issue, `certificate_revision` incremented on re-issue.
- Template includes ISO 17025 Clause 7.8 mandatory elements: lab/customer identity, unique cert number, equipment identification, dates, environmental conditions, measurement results table (nominal, as-found, as-left, tolerance, expanded uncertainty, k, conformity), reference-standard traceability statement, decision rule (ILAC G8), authorising signatory.
- Tenant-customisable template (logo, header/footer) via a stored template override.
- `GET /calibrations/{id}/certificate.pdf` → presigned download; certificate is generated only for `approved` records.

**Testing**:
- `Integration: generate cert for an approved record → valid PDF/A, contains cert number, all measurement rows, uncertainty column, traceability statement` (assert via `pikepdf`/text extraction).
- `Integration: request cert for non-approved record → 409`.
- `Integration: re-issue → certificate_revision increments, prior PDF retained`.

#### 4.4 — Review/approval workflow with e-signatures (Part 11)

**What**: State machine enforcing performer → review → approval with role separation and electronic signatures.

**Design**:
- States: `in_progress → pending_review → approved | rejected`, plus `voided`.
- Transitions (each requires re-authentication / e-signature → writes `audit_log` row with `signature_meaning`):
  - `submit` (technician, `calibration:perform`): in_progress → pending_review.
  - `approve` (`calibration:approve`): pending_review → approved. **Rejected if approver == performer** (ISO 17025 independence, enforced in service + RBAC policy 1.3).
  - `reject`: pending_review → in_progress with `reviewer_notes`.
  - `void`: any → voided (reason mandatory).
- On `approve`: schedule's `last_calibration_date`/`next_due_date` advanced; equipment status → `in_service` or `out_of_tolerance`.

**Testing**:
- `Unit: state machine rejects illegal transition (in_progress → approved) → error`.
- `Integration: performer attempts to approve own record → 403, audit notes denied`.
- `Integration: approve → audit_log row signature_meaning=approved, schedule advanced, equipment status updated`.
- `Integration: void approved record → status voided, certificate marked invalid`.

---

## Phase 5: Out-of-Tolerance Workflow & CAPA

### Purpose
Turn OOT findings into a governed investigation: severity, IATF 16949 retroactive impact assessment, disposition, and linked corrective/preventive actions through to verification. This is the compliance backbone automotive (IATF 16949) and aerospace (AS9100) buyers require.

### Tasks

#### 5.1 — OOT event lifecycle

**What**: Manage the `oot_event` created in 4.2 through disposition.

**Design**:
- `oot_event` + `corrective_action` tables from Suggestion 1. Disposition ∈ `pending | use_as_is | adjust_recalibrate | repair | retire | quarantine`. Setting `quarantine`/`retire` updates linked equipment status.
- IATF fields: `impact_assessment`, `products_affected`, `retroactive_action_required`, `retroactive_period_start/end`.
- Endpoints: `GET /oot-events`, `GET/PATCH /oot-events/{id}`, `POST /oot-events/{id}/disposition`, `POST /oot-events/{id}/impact-assessment`.

**Testing**:
- `Integration: disposition=quarantine → equipment status quarantined + audit row`.
- `Integration: impact assessment requiring retroactive action without period dates → 422`.

#### 5.2 — Corrective actions (CAPA)

**What**: Track corrective/preventive actions with assignment, due date, and effectiveness verification.

**Design**:
- States: `open → in_progress → completed → verified → closed`. `verified_by` must differ from `assigned_to`. Endpoints CRUD under `/oot-events/{id}/actions` and `/actions/{id}/verify`.
- Overdue actions feed the notification engine (Phase 6).

**Testing**:
- `Unit: verifier == assignee → rejected`.
- `Integration: complete then verify → state verified, audit trail intact`.

---

## Phase 6: Notifications, Webhooks & Public REST API / OpenAPI

### Purpose
Close the integration loop required by the market: configurable email/webhook notifications for due/overdue/OOT/approval events, reliable webhook delivery, and a documented, versioned public REST API with a committed OpenAPI 3.1 spec — the interoperability standard standards.md says the domain lacks.

### Tasks

#### 6.1 — Notification rules and email delivery

**What**: Rule-driven notifications consumed from the scheduling sweep and workflow events.

**Design**:
- `notification_rule`, `notification_log` tables from Suggestion 1. Event types: `calibration_due`, `calibration_overdue`, `oot_event`, `approval_required`, `certificate_expiring`, `action_overdue`. Channels: `email`, `webhook`, `in_app`.
- Celery task `dispatch_notifications` renders Jinja2 email templates and sends via SMTP (config-driven); logs to `notification_log` with status.

**Testing**:
- `Integration (mock SMTP): due event matching a rule → one email rendered with equipment + due date, notification_log status=sent`.
- `Integration: failed SMTP → status=failed, retried by Celery, no duplicate on success`.

#### 6.2 — Webhook delivery with signing and retry

**What**: Outbound webhooks for event subscriptions.

**Design**:
- `webhook_endpoint` table (url, secret, subscribed_event_types, is_active). Task `deliver_webhook` POSTs a JSON Schema-described payload with an HMAC-SHA256 `X-TCM-Signature` header; exponential backoff retry (max 5), dead-letter on exhaustion. Payloads versioned (`schema_version`).

**Testing**:
- `Integration: subscribed endpoint receives signed payload; signature verifies with secret`.
- `Integration: endpoint returning 500 → retried with backoff; after max attempts → dead-lettered`.

#### 6.3 — Public REST API surface, OpenAPI 3.1, API keys

**What**: Finalise the externally documented API and publish the spec.

**Design**:
- All resource routers expose stable, versioned paths under `/api/v1`. Pydantic response models carry JSON Schema (Draft 2020-12). `scripts/export_openapi.py` writes `openapi.json`; CI fails if it diverges from the live app (`spectral lint` + diff).
- API-key auth (1.3) with scopes for machine integrations (CMMS/ERP/QMS).
- Define a candidate open **Calibration Record JSON schema** (`schemas/public/calibration_record.schema.json`) as the project's proposed interoperability format (standards.md opportunity).

**Testing**:
- `Integration: openapi.json validates against OpenAPI 3.1 meta-schema`.
- `CI: regenerate spec and diff against committed openapi.json → must match`.
- `Integration: API key with calibration:read can GET a record as the published JSON schema; schema validates`.

---

## Phase 7: Frontend — Dashboard, Registry, Calibration Entry, Multi-Site

### Purpose
Ship the web UI metrology and quality managers actually use day-to-day: a multi-site dashboard, equipment registry with scanning, the calibration measurement-entry form, the review/approval queue, and audit views. Delivers the "modern UI, multi-site visibility" advantage incumbents lack.

### Tasks

#### 7.1 — Auth, app shell, typed API client

**What**: Next.js app with login (local + OIDC), session handling, and a typed client generated from `openapi.json`.

**Design**:
- `web/lib/api` generated via `openapi-typescript`. Auth via httpOnly refresh cookie + in-memory access token. Route groups: `(auth)`, `(app)`. shadcn/ui + Tailwind.

**Testing**:
- `E2E (Playwright): login → redirected to dashboard; logout clears session`.
- `E2E: OIDC button redirects to mocked IdP and returns authenticated`.

#### 7.2 — Multi-site dashboard

**What**: Cross-site KPIs: due-within-30, overdue, in-calibration, quarantined, open OOT.

**Design**:
- Server Component fetching an aggregate endpoint `GET /api/v1/dashboard?site_id=`. Cards + a due-soon table; site selector for tenant-admins, scoped automatically for site users.

**Testing**:
- `E2E: dashboard shows correct overdue count for seeded data; switching site updates figures`.

#### 7.3 — Equipment registry and web scanner

**What**: Searchable equipment table, detail view, create/edit form with dynamic attribute fields, and camera-based barcode/QR scanning.

**Design**:
- Form renders attribute inputs dynamically from the instrument-type JSON Schema (2.1). `@zxing/browser` scanner resolves codes via `GET /scan/{barcode}` and routes to the equipment detail.

**Testing**:
- `E2E: create pressure gauge → attribute fields driven by schema; invalid attribute shows inline error`.
- `E2E: scanning a known QR opens the right equipment (mocked camera input)`.

#### 7.4 — Calibration entry, review queue, audit view

**What**: Measurement-entry grid, submit-for-review, approval queue, and per-record audit trail display.

**Design**:
- Calibration form: editable points grid (nominal/as-found/as-left/tolerance), live pass/fail colouring from 4.2 logic mirrored client-side, environment panel, reference-standard picker. Submit triggers e-signature modal. Reviewer queue lists `pending_review`; approve/reject with signature. Audit tab renders `audit_log` diffs.

**Testing**:
- `E2E: enter a failing point → row flagged red, record result=fail on submit`.
- `E2E: technician cannot see approve action on own record; reviewer can approve another's`.
- `E2E: approved record exposes a Download Certificate button → PDF downloads`.

---

## Phase 8: GUM Uncertainty Budgets & MSA / Gauge R&R

### Purpose
Add the metrology depth that separates this tool from CMMS-bundled modules: GUM-compliant uncertainty budget computation feeding measurement points, and a full MSA/gauge R&R study module (variable + attribute) required by IATF 16949.

### Tasks

#### 8.1 — GUM uncertainty engine

**What**: Compute combined and expanded uncertainty from Type A/B components per JCGM 100.

**Design**:
- `uncertainty_budget`, `uncertainty_component` tables from Suggestion 1. `services/uncertainty.py`:
  - per component standard uncertainty `u_i = input_value / divisor` (divisor √3 rectangular, √6 triangular, 2 normal-95, etc.), contribution `(c_i · u_i)²`.
  - combined `u_c = sqrt(Σ (c_i·u_i)²)`; effective DoF via **Welch–Satterthwaite**; coverage factor `k` from Student-t at `effective_dof` (default 2); `U = k · u_c`.
  - Result written back to the linked `measurement_point.expanded_uncertainty`/`coverage_factor`.
- Endpoints: `POST /calibrations/{id}/uncertainty-budgets`, `GET .../uncertainty-budgets/{id}`.

**Testing**:
- `Unit: rectangular component input 0.05 → u_i = 0.05/√3 ≈ 0.0289`.
- `Unit: known textbook budget (GUM example) → U matches reference within 1e-6`.
- `Unit: Welch–Satterthwaite effective DoF matches reference for mixed A/B components`.

#### 8.2 — MSA / Gauge R&R

**What**: Variable (ANOVA + X̄-R) and attribute gauge studies with acceptance verdicts.

**Design**:
- `msa_study`, `msa_measurement` tables from Suggestion 1. `services/msa.py` computes Repeatability (EV), Reproducibility (AV), Gage R&R, Part Variation, Total Variation, %GRR (vs tolerance and study), NDC. Verdict: `acceptable <10%`, `marginal 10–30%`, `unacceptable >30%` (AIAG MSA 4th Ed methodology, implemented from the published manual per features.md legal note).
- Endpoints: create study, bulk-record measurements (operator × part × trial), `POST .../analyze`.

**Testing**:
- `Unit: ANOVA gauge R&R on canonical AIAG dataset → %GRR, NDC match published values within tolerance`.
- `Unit: %GRR 22% → verdict marginal`.
- `Integration: incomplete measurement matrix (missing trial) → analyze returns 422`.

---

## Phase 9: AI — Drift Feature Store, Interval Prediction & OOT Anomaly Detection

### Purpose
Deliver the headline AI-native differentiators: replace fixed intervals with drift- and usage-informed predictions, and flag instruments likely to drift OOT before their due date — capabilities research.md identifies as absent across all incumbents.

### Tasks

#### 9.1 — Drift observation feature store

**What**: Populate a purpose-built feature table from approved calibrations for ML.

**Design**:
- `drift_observation` table (from Suggestion 2's `read_drift_history`): one row per measurement point per approved calibration — `equipment_id, date_calibrated, point_label, nominal_value, unit, as_found_deviation, expanded_uncertainty, point_result, interval_days_at_time`. Populated by a service hook on calibration approval (4.4) and backfillable.

**Testing**:
- `Integration: approving a calibration inserts one drift_observation per measurement point with the interval in effect`.
- `Integration: backfill job over historical records is idempotent`.

#### 9.2 — Interval prediction

**What**: Per-equipment recommended calibration interval from drift history.

**Design**:
- `services/ai/interval_predictor.py`: robust linear regression of `|as_found_deviation|` vs days-since-prior-cal per point; predict interval at which projected drift hits a fraction (e.g. 70%) of tolerance, with a confidence margin. Requires ≥4 historical cycles; otherwise returns `insufficient_data`.
- `POST /equipment/{id}/predict-interval` → `{recommended_interval_days, current_interval_days, confidence, rationale, basis_points}`. Applying a prediction sets `calibration_schedule.interval_source='ai_predicted'` and is fully audited; never auto-applied without user confirmation.

**Testing**:
- `Unit: synthetic linear drift 0.1/30d, tolerance ±0.5 → interval ≈ point reaching 0.35`.
- `Unit: <4 cycles → insufficient_data, no recommendation`.
- `Integration: apply prediction → schedule interval_source=ai_predicted, next_due recomputed, audit row written`.

#### 9.3 — Predictive OOT anomaly alerts

**What**: Flag instruments whose drift trajectory is anomalous before the due date.

**Design**:
- `services/ai/oot_anomaly.py`: per-equipment IsolationForest / trend test on the deviation time series; a daily Celery task scores active equipment and raises an `in_app`/email notification (`predicted_oot`) with a risk score and the contributing trend. Explainable: returns the deviation slope and projected exceedance date, not just a black-box score.

**Testing**:
- `Unit: monotonically accelerating drift → flagged high risk; stable series → low risk`.
- `Integration: scoring task raises a predicted_oot notification for a high-risk instrument`.

---

## Phase 10: MCP Server & NL Audit Assistant

### Purpose
Expose calibration data through the Model Context Protocol and provide a natural-language audit-preparation assistant — the AI-native differentiator standards.md flags as implemented by no commercial product.

### Tasks

#### 10.1 — MCP server

**What**: MCP server exposing read-only calibration resources and tools to LLM clients.

**Design**:
- `tcm/mcp/server.py` (Python `mcp` SDK). Resources/tools (read-only, RBAC- and tenant-scoped via a scoped API key): list/query equipment, fetch calibration records, OOT events + linked CAPA, schedule/due data, and an ISO 17025 clause-mapping checklist resource. All responses use the published JSON schema (6.3).

**Testing**:
- `Integration (MCP client harness): query equipment due next 30 days → correct list; auditor-scoped key cannot invoke any mutating tool (none exposed)`.

#### 10.2 — NL audit assistant

**What**: Answer audit-prep questions and surface compliance gaps against ISO 17025 / IATF 16949 clauses.

**Design**:
- `services/ai/audit_assistant.py`: Claude (Anthropic SDK) with tool access to the MCP resources; a clause-mapping table links standard clauses (ISO 17025 6.4/7.5/7.6/7.8; IATF 7.1.5.1) to queryable data checks (e.g. "all equipment has current calibration", "all OOT events have impact assessment", "approver ≠ performer on every record"). Endpoint `POST /audit-assistant/query` `{question}` → grounded answer + cited records + a gap list. Answers are advisory and cite record IDs; no data is mutated.

**Testing**:
- `Integration (mocked LLM): "which instruments are overdue?" → answer enumerates the seeded overdue items with IDs`.
- `Unit: clause check "approver≠performer" detects a seeded violation and reports the record`.
- `Integration: assistant never calls a mutating endpoint (tool allowlist enforced)`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, tenancy, RBAC, audit)        ── required by everything
   │
Phase 2: Equipment Inventory                             ── requires 1
   │
Phase 3: Scheduling & Due-Date Engine                    ── requires 2
   │
Phase 4: Calibration Execution & Review                  ── requires 2,3
   │
   ├── Phase 5: OOT Workflow & CAPA                       ── requires 4
   ├── Phase 6: Notifications, Webhooks, Public API       ── requires 3,4 (5 for OOT events)
   └── Phase 8: Uncertainty & MSA                         ── requires 4 (can parallel 5/6)
        │
Phase 7: Frontend                                        ── requires 4,6 (consumes 5,8 as built)
   │
Phase 9: AI Drift/Interval/Anomaly                       ── requires 4 (8 improves features)
   │
Phase 10: MCP Server & NL Audit Assistant                ── requires 4,5,6
```

Parallelism:
- After Phase 4, **Phases 5, 6, and 8** can be developed concurrently.
- **Phase 7 (frontend)** can begin against Phase 6's API and absorb 5/8 UI as those land.
- **Phases 9 and 10** can proceed in parallel once Phase 4 (and 5/6 for Phase 10) are complete.

Scope mapping to features.md: Phases 1–7 deliver the full **MVP (Must-have)**; Phases 8–9 deliver **Should-have (v1.1)** depth (uncertainty, MSA, AI intervals); Phase 10 plus 9.3 deliver **Nice-to-have (backlog)** AI items (NL audit assistant, predictive OOT). Bidirectional SAP/Maximo connectors, native mobile, and GxP validation pack remain post-plan backlog built atop the Phase 6 webhook/API surface.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase implemented.
2. All unit and integration tests for the phase pass; coverage ≥ 85% on new service code.
3. `ruff check` and `ruff format --check` pass; `mypy --strict` passes on `src/tcm`.
4. Alembic migration(s) created, reversible, and applied cleanly to a fresh database; `alembic upgrade head` then `downgrade base` succeeds in CI.
5. Docker image builds; `docker compose up` brings the stack to healthy `/healthz`.
6. The phase's feature works end-to-end (demonstrated by an integration or E2E test).
7. New configuration options documented in README and present in `.env.example`.
8. New/changed API endpoints appear in regenerated `openapi.json`, which matches the committed spec (CI diff check) and passes `spectral lint`.
9. All new mutating endpoints write `audit_log` rows; RLS confirmed to isolate tenants for new tables.
10. For frontend phases: Playwright E2E green in CI headless run.
```
