# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Global Employment Platform (EOR) · Candidate #420

## Summary

A fully normalized relational data model using PostgreSQL as the primary store. Every entity is decomposed into well-defined tables with strict foreign key relationships, enforced data types, and referential integrity. Jurisdiction-specific rules, tax tables, and benefit catalogues are stored as structured configuration data in dedicated tables rather than embedded in application code. Multi-tenancy is achieved via a shared database with `tenant_id` (client organization) on all tenant-scoped tables, enforced by Row-Level Security (RLS).

This approach prioritises **data integrity, auditability, and regulatory compliance** above all else — the exact qualities demanded by a platform that processes payroll, holds employment contracts, and files statutory reports across 80+ jurisdictions.

---

## Key Entities and Relationships

### Entity-Relationship Overview

```
Organization (client company)
  └── has many → Employees
  └── has many → Contracts
  └── belongs to → Subscription/Plan

Employee
  └── has one → Contract (active)
  └── has many → Contracts (historical)
  └── has many → PayrollRuns (via Contract)
  └── has many → BenefitEnrollments
  └── has many → Documents
  └── belongs to → Jurisdiction

Contract
  └── belongs to → Employee
  └── belongs to → Organization
  └── belongs to → LegalEntity (EOR entity in-country)
  └── belongs to → Jurisdiction
  └── has one → CompensationPackage
  └── has many → ContractAmendments
  └── has many → ContractVersions

LegalEntity (EOR-owned entities per country)
  └── belongs to → Jurisdiction
  └── has many → Contracts
  └── has many → PayrollRuns

Jurisdiction
  └── has many → TaxRules
  └── has many → StatutoryBenefits
  └── has many → LeaveEntitlements
  └── has many → ComplianceRules
  └── has many → MinimumWageRules

PayrollRun
  └── belongs to → Contract
  └── belongs to → PayrollBatch
  └── has many → PayrollLineItems
  └── has many → TaxWithholdings
  └── has many → StatutoryDeductions

BenefitPlan
  └── belongs to → Jurisdiction
  └── has many → BenefitEnrollments
  └── has one → BenefitProvider
```

### Core Schema Snippets

#### Organizations and Employees

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    legal_name      TEXT NOT NULL,
    billing_country VARCHAR(2) NOT NULL,
    currency        VARCHAR(3) NOT NULL,
    plan_tier       VARCHAR(20) NOT NULL DEFAULT 'standard',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE employees (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT NOT NULL,
    date_of_birth   DATE,
    nationality     VARCHAR(2),
    tax_id          TEXT,  -- encrypted at rest
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'onboarding',
    -- status: onboarding, active, on_leave, offboarding, terminated
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Jurisdictions and Legal Entities

```sql
CREATE TABLE jurisdictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    VARCHAR(2) NOT NULL UNIQUE,
    country_name    TEXT NOT NULL,
    region          TEXT,  -- e.g. state/province for US, CA, AU
    currency        VARCHAR(3) NOT NULL,
    timezone        TEXT NOT NULL,
    eor_restricted  BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT
);

CREATE TABLE legal_entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    entity_name     TEXT NOT NULL,
    entity_type     VARCHAR(30) NOT NULL,  -- owned, partner
    registration_no TEXT,
    address         TEXT,
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Contracts and Amendments

```sql
CREATE TABLE contracts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    legal_entity_id     UUID NOT NULL REFERENCES legal_entities(id),
    jurisdiction_id     UUID NOT NULL REFERENCES jurisdictions(id),
    contract_type       VARCHAR(20) NOT NULL,  -- permanent, fixed_term, part_time
    job_title           TEXT NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE,  -- NULL for permanent
    probation_end_date  DATE,
    notice_period_days  INT,
    work_hours_per_week NUMERIC(4,1),
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    -- status: draft, pending_signature, active, amended, terminated
    version             INT NOT NULL DEFAULT 1,
    signed_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE contract_amendments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES contracts(id),
    amendment_type  VARCHAR(30) NOT NULL,
    -- types: salary_change, role_change, regulatory_update, benefit_change
    reason          TEXT NOT NULL,
    effective_date  DATE NOT NULL,
    old_values      JSONB,  -- snapshot of changed fields before
    new_values      JSONB,  -- snapshot of changed fields after
    approved_by     UUID,
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compensation_packages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES contracts(id),
    base_salary     NUMERIC(12,2) NOT NULL,
    salary_currency VARCHAR(3) NOT NULL,
    salary_period   VARCHAR(10) NOT NULL DEFAULT 'annual',
    -- period: annual, monthly
    bonus_structure TEXT,
    equity_grants   TEXT,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Payroll

```sql
CREATE TABLE payroll_batches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    legal_entity_id UUID NOT NULL REFERENCES legal_entities(id),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    pay_period_start DATE NOT NULL,
    pay_period_end  DATE NOT NULL,
    payment_date    DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    -- status: draft, calculating, review, approved, submitted, paid, failed
    currency        VARCHAR(3) NOT NULL,
    total_gross     NUMERIC(14,2),
    total_net       NUMERIC(14,2),
    total_employer_cost NUMERIC(14,2),
    approved_by     UUID,
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payroll_runs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payroll_batch_id    UUID NOT NULL REFERENCES payroll_batches(id),
    contract_id         UUID NOT NULL REFERENCES contracts(id),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    gross_salary        NUMERIC(12,2) NOT NULL,
    net_salary          NUMERIC(12,2) NOT NULL,
    employer_cost       NUMERIC(12,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL,
    exchange_rate       NUMERIC(12,6),
    exchange_rate_date  DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'calculated',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payroll_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payroll_run_id  UUID NOT NULL REFERENCES payroll_runs(id),
    line_type       VARCHAR(30) NOT NULL,
    -- types: base_salary, overtime, bonus, commission, allowance,
    --        income_tax, social_security, pension, health_insurance,
    --        employer_social_security, employer_pension, employer_health
    description     TEXT NOT NULL,
    amount          NUMERIC(12,2) NOT NULL,
    is_deduction    BOOLEAN NOT NULL DEFAULT false,
    is_employer_cost BOOLEAN NOT NULL DEFAULT false,
    tax_rule_id     UUID REFERENCES tax_rules(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Jurisdiction-Specific Tax and Compliance Rules

```sql
CREATE TABLE tax_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    tax_type        VARCHAR(30) NOT NULL,
    -- types: income_tax, social_security_employee, social_security_employer,
    --        pension_employee, pension_employer, health_insurance, solidarity_tax
    rate_type       VARCHAR(10) NOT NULL,  -- flat, progressive, tiered
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tax_brackets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tax_rule_id     UUID NOT NULL REFERENCES tax_rules(id),
    lower_bound     NUMERIC(14,2) NOT NULL,
    upper_bound     NUMERIC(14,2),  -- NULL for top bracket
    rate            NUMERIC(6,4) NOT NULL,  -- e.g. 0.3200 = 32%
    fixed_amount    NUMERIC(12,2) DEFAULT 0
);

