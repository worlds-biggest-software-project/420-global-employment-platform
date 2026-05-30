# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Global Employment Platform (EOR) · Candidate #420

## Summary

A hybrid data model using PostgreSQL as the single database engine, combining normalized relational tables for core entities (organizations, employees, contracts, payroll) with JSONB columns for jurisdiction-specific variations, custom fields, and semi-structured data. This approach treats PostgreSQL as both a relational database and a document store, leveraging JSONB's indexing capabilities (GIN indexes) to maintain queryability while accommodating the enormous variability across 80+ jurisdictions without schema migrations.

The key insight: in a global employment platform, roughly 60-70% of the data model is universal (every employee has a name, every contract has a start date, every payroll run has a gross and net amount), but 30-40% varies wildly by jurisdiction (France requires 40+ payslip line items, Germany has church tax, Brazil has 13th-month salary, India has PF/ESI contributions). The hybrid model stores the universal part in typed columns and the variable part in JSONB, avoiding both the rigidity of full normalization and the chaos of pure document storage.

---

## Design Principles

1. **Typed columns for universal fields.** Any field that exists in every jurisdiction (employee name, contract start date, gross salary, currency) gets a proper column with a type, constraints, and indexes.
2. **JSONB for jurisdiction-specific extensions.** Fields that only apply to certain countries or vary in structure across jurisdictions go into a `jurisdiction_data` JSONB column with a validated schema.
3. **JSON Schema validation.** Each jurisdiction has a registered JSON Schema that defines the expected structure of its JSONB extensions. Application-level validation ensures JSONB data conforms to these schemas before insert.
4. **GIN indexes on JSONB paths.** Frequently queried JSONB fields get GIN or expression indexes for performance parity with regular columns.
5. **Single database engine.** No separate document store (MongoDB, etc.) to manage. PostgreSQL handles both relational and document workloads.

---

## Key Entities and Schema

