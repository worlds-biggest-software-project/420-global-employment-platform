# Data Model Suggestion 4: Bi-Temporal Core + Regulatory Knowledge Graph

> Project: Global Employment Platform (EOR) · Candidate #420

## Summary

A domain-specific architecture combining two patterns purpose-built for the unique challenges of global employment:

1. **Bi-temporal relational tables** (PostgreSQL with SQL:2011 temporal extensions) for all employment and payroll data, tracking both "valid time" (when a fact was true in the real world) and "transaction time" (when the system recorded it). This enables precise historical queries, retroactive corrections, and audit trails that satisfy regulatory requirements across all jurisdictions.

2. **A regulatory knowledge graph** (Neo4j or PostgreSQL with Apache AGE) for modelling the complex, interconnected web of labour laws, tax rules, benefit requirements, and compliance dependencies across 80+ jurisdictions. The graph structure naturally represents relationships like "German church tax depends on employee's tax class AND denomination AND state" or "Brazilian 13th salary affects INSS calculation which affects FGTS deposit."

The bi-temporal model answers "what was true and what did we know?" The knowledge graph answers "what rules apply and how do they interact?"

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────┐
│                   Application Layer                     │
│  (Contract Mgmt, Payroll Engine, Compliance Monitor)   │
└───────────────┬──────────────────────┬─────────────────┘
                │                      │
    ┌───────────▼───────────┐  ┌───────▼──────────────┐
    │  Bi-Temporal Store    │  │  Regulatory Graph     │
    │  (PostgreSQL)         │  │  (Neo4j / AGE)        │
    │                       │  │                       │
    │  • Employees          │  │  • Jurisdictions      │
    │  • Contracts          │  │  • Tax Rules          │
    │  • Compensation       │  │  • Leave Policies     │
    │  • Payroll Runs       │  │  • Benefit Mandates   │
    │  • Benefit Enroll.    │  │  • Compliance Deps    │
    │  • Documents          │  │  • PE Risk Factors    │
    │                       │  │  • CBA Agreements     │
    │  valid_from/valid_to  │  │  • Rule Dependencies  │
    │  sys_from/sys_to      │  │  • Treaty Networks    │
    └───────────────────────┘  └──────────────────────┘
                │                      │
                └──────────┬───────────┘
                           │
                ┌──────────▼──────────┐
                │   Analytics Store   │
                │  (ClickHouse /      │
                │   TimescaleDB)      │
                │                     │
                │  • Cost Analytics   │
                │  • Compliance KPIs  │
                │  • Payroll Trends   │
                └─────────────────────┘
```

---

## Part 1: Bi-Temporal Relational Model

### What is Bi-Temporal?

Every row in a bi-temporal table has four timestamp columns:

| Column | Meaning |
|---|---|
| `valid_from` | When this fact became true in the real world |
| `valid_to` | When this fact stopped being true in the real world |
| `sys_from` | When this row was inserted into the database |
| `sys_to` | When this row was superseded in the database |

This allows the system to answer four types of questions:
- **Current state:** What is true now? (`valid_to IS NULL AND sys_to IS NULL`)
- **Historical state:** What was true on a specific past date? (`valid_from <= date AND valid_to > date`)
- **What we knew then:** What did the system believe was true on a specific past date? (`sys_from <= date AND sys_to > date`)
- **Retroactive correction:** An employee's tax class was wrong for Q2. The correction is recorded with the current `sys_from` but the original `valid_from`/`valid_to` range. Both the original (incorrect) and corrected records are preserved.

### Core Bi-Temporal Tables

#### Employees

```sql
CREATE TABLE employees (
    id              UUID NOT NULL,
    organization_id UUID NOT NULL,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT NOT NULL,
    date_of_birth   DATE,
    nationality     VARCHAR(2),
    jurisdiction_code VARCHAR(2) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    jurisdiction_data JSONB NOT NULL DEFAULT '{}',
    
    -- Bi-temporal columns
    valid_from      DATE NOT NULL,
    valid_to        DATE NOT NULL DEFAULT '9999-12-31',
    sys_from        TIMESTAMPTZ NOT NULL DEFAULT now(),
    sys_to          TIMESTAMPTZ NOT NULL DEFAULT '9999-12-31',
    
    -- The PK includes temporal dimensions
    PRIMARY KEY (id, valid_from, sys_from),
    
    -- Exclude overlapping valid periods for the same employee
    -- (within current system time)
    EXCLUDE USING gist (
        id WITH =,
        daterange(valid_from, valid_to) WITH &&
    ) WHERE (sys_to = '9999-12-31')
);