CREATE TABLE minimum_wage_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    wage_type       VARCHAR(20) NOT NULL,  -- hourly, monthly, annual
    amount          NUMERIC(12,2) NOT NULL,
    currency        VARCHAR(3) NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    notes           TEXT
);

CREATE TABLE leave_entitlements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    leave_type      VARCHAR(30) NOT NULL,
    -- types: annual, sick, maternity, paternity, parental, bereavement, public_holiday
    min_days        NUMERIC(5,1) NOT NULL,
    accrual_method  VARCHAR(20),  -- immediate, monthly, annual
    paid            BOOLEAN NOT NULL DEFAULT true,
    pay_rate        NUMERIC(5,2) DEFAULT 1.00,  -- 1.00 = 100% pay
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    notes           TEXT
);

CREATE TABLE compliance_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    rule_category   VARCHAR(30) NOT NULL,
    -- categories: notice_period, severance, probation, working_hours,
    --             overtime, data_privacy, pe_risk, collective_bargaining
    rule_key        TEXT NOT NULL,
    rule_value      TEXT NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    source_law      TEXT,
    notes           TEXT
);
```

#### Benefits

```sql
CREATE TABLE benefit_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    plan_name       TEXT NOT NULL,
    plan_type       VARCHAR(30) NOT NULL,
    -- types: health_insurance, pension, life_insurance, dental, vision,
    --        wellness, meal_voucher, transport
    is_statutory    BOOLEAN NOT NULL DEFAULT false,
    provider_name   TEXT,
    monthly_cost_employee NUMERIC(10,2),
    monthly_cost_employer NUMERIC(10,2),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE benefit_enrollments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id),
    benefit_plan_id UUID NOT NULL REFERENCES benefit_plans(id),
    enrollment_date DATE NOT NULL,
    termination_date DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    dependents_count INT DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

#### Document Management

