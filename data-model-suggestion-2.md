# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Global Employment Platform (EOR) · Candidate #420

## Summary

An event-sourced architecture with Command Query Responsibility Segregation (CQRS) where every state change in the system — contract creation, salary adjustment, payroll calculation, benefit enrollment, compliance rule update — is captured as an immutable event in an append-only event store. The current state of any entity is derived by replaying its event stream. Read models (projections) are built asynchronously from the event stream to serve queries, dashboards, and reports.

This approach prioritises **complete auditability, regulatory traceability, and temporal correctness** — critical for a platform where regulators may ask "what was this employee's tax withholding in March 2024, and what rule was applied?" and the system must answer with provable certainty.

---

## Architecture Overview

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
     ┌────────▼────────┐          ┌─────────▼─────────┐
     │  Command Side   │          │   Query Side       │
     │  (Write Model)  │          │   (Read Models)    │
     └────────┬────────┘          └─────────▲─────────┘
              │                             │
              │  append                     │  project
              │                             │
     ┌────────▼─────────────────────────────┤
     │         Event Store                  │
     │   (append-only, immutable log)       │
     └──────────────────────────────────────┘
              │
              │  subscribe
              │
     ┌────────▼────────┐
     │   Projections   │
     │  (Read Models)  │
     ├─────────────────┤
     │ • Employee View │
     │ • Payroll View  │
     │ • Compliance    │
     │ • Analytics     │
     │ • Audit Trail   │
     └─────────────────┘
```

---

## Key Entities (Aggregates) and Event Streams

### Aggregate: Employee

The Employee aggregate owns the lifecycle of a person employed through the platform.

```
Stream: employee-{employee_id}

Events:
  EmployeeOnboarded
  EmployeeProfileUpdated
  EmployeeDocumentUploaded
  EmployeeDocumentVerified
  EmployeeStatusChanged        (active, on_leave, offboarding, terminated)
  EmployeeBankDetailsUpdated
  EmployeeEmergencyContactUpdated
  EmployeeTaxIdRegistered
```

### Aggregate: Contract

The Contract aggregate tracks the full lifecycle of an employment agreement.

```
Stream: contract-{contract_id}

Events:
  ContractDrafted
    { employee_id, organization_id, legal_entity_id, jurisdiction_id,
      contract_type, job_title, start_date, end_date,
      work_hours_per_week, notice_period_days }
  ContractCompensationSet
    { base_salary, salary_currency, salary_period, bonus_structure }
  ContractBenefitsAttached
    { benefit_plan_ids[], statutory_benefits[], supplemental_benefits[] }
  ContractSentForSignature
    { sent_to, sent_at, document_id }
  ContractSignedByEmployee
    { signed_at, signature_id }
  ContractSignedByEmployer
    { signed_at, signatory_id }
  ContractActivated
    { activated_at }
  ContractAmended
    { amendment_type, reason, effective_date, 
      changed_fields: { field: { old_value, new_value } } }
  ContractRenewalInitiated
    { new_end_date, renewal_terms }
  ContractTerminationInitiated
    { termination_type, reason, notice_date, last_working_day,
      severance_amount, severance_currency }
  ContractTerminated
    { terminated_at, final_pay_processed }
```

### Aggregate: PayrollCycle

The PayrollCycle aggregate handles a single payroll run for a jurisdiction/legal entity.

```
Stream: payroll-cycle-{cycle_id}

Events:
  PayrollCycleOpened
    { legal_entity_id, jurisdiction_id, pay_period_start, pay_period_end,
      payment_date, currency }
  PayrollEmployeeIncluded
    { employee_id, contract_id, gross_salary, adjustments[] }
  PayrollCalculated
    { employee_id, gross, deductions[], employer_costs[],
      net, tax_rules_applied[], exchange_rate, exchange_rate_date }
  PayrollAnomalyFlagged
    { employee_id, anomaly_type, description, severity, 
      suggested_action }
  PayrollReviewApproved
    { approved_by, approved_at, total_gross, total_net,
      total_employer_cost }
  PayrollSubmittedToAuthority
    { authority_name, submission_reference, submitted_at }
  PayrollPaymentDispatched
    { payment_batch_id, payment_method, dispatched_at }
  PayrollPaymentConfirmed
    { confirmation_reference, confirmed_at }
  PayrollCycleClosed
    { closed_at, summary_stats }