CREATE INDEX idx_employees_current 
    ON employees (id) 
    WHERE valid_to = '9999-12-31' AND sys_to = '9999-12-31';

CREATE INDEX idx_employees_org_current 
    ON employees (organization_id) 
    WHERE valid_to = '9999-12-31' AND sys_to = '9999-12-31';
```

#### Contracts

```sql
CREATE TABLE contracts (
    id                  UUID NOT NULL,
    employee_id         UUID NOT NULL,
    organization_id     UUID NOT NULL,
    legal_entity_id     UUID NOT NULL,
    jurisdiction_code   VARCHAR(2) NOT NULL,
    contract_type       VARCHAR(20) NOT NULL,
    job_title           TEXT NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE,
    probation_end_date  DATE,
    notice_period_days  INT,
    work_hours_per_week NUMERIC(4,1),
    status              VARCHAR(20) NOT NULL,
    version             INT NOT NULL DEFAULT 1,
    jurisdiction_terms  JSONB NOT NULL DEFAULT '{}',
    
    -- Bi-temporal columns
    valid_from      DATE NOT NULL,
    valid_to        DATE NOT NULL DEFAULT '9999-12-31',
    sys_from        TIMESTAMPTZ NOT NULL DEFAULT now(),
    sys_to          TIMESTAMPTZ NOT NULL DEFAULT '9999-12-31',
    
    PRIMARY KEY (id, valid_from, sys_from)
);

-- View for current contract state
CREATE VIEW current_contracts AS
SELECT * FROM contracts
WHERE valid_to = '9999-12-31' AND sys_to = '9999-12-31';
```

#### Compensation (Separate Bi-Temporal Table)

```sql
CREATE TABLE compensation (
    id              UUID NOT NULL,
    contract_id     UUID NOT NULL,
    base_salary     NUMERIC(12,2) NOT NULL,
    salary_currency VARCHAR(3) NOT NULL,
    salary_period   VARCHAR(10) NOT NULL DEFAULT 'annual',
    bonus_target    NUMERIC(5,2),
    equity_details  JSONB DEFAULT '{}',
    allowances      JSONB DEFAULT '[]',
    
    -- Bi-temporal columns
    valid_from      DATE NOT NULL,  -- when the salary became effective
    valid_to        DATE NOT NULL DEFAULT '9999-12-31',
    sys_from        TIMESTAMPTZ NOT NULL DEFAULT now(),
    sys_to          TIMESTAMPTZ NOT NULL DEFAULT '9999-12-31',
    
    PRIMARY KEY (id, valid_from, sys_from)
);

-- Example: Query salary history for an employee
-- "What was employee X's salary on 2024-06-15?"
-- SELECT * FROM compensation
-- WHERE contract_id = :contract_id
--   AND valid_from <= '2024-06-15' AND valid_to > '2024-06-15'
--   AND sys_to = '9999-12-31';
--
-- "What did the system believe the salary was on 2024-06-15
--  as of the knowledge we had on 2024-07-01?"
-- SELECT * FROM compensation
-- WHERE contract_id = :contract_id
--   AND valid_from <= '2024-06-15' AND valid_to > '2024-06-15'
--   AND sys_from <= '2024-07-01' AND sys_to > '2024-07-01';
```

#### Payroll Runs (Bi-Temporal with Correction Support)

```sql
CREATE TABLE payroll_runs (
    id                  UUID NOT NULL,
    payroll_batch_id    UUID NOT NULL,
    contract_id         UUID NOT NULL,
    employee_id         UUID NOT NULL,
    
    gross_salary        NUMERIC(12,2) NOT NULL,
    total_deductions    NUMERIC(12,2) NOT NULL,
    total_employer_cost NUMERIC(12,2) NOT NULL,
    net_salary          NUMERIC(12,2) NOT NULL,
    currency            VARCHAR(3) NOT NULL,
    exchange_rate       NUMERIC(12,6),
    
    payslip_data        JSONB NOT NULL DEFAULT '{}',
    tax_rules_snapshot  JSONB NOT NULL DEFAULT '{}',
    -- Snapshot of tax rules used for this calculation,
    -- enabling exact reproduction of the calculation later
    
    is_correction       BOOLEAN NOT NULL DEFAULT false,
    corrects_run_id     UUID,  -- links to the original run being corrected
    correction_reason   TEXT,
    
    -- Bi-temporal columns
    valid_from      DATE NOT NULL,  -- pay period start
    valid_to        DATE NOT NULL,  -- pay period end
    sys_from        TIMESTAMPTZ NOT NULL DEFAULT now(),
    sys_to          TIMESTAMPTZ NOT NULL DEFAULT '9999-12-31',
    
    PRIMARY KEY (id, valid_from, sys_from)
);