```sql
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID REFERENCES employees(id),
    contract_id     UUID REFERENCES contracts(id),
    document_type   VARCHAR(30) NOT NULL,
    -- types: contract, amendment, id_verification, right_to_work,
    --        tax_form, payslip, offer_letter, termination_letter
    file_name       TEXT NOT NULL,
    file_path       TEXT NOT NULL,  -- S3/object store path
    mime_type       TEXT,
    file_size       BIGINT,
    uploaded_by     UUID NOT NULL,
    e_signed        BOOLEAN NOT NULL DEFAULT false,
    signed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Pros

1. **Maximum data integrity.** Foreign keys, CHECK constraints, and typed columns prevent invalid data from entering the system. In a domain where a payroll error can violate employment law, this is not optional.

2. **Regulatory auditability.** Normalized tables with timestamps and versioned compliance rules produce clean, queryable audit trails. Auditors and regulators can trace any payroll calculation back to the specific tax rule version that governed it.

3. **Proven at scale in financial systems.** Banks, payroll processors, and tax authorities worldwide run on normalized relational databases. The patterns are battle-tested for exactly this kind of high-integrity, multi-jurisdiction financial data.

4. **Strong tooling ecosystem.** PostgreSQL offers mature tooling for backups, replication, monitoring, migration (Flyway, Liquibase), and Row-Level Security. ORMs and query builders in every major language support it natively.

5. **Complex reporting.** JOIN-heavy analytics queries (e.g., "total employer cost by jurisdiction and benefit type for Q3") are natural in a normalized schema. No denormalization or aggregation pipelines needed for most operational reports.

6. **Multi-tenancy with RLS.** PostgreSQL Row-Level Security policies on `organization_id` provide database-enforced tenant isolation without schema-per-tenant overhead.

7. **Referential integrity across jurisdictions.** The jurisdiction table acts as a hub connecting tax rules, leave entitlements, minimum wages, benefit plans, and compliance rules — ensuring every employee's payroll is calculated against the correct regulatory configuration.

---

## Cons

1. **Schema rigidity for jurisdiction-specific variations.** Every country has unique payroll components (France has ~40 payslip line items; the US has federal + state + local taxes). A fully normalized model either requires very generic tables (losing specificity) or country-specific tables (creating schema sprawl).

2. **Migration complexity.** Adding a new jurisdiction or regulatory change may require schema migrations (new columns, new lookup tables). In a system covering 80+ countries with laws changing constantly, this creates operational risk.

3. **Performance at extreme scale.** Deep JOIN queries across employees, contracts, payroll runs, line items, tax rules, and benefits can become expensive when processing tens of thousands of employees across dozens of jurisdictions in a single payroll cycle. Read replicas and materialized views will be needed.

4. **Contract/document flexibility.** Employment contracts vary enormously by jurisdiction. A normalized approach struggles to model the full diversity of contract clauses, annexes, and country-specific fields without either over-generalizing or over-specializing the schema.

5. **Historical rule versioning is manual.** Tracking which tax rate applied at a specific point in time requires explicit `effective_from`/`effective_to` date ranges on every rule table. This works but is tedious to manage and query correctly, especially when rules change retroactively.

6. **Cold start for new jurisdictions.** Each new country requires populating tax rules, leave entitlements, minimum wages, benefit plans, and compliance rules across multiple tables before a single employee can be onboarded.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| **Primary database** | PostgreSQL 16+ with RLS enabled |
| **Schema migrations** | Flyway or Liquibase for version-controlled DDL |
| **Encryption** | pgcrypto for column-level encryption of PII (tax IDs, bank details) |
| **Read scaling** | PostgreSQL streaming replication with read replicas for analytics |
| **Connection pooling** | PgBouncer for connection management under multi-tenant load |
| **Backup** | pg_basebackup + WAL archiving for point-in-time recovery |
| **Monitoring** | pg_stat_statements + Prometheus/Grafana for query performance |
| **ORM** | Prisma, Drizzle, or SQLAlchemy depending on application language |
| **Audit logging** | pgAudit extension for statement-level audit logging |

---

## Migration and Scaling Considerations

### Data Migration

- **Jurisdiction bootstrapping:** Pre-populate jurisdiction configuration tables (tax rules, leave entitlements, minimum wages) from authoritative government sources. Automate this with structured data feeds where available.
- **Incremental country rollout:** Schema supports adding countries without migrations — new rows in `jurisdictions`, `tax_rules`, `leave_entitlements`, etc.
- **Contract migration:** If migrating from a legacy system, map existing employee/contract records into the normalized structure with a dedicated ETL pipeline. Validate referential integrity post-migration.

### Scaling Strategy

- **Vertical first:** PostgreSQL handles millions of rows per table comfortably. For the first 80 countries and up to ~50,000 employees, a single primary with read replicas is sufficient.
- **Partitioning:** Partition `payroll_runs` and `payroll_line_items` by `pay_period_start` (range partitioning) to keep payroll history queries fast as data grows.
- **Read replicas:** Route analytics, reporting, and dashboard queries to read replicas. Keep the primary for writes (payroll calculations, contract updates).
- **Citus for horizontal scaling:** If the platform grows beyond a single PostgreSQL instance, Citus (distributed PostgreSQL) can shard by `organization_id` for horizontal multi-tenant scaling.
- **Archival:** Move payroll data older than the statutory retention period (typically 5-7 years) to cold storage (S3 + Parquet) with a queryable archive layer (Athena, DuckDB).