### Organizations and Employees

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    legal_name      TEXT NOT NULL,
    billing_country VARCHAR(2) NOT NULL,
    currency        VARCHAR(3) NOT NULL,
    plan_tier       VARCHAR(20) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings may include:
    -- { "approval_workflow": "two_level",
    --   "payroll_notification_emails": ["finance@acme.com"],
    --   "custom_fields_config": [...],
    --   "integrations": { "hris": "bamboohr", "ats": "greenhouse" } }
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
    jurisdiction_code VARCHAR(2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'onboarding',
    
    -- JSONB for jurisdiction-specific employee data
    jurisdiction_data JSONB NOT NULL DEFAULT '{}',
    -- Examples by country:
    --
    -- US: { "ssn_encrypted": "...", "w4_filing_status": "single",
    --        "w4_allowances": 2, "state": "CA",
    --        "state_tax_elections": {...} }
    --
    -- DE: { "steuer_id": "...", "tax_class": "I",
    --        "church_tax": true, "church_denomination": "catholic",
    --        "social_insurance_number": "..." }
    --
    -- BR: { "cpf": "...", "pis_pasep": "...",
    --        "ctps_number": "...", "ctps_series": "..." }
    --
    -- FR: { "numero_securite_sociale": "...",
    --        "mutuelle_enrollment": true,
    --        "convention_collective": "SYNTEC" }

    -- JSONB for tenant-specific custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- { "department": "Engineering", "cost_center": "CC-4420",
    --   "badge_number": "ENG-0042" }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- GIN index for jurisdiction-specific queries
CREATE INDEX idx_employees_jurisdiction_data
    ON employees USING gin (jurisdiction_data jsonb_path_ops);

-- Expression index for common JSONB lookups
CREATE INDEX idx_employees_us_state 
    ON employees ((jurisdiction_data->>'state'))
    WHERE jurisdiction_code = 'US';
```

### Contracts

```sql
CREATE TABLE contracts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    legal_entity_id     UUID NOT NULL REFERENCES legal_entities(id),
    jurisdiction_code   VARCHAR(2) NOT NULL,
    
    -- Universal contract fields (typed columns)
    contract_type       VARCHAR(20) NOT NULL,
    job_title           TEXT NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE,
    probation_end_date  DATE,
    notice_period_days  INT,
    work_hours_per_week NUMERIC(4,1),
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    version             INT NOT NULL DEFAULT 1,
    
    -- Compensation (universal)
    base_salary         NUMERIC(12,2) NOT NULL,
    salary_currency     VARCHAR(3) NOT NULL,
    salary_period       VARCHAR(10) NOT NULL DEFAULT 'annual',

    -- JSONB for jurisdiction-specific contract terms
    jurisdiction_terms  JSONB NOT NULL DEFAULT '{}',
    -- Examples:
    --
    -- DE: { "collective_agreement": "IG Metall",
    --        "pay_grade": "EG12", "christmas_bonus": true,
    --        "vacation_days": 30,
    --        "additional_pension_contribution": 2.5 }
    --
    -- FR: { "convention_collective": "SYNTEC",
    --        "coefficient": 170, "position": "cadre",
    --        "rtt_days": 12,
    --        "mutuelle_employer_share": 50.0 }
    --
    -- BR: { "thirteenth_salary": true,
    --        "vale_transporte": true,
    --        "vale_refeicao_daily": 35.00,
    --        "fgts_rate": 8.0 }
    --
    -- JP: { "commuting_allowance_monthly": 20000,
    --        "overtime_premium_rate": 1.25,
    --        "social_insurance_grade": 22 }

    -- JSONB for supplemental compensation details
    compensation_details JSONB NOT NULL DEFAULT '{}',
    -- { "bonus": { "type": "annual", "target_percentage": 15 },
    --   "equity": { "grant_type": "RSU", "shares": 1000,
    --               "vesting_schedule": "4_year_1_cliff" },
    --   "allowances": [
    --     { "type": "housing", "amount": 2000, "currency": "USD" },
    --     { "type": "remote_work", "amount": 100, "currency": "USD" }
    --   ] }

    signed_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Jurisdiction Configuration

```sql
CREATE TABLE jurisdictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code    VARCHAR(2) NOT NULL UNIQUE,
    country_name    TEXT NOT NULL,
    currency        VARCHAR(3) NOT NULL,
    timezone        TEXT NOT NULL,
    eor_restricted  BOOLEAN NOT NULL DEFAULT false,
    
    -- JSONB for the full regulatory configuration
    payroll_config  JSONB NOT NULL DEFAULT '{}',
    -- { "pay_frequency": ["monthly"],
    --   "pay_day_rules": "last_business_day",
    --   "fiscal_year_start": "01-01",
    --   "13th_salary": false,
    --   "14th_salary": false,
    --   "currency_decimals": 2 }

    tax_config      JSONB NOT NULL DEFAULT '{}',
    -- { "income_tax": {
    --     "type": "progressive",
    --     "brackets": [
    --       { "lower": 0, "upper": 11604, "rate": 0 },
    --       { "lower": 11604, "upper": 27478, "rate": 0.14 },
    --       { "lower": 27478, "upper": 78570, "rate": 0.30 },
    --       { "lower": 78570, "upper": null, "rate": 0.45 }
    --     ],
    --     "effective_from": "2024-01-01"
    --   },
    --   "social_security_employee": { "rate": 0.0945, "cap": 58050 },
    --   "social_security_employer": { "rate": 0.1545, "cap": 58050 },
    --   "church_tax": { "applicable": true, "rate": 0.08 },
    --   "solidarity_surcharge": { "rate": 0.055, "threshold": 18130 }
    -- }

    leave_config    JSONB NOT NULL DEFAULT '{}',
    -- { "annual_leave": { "days": 20, "accrual": "immediate" },
    --   "sick_leave": { "days": "unlimited", "pay_rate": 1.0,
    --                   "employer_period_weeks": 6 },
    --   "maternity_leave": { "weeks": 14, "pay_rate": 1.0 },
    --   "paternity_leave": { "weeks": 0, "pay_rate": 0 },
    --   "public_holidays": 10 }

    compliance_rules JSONB NOT NULL DEFAULT '{}',
    -- { "max_probation_months": 6,
    --   "min_notice_period": { "under_2_years": 30, "2_to_5_years": 60 },
    --   "severance": { "formula": "0.5_months_per_year", "cap_months": 12 },
    --   "working_hours": { "max_weekly": 48, "max_daily": 10 },
    --   "overtime_rules": { "premium": 1.25, "max_monthly_hours": 20 },
    --   "data_privacy_law": "GDPR",
    --   "pe_risk_notes": "EOR model accepted but requires AUG licence" }

    -- Schema version for migration tracking
    config_version  INT NOT NULL DEFAULT 1,
    
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Historical versions of jurisdiction configs
CREATE TABLE jurisdiction_config_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    config_type     VARCHAR(20) NOT NULL,
    -- types: payroll_config, tax_config, leave_config, compliance_rules
    config_data     JSONB NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    source_law      TEXT,
    change_summary  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jurisdiction_history_lookup
    ON jurisdiction_config_history (jurisdiction_id, config_type, effective_from);
```

### Payroll

```sql
CREATE TABLE payroll_batches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    legal_entity_id UUID NOT NULL REFERENCES legal_entities(id),
    jurisdiction_code VARCHAR(2) NOT NULL,
    pay_period_start DATE NOT NULL,
    pay_period_end  DATE NOT NULL,
    payment_date    DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    currency        VARCHAR(3) NOT NULL,
    
    -- Aggregated totals (typed for queries and reporting)
    total_gross     NUMERIC(14,2),
    total_net       NUMERIC(14,2),
    total_employer_cost NUMERIC(14,2),
    employee_count  INT,
    
    -- JSONB for batch-level metadata
    batch_metadata  JSONB NOT NULL DEFAULT '{}',
    -- { "tax_config_version": 3,
    --   "exchange_rates_snapshot": { "USD": 1.0, "EUR": 0.92 },
    --   "anomalies_flagged": 2,
    --   "approval_chain": [
    --     { "approver_id": "...", "approved_at": "...", "level": 1 }
    --   ] }
    
    approved_by     UUID,
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payroll_runs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payroll_batch_id    UUID NOT NULL REFERENCES payroll_batches(id),
    contract_id         UUID NOT NULL REFERENCES contracts(id),
    employee_id         UUID NOT NULL REFERENCES employees(id),
    
    -- Universal payroll fields (typed columns)
    gross_salary        NUMERIC(12,2) NOT NULL,
    total_deductions    NUMERIC(12,2) NOT NULL,
    total_employer_cost NUMERIC(12,2) NOT NULL,
    net_salary          NUMERIC(12,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL,
    exchange_rate       NUMERIC(12,6),
    
    -- JSONB for the full payslip breakdown
    payslip_data    JSONB NOT NULL DEFAULT '{}',
    -- This is where jurisdiction-specific line items live.
    -- The structure varies dramatically by country.
    --
    -- DE example:
    -- { "earnings": [
    --     { "code": "BASE", "description": "Grundgehalt", "amount": 5000.00 },
    --     { "code": "XMAS", "description": "Weihnachtsgeld", "amount": 416.67 }
    --   ],
    --   "employee_deductions": [
    --     { "code": "LST", "description": "Lohnsteuer", "amount": 892.33 },
    --     { "code": "SOLI", "description": "Solidaritätszuschlag", "amount": 49.08 },
    --     { "code": "KiSt", "description": "Kirchensteuer", "amount": 71.39 },
    --     { "code": "KV_AN", "description": "Krankenversicherung AN", "amount": 365.00 },
    --     { "code": "RV_AN", "description": "Rentenversicherung AN", "amount": 465.00 },
    --     { "code": "AV_AN", "description": "Arbeitslosenversicherung AN", "amount": 65.00 },
    --     { "code": "PV_AN", "description": "Pflegeversicherung AN", "amount": 85.25 }
    --   ],
    --   "employer_costs": [
    --     { "code": "KV_AG", "description": "Krankenversicherung AG", "amount": 365.00 },
    --     { "code": "RV_AG", "description": "Rentenversicherung AG", "amount": 465.00 },
    --     { "code": "AV_AG", "description": "Arbeitslosenversicherung AG", "amount": 65.00 },
    --     { "code": "PV_AG", "description": "Pflegeversicherung AG", "amount": 85.25 },
    --     { "code": "UV", "description": "Unfallversicherung", "amount": 60.00 },
    --     { "code": "U1", "description": "Umlage U1", "amount": 14.00 },
    --     { "code": "U2", "description": "Umlage U2", "amount": 19.50 },
    --     { "code": "INSOLVENZ", "description": "Insolvenzgeldumlage", "amount": 3.00 }
    --   ],
    --   "tax_rules_applied": { "income_tax_version": 3, "social_version": 2 }
    -- }
    --
    -- FR example would have ~40 distinct line items
    -- BR example would include INSS, IRRF, FGTS, vale transporte, etc.
    
    -- JSONB for ML anomaly detection results
    anomaly_flags   JSONB DEFAULT '[]',
    -- [{ "type": "salary_deviation", "severity": "warning",
    --    "message": "Net pay 15% lower than previous month",
    --    "suggested_action": "review" }]
    
    status          VARCHAR(20) NOT NULL DEFAULT 'calculated',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- GIN index for payslip queries
CREATE INDEX idx_payroll_payslip ON payroll_runs 
    USING gin (payslip_data jsonb_path_ops);
```

### Benefits

```sql
CREATE TABLE benefit_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_code VARCHAR(2) NOT NULL,
    plan_name       TEXT NOT NULL,
    plan_type       VARCHAR(30) NOT NULL,
    is_statutory    BOOLEAN NOT NULL DEFAULT false,
    provider_name   TEXT,
    
    -- JSONB for plan-specific details that vary by type and jurisdiction
    plan_details    JSONB NOT NULL DEFAULT '{}',
    -- Health insurance example:
    -- { "coverage_type": "family",
    --   "deductible": 500, "copay_percentage": 20,
    --   "network": "PPO", "dental_included": true,
    --   "vision_included": false,
    --   "employee_monthly_cost": 150.00,
    --   "employer_monthly_cost": 450.00,
    --   "dependent_tiers": [
    --     { "tier": "employee_only", "cost": 150 },
    --     { "tier": "employee_spouse", "cost": 280 },
    --     { "tier": "family", "cost": 420 }
    --   ] }
    --
    -- Pension example:
    -- { "scheme_type": "defined_contribution",
    --   "employer_match_rate": 0.05,
    --   "employee_min_contribution": 0.03,
    --   "vesting_years": 3,
    --   "annual_cap": 23000 }
    
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
    
    -- JSONB for enrollment-specific selections
    enrollment_details JSONB NOT NULL DEFAULT '{}',
    -- { "coverage_tier": "family",
    --   "dependents": [
    --     { "name": "...", "relationship": "spouse", "dob": "..." },
    --     { "name": "...", "relationship": "child", "dob": "..." }
    --   ],
    --   "employee_contribution": 280.00,
    --   "employer_contribution": 450.00 }
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### JSON Schema Registry

```sql
-- Stores the expected JSONB structure for each jurisdiction's data
CREATE TABLE json_schema_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_code VARCHAR(2) NOT NULL,
    schema_target   VARCHAR(30) NOT NULL,
    -- targets: employee_jurisdiction_data, contract_jurisdiction_terms,
    --          payslip_data, tax_config, leave_config, compliance_rules
    json_schema     JSONB NOT NULL,
    -- Standard JSON Schema (Draft 2020-12) defining required fields,
    -- types, validation rules for the jurisdiction's JSONB data
    version         INT NOT NULL DEFAULT 1,
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (jurisdiction_code, schema_target, version)
);
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name      TEXT NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL,  -- INSERT, UPDATE, DELETE
    old_values      JSONB,
    new_values      JSONB,
    changed_by      UUID,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    organization_id UUID
);