```

### Aggregate: Jurisdiction (Configuration)

The Jurisdiction aggregate tracks regulatory rules and their changes over time.

```
Stream: jurisdiction-{jurisdiction_id}

Events:
  JurisdictionRegistered
    { country_code, country_name, currency, timezone }
  TaxRulePublished
    { tax_type, rate_type, brackets[], effective_from, source_law }
  TaxRuleSuperseded
    { old_rule_id, new_rule_id, effective_from, reason }
  MinimumWageUpdated
    { wage_type, amount, currency, effective_from, source_law }
  LeaveEntitlementUpdated
    { leave_type, min_days, paid, pay_rate, effective_from }
  ComplianceRulePublished
    { rule_category, rule_key, rule_value, effective_from, source_law }
  ComplianceAlertIssued
    { alert_type, description, impact_assessment, 
      affected_contracts_count, effective_date }
  BenefitPlanRegistered
    { plan_name, plan_type, is_statutory, provider, costs }
  BenefitPlanDeactivated
    { plan_id, reason, effective_date }
```

### Aggregate: BenefitEnrollment

```
Stream: benefit-enrollment-{enrollment_id}

Events:
  BenefitEnrolled
    { employee_id, benefit_plan_id, enrollment_date, dependents_count }
  BenefitDependentAdded
    { dependent_name, relationship, effective_date }
  BenefitDependentRemoved
    { dependent_name, effective_date }
  BenefitEnrollmentTerminated
    { termination_date, reason }
```

### Aggregate: MisclassificationAssessment

```
Stream: misclassification-{assessment_id}

Events:
  AssessmentInitiated
    { organization_id, worker_id, jurisdiction_id }
  QuestionnaireCompleted
    { responses[], control_score, integration_score, economic_dependency_score }
  RiskScoreCalculated
    { overall_risk: "low" | "medium" | "high", factors[], recommendation }
  ConversionPathwayRecommended
    { recommended_contract_type, estimated_cost_difference, timeline }
  ConversionApproved
    { approved_by, target_start_date }
  ConversionCompleted
    { new_contract_id, conversion_date }
```

---

## Event Store Schema

```sql
-- Core event store table (append-only)
CREATE TABLE events (
    global_position   BIGSERIAL PRIMARY KEY,
    stream_id         TEXT NOT NULL,
    stream_position   INT NOT NULL,
    event_type        TEXT NOT NULL,
    data              JSONB NOT NULL,
    metadata          JSONB NOT NULL DEFAULT '{}',
    -- metadata includes: causation_id, correlation_id, user_id,
    --   tenant_id, timestamp, source_service
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, stream_position)
);

CREATE INDEX idx_events_stream ON events (stream_id, stream_position);
CREATE INDEX idx_events_type ON events (event_type);
CREATE INDEX idx_events_created ON events (created_at);
CREATE INDEX idx_events_tenant ON events (
    (metadata->>'tenant_id')
);