-- Example: Retroactive payroll correction
-- 1. Mark original run as superseded (set sys_to = now())
-- 2. Insert corrected run with same valid_from/valid_to
--    but new sys_from = now(), is_correction = true
-- Both records are preserved for audit
```

#### Benefit Enrollments

```sql
CREATE TABLE benefit_enrollments (
    id              UUID NOT NULL,
    employee_id     UUID NOT NULL,
    benefit_plan_id UUID NOT NULL,
    status          VARCHAR(20) NOT NULL,
    dependents_count INT DEFAULT 0,
    enrollment_details JSONB NOT NULL DEFAULT '{}',
    
    -- Bi-temporal columns
    valid_from      DATE NOT NULL,  -- enrollment effective date
    valid_to        DATE NOT NULL DEFAULT '9999-12-31',
    sys_from        TIMESTAMPTZ NOT NULL DEFAULT now(),
    sys_to          TIMESTAMPTZ NOT NULL DEFAULT '9999-12-31',
    
    PRIMARY KEY (id, valid_from, sys_from)
);
```

### Helper Functions for Bi-Temporal Queries

```sql
-- Get current state of any bi-temporal entity
CREATE FUNCTION current_state(tbl regclass, entity_id UUID)
RETURNS SETOF RECORD AS $$
BEGIN
    RETURN QUERY EXECUTE format(
        'SELECT * FROM %s WHERE id = $1 
         AND valid_to = ''9999-12-31'' AND sys_to = ''9999-12-31''',
        tbl
    ) USING entity_id;
END;
$$ LANGUAGE plpgsql;

-- Get state as-of a specific valid date
CREATE FUNCTION state_as_of(tbl regclass, entity_id UUID, as_of DATE)
RETURNS SETOF RECORD AS $$
BEGIN
    RETURN QUERY EXECUTE format(
        'SELECT * FROM %s WHERE id = $1
         AND valid_from <= $2 AND valid_to > $2
         AND sys_to = ''9999-12-31''',
        tbl
    ) USING entity_id, as_of;
END;
$$ LANGUAGE plpgsql;

-- Get state as known at a specific system time
CREATE FUNCTION state_known_at(
    tbl regclass, entity_id UUID, 
    as_of DATE, known_at TIMESTAMPTZ
)
RETURNS SETOF RECORD AS $$
BEGIN
    RETURN QUERY EXECUTE format(
        'SELECT * FROM %s WHERE id = $1
         AND valid_from <= $2 AND valid_to > $2
         AND sys_from <= $3 AND sys_to > $3',
        tbl
    ) USING entity_id, as_of, known_at;
END;
$$ LANGUAGE plpgsql;