CREATE INDEX idx_audit_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_org ON audit_log (organization_id, changed_at);
```

---

## Pros

1. **Best of both worlds.** Universal fields get relational integrity (foreign keys, NOT NULL, CHECK constraints, typed indexes). Jurisdiction-specific fields get document flexibility (no schema migrations when adding a new country's tax fields).

2. **Single database engine.** PostgreSQL handles both workloads. No operational overhead of running and syncing a separate document database. One backup strategy, one monitoring stack, one connection pool, one team to maintain.

3. **Zero-migration country onboarding.** Adding a new jurisdiction requires inserting rows (jurisdiction config, JSON Schema registry entries, benefit plans) — not running DDL migrations. A new country can go live without a deployment.

4. **JSONB is queryable and indexable.** GIN indexes on JSONB columns provide fast lookups. Queries like `WHERE jurisdiction_data->>'tax_class' = 'I'` or `WHERE payslip_data @> '{"earnings": [{"code": "XMAS"}]}'` work with index support.

5. **Natural fit for payslip data.** Payslip line items vary from 8 (Singapore) to 40+ (France). A JSONB `payslip_data` column accommodates this without a `payroll_line_items` table with 40 optional columns or a generic EAV structure.

6. **Tenant-specific custom fields for free.** Organizations can define custom employee fields (department, cost center, badge number) stored in the `custom_fields` JSONB column. No schema changes, no multi-tenant column sprawl.

7. **JSON Schema validation.** The JSON Schema registry provides structural validation of JSONB data at the application layer, preventing garbage data while maintaining flexibility. Schemas are versioned per jurisdiction.

8. **Progressive formalization.** If a JSONB field becomes universally needed (e.g., every jurisdiction ends up needing `overtime_premium_rate`), it can be promoted to a typed column in a future migration without losing existing data.

---

## Cons

1. **Weaker integrity for JSONB data.** PostgreSQL cannot enforce foreign keys, NOT NULL, or CHECK constraints inside JSONB columns. Validation must happen at the application layer, which means bugs in validation code can allow invalid data.

2. **JSON Schema maintenance burden.** Each jurisdiction needs a maintained JSON Schema for each entity type (employee data, contract terms, payslip data). With 80+ jurisdictions, that is 240+ schemas to maintain and version.

3. **JSONB query performance caveats.** While GIN indexes help, complex nested JSONB queries are slower than equivalent typed-column queries. Aggregations across JSONB fields (e.g., "sum of church tax deductions across all German employees") require careful query construction.

4. **Inconsistent reporting.** Reporting across jurisdictions becomes harder when each country's payslip has a different JSONB structure. Building a unified "total employer cost by deduction type" report requires jurisdiction-aware query logic.

5. **Schema drift risk.** Without discipline, JSONB fields can diverge from their schemas. Over time, different versions of the application may write slightly different JSONB structures, creating data quality issues that are hard to detect.

6. **Developer experience varies.** Developers working with typed columns get autocomplete, type checking, and ORM support. Developers working with JSONB fields get string-keyed access with runtime type assertions. The two halves of the model have different ergonomics.

7. **Audit trail for JSONB changes is coarser.** The audit log captures old/new JSONB values as whole objects. Identifying exactly which nested field changed requires application-level JSONB diff logic.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| **Database** | PostgreSQL 16+ (native JSONB, GIN indexes, RLS) |
| **JSONB validation** | Ajv (Node.js) or jsonschema (Python) for JSON Schema Draft 2020-12 validation at the application layer |
| **Schema management** | JSON Schema registry table + versioning; consider json-schema-to-typescript for type generation |
| **Indexing strategy** | GIN indexes with `jsonb_path_ops` for containment queries; expression indexes for frequently queried scalar JSONB fields |
| **ORM** | Prisma (with Json field support), Drizzle (with JSONB operators), or SQLAlchemy (with JSONB column type) |
| **Migration tool** | Flyway or Liquibase for DDL; application-level migrations for JSON Schema registry |
| **Monitoring** | Track JSONB column sizes (pg_column_size); alert on unexpectedly large JSONB values |
| **Encryption** | pgcrypto for PII in typed columns; application-level encryption for sensitive JSONB fields (SSN, tax IDs) |
| **Full-text search** | PostgreSQL tsvector on text fields; Elasticsearch if advanced payslip search is needed |

---

## Migration and Scaling Considerations

### Data Migration

- **From pure relational:** Existing normalized jurisdiction tables (tax_rules, leave_entitlements, etc.) can be aggregated into JSONB config objects with a SQL migration script. The reverse direction (JSONB to normalized) is also straightforward.
- **From document store:** If migrating from MongoDB or similar, the JSONB columns provide a natural landing zone. The migration adds relational structure (foreign keys, typed columns) around the existing document data.
- **Jurisdiction data seeding:** Create a seeding pipeline that reads authoritative jurisdiction data (tax brackets, leave entitlements, minimum wages) from structured sources and generates JSONB config objects with JSON Schema validation.

### Scaling Strategy

- **Vertical scaling first.** A single PostgreSQL instance with JSONB handles 80+ jurisdictions and tens of thousands of employees comfortably. JSONB storage is compact (binary format, deduplicated keys).
- **Partitioning.** Partition `payroll_runs` by `pay_period_start` (range partitioning) to manage payroll history growth. Partition `audit_log` by `changed_at`.
- **Read replicas.** Route reporting queries (especially cross-jurisdiction aggregations over JSONB) to read replicas. Keep writes on the primary.
- **JSONB size monitoring.** Monitor average JSONB column sizes per table. If a single JSONB column grows beyond ~100KB (unlikely for payroll data but possible for complex contract terms), consider normalizing the largest nested structures.
- **Caching.** Cache jurisdiction configs in Redis or application memory (they change infrequently). Invalidate on `jurisdiction_config_history` inserts.
- **TOAST management.** PostgreSQL automatically TOASTs (compresses and out-of-lines) large JSONB values. Monitor TOAST table sizes and vacuum frequency.
- **Archival.** Same as the relational model: move historical payroll data beyond the retention period to cold storage. JSONB payslip data is self-contained, making archival simpler than with normalized line items across multiple tables.
