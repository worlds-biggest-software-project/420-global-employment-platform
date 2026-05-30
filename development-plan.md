# Global Employment Platform (EOR) — Phased Development Plan

> Project: 420-global-employment-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, phased build. The chosen data model is **Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** — typed columns for the ~60-70% of the model that is universal across jurisdictions, JSONB (with JSON Schema validation) for the ~30-40% that varies wildly by country. We borrow bi-temporal `effective_from`/`effective_to` versioning on regulatory tables from Suggestion 1/4, and an append-only `audit_events` table (Suggestion 2's audit-trail projection, without full event sourcing) to satisfy regulatory traceability.

Scope note: this is open-source EOR *software*. The legal entity network, banking rails, and country-by-country legal review described in `research.md` are operational/business prerequisites that sit outside the codebase. The platform models them as data (`legal_entities`, partner records) and integrates with external payment/KYC providers via adapters, but does not itself provide the legal entities.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | **Python 3.12** | The differentiating value (`research.md` AI-native advantage; `features.md` AI-augmentation candidates) is ML/LLM-heavy: misclassification scoring, payroll anomaly detection, compliance forecasting, contract amendment generation. Python has the strongest ecosystem (pandas, scikit-learn, openai/anthropic SDKs) for this, and excellent numeric/decimal handling for payroll. |
| API framework | **FastAPI** | Generates OpenAPI 3.1 (required by `standards.md`) automatically, native Pydantic validation maps directly to JSON Schema 2020-12 (also required), async support for webhooks and long-running AI calls. |
| Money/decimal | **Python `decimal.Decimal` + `py-moneyed`** | Payroll errors carry legal risk (`research.md`). Never use floats for money. `NUMERIC` in Postgres maps to `Decimal`. |
| Database | **PostgreSQL 16** | Mandated by the hybrid data model: relational integrity + JSONB + GIN indexes in a single engine. RLS for multi-tenancy. `pgcrypto` for PII encryption. |
| ORM / query builder | **SQLAlchemy 2.0 (async) + Alembic** | Mature JSONB support, async engine for FastAPI, Alembic for version-controlled migrations. |
| Validation / schemas | **Pydantic v2 + `jsonschema`** | Pydantic for API models; `jsonschema` (Draft 2020-12) to validate jurisdiction-specific JSONB blobs against registered per-country schemas. |
| Task queue | **Celery + Redis** | Async workloads: payroll calculation batches, webhook delivery, compliance-feed polling, AI inference, e-signature callbacks. Redis doubles as cache + broker. |
| Object storage | **S3-compatible (boto3); MinIO in dev** | Contracts, payslips, ID documents, tax forms. Path stored in `documents.file_path`. |
| LLM / AI | **Provider-agnostic adapter over Anthropic + OpenAI SDKs** | Contract generation, amendment drafting, compliance impact summaries. Classical ML (scikit-learn) for anomaly detection and misclassification scoring where explainability and determinism matter. |
| Auth | **OAuth 2.0 (RFC 6749) + JWT (RFC 7519); SAML 2.0 for enterprise SSO** | Required by `standards.md`; matches Deel/Remote/Oyster auth patterns. `authlib` + `python-jose`. |
| Frontend | **Next.js 15 (React, TypeScript) + shadcn/ui + TanStack Query** | Dashboard-centric UX (`features.md`): payroll batches, compliance timeline, workforce metrics. Server components for data-heavy tables. Consumes the OpenAPI spec via generated client. |
| E-signature | **Adapter interface; DocuSign + a self-hosted PDF-signing fallback** | Contract signing workflow is table-stakes. |
| Containerisation | **Docker + docker-compose** | Hybrid deployment (`README.md`): self-hostable. Compose runs api, worker, postgres, redis, minio. |
| Testing | **pytest + pytest-asyncio + testcontainers; Playwright for frontend e2e** | testcontainers spins real Postgres/Redis for integration tests; Playwright for dashboard flows. |
| Code quality | **ruff (lint+format) + mypy (strict) + pre-commit** | Strict typing critical for money/compliance logic. |
| Package manager | **uv** (backend) / **pnpm** (frontend) | Fast, reproducible. |
| Currency / FX | **`exchangerate.host`/ECB feed adapter, cached daily** | Multi-currency payroll needs auditable, dated FX rates (stored on each `payroll_run`). |
| Observability | **OpenTelemetry + Prometheus/Grafana; structured JSON logs** | NIST CSF continuous monitoring (`standards.md`). |

### Project Structure

```
global-employment-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .pre-commit-config.yaml
├── README.md
├── migrations/                      # Alembic versioned migrations
│   └── versions/
├── jurisdictions/                   # Versioned per-country config (data, not code)
│   ├── _schema/                     # JSON Schema 2020-12 per jurisdiction-data shape
│   │   ├── base.schema.json
│   │   ├── DE.schema.json
│   │   └── BR.schema.json
│   ├── DE/
│   │   ├── tax_rules.yaml
│   │   ├── leave_entitlements.yaml
│   │   ├── minimum_wage.yaml
│   │   ├── benefits.yaml
│   │   └── compliance_rules.yaml
│   ├── BR/...
│   └── manifest.yaml                # which jurisdictions are active / eor_restricted
├── contract_templates/              # Jinja2 templates per jurisdiction + role
│   ├── DE/permanent.md.j2
│   └── ...
├── src/
│   └── gep/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── base.py              # async engine, session factory, RLS helpers
│       │   ├── models/              # SQLAlchemy ORM models
│       │   │   ├── organization.py
│       │   │   ├── employee.py
│       │   │   ├── contract.py
│       │   │   ├── jurisdiction.py
│       │   │   ├── payroll.py
│       │   │   ├── benefit.py
│       │   │   ├── document.py
│       │   │   └── audit.py
│       │   └── repositories/        # data-access layer (one per aggregate)
│       ├── schemas/                 # Pydantic request/response models
│       ├── api/
│       │   ├── deps.py              # auth, tenant, db-session dependencies
│       │   └── routers/            # one router per resource
│       ├── domain/                  # business logic (no framework deps)
│       │   ├── payroll/
│       │   │   ├── engine.py        # gross→net calculation pipeline
│       │   │   ├── tax.py           # bracket/flat/tiered evaluators
│       │   │   └── fx.py
│       │   ├── contracts/
│       │   │   ├── generator.py     # template + LLM contract assembly
│       │   │   └── amendments.py
│       │   ├── compliance/
│       │   │   ├── monitor.py       # rule-change detection + alerts
│       │   │   └── forecast.py      # predictive impact (Phase 9)
│       │   ├── classification/      # misclassification risk (Phase 8)
│       │   └── benefits/
│       ├── jurisdictions/
│       │   ├── loader.py            # loads YAML config + JSON Schema validators
│       │   └── registry.py          # in-memory jurisdiction config cache
│       ├── ai/
│       │   ├── client.py            # provider-agnostic LLM adapter
│       │   ├── anomaly.py           # payroll anomaly model wrapper
│       │   └── prompts/             # prompt templates
│       ├── integrations/
│       │   ├── base.py              # connector interface
│       │   ├── hris/                # Workday, BambooHR, HiBob
│       │   ├── ats/                 # Greenhouse, Lever
│       │   ├── accounting/          # Xero, QuickBooks
│       │   ├── esignature/          # DocuSign adapter
│       │   └── payments/            # disbursement adapter
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks/
│       ├── webhooks/
│       │   ├── inbound.py           # signed inbound webhook handler
│       │   └── outbound.py          # event delivery with retries
│       └── audit/
│           └── recorder.py          # append-only audit_events writer
├── tests/
│   ├── conftest.py                  # testcontainers fixtures, factory_boy
│   ├── fixtures/                    # sample diffs, payroll inputs, jurisdiction data
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── frontend/
    ├── package.json
    ├── app/                         # Next.js App Router
    ├── components/
    └── lib/api/                     # generated OpenAPI client
```

---

## Phase 1: Foundation & Project Skeleton

### Purpose
Establish the runnable scaffold: config, database connection, migrations, Docker, CI, auth primitives, and the multi-tenant request pipeline. Nothing domain-specific ships, but every later phase plugs into this skeleton without restructuring. After this phase a developer can run `docker compose up`, hit a health endpoint, and authenticate.

### Tasks

#### 1.1 — Project scaffold, tooling, and config

**What**: Create the repository structure, dependency management, linting/typing/CI, and environment-driven configuration.

**Design**:
- `pyproject.toml` with dependencies pinned via `uv`. Dev group: pytest, ruff, mypy, testcontainers, factory_boy.
- `src/gep/config.py` using Pydantic `BaseSettings`:
  ```python
  class Settings(BaseSettings):
      database_url: PostgresDsn
      redis_url: RedisDsn = "redis://localhost:6379/0"
      jwt_secret: SecretStr
      jwt_algorithm: str = "HS256"
      access_token_ttl_seconds: int = 3600
      s3_endpoint: str | None = None        # MinIO in dev
      s3_bucket: str = "gep-documents"
      pii_encryption_key: SecretStr         # for pgcrypto / app-layer envelope
      llm_provider: Literal["anthropic", "openai"] = "anthropic"
      llm_api_key: SecretStr | None = None
      environment: Literal["dev", "staging", "prod"] = "dev"
      model_config = SettingsConfigDict(env_prefix="GEP_", env_file=".env")
  ```
- `.pre-commit-config.yaml` runs ruff (lint+format) and mypy --strict.
- `Dockerfile` (multi-stage: builder installs deps, runtime slim image), `docker-compose.yml` with services: `api`, `worker`, `postgres:16`, `redis:7`, `minio`.
- GitHub Actions: lint → typecheck → test on PR.

**Testing**:
- `Unit: Settings loads from env with GEP_ prefix → fields populated, defaults applied`
- `Unit: missing required GEP_DATABASE_URL → ValidationError naming the field`
- `Integration: docker compose up → all 5 services reach healthy state`
- `CI: ruff + mypy --strict pass on empty skeleton`

#### 1.2 — Database engine, base models, and migrations

**What**: Async SQLAlchemy engine, declarative base with shared mixins, Alembic wired up.

**Design**:
- `src/gep/db/base.py`: async engine from `settings.database_url`, `async_sessionmaker`, `get_session()` dependency.
- Shared mixins:
  ```python
  class UUIDMixin:
      id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
  class TimestampMixin:
      created_at: Mapped[datetime] = mapped_column(server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
  class TenantMixin:
      organization_id: Mapped[UUID] = mapped_column(ForeignKey("organizations.id"), index=True)
  ```
- Alembic configured for async; `alembic revision --autogenerate` works against the models.
- Enable extensions in the first migration: `pgcrypto`, `uuid-ossp`.

**Testing**:
- `Integration (testcontainers): run all migrations up then down → no errors, schema empty after downgrade`
- `Integration: autogenerate after models defined → no spurious diff (models match migrations)`
- `Unit: TimestampMixin sets created_at/updated_at on insert/update`

#### 1.3 — Auth, RBAC, and multi-tenancy

**What**: OAuth2 password + client-credentials flows issuing JWTs, role-based scopes, and Postgres RLS tenant isolation.

**Design**:
- `users` and `organizations` tables (minimal here; org expands in Phase 2):
  ```sql
  CREATE TABLE users (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      organization_id UUID NOT NULL REFERENCES organizations(id),
      email TEXT NOT NULL UNIQUE,
      hashed_password TEXT NOT NULL,
      role VARCHAR(20) NOT NULL DEFAULT 'admin',
      -- role: admin, hr_manager, payroll_operator, employee, auditor, api
      created_at TIMESTAMPTZ NOT NULL DEFAULT now()
  );
  ```
- JWT claims: `sub` (user id), `org` (organization_id), `role`, `scope` (space-delimited), `exp`.
- `api/deps.py`: `get_current_user`, `require_scope("payroll:write")`, `get_tenant_id`.
- RLS: every tenant-scoped table gets a policy `USING (organization_id = current_setting('gep.tenant_id')::uuid)`. The DB session dependency runs `SET LOCAL gep.tenant_id = :org` per request.
- Role→scope matrix defined in `auth/scopes.py`.

**Testing**:
- `Unit: valid credentials → JWT with correct org/role/scope claims`
- `Unit: expired token → 401`
- `Integration: request with payroll_operator role hitting admin-only endpoint → 403`
- `Integration (real PG): user from org A queries employees, sees only org A rows (RLS enforced) → org B rows absent`
- `Integration: client-credentials flow for role=api → token with api scopes, no employee scope`

#### 1.4 — Audit recorder & request middleware

**What**: Append-only audit log capturing every state-changing action, plus structured logging/OTel middleware.

**Design**:
- `audit_events` table (from Suggestion 2's audit projection, used directly):
  ```sql
  CREATE TABLE audit_events (
      id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      entity_type   TEXT NOT NULL,      -- employee, contract, payroll_batch, ...
      entity_id     UUID NOT NULL,
      action        TEXT NOT NULL,      -- created, updated, approved, terminated
      actor_id      UUID,
      actor_type    TEXT NOT NULL,      -- user, system, cron, api
      organization_id UUID NOT NULL,
      before        JSONB,
      after         JSONB,
      correlation_id UUID,
      occurred_at   TIMESTAMPTZ NOT NULL DEFAULT now()
  );
  CREATE INDEX idx_audit_entity ON audit_events (entity_type, entity_id);
  CREATE INDEX idx_audit_org_time ON audit_events (organization_id, occurred_at);
  ```
- `audit/recorder.py`: `record(entity_type, entity_id, action, before, after)` reads actor/correlation from request context (contextvars). Append-only enforced by a DB trigger rejecting UPDATE/DELETE.
- Middleware injects a `correlation_id` per request, structured JSON logging, OTel span.

**Testing**:
- `Unit: record() writes a row with actor from context`
- `Integration: UPDATE on audit_events → DB error (immutable)`
- `Integration: a state-changing API call writes exactly one audit row with before/after diff`

---

## Phase 2: Jurisdiction Configuration Engine

### Purpose
The jurisdiction is the hub of the entire data model (per Suggestion 1's ER overview) — payroll, leave, benefits, and compliance all hang off it. This phase builds the system that loads versioned per-country configuration (tax rules, minimum wage, leave, benefits, compliance rules) from `jurisdictions/*.yaml`, validates jurisdiction-specific JSONB shapes against registered JSON Schemas, and exposes them through a cached registry. After this phase the platform can answer "what are the statutory rules for Germany effective on date X?" — the prerequisite for onboarding any employee.

### Tasks

#### 2.1 — Jurisdiction & regulatory rule tables (bi-temporal)

**What**: Tables for jurisdictions, legal entities, and the date-versioned regulatory rule sets.

**Design**: Adopt Suggestion 1's schema for `jurisdictions`, `legal_entities`, `tax_rules`, `tax_brackets`, `minimum_wage_rules`, `leave_entitlements`, `compliance_rules`, plus a JSON Schema reference column. All rule tables carry `effective_from`/`effective_to` (bi-temporal valid-time, from Suggestion 4). Key additions:
```sql
ALTER TABLE jurisdictions ADD COLUMN jurisdiction_schema_id TEXT;  -- key into jurisdictions/_schema
-- tax_rules.rate_type CHECK IN ('flat','progressive','tiered')
-- compliance_rules.rule_category CHECK IN (...notice_period, severance, pe_risk, ...)
```
Active-rule lookup helper: `effective_from <= :as_of AND (effective_to IS NULL OR effective_to > :as_of)`.

**Testing**:
- `Unit: active-rule query returns the rule whose date range covers as_of`
- `Unit: two overlapping tax_rules of same type for same jurisdiction → validation rejects on load`
- `Integration: insert DE jurisdiction + 2 historical minimum-wage rows → query as_of past date returns old wage, present date returns new`

#### 2.2 — Config loader & JSON Schema validation

**What**: Load `jurisdictions/<CC>/*.yaml` into the rule tables and validate JSONB extension shapes.

**Design**:
- `jurisdictions/loader.py`: parse YAML, validate each file against `jurisdictions/_schema/<CC>.schema.json` (Draft 2020-12 via `jsonschema`), upsert into rule tables inside a transaction. Idempotent (re-running same config = no-op).
- `jurisdictions/_schema/base.schema.json` defines the universal shape; per-country schemas `allOf`-extend it (e.g. `DE.schema.json` requires `church_tax`, `tax_class`; `BR.schema.json` requires `thirteenth_salary`, `fgts`).
- CLI: `gep jurisdictions load DE` and `gep jurisdictions load --all`.
- `manifest.yaml` lists active jurisdictions and `eor_restricted` flags (Germany/France/Italy noted in `research.md` build considerations).

**Testing**:
- `Unit: valid DE tax_rules.yaml → tax_rules + tax_brackets rows created`
- `Unit: BR config missing required thirteenth_salary → SchemaValidationError naming the path`
- `Unit: re-running loader on unchanged config → zero new rows`
- `Fixture-based: load committed DE + BR fixtures → registry reports both jurisdictions active`

#### 2.3 — Jurisdiction registry & API

**What**: In-memory cache of active jurisdiction config and read-only API endpoints.

**Design**:
- `jurisdictions/registry.py`: loads all active jurisdictions on startup, refreshable on config change; `get(country_code, as_of)` returns a resolved `JurisdictionConfig` dataclass (tax rules, wage, leave, benefits, compliance, JSONB schema).
- Endpoints (all `GET`, scope `jurisdiction:read`):
  - `GET /jurisdictions` → list with country_code, name, currency, eor_restricted
  - `GET /jurisdictions/{cc}` → full resolved config for `as_of` (default today)
  - `GET /jurisdictions/{cc}/minimum-wage?as_of=`
- Returns `409`/warning flag when `eor_restricted` is true.

**Testing**:
- `Unit: registry.get("DE", as_of) → resolved config with correct effective rules`
- `Integration: GET /jurisdictions/DE → 200 with tax+wage+leave; eor_restricted flag present`
- `Integration: GET /jurisdictions/ZZ (unknown) → 404`

---

## Phase 3: Organizations, Employees & Contracts (Core Domain)

### Purpose
The heart of the product. This phase delivers the central entities — client organizations, the employees they hire, and the locally compliant employment contracts that bind them to an EOR legal entity. It uses the hybrid model: typed columns for universal fields, a validated `jurisdiction_data` JSONB column for country-specific fields. After this phase the platform can onboard an employee and produce a draft contract record (generation/signature follow in Phase 4).

### Tasks

#### 3.1 — Organization & employee models with PII encryption

**What**: `organizations` and `employees` tables with encrypted PII and a JSONB extension column.

**Design**: Suggestion 1's `organizations`/`employees` schema, extended hybrid-style (Suggestion 3):
```sql
ALTER TABLE employees ADD COLUMN jurisdiction_data JSONB NOT NULL DEFAULT '{}';
CREATE INDEX idx_emp_jurisdiction_data ON employees USING GIN (jurisdiction_data);
-- tax_id, bank_details stored encrypted: pgp_sym_encrypt at write, app-layer envelope key
-- status CHECK IN ('onboarding','active','on_leave','offboarding','terminated')
```
- Repository layer encrypts `tax_id`/`bank_details` via `pgcrypto` keyed by `settings.pii_encryption_key`; decryption only for `payroll_operator`/`admin` scopes.
- On create, `jurisdiction_data` validated against the employee's jurisdiction JSON Schema.
- GDPR/LGPD support hooks: `DELETE /employees/{id}` performs crypto-shred + tombstone (data-subject erasure), audited.

**Testing**:
- `Unit: create employee with DE jurisdiction_data → validates, tax_id stored encrypted (raw col not plaintext)`
- `Unit: employee with jurisdiction_data violating schema → 422`
- `Integration: payroll_operator reads tax_id → decrypted; hr_manager → masked`
- `Integration: erasure request → PII fields nulled/crypto-shredded, audit_event recorded`

#### 3.2 — Contract, compensation & amendment models

**What**: Contract lifecycle tables with versioning and a compensation package.

**Design**: Suggestion 1's `contracts`, `contract_amendments`, `compensation_packages`, plus `jurisdiction_data JSONB` on contracts. Status state machine:
```
draft → pending_signature → active → amended → terminated
                         ↘ (expired for fixed_term)
```
- Repository enforces legal transitions; illegal transition → `ConflictError`.
- `contract_type CHECK IN ('permanent','fixed_term','part_time','project','fractional')` (hybrid-employment opportunity from `features.md`).
- Amendments write `old_values`/`new_values` JSONB snapshots and bump `contracts.version`.

**Testing**:
- `Unit: draft → active without signature → ConflictError`
- `Unit: amend active contract → version increments, amendment row with before/after snapshot`
- `Integration: create contract referencing eor_restricted jurisdiction → 200 with restriction warning in payload`

#### 3.3 — Onboarding workflow & employee/contract API

**What**: REST endpoints and a guided onboarding flow assembling employee + contract + document checklist.

**Design**:
- Endpoints (scopes noted):
  - `POST /employees` (`employee:write`) → 201
  - `GET /employees?status=&jurisdiction=` (`employee:read`, paginated, RFC 8288 `Link` headers)
  - `POST /contracts` (`contract:write`)
  - `POST /contracts/{id}/amendments` (`contract:write`)
  - `POST /onboarding` (`employee:write`) → orchestrates employee creation, contract draft, and a jurisdiction-specific document checklist (id_verification, right_to_work, tax_form) returned as `required_documents[]`.
- Pagination: cursor-based; OpenAPI 3.1 auto-doc with examples.

**Testing**:
- `Integration: POST /onboarding for DE → employee created, draft contract, DE document checklist returned`
- `Integration: POST /employees missing required jurisdiction field → 422 with field path`
- `E2E: onboard → list employees → fetch contract → all consistent, audit trail has 3 events`

---

## Phase 4: Contract Generation, Documents & E-Signature

### Purpose
Turn contract *records* into compliant, signed *documents* — a table-stakes feature (`features.md`). Combines Jinja2 jurisdiction templates with an LLM assist for clause localisation, renders to PDF, stores in object storage, and drives an e-signature workflow via a pluggable adapter. After this phase a contract can go from draft to legally signed and stored with version control.

### Tasks

#### 4.1 — Document store & model

**What**: `documents` table + S3-backed storage service.

**Design**: Suggestion 1's `documents` schema. `integrations`-agnostic `DocumentStore` interface: `put(bytes, key) -> path`, `get(path) -> bytes`, `presigned_url(path, ttl)`. Backed by boto3 (MinIO in dev). Documents linked to employee and/or contract; `document_type` enum as in Suggestion 1.

**Testing**:
- `Integration (MinIO): put then get → bytes round-trip; presigned_url downloads`
- `Unit: document row links to contract; audit event on upload`

#### 4.2 — Contract generator (template + LLM)

**What**: Render a jurisdiction- and role-specific contract from template + contract data, with LLM localisation of variable clauses.

**Design**:
- `domain/contracts/generator.py`: loads `contract_templates/<CC>/<contract_type>.md.j2`, fills with contract + compensation + jurisdiction config, renders Markdown → PDF (WeasyPrint).
- LLM step (`ai/prompts/contract_clause.txt`) only fills *flagged* `{{ ai_clause:... }}` slots (e.g., a bespoke role-scope clause), constrained by jurisdiction compliance rules passed in context. Deterministic fields never go through the LLM.
  - System prompt skeleton: *"You are a legal drafting assistant for an Employer of Record. Draft ONLY the requested clause for {country}, consistent with these statutory rules: {compliance_rules}. Do not invent obligations. Output plain text, no preamble."*
- Output stored as a `contract` document; SHA-256 hash recorded for tamper-evidence.

**Testing**:
- `Unit (mocked LLM): generate DE permanent contract → PDF contains job title, salary, statutory notice period from config`
- `Unit: template references unknown jurisdiction → TemplateNotFound error`
- `Unit: deterministic fields identical across two runs (only ai_clause slot may differ)`
- `Fixture: rendered contract hash stored and stable for fixed inputs (mocked LLM)`

#### 4.3 — E-signature workflow

**What**: Pluggable e-signature adapter with status-tracking and webhook callback.

**Design**:
- `integrations/esignature/base.py`: `send(document, signers) -> envelope_id`, `status(envelope_id)`. DocuSign adapter + a self-hosted fallback (PAdES PDF signature).
- Contract transitions to `pending_signature` on send; inbound signature webhook (Phase 6 infra) flips employee/employer signed flags, then `active` once both signed; writes `signed_at` and `signature_id`.
- States mirror Suggestion 2's `ContractSentForSignature` / `SignedByEmployee` / `SignedByEmployer` / `Activated`.

**Testing**:
- `Integration (mocked DocuSign): send → envelope_id stored, contract pending_signature`
- `Integration: signature-complete webhook (valid sig) → contract active, both signed_at set`
- `Integration: webhook with bad signature → 401, no state change`

---

## Phase 5: Payroll Engine

### Purpose
The highest-risk, highest-value subsystem. Calculates multi-currency gross→net payroll with correct statutory deductions and employer costs, pulling rules from the jurisdiction registry, applying dated FX rates, and producing reviewable, approvable payroll batches with full line-item breakdowns. Payroll errors carry legal risk (`research.md`), so the engine is deterministic, `Decimal`-based, and fully audited. AI anomaly review is layered on in Phase 8.

### Tasks

#### 5.1 — Payroll tables & FX service

**What**: `payroll_batches`, `payroll_runs`, `payroll_line_items` plus a dated FX-rate service.

**Design**: Suggestion 1's payroll schema verbatim (range-partition `payroll_runs`/`payroll_line_items` by `pay_period_start`). FX service (`domain/payroll/fx.py`): fetches daily rates from ECB/exchangerate.host, caches in Redis + persists `(date, base, quote, rate)`; each `payroll_run` records `exchange_rate` and `exchange_rate_date` for auditability.

**Testing**:
- `Unit: FX lookup for a date returns persisted rate; missing date → fetch + store`
- `Integration: payroll_runs partition created for new pay period`

#### 5.2 — Tax & deduction calculators

**What**: Pure functions evaluating flat/progressive/tiered tax and statutory deductions for a jurisdiction.

**Design**: `domain/payroll/tax.py`:
```python
def apply_progressive(gross: Decimal, brackets: list[TaxBracket]) -> Decimal: ...
def apply_flat(gross: Decimal, rate: Decimal) -> Decimal: ...
def evaluate_jurisdiction(gross: Decimal, jc: JurisdictionConfig, emp_data: dict) -> list[LineItem]: ...
```
- `evaluate_jurisdiction` produces employee deductions (income tax, social security, pension, health) and employer costs (employer social security/pension/health) as `LineItem`s, reading rule rates effective on the pay period. Jurisdiction-specific items (DE church tax, BR 13th salary/FGTS, India PF/ESI) driven by `emp_data` JSONB + the country's rule set — no hardcoded country logic.
- All arithmetic `Decimal`, rounded per jurisdiction rounding rule (config field).

**Testing**:
- `Unit: progressive brackets, known DE gross → expected income tax (golden value)`
- `Unit: BR gross with thirteenth_salary flag → 13th-month line item present`
- `Unit: rounding rule applied per jurisdiction; sum(deductions)+net == gross`
- `Unit: zero gross → no negative line items`

#### 5.3 — Payroll run orchestration, review & approval

**What**: Build a batch for a legal entity/period, calculate every employee, surface totals, and gate submission behind approval.

**Design**:
- Celery task `calculate_batch(batch_id)`: for each active contract in the entity/period, run `evaluate_jurisdiction`, persist `payroll_run` + line items, roll up batch totals. Batch state machine: `draft → calculating → review → approved → submitted → paid` (plus `failed`).
- `POST /payroll/batches`, `POST /payroll/batches/{id}/calculate`, `GET /payroll/batches/{id}` (totals + per-employee breakdown), `POST /payroll/batches/{id}/approve` (`payroll:approve` scope; records `approved_by`/`approved_at`). Approval is a hard gate — only `approved` batches can be submitted/paid.

**Testing**:
- `Integration: calculate batch of 3 DE employees → 3 runs, line items, batch totals correct`
- `Integration: approve without payroll:approve scope → 403`
- `Integration: attempt submit on un-approved batch → ConflictError`
- `E2E: create batch → calculate → review totals → approve → status=approved, audit trail complete`

---

## Phase 6: Integrations & Webhooks Infrastructure

### Purpose
EOR platforms must avoid data silos (`features.md` table-stakes; `standards.md` API patterns). This phase builds the bidirectional integration substrate: a signed inbound-webhook handler (used already by e-signature in Phase 4), a reliable outbound event-delivery system, and the first concrete connectors (HRIS, ATS, accounting). After this phase the platform syncs with the client's existing HR stack.

### Tasks

#### 6.1 — Inbound webhook handler & outbound delivery

**What**: HMAC-verified inbound webhook intake and a retrying outbound webhook dispatcher.

**Design**:
- `webhooks/inbound.py`: `POST /webhooks/{provider}` verifies provider HMAC signature, enqueues a Celery task, returns 200 fast. Replay protection via timestamp + nonce.
- `webhooks/outbound.py`: `webhook_endpoints` table (org-scoped url + secret + subscribed event types). On domain events (contract.activated, payroll.approved, compliance.alert), enqueue delivery with exponential backoff (max retries, then dead-letter). Payloads signed with per-endpoint secret.

**Testing**:
- `Integration: inbound with valid HMAC → task enqueued, 200; invalid → 401`
- `Integration: outbound delivery fails twice then succeeds → 3 attempts, marked delivered`
- `Integration: persistent failure → dead-letter row + alert`

#### 6.2 — Connector framework & HRIS/ATS/accounting adapters

**What**: A uniform connector interface and the first concrete integrations.

**Design**:
- `integrations/base.py`:
  ```python
  class Connector(Protocol):
      provider: str
      def authenticate(self, creds: dict) -> None: ...
      def sync_employees(self) -> list[EmployeeSync]: ...     # HRIS inbound
      def push_employee(self, e: Employee) -> None: ...
      def export_payroll(self, batch: PayrollBatch) -> None:  # accounting outbound
  ```
- Concrete: `hris/bamboohr.py`, `hris/hibob.py`, `hris/workday.py`, `ats/greenhouse.py`, `ats/lever.py`, `accounting/xero.py`, `accounting/quickbooks.py`. OAuth2 (RFC 6749) per provider; tokens stored encrypted.
- Field mapping config per provider; conflict policy = "BambooHR as system of truth for PTO" pattern (`features.md`).
- `POST /integrations/{provider}/connect`, `POST /integrations/{provider}/sync`.

**Testing**:
- `Integration (mocked BambooHR API): sync_employees → employees upserted, field mapping correct`
- `Integration (mocked Xero): export approved payroll batch → journal entries pushed`
- `Unit: connector with expired OAuth token → refresh attempted`
- `Integration: ATS (Greenhouse) hired candidate → onboarding draft created`

---

## Phase 7: Benefits, Leave, Onboarding/Offboarding Completion

### Purpose
Round out the employment lifecycle: statutory and supplemental benefits enrolment per jurisdiction, leave/time-off management synced to payroll, and compliant offboarding with severance and final pay. Completes the table-stakes feature set so the platform covers hire-to-terminate.

### Tasks

#### 7.1 — Benefits administration

**What**: Benefit plan catalogue, enrolment, and auto-enrolment in statutory benefits.

**Design**: Suggestion 1's `benefit_plans` + `benefit_enrollments`. On contract activation, a service auto-enrols the employee in all `is_statutory=true` plans for their jurisdiction. Supplemental plans selectable via API. Costs feed payroll line items (employee/employer split).
- `GET /jurisdictions/{cc}/benefits`, `POST /employees/{id}/benefits` (enrol), `DELETE .../benefits/{enrollment_id}`.

**Testing**:
- `Integration: activate DE contract → auto-enrolled in statutory health + pension`
- `Unit: supplemental enrolment adds employer/employee cost line items to next payroll`
- `Integration: enrol in non-existent plan → 404`

#### 7.2 — Leave & time-off management

**What**: Leave balances per jurisdiction entitlement, requests, and payroll sync.

**Design**: `leave_balances` and `leave_requests` tables. Accrual from `leave_entitlements` (immediate/monthly/annual). Approved unpaid leave reduces payroll gross for the period; paid leave at configured `pay_rate`. Optional inbound sync from HRIS (BambooHR PTO → leave).

**Testing**:
- `Unit: monthly accrual adds correct days after a month`
- `Unit: unpaid leave days reduce gross in payroll calc`
- `Integration: leave request exceeding balance → 422`

#### 7.3 — Offboarding, severance & final pay

**What**: Compliant termination workflow computing notice period, severance, and final pay.

**Design**: `domain/contracts/amendments.py` extended with offboarding service. Reads jurisdiction `compliance_rules` (notice_period, severance) to compute statutory notice + severance (e.g., tenure-based formula from config), generates a `termination_letter` document, schedules a final `payroll_run` including accrued-but-unused leave payout. Contract → `terminated`.

**Testing**:
- `Unit: severance for 5-yr DE tenure → matches statutory formula from config`
- `Integration: offboard → termination letter generated, final payroll run scheduled, contract terminated`
- `Unit: notice period below jurisdiction minimum → validation error`

---

## Phase 8: AI Layer — Misclassification & Payroll Anomaly Detection

### Purpose
Deliver the first wave of the AI-native advantage (`README.md`, `features.md` AI-augmentation candidates). Two high-value, explainable AI features: misclassification risk scoring for contractor relationships, and payroll anomaly detection that flags errors before submission. These differentiate the platform without requiring the legal-entity scale of incumbents.

### Tasks

#### 8.1 — Misclassification risk assessment

**What**: A questionnaire-driven, jurisdiction-aware risk model scoring contractor misclassification probability with a conversion recommendation.

**Design**: Tables/flow from Suggestion 2's MisclassificationAssessment aggregate (modelled relationally here):
```sql
CREATE TABLE misclassification_assessments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    worker_id UUID, jurisdiction_id UUID NOT NULL,
    control_score NUMERIC(4,2), integration_score NUMERIC(4,2),
    economic_dependency_score NUMERIC(4,2),
    overall_risk VARCHAR(10),       -- low | medium | high
    factors JSONB, recommendation TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- `domain/classification/`: jurisdiction-weighted scoring (EU/UK/Brazil weight control + economic dependency more heavily, per `research.md`). A transparent weighted-rubric model (not a black box) plus an LLM-generated plain-language rationale. Outputs `overall_risk` + recommended conversion pathway with estimated cost delta.
- `POST /assessments` (answers) → score; `GET /assessments/{id}`.

**Testing**:
- `Unit: high control + high economic dependency (UK) → high risk`
- `Unit: independent multi-client contractor → low risk`
- `Integration (mocked LLM): assessment returns score + rationale + conversion pathway`

#### 8.2 — Payroll anomaly detection

**What**: ML/statistical review of a calculated payroll batch flagging outliers before approval.

**Design**: `ai/anomaly.py`: per-employee features (gross vs trailing average, deduction ratio vs jurisdiction norm, net change %, new line-item types). Model = IsolationForest + rule thresholds (explainable). Runs automatically after `calculate_batch`, writing flags:
```python
@dataclass
class PayrollAnomaly:
    employee_id: UUID; anomaly_type: str; severity: Literal["low","medium","high"]
    description: str; suggested_action: str
```
Flags surface on the batch review screen; high-severity flags block approval until acknowledged.

**Testing**:
- `Unit: employee gross 3x trailing average → high-severity anomaly`
- `Unit: stable payroll → no anomalies`
- `Integration: calculate batch with one outlier → anomaly attached, approval requires acknowledgement`

---

## Phase 9: Compliance Monitoring & Predictive Alerts

### Purpose
Deliver the platform's headline differentiator (`README.md`): proactive, predictive compliance rather than reactive alerts. Monitors regulatory feeds, detects when changes affect active contracts, forecasts financial impact 60+ days ahead, and auto-drafts contract amendments. This is the AI-native compliance engine the research identifies as the key opportunity.

### Tasks

#### 9.1 — Compliance change detection & impact analysis

**What**: Detect new/superseded jurisdiction rules and compute their concrete impact on the tenant's workforce.

**Design**: `domain/compliance/monitor.py`. A scheduled Celery beat task ingests rule updates (new rows in `tax_rules`/`minimum_wage_rules`/`compliance_rules` loaded from feeds or manual review). On a detected change, an analyzer finds affected contracts and computes financial impact:
```python
@dataclass
class ComplianceImpact:
    jurisdiction: str; change_type: str; effective_date: date
    affected_contracts: int
    cost_delta: Decimal; cost_delta_pct: Decimal   # e.g. "+2.3% labour cost in FR"
    required_action: Literal["none","amendment","payroll_adjustment","notification"]
```
- Example: `MinimumWageUpdated` → find contracts below new minimum, compute uplift cost, flag for amendment. Writes a `compliance_alerts` row and emits a `compliance.alert` outbound webhook.
- `GET /compliance/alerts`, `GET /compliance/timeline?jurisdiction=` (the read-projection from Suggestion 2).

**Testing**:
- `Unit: minimum wage rise → contracts below new min identified, cost delta computed`
- `Integration: new tax rule effective in 45 days → alert created with impact, webhook emitted`
- `Unit: change affecting zero contracts → no alert`

#### 9.2 — Predictive forecasting & amendment auto-drafting

**What**: Forecast upcoming regulatory impact and auto-generate compliant contract amendments.

**Design**:
- `domain/compliance/forecast.py`: projects known future-dated rules (effective_from in the future) into a 60/90-day labour-cost forecast per jurisdiction; surfaces "this change will affect payroll in N days" alerts ahead of effective date.
- Amendment auto-draft: when `required_action == "amendment"`, the contract generator (Phase 4) produces a draft `ContractAmendment` with the changed fields (e.g., new minimum-compliant salary) and an LLM-written change summary, left in `draft` for human approval — never auto-applied.
- `POST /compliance/alerts/{id}/draft-amendment` → creates draft amendment(s).

**Testing**:
- `Unit: future-dated minimum wage → appears in 60-day forecast before effective date`
- `Integration (mocked LLM): draft-amendment for sub-minimum contracts → draft amendments created, not auto-activated`
- `Integration: 90-day forecast aggregates cost delta across multiple pending changes`

---

## Phase 10: Analytics Dashboard & Frontend

### Purpose
Surface everything through a dashboard-centric UX (`features.md`): real-time spend analytics by country/currency/contract type, payroll batch visibility, compliance timeline, and workforce metrics aligned to ISO 30414 human-capital reporting (`standards.md`). After this phase the platform is usable by non-developers end-to-end.

### Tasks

#### 10.1 — Analytics API & read projections

**What**: Aggregation endpoints for spend, headcount, and compliance, backed by read-optimised views.

**Design**: Materialised views / read tables (Suggestion 2's `read_payroll_summary`, `read_compliance_timeline`) refreshed after batch approval and rule changes.
- `GET /analytics/spend?group_by=country|currency|contract_type&period=` → labour cost, employer cost, headcount.
- `GET /analytics/workforce` → ISO 30414 metrics: headcount, turnover, cost-per-head by jurisdiction.
- `GET /analytics/compliance` → open alerts, upcoming changes, forecast cost delta.

**Testing**:
- `Integration: spend grouped by country → totals reconcile with sum of approved payroll runs`
- `Unit: turnover metric = terminations / avg headcount over period`
- `Integration: analytics scoped to tenant via RLS (org A cannot see org B spend)`

#### 10.2 — Next.js dashboard

**What**: The web UI consuming the OpenAPI client.

**Design**: Next.js App Router + shadcn/ui + TanStack Query. Generated typed API client from the OpenAPI 3.1 spec. Screens: Dashboard (spend + compliance overview), Employees, Contracts (with signature status), Payroll (batch list → review with anomaly flags → approve), Compliance Timeline, Jurisdictions, Integrations, Settings/SSO. Auth via OAuth2 + JWT; SAML SSO for enterprise.

**Testing**:
- `E2E (Playwright): log in → onboard employee → create + calculate payroll batch → review anomalies → approve`
- `E2E: compliance timeline shows alert created in Phase 9`
- `Component: payroll review table renders line items + anomaly badges`
- `Accessibility: axe scan on key screens → no critical violations`

---

## Phase 11: Hardening, Compliance & Self-Host Packaging

### Purpose
Make the platform production- and audit-ready: security per OWASP/NIST/ISO 27001, data-protection workflows for GDPR/CCPA/LGPD, performance scaling, and a clean self-hosted deployment artefact (the open-source, self-hostable promise in `README.md`).

### Tasks

#### 11.1 — Security & data-protection hardening

**What**: OWASP Top 10 mitigations, encryption-at-rest verification, and data-subject-rights workflows.

**Design**: Rate limiting, input validation audit, dependency scanning (CI), secret management, TLS enforcement. Data-protection endpoints: `GET /data-subject/{employee_id}/export` (GDPR portability, machine-readable JSON), `POST /data-subject/{employee_id}/erase` (crypto-shred + tombstone, statutory-retention-aware — payroll/tax records retained per jurisdiction retention period). Breach-notification hook (72h GDPR). Map controls to ISO 27001 Annex A in `docs/compliance/`.

**Testing**:
- `Security: OWASP A01 (access control) — cross-tenant access attempts blocked (RLS + scope)`
- `Security: A03 injection — parameterised queries everywhere, fuzz key endpoints`
- `Integration: data export returns all PII for subject in portable JSON`
- `Integration: erase preserves statutory-retained payroll records, removes erasable PII`

#### 11.2 — Performance, scaling & deployment

**What**: Load handling for payroll cycles, read replicas/partitioning, and packaged self-host deployment.

**Design**: Partition validation on `payroll_runs`/`payroll_line_items`; PgBouncer pooling; route analytics to read replica; Redis caching of jurisdiction registry. Helm chart + hardened docker-compose for self-host; `gep init` bootstrap CLI (run migrations, load jurisdictions, create first admin). Backups: WAL archiving guidance.

**Testing**:
- `Load: payroll batch of 5,000 employees → completes within target, partitions used`
- `Integration: read replica serves analytics; primary handles writes`
- `E2E: fresh self-host (compose up + gep init) → reach functional dashboard with DE+BR loaded`
- `CI: Docker build succeeds; Helm chart lints`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Skeleton          ─── required by everything
    │
Phase 2: Jurisdiction Config Engine     ─── requires 1
    │
Phase 3: Orgs / Employees / Contracts   ─── requires 1, 2  (core value)
    │
    ├── Phase 4: Contract Gen & E-Sign   ─── requires 3        ┐ can parallel
    ├── Phase 5: Payroll Engine          ─── requires 2, 3     ┘ (4 ∥ 5)
    │
    ├── Phase 6: Integrations & Webhooks ─── requires 3 (e-sign in 4 uses its inbound infra)
    ├── Phase 7: Benefits/Leave/Offboard ─── requires 3, 5
    │
    ├── Phase 8: AI (classification +    ─── requires 5 (anomaly), 3 (classification)
    │            payroll anomaly)             8 ∥ 9 after their deps
    └── Phase 9: Compliance Monitoring   ─── requires 2, 3, 4 (amendment draft), 6 (webhooks)
         │
Phase 10: Analytics & Frontend          ─── requires 5, 9 (and surfaces 8)
    │
Phase 11: Hardening & Self-Host         ─── requires all
```

Parallelism opportunities:
- **Phases 4 and 5** can be built concurrently once Phase 3 lands (different domains: documents vs payroll). Note Phase 4's signature callback depends on Phase 6's inbound webhook infra — sequence the *callback* after 6, or stub it.
- **Phases 6 and 7** can proceed in parallel after Phase 5.
- **Phases 8 and 9** can be built concurrently once their respective dependencies (5 for anomaly; 2/3/4/6 for compliance) are met.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks implemented as specified.
2. All unit and integration tests pass (`pytest`); frontend e2e pass where applicable (`playwright`).
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict` passes (zero type errors).
5. Alembic migration(s) created, and `upgrade`→`downgrade`→`upgrade` round-trips cleanly with no autogenerate drift.
6. Docker build succeeds; `docker compose up` reaches healthy state.
7. New/changed endpoints appear correctly in the auto-generated OpenAPI 3.1 spec with examples.
8. New config options documented in README and present in `.env.example`.
9. All state-changing operations write an `audit_events` row; RLS verified for any new tenant-scoped table.
10. Money handling uses `Decimal` end-to-end (no floats); payroll-touching code has golden-value tests.
11. Feature demonstrated end-to-end against a real Postgres/Redis (testcontainers) — not only mocks.
</content>
</invoke>