-- Snapshot table for performance (periodic state snapshots)
CREATE TABLE snapshots (
    stream_id         TEXT PRIMARY KEY,
    stream_position   INT NOT NULL,
    state             JSONB NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Subscription checkpoints for projections
CREATE TABLE projection_checkpoints (
    projection_name   TEXT PRIMARY KEY,
    last_position     BIGINT NOT NULL DEFAULT 0,
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Models (Projections)

### Employee Directory Projection

```sql
CREATE TABLE read_employees (
    employee_id       UUID PRIMARY KEY,
    organization_id   UUID NOT NULL,
    full_name         TEXT NOT NULL,
    email             TEXT NOT NULL,
    jurisdiction      VARCHAR(2) NOT NULL,
    status            TEXT NOT NULL,
    job_title         TEXT,
    start_date        DATE,
    current_salary    NUMERIC(12,2),
    salary_currency   VARCHAR(3),
    active_benefits   INT DEFAULT 0,
    last_payroll_date DATE,
    updated_at        TIMESTAMPTZ NOT NULL
);
```

### Payroll Dashboard Projection

```sql
CREATE TABLE read_payroll_summary (
    cycle_id          UUID PRIMARY KEY,
    jurisdiction      VARCHAR(2) NOT NULL,
    legal_entity_id   UUID NOT NULL,
    pay_period_start  DATE NOT NULL,
    pay_period_end    DATE NOT NULL,
    status            TEXT NOT NULL,
    employee_count    INT NOT NULL,
    total_gross       NUMERIC(14,2),
    total_deductions  NUMERIC(14,2),
    total_employer_cost NUMERIC(14,2),
    total_net         NUMERIC(14,2),
    currency          VARCHAR(3) NOT NULL,
    anomalies_count   INT DEFAULT 0,
    updated_at        TIMESTAMPTZ NOT NULL
);
```

### Compliance Timeline Projection

```sql
CREATE TABLE read_compliance_timeline (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction      VARCHAR(2) NOT NULL,
    event_type        TEXT NOT NULL,
    category          TEXT NOT NULL,
    description       TEXT NOT NULL,
    effective_date    DATE NOT NULL,
    impact_assessment TEXT,
    affected_contracts INT DEFAULT 0,
    event_timestamp   TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_compliance_jurisdiction_date 
    ON read_compliance_timeline (jurisdiction, effective_date);
```

### Full Audit Trail Projection

```sql
CREATE TABLE read_audit_trail (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type       TEXT NOT NULL,  -- employee, contract, payroll, benefit
    entity_id         UUID NOT NULL,
    action            TEXT NOT NULL,
    actor_id          UUID,
    actor_type        TEXT,  -- user, system, cron, api
    tenant_id         UUID NOT NULL,
    details           JSONB NOT NULL,
    occurred_at       TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_audit_entity ON read_audit_trail (entity_type, entity_id);
CREATE INDEX idx_audit_tenant ON read_audit_trail (tenant_id, occurred_at);
```

---

## Pros

1. **Provable audit trail by design.** Every state change is an immutable event. Regulators, auditors, and legal teams can see exactly what happened, when, in what order, and who initiated it. This is not bolted on — it is the core architecture.

2. **Perfect temporal queries.** "What was employee X's salary on March 15, 2024?" is answered by replaying events up to that timestamp. No need for complex `effective_from`/`effective_to` range queries. The event stream IS the history.

3. **Regulatory compliance traceability.** When a tax rule changes, the `TaxRulePublished` and `TaxRuleSuperseded` events create a clear chain showing which rule governed which payroll calculation. This is invaluable during tax audits.

4. **Retroactive corrections without data loss.** If a payroll error is discovered months later, a corrective event is appended (never modifying the original). The system maintains both the original (incorrect) and corrected states, which is often a legal requirement.

5. **Independent read/write scaling.** Write side (event appending) and read side (projections) scale independently. Payroll calculations can run without blocking dashboard queries, and new reporting views can be built without touching the write model.

6. **Replayability.** If a new regulatory requirement demands a new report, a new projection can be built by replaying the entire event history. No data migration needed — the events contain everything.

7. **Natural fit for compliance monitoring.** Jurisdiction events (TaxRulePublished, ComplianceAlertIssued) flow through the same event infrastructure as business events, enabling automated reactions: "When MinimumWageUpdated, find all contracts below new minimum and flag for amendment."

---

## Cons

1. **Significant implementation complexity.** Event sourcing requires building event stores, projection engines, snapshot management, idempotent event handlers, and saga/process managers. The learning curve is steep, and few teams have production experience with this pattern.

2. **Eventual consistency in read models.** Projections update asynchronously. After a contract is signed, the employee directory view may take milliseconds to seconds to reflect the change. For most EOR operations this is acceptable, but payroll approvals may need stronger consistency guarantees.

3. **Event schema evolution is hard.** As the platform evolves, event schemas will change. Managing backward-compatible event versioning (upcasting old events to new schemas) requires careful design and tooling. In a domain with 80+ jurisdictions adding new fields, this is a recurring challenge.

4. **Debugging complexity.** Understanding the current state of an entity requires reasoning about its event stream. A contract with 50 events (drafted, amended 8 times, benefits changed 4 times, etc.) is harder to inspect than a single row in a `contracts` table.

5. **Storage growth.** An append-only store grows without bound. A platform with 50,000 employees generating payroll events monthly, plus compliance events across 80 jurisdictions, will accumulate billions of events over years. Archival and compaction strategies are essential.

6. **Snapshot management overhead.** Without snapshots, replaying long event streams to rebuild state becomes slow. Snapshot frequency must balance between storage cost and rebuild performance.

7. **Limited ORM/framework support.** Unlike relational models that work naturally with every ORM, event-sourced systems require specialized frameworks (Marten, EventStoreDB, Axon). The ecosystem is smaller and the tooling less mature.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| **Event store** | EventStoreDB (purpose-built) or PostgreSQL with append-only table + advisory locks |
| **Event bus** | Apache Kafka or NATS JetStream for event distribution to projections |
| **Projection database** | PostgreSQL for relational projections, Elasticsearch for full-text search projections |
| **Snapshot store** | PostgreSQL or Redis for hot snapshots |
| **Framework** | Marten (.NET), Axon (Java), or custom implementation with Node.js/Python |
| **Schema registry** | Confluent Schema Registry or custom event versioning layer |
| **Monitoring** | Event lag monitoring (projection delay), dead letter queues for failed projections |
| **Serialization** | JSON with explicit version fields; Avro or Protobuf for high-throughput streams |
| **Process managers** | Saga pattern for multi-aggregate workflows (onboarding, payroll cycle, offboarding) |

---

## Migration and Scaling Considerations

### Migrating to Event Sourcing

- **Greenfield is ideal.** Event sourcing is most practical when starting from scratch. Migrating an existing relational system to event sourcing requires "event generation" from current state, which produces synthetic events that lack the fidelity of real event streams.
- **Hybrid introduction.** Start with event sourcing for the highest-value aggregates (contracts, payroll cycles) while keeping simpler entities (benefit plans, jurisdiction configuration) in a traditional relational model. This reduces initial complexity while capturing the most important audit trails.
- **Import events for initial state.** When onboarding existing client data, generate `EmployeeOnboarded` and `ContractActivated` events from the import, establishing a known starting state.

### Scaling Strategy

- **Event store partitioning.** Partition the event store by stream prefix (e.g., `contract-*`, `payroll-*`) or by tenant/organization for write throughput.
- **Projection scaling.** Run multiple projection instances in parallel, partitioned by jurisdiction or organization. Each instance subscribes to a subset of events.
- **Kafka for distribution.** Use Kafka topics partitioned by organization_id to distribute events to projection workers. This naturally load-balances across jurisdictions.
- **Event archival.** Compact old event streams after a configurable retention period (e.g., 10 years for payroll events). Archived events move to cold storage (S3 + Parquet) but remain replayable if needed.
- **Snapshot frequency.** Generate snapshots every N events (e.g., every 100 events per stream) to bound replay time. For the PayrollCycle aggregate (which may have hundreds of `PayrollCalculated` events per cycle), snapshots after each batch approval are recommended.
- **Read model rebuilds.** Design projections to be fully rebuildable from the event store. Maintain projection versioning so that schema changes trigger automatic rebuilds.