-- Perform a bi-temporal update (close current, insert new)
CREATE FUNCTION bitemporal_update(
    tbl regclass, entity_id UUID,
    new_valid_from DATE, new_data JSONB
)
RETURNS VOID AS $$
BEGIN
    -- Close current version (set sys_to and valid_to)
    EXECUTE format(
        'UPDATE %s SET sys_to = now()
         WHERE id = $1 AND sys_to = ''9999-12-31''',
        tbl
    ) USING entity_id;
    
    -- Insert new version
    -- (caller must provide the full INSERT with new_valid_from)
END;
$$ LANGUAGE plpgsql;
```

---

## Part 2: Regulatory Knowledge Graph

### Why a Graph?

Labour law compliance across 80+ jurisdictions involves deeply interconnected rules:

- A **tax rule** in Germany depends on the employee's **tax class**, which depends on their **marital status** and whether their **spouse** works in Germany.
- **Church tax** applies only if the employee belongs to a **recognized denomination**, and the rate varies by **federal state**.
- In France, the applicable **collective bargaining agreement (convention collective)** determines minimum salaries, overtime rules, leave entitlements, and mandatory benefits — and a single company may operate under multiple CCAs.
- **Tax treaties** between countries determine withholding rates for employees who are tax residents of one country but employed through an entity in another.
- **Permanent establishment risk** depends on the combination of employee activities, duration, jurisdiction rules, and the client company's existing presence.

These are **graph problems**: nodes are rules, entities, conditions, and jurisdictions; edges are dependencies, applicabilities, and overrides.

### Graph Schema (Neo4j / Cypher)

```cypher
// ─── Jurisdiction Nodes ───
(:Jurisdiction {
    code: "DE",
    name: "Germany",
    currency: "EUR",
    timezone: "Europe/Berlin",
    eor_model: "accepted",  // accepted, restricted, prohibited
    data_privacy_law: "GDPR"
})

(:SubJurisdiction {
    code: "DE-BY",
    name: "Bavaria",
    parent_code: "DE",
    church_tax_rate: 0.08
})

// ─── Tax Rule Nodes ───
(:TaxRule {
    id: "DE-income-tax-2024",
    jurisdiction: "DE",
    tax_type: "income_tax",
    rate_type: "progressive",
    effective_from: date("2024-01-01"),
    effective_to: null,
    source_law: "EStG §32a"
})

(:TaxBracket {
    lower_bound: 11604,
    upper_bound: 17005,
    rate: 0.14,
    formula: "linear_progressive"
})

(:TaxRule)-[:HAS_BRACKET {order: 1}]->(:TaxBracket)
(:TaxRule)-[:APPLIES_IN]->(:Jurisdiction)
(:TaxRule)-[:SUPERSEDES]->(:TaxRule)  // links to previous version

// ─── Dependency Edges ───
(:TaxRule {tax_type: "church_tax"})
    -[:DEPENDS_ON {condition: "employee.church_denomination IS NOT NULL"}]->
    (:TaxRule {tax_type: "income_tax"})
    // Church tax is calculated as % of income tax

(:TaxRule {tax_type: "solidarity_surcharge"})
    -[:DEPENDS_ON {condition: "income_tax > threshold"}]->
    (:TaxRule {tax_type: "income_tax"})

// ─── Social Insurance Rules ───
(:SocialInsuranceRule {
    id: "DE-KV-2024",
    type: "health_insurance",
    employee_rate: 0.073,
    employer_rate: 0.073,
    additional_rate: 0.017,
    contribution_ceiling_annual: 62100,
    effective_from: date("2024-01-01")
})

(:SocialInsuranceRule)-[:APPLIES_IN]->(:Jurisdiction)
(:SocialInsuranceRule)-[:PART_OF]->(:SocialInsuranceSystem {name: "Gesetzliche Krankenversicherung"})

// ─── Leave Policies ───
(:LeavePolicy {
    id: "DE-annual-leave-2024",
    leave_type: "annual",
    min_days: 20,
    standard_days: 30,
    accrual_method: "annual",
    paid: true,
    effective_from: date("2024-01-01"),
    source_law: "BUrlG §3"
})

(:LeavePolicy)-[:APPLIES_IN]->(:Jurisdiction)
(:LeavePolicy)-[:MAY_BE_ENHANCED_BY]->(:CollectiveAgreement)

// ─── Collective Bargaining Agreements ───
(:CollectiveAgreement {
    id: "DE-IG-Metall-2024",
    name: "IG Metall Tarifvertrag",
    industry: "metalworking",
    effective_from: date("2024-04-01"),
    effective_to: date("2026-03-31")
})

(:CollectiveAgreement)-[:APPLIES_IN]->(:Jurisdiction)
(:CollectiveAgreement)-[:OVERRIDES]->(:LeavePolicy)
(:CollectiveAgreement)-[:SETS_MINIMUM]->(:PayGrade {
    grade: "EG12",
    monthly_minimum: 4800.00
})

// ─── Tax Treaties ───
(:TaxTreaty {
    id: "DE-US-treaty",
    country_a: "DE",
    country_b: "US",
    signed_date: date("1989-08-29"),
    effective_from: date("1990-01-01"),
    withholding_rate_dividends: 0.15,
    withholding_rate_interest: 0.0,
    withholding_rate_royalties: 0.0
})

(:TaxTreaty)-[:BETWEEN]->(:Jurisdiction {code: "DE"})
(:TaxTreaty)-[:BETWEEN]->(:Jurisdiction {code: "US"})

// ─── PE Risk Factors ───
(:PERiskFactor {
    id: "DE-PE-fixed-place",
    factor: "fixed_place_of_business",
    risk_weight: 0.8,
    description: "Employee has a dedicated office or workspace"
})

(:PERiskFactor)-[:CONTRIBUTES_TO]->(:PERiskAssessment)
(:PERiskFactor)-[:APPLIES_IN]->(:Jurisdiction)
(:PERiskAssessment)-[:FOR_JURISDICTION]->(:Jurisdiction)

// ─── Benefit Mandates ───
(:BenefitMandate {
    id: "DE-health-insurance-mandate",
    benefit_type: "health_insurance",
    mandatory: true,
    employer_contribution_required: true,
    effective_from: date("2024-01-01"),
    source_law: "SGB V"
})

(:BenefitMandate)-[:APPLIES_IN]->(:Jurisdiction)
(:BenefitMandate)-[:REQUIRES]->(:SocialInsuranceRule)
```

### Example Graph Queries

```cypher
// 1. Get all tax rules and their dependencies for Germany
MATCH (j:Jurisdiction {code: "DE"})<-[:APPLIES_IN]-(rule)
WHERE rule:TaxRule OR rule:SocialInsuranceRule
OPTIONAL MATCH (rule)-[dep:DEPENDS_ON]->(dependency)
RETURN rule, dep, dependency

// 2. Find all rules affected by a change to income tax
MATCH (changed:TaxRule {id: "DE-income-tax-2024"})
MATCH (dependent)-[:DEPENDS_ON*]->(changed)
RETURN dependent.id, dependent.tax_type, labels(dependent)
// Returns: church_tax, solidarity_surcharge (both depend on income tax)

// 3. Calculate PE risk for a specific jurisdiction
MATCH (j:Jurisdiction {code: "DE"})<-[:APPLIES_IN]-(f:PERiskFactor)
RETURN f.factor, f.risk_weight, f.description
ORDER BY f.risk_weight DESC

// 4. Find applicable collective agreement and its overrides
MATCH (cba:CollectiveAgreement {industry: "metalworking"})
      -[:APPLIES_IN]->(j:Jurisdiction {code: "DE"})
OPTIONAL MATCH (cba)-[:OVERRIDES]->(overridden)
OPTIONAL MATCH (cba)-[:SETS_MINIMUM]->(grade:PayGrade)
RETURN cba.name, collect(overridden.id) AS overridden_policies,
       collect(grade) AS pay_grades

// 5. Find tax treaty between two countries
MATCH (treaty:TaxTreaty)-[:BETWEEN]->(j1:Jurisdiction {code: "DE"}),
      (treaty)-[:BETWEEN]->(j2:Jurisdiction {code: "US"})
RETURN treaty

// 6. Impact analysis: what changes if minimum wage increases?
MATCH (j:Jurisdiction {code: "FR"})
MATCH (j)<-[:APPLIES_IN]-(rule)
WHERE rule.effective_from > date("2024-06-01")
OPTIONAL MATCH (dependent)-[:DEPENDS_ON*]->(rule)
RETURN rule.id, labels(rule), collect(dependent.id) AS affected_rules
```

---

## Integration Between the Two Stores

```
┌──────────────────────────────────────────────────┐
│                Payroll Engine                      │
│                                                    │
│  1. Load employee + contract from bi-temporal DB  │
│     (as of pay period date)                       │
│                                                    │
│  2. Query knowledge graph for applicable rules     │
│     MATCH (j:Jurisdiction {code: employee.jurisdiction})
│     <-[:APPLIES_IN]-(rule)                        │
│     WHERE rule.effective_from <= pay_period_end    │
│                                                    │
│  3. Resolve rule dependencies via graph traversal  │
│     (income tax → church tax → solidarity)         │
│                                                    │
│  4. Calculate payroll using resolved rules          │
│                                                    │
│  5. Store results in bi-temporal payroll_runs       │
│     with tax_rules_snapshot for reproducibility    │
└──────────────────────────────────────────────────┘
```

The payroll engine reads employee/contract state from the bi-temporal store (as of the pay period date), resolves applicable rules and their dependencies from the knowledge graph, performs calculations, and writes results back to the bi-temporal store with a snapshot of the rules used.

---

## Pros

1. **Precise historical accuracy.** Bi-temporal queries answer both "what was true" and "what did we know" at any point in time. When a German tax audit asks "what income tax rate was applied to employee X for March 2024, and when was it recorded?", the answer is a single query.

2. **Retroactive corrections without data loss.** Payroll corrections, backdated salary changes, and retroactive tax adjustments are first-class operations. The original and corrected values coexist with full traceability.

3. **Natural modelling of rule dependencies.** The graph captures that church tax depends on income tax, that collective agreements override statutory leave, and that PE risk is a function of multiple interconnected factors. No amount of relational JOINs models this as naturally.

4. **Compliance impact analysis.** "If France increases minimum wage, which employees are affected, and what is the cost impact?" becomes a graph traversal followed by a bi-temporal query — computable in milliseconds, not days of manual analysis.

5. **Tax treaty navigation.** The graph naturally represents the network of bilateral tax treaties between countries. Finding the applicable treaty for a specific employee's tax residency and employment jurisdiction is a simple path query.

6. **Regulatory change propagation.** When a tax rule is updated, the graph reveals all dependent rules that must also be recalculated. This enables automated compliance alerts: "Germany's income tax brackets changed; church tax and solidarity surcharge calculations are also affected for 1,247 employees."

7. **AI-ready structure.** The knowledge graph provides structured context for AI models doing compliance monitoring, risk assessment, and benefit optimization. Graph-augmented retrieval (GraphRAG) enables the AI compliance assistant to traverse rule relationships when answering questions.

---

## Cons

1. **Operational complexity of two database engines.** Running PostgreSQL AND Neo4j (or AGE) requires two sets of operations: backups, monitoring, failover, schema management, and team expertise. This is the most significant drawback.

2. **Data synchronization between stores.** Employee data lives in PostgreSQL; jurisdiction rules live in the graph. Cross-store consistency requires careful synchronization. A payroll calculation that reads from both stores must ensure it sees a consistent snapshot.

3. **Bi-temporal queries are complex to write.** Developers must remember to include temporal predicates in every query. Forgetting the `sys_to = '9999-12-31'` filter returns historical rows that appear as duplicates. This is a persistent source of bugs.

4. **Graph database learning curve.** Most backend developers are fluent in SQL but not Cypher (Neo4j's query language). Building and maintaining the regulatory graph requires graph modelling expertise.

5. **Graph population and maintenance burden.** The regulatory knowledge graph must be populated with accurate, up-to-date labour law data for 80+ jurisdictions. This is an ongoing legal research effort that cannot be fully automated.

6. **Bi-temporal storage overhead.** Every update creates a new row (the old row has its `sys_to` closed). Active tables grow significantly faster than non-temporal tables. A contract amended 10 times has 10 rows, not 1.

7. **ORM incompatibility.** Standard ORMs (Prisma, Sequelize, Django ORM) do not natively support bi-temporal queries. A custom data access layer or query builder is needed, adding implementation effort.

8. **Testing complexity.** Testing bi-temporal logic requires setting up scenarios across multiple time dimensions. Testing graph queries requires a populated test graph. The combination makes integration testing significantly more complex.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| **Bi-temporal store** | PostgreSQL 16+ with temporal table patterns (using EXCLUDE constraints and range types) |
| **Knowledge graph** | Neo4j 5.x (standalone) or Apache AGE extension for PostgreSQL (reduces to single DB engine) |
| **Graph query language** | Cypher (Neo4j) or openCypher (AGE). Future-proof: GQL (ISO/IEC 39075, the new SQL-for-graphs standard) |
| **Bi-temporal library** | Custom data access layer wrapping temporal predicates; or SQB with temporal query builder |
| **Change data capture** | Debezium to sync PostgreSQL changes to the graph (if using separate databases) |
| **Analytics** | TimescaleDB (time-series extension for PostgreSQL) for payroll trends and cost analytics |
| **Caching** | Redis for frequently accessed jurisdiction configs and resolved rule sets |
| **Graph visualization** | Neo4j Bloom or custom D3.js visualization for compliance rule exploration |
| **Monitoring** | PostgreSQL pg_stat_statements + Neo4j Ops Manager; track bi-temporal table growth |
| **AI integration** | LangChain/LlamaIndex with Neo4j vector + graph retrieval for compliance AI assistant |

### Reducing Operational Complexity: Apache AGE

If running two separate database engines is unacceptable, **Apache AGE** (A Graph Extension) adds graph database functionality directly to PostgreSQL. This allows Cypher queries to run inside PostgreSQL, accessing the same database as the bi-temporal tables. The trade-off is reduced graph performance compared to native Neo4j, but for the regulatory graph (relatively small, read-heavy, slowly changing data), AGE is sufficient.

```sql
-- Using Apache AGE within PostgreSQL
-- Create graph
SELECT create_graph('regulatory');

-- Add a jurisdiction node
SELECT * FROM cypher('regulatory', $$
    CREATE (:Jurisdiction {code: 'DE', name: 'Germany', currency: 'EUR'})
$$) AS (v agtype);

-- Query tax rules with dependencies
SELECT * FROM cypher('regulatory', $$
    MATCH (j:Jurisdiction {code: 'DE'})<-[:APPLIES_IN]-(rule:TaxRule)
    OPTIONAL MATCH (rule)-[:DEPENDS_ON]->(dep)
    RETURN rule.tax_type, dep.tax_type
$$) AS (rule_type agtype, depends_on agtype);
```

---

## Migration and Scaling Considerations

### Migration Path

- **Phase 1: Bi-temporal tables.** Start with bi-temporal tables for contracts and compensation (the highest-value entities for historical queries). Use regular tables for everything else.
- **Phase 2: Regulatory graph.** Build the knowledge graph for 5-10 initial jurisdictions. Validate the model with real payroll calculations before expanding.
- **Phase 3: Full bi-temporal.** Extend bi-temporal patterns to employees, benefits, and payroll as the system matures.
- **Phase 4: Graph expansion.** Add collective agreements, tax treaties, and PE risk factors as the platform expands to more jurisdictions.

### Graph Population Strategy

- **Initial load:** Manual curation by legal/compliance team for the first 10 jurisdictions. Structure the process as jurisdiction profiles with standardized node/edge templates.
- **Ongoing updates:** Compliance team adds/modifies graph nodes when regulations change. AI-assisted extraction from legal documents can accelerate but must be human-verified.
- **Versioning:** Graph nodes carry `effective_from`/`effective_to` dates. New regulations create new nodes linked with `SUPERSEDES` edges to predecessors.

### Scaling

- **Bi-temporal partitioning:** Partition temporal tables by `valid_from` (range) to keep active partitions small. Archive closed `sys_to` rows to cold storage on a schedule.
- **Graph scaling:** The regulatory graph is relatively small (thousands of nodes, tens of thousands of edges for 80+ jurisdictions). A single Neo4j instance handles this easily. Scaling concerns are minimal for this workload.
- **Read replicas:** PostgreSQL read replicas for payroll reporting. Neo4j read replicas (causal cluster) if graph query volume grows.
- **Materialized views:** Pre-compute common bi-temporal queries (current employee state, current compensation) as materialized views refreshed on change, reducing query complexity for application code.
- **Retention policy:** Bi-temporal history grows linearly. Implement a retention policy: after the statutory retention period (5-10 years depending on jurisdiction), archive closed temporal rows to Parquet/S3.
