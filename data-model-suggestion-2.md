# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Budget & Appropriations Management · Created: 2026-05-22

## Philosophy

This model treats every budget action as an immutable event in an append-only event store. The current state of any budget, appropriation, or expenditure is derived by replaying the sequence of events that produced it. The write side records events (commands produce events); the read side maintains materialised projections optimised for specific query patterns (budget dashboards, appropriation balances, SEFA schedules).

This approach is particularly well-suited to government budgeting because the domain inherently demands a complete, tamper-proof audit trail. Every budget amendment, appropriation transfer, encumbrance, and expenditure must be traceable to who authorised it, when, and why. In a traditional CRUD model, audit trails are a secondary concern bolted on via triggers or application logging. In an event-sourced model, the audit trail IS the data — it is impossible to lose history because the events are the source of truth.

The pattern draws from double-entry accounting principles (every financial transaction is an immutable journal entry) and is used in production by modern financial systems including Stripe's ledger, Square's financial infrastructure, and government-adjacent systems like the UK's GOV.UK Pay platform. The DAIMS/GSDM model at USASpending.gov similarly tracks federal spending as a sequence of submissions and corrections over time.

**Best for:** Organisations that require absolute auditability (federal agencies under FISMA/FedRAMP, state agencies under strict GASB compliance), need temporal queries ("what was the budget as of March 15?"), plan to build AI-powered analytics on budget change patterns, or want to support complex undo/redo workflows during budget formulation.

**Trade-offs:**
- (+) Perfect, immutable audit trail by construction — no additional logging infrastructure needed
- (+) Full temporal query capability — reconstruct any budget state at any point in time
- (+) Supports complex undo/redo and "what-if" by branching event streams
- (+) AI/ML analytics can operate directly on the event stream to detect patterns and anomalies
- (+) Natural fit for real-time event-driven integrations (ERP sync, alerts, notifications)
- (-) Higher read complexity — queries require materialised views or projections rather than direct table scans
- (-) Event schema evolution requires careful versioning (upcasting old events)
- (-) Larger storage footprint than CRUD — events accumulate forever
- (-) Steeper learning curve for teams unfamiliar with event sourcing / CQRS
- (-) Eventual consistency between event store and read projections must be managed

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GASB Fund Accounting | Fund, department, and account identifiers are embedded in every budget event as structured metadata, enabling fund-level projections |
| GFOA Chart of Accounts | Account segment codes are event properties; the chart of accounts itself is an entity with its own event stream |
| COFOG | COFOG codes attached to expenditure and appropriation events enable international classification projections |
| DAIMS / GSDM | Federal reporting projections materialise DAIMS-compliant data from the event stream, similar to how USASpending.gov ingests agency submission files |
| OMB Circular A-11 | Budget submission events carry A-11 object class codes and programme activity identifiers |
| ISO 8601 | All event timestamps use ISO 8601 with timezone; fiscal periods are ISO 8601 date ranges |
| XBRL / FDTA | XBRL-compatible projections can be rebuilt at any time from the event stream as GASB taxonomy elements are finalised |
| OCSF | Audit event structure draws from Open Cybersecurity Schema Framework patterns for structured security event logging |

---

## Event Store (Source of Truth)

```sql
-- The single source of truth: an append-only event store
CREATE TABLE budget_events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                  -- aggregate root ID (budget, appropriation, etc.)
    stream_type     TEXT NOT NULL,                  -- 'budget', 'appropriation', 'encumbrance', 'expenditure', 'fund', 'department', 'account', 'workflow'
    event_type      TEXT NOT NULL,                  -- e.g., 'BudgetLineItemCreated', 'AppropriationAmended', 'EncumbranceRecorded'
    event_version   INT NOT NULL,                   -- monotonically increasing per stream
    -- Event payload
    payload         JSONB NOT NULL,                 -- the full event data (see examples below)
    -- Metadata
    org_id          UUID NOT NULL,
    fiscal_year     TEXT,                           -- e.g., 'FY2026' — denormalised for partitioning
    caused_by       UUID,                           -- user who triggered the event
    correlation_id  UUID,                           -- links related events across aggregates
    causation_id    UUID,                           -- the event or command that caused this event
    ip_address      INET,
    user_agent      TEXT,
    schema_version  INT NOT NULL DEFAULT 1,         -- for event schema evolution / upcasting
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Immutability constraint: no UPDATE or DELETE allowed (enforced by application + RLS)
    UNIQUE (stream_id, event_version)
);

-- Partitioned by fiscal year for performance on historical queries
-- In production, use declarative partitioning:
-- CREATE TABLE budget_events (...) PARTITION BY LIST (fiscal_year);

CREATE INDEX idx_events_stream ON budget_events (stream_id, event_version);
CREATE INDEX idx_events_type ON budget_events (stream_type, event_type);
CREATE INDEX idx_events_org_time ON budget_events (org_id, occurred_at);
CREATE INDEX idx_events_correlation ON budget_events (correlation_id);
CREATE INDEX idx_events_caused_by ON budget_events (caused_by);
CREATE INDEX idx_events_fiscal_year ON budget_events (fiscal_year);
CREATE INDEX idx_events_payload ON budget_events USING gin (payload);

-- Snapshots for performance (avoid replaying thousands of events)
CREATE TABLE event_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INT NOT NULL,                  -- event_version at snapshot time
    state           JSONB NOT NULL,                 -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshots_stream ON event_snapshots (stream_id, snapshot_version DESC);
```

### Event Type Catalogue

```sql
-- Event type registry (documentation and schema validation)
CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    stream_type     TEXT NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,                 -- JSON Schema for the event payload
    schema_version  INT NOT NULL DEFAULT 1,
    introduced_at   TEXT NOT NULL,                  -- software version that introduced this event
    deprecated_at   TEXT,                           -- software version that deprecated this event
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Event Payloads

```sql
-- Event: BudgetLineItemCreated
-- {
--   "budget_version_id": "uuid",
--   "fund_id": "uuid",
--   "fund_code": "100",
--   "dept_id": "uuid",
--   "dept_code": "4100",
--   "account_id": "uuid",
--   "account_code": "51100",
--   "programme_id": "uuid",
--   "requested_amount": 150000.00,
--   "justification": "Three new patrol officers to meet staffing requirements",
--   "personnel_detail": {
--     "positions": [
--       {"title": "Police Officer", "fte": 3.0, "base_salary": 42000.00, "benefits_rate": 0.35}
--     ]
--   }
-- }

-- Event: AppropriationAdopted
-- {
--   "fund_id": "uuid",
--   "dept_id": "uuid",
--   "account_id": "uuid",
--   "original_amount": 150000.00,
--   "authority_type": "annual",
--   "legislation_ref": "Ordinance 2026-042",
--   "effective_date": "2025-10-01"
-- }

-- Event: AppropriationAmended
-- {
--   "appropriation_stream_id": "uuid",
--   "amendment_type": "increase",
--   "amount": 25000.00,
--   "new_total": 175000.00,
--   "reason": "Mid-year supplemental for emergency equipment",
--   "ordinance_ref": "Ordinance 2026-078",
--   "approved_by": "uuid"
-- }

-- Event: EncumbranceRecorded
-- {
--   "appropriation_stream_id": "uuid",
--   "encumbrance_type": "purchase_order",
--   "reference_number": "PO-2026-00451",
--   "vendor_name": "Acme Office Supplies",
--   "amount": 12500.00,
--   "description": "Annual office supply contract"
-- }

-- Event: EncumbranceLiquidated
-- {
--   "encumbrance_stream_id": "uuid",
--   "liquidation_amount": 4200.00,
--   "remaining_amount": 8300.00,
--   "expenditure_reference": "INV-2026-03344"
-- }

-- Event: ExpenditureRecorded
-- {
--   "appropriation_stream_id": "uuid",
--   "encumbrance_stream_id": "uuid",
--   "amount": 4200.00,
--   "vendor_name": "Acme Office Supplies",
--   "invoice_number": "INV-2026-03344",
--   "transaction_date": "2026-02-15",
--   "object_class_code": "25.3",
--   "cofog_division": "01",
--   "erp_reference": "MUNIS-TX-889921",
--   "federal_award": {
--     "cfda_number": "16.710",
--     "award_number": "2025-CK-WX-0042"
--   }
-- }

-- Event: WorkflowStepApproved
-- {
--   "workflow_instance_id": "uuid",
--   "step_name": "Finance Director Review",
--   "approver_id": "uuid",
--   "decision": "approved",
--   "comments": "Budget within departmental allocation limits"
-- }

-- Event: ScenarioCreated
-- {
--   "parent_version_id": "uuid",
--   "scenario_name": "10% Revenue Shortfall",
--   "description": "Model impact of 10% property tax revenue decline",
--   "assumptions": {
--     "revenue_adjustment": -0.10,
--     "affected_sources": ["property_tax"]
--   }
-- }
```

---

## Read-Side Projections (Materialised Views)

```sql
-- =============================================================
-- PROJECTION: Current Budget State
-- Rebuilt by replaying BudgetLineItem* events
-- =============================================================
CREATE TABLE proj_budget_current (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year     TEXT NOT NULL,
    budget_version_stream_id UUID NOT NULL,
    version_name    TEXT NOT NULL,
    version_type    TEXT NOT NULL,
    is_adopted      BOOLEAN NOT NULL DEFAULT false,
    fund_code       TEXT NOT NULL,
    fund_name       TEXT NOT NULL,
    dept_code       TEXT NOT NULL,
    dept_name       TEXT NOT NULL,
    account_code    TEXT NOT NULL,
    account_name    TEXT NOT NULL,
    programme_code  TEXT,
    programme_name  TEXT,
    -- Amounts (projected from latest events)
    prior_year_actual    NUMERIC(18,2) DEFAULT 0,
    current_year_budget  NUMERIC(18,2) DEFAULT 0,
    requested_amount     NUMERIC(18,2) DEFAULT 0,
    recommended_amount   NUMERIC(18,2) DEFAULT 0,
    adopted_amount       NUMERIC(18,2) DEFAULT 0,
    justification   TEXT,
    last_event_id   UUID NOT NULL,                  -- tracks projection currency
    last_event_at   TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_budget_org_fy ON proj_budget_current (org_id, fiscal_year);
CREATE INDEX idx_proj_budget_version ON proj_budget_current (budget_version_stream_id);
CREATE INDEX idx_proj_budget_fund_dept ON proj_budget_current (fund_code, dept_code);

-- =============================================================
-- PROJECTION: Appropriation Balances (real-time)
-- Rebuilt by replaying Appropriation*, Encumbrance*, Expenditure* events
-- =============================================================
CREATE TABLE proj_appropriation_balances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year     TEXT NOT NULL,
    appropriation_stream_id UUID NOT NULL UNIQUE,
    fund_code       TEXT NOT NULL,
    dept_code       TEXT NOT NULL,
    account_code    TEXT NOT NULL,
    programme_code  TEXT,
    -- Balance components
    original_appropriation NUMERIC(18,2) NOT NULL DEFAULT 0,
    amendments_net  NUMERIC(18,2) NOT NULL DEFAULT 0,
    current_appropriation NUMERIC(18,2) NOT NULL DEFAULT 0,  -- original + amendments
    encumbrances    NUMERIC(18,2) NOT NULL DEFAULT 0,
    expenditures    NUMERIC(18,2) NOT NULL DEFAULT 0,
    available_balance NUMERIC(18,2) NOT NULL DEFAULT 0,      -- current - encumbrances - expenditures
    -- Percentage consumed
    pct_consumed    NUMERIC(5,2) GENERATED ALWAYS AS (
        CASE WHEN current_appropriation > 0
             THEN ((encumbrances + expenditures) / current_appropriation * 100)
             ELSE 0 END
    ) STORED,
    -- Alert threshold
    alert_triggered BOOLEAN NOT NULL DEFAULT false,
    -- Tracking
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_approp_org_fy ON proj_appropriation_balances (org_id, fiscal_year);
CREATE INDEX idx_proj_approp_fund ON proj_appropriation_balances (fund_code);
CREATE INDEX idx_proj_approp_alert ON proj_appropriation_balances (alert_triggered) WHERE alert_triggered = true;
CREATE INDEX idx_proj_approp_pct ON proj_appropriation_balances (pct_consumed DESC);

-- =============================================================
-- PROJECTION: Revenue Tracking
-- Rebuilt by replaying Revenue* events
-- =============================================================
CREATE TABLE proj_revenue_tracking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year     TEXT NOT NULL,
    fund_code       TEXT NOT NULL,
    account_code    TEXT NOT NULL,
    source_name     TEXT NOT NULL,
    estimated_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    actual_amount   NUMERIC(18,2) NOT NULL DEFAULT 0,
    variance        NUMERIC(18,2) GENERATED ALWAYS AS (actual_amount - estimated_amount) STORED,
    variance_pct    NUMERIC(5,2) GENERATED ALWAYS AS (
        CASE WHEN estimated_amount > 0
             THEN ((actual_amount - estimated_amount) / estimated_amount * 100)
             ELSE 0 END
    ) STORED,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_revenue_org_fy ON proj_revenue_tracking (org_id, fiscal_year);

-- =============================================================
-- PROJECTION: SEFA Schedule
-- Rebuilt by replaying Expenditure* events with federal award data
-- =============================================================
CREATE TABLE proj_sefa_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year     TEXT NOT NULL,
    cfda_number     TEXT NOT NULL,
    federal_agency  TEXT NOT NULL,
    award_name      TEXT NOT NULL,
    award_number    TEXT,
    pass_through_entity TEXT,
    is_major_programme BOOLEAN NOT NULL DEFAULT false,
    total_expenditures NUMERIC(18,2) NOT NULL DEFAULT 0,
    expenditure_count INT NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_sefa_org_fy ON proj_sefa_schedule (org_id, fiscal_year);
CREATE INDEX idx_proj_sefa_cfda ON proj_sefa_schedule (cfda_number);

-- =============================================================
-- PROJECTION: COFOG Expenditure Classification
-- Rebuilt by replaying Expenditure* events with COFOG codes
-- =============================================================
CREATE TABLE proj_cofog_expenditures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year     TEXT NOT NULL,
    cofog_division  TEXT NOT NULL,                  -- 2-digit
    cofog_group     TEXT,                           -- 3-digit
    cofog_class     TEXT,                           -- 4-digit
    cofog_label     TEXT NOT NULL,
    total_amount    NUMERIC(18,2) NOT NULL DEFAULT 0,
    transaction_count INT NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_cofog_org_fy ON proj_cofog_expenditures (org_id, fiscal_year);

-- =============================================================
-- PROJECTION: Workflow Status Dashboard
-- Rebuilt by replaying Workflow* events
-- =============================================================
CREATE TABLE proj_workflow_status (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    workflow_instance_stream_id UUID NOT NULL UNIQUE,
    workflow_type   TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_stream_id UUID NOT NULL,
    entity_label    TEXT NOT NULL,                  -- human-readable label
    submitted_by_name TEXT NOT NULL,
    submitted_at    TIMESTAMPTZ NOT NULL,
    current_step    TEXT NOT NULL,
    current_assignee TEXT,
    status          TEXT NOT NULL,
    last_action_at  TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_wf_org_status ON proj_workflow_status (org_id, status);
```

---

## Reference Data Tables (Non-Event-Sourced)

```sql
-- These tables hold relatively static reference data that does not benefit
-- from event sourcing. They are maintained via standard CRUD with
-- change-data-capture events written to the event store for auditability.

CREATE TABLE ref_organisations (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    org_type        TEXT NOT NULL,
    jurisdiction    TEXT,
    fips_code       TEXT,
    uei             TEXT,
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_funds (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES ref_organisations(id),
    fund_code       TEXT NOT NULL,
    name            TEXT NOT NULL,
    fund_category   TEXT NOT NULL,
    fund_type       TEXT NOT NULL,
    is_major        BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (org_id, fund_code)
);

CREATE TABLE ref_departments (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES ref_organisations(id),
    parent_id       UUID REFERENCES ref_departments(id),
    dept_code       TEXT NOT NULL,
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (org_id, dept_code)
);

CREATE TABLE ref_chart_of_accounts (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES ref_organisations(id),
    account_code    TEXT NOT NULL,
    account_name    TEXT NOT NULL,
    account_type    TEXT NOT NULL,
    object_class_code TEXT,
    cofog_division  TEXT,
    cofog_group     TEXT,
    cofog_class     TEXT,
    xbrl_element_id TEXT,
    parent_id       UUID REFERENCES ref_chart_of_accounts(id),
    is_posting      BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (org_id, account_code)
);

CREATE TABLE ref_programmes (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES ref_organisations(id),
    dept_id         UUID REFERENCES ref_departments(id),
    programme_code  TEXT NOT NULL,
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (org_id, programme_code)
);

CREATE TABLE ref_cofog_codes (
    division        TEXT NOT NULL,
    "group"         TEXT,
    class           TEXT,
    label           TEXT NOT NULL,
    PRIMARY KEY (division, "group", class)
);

CREATE TABLE ref_object_class_codes (
    code            TEXT PRIMARY KEY,               -- OMB A-11 object class code
    title           TEXT NOT NULL,
    category        TEXT NOT NULL                   -- 'personnel', 'contractual', 'grants', 'other'
);

-- Users and Roles (standard CRUD with audit events)
CREATE TABLE ref_users (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES ref_organisations(id),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_roles (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES ref_organisations(id),
    role_name       TEXT NOT NULL,
    UNIQUE (org_id, role_name)
);

CREATE TABLE ref_user_roles (
    user_id         UUID NOT NULL REFERENCES ref_users(id),
    role_id         UUID NOT NULL REFERENCES ref_roles(id),
    dept_id         UUID REFERENCES ref_departments(id),
    fund_id         UUID REFERENCES ref_funds(id),
    PRIMARY KEY (user_id, role_id, dept_id, fund_id)
);
```

---

## Event Processing Infrastructure

```sql
-- Projection tracking (which projections are current, which need rebuilding)
CREATE TABLE projection_status (
    projection_name TEXT PRIMARY KEY,               -- e.g., 'proj_budget_current', 'proj_appropriation_balances'
    last_event_id   UUID,
    last_event_at   TIMESTAMPTZ,
    events_behind   BIGINT NOT NULL DEFAULT 0,      -- how many unprocessed events
    status          TEXT NOT NULL DEFAULT 'current' CHECK (status IN ('current', 'rebuilding', 'stale', 'error')),
    error_message   TEXT,
    last_rebuilt_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for failed event processing
CREATE TABLE event_dead_letters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL,
    projection_name TEXT NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INT NOT NULL DEFAULT 0,
    last_retry_at   TIMESTAMPTZ,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dead_letters_unresolved ON event_dead_letters (resolved, projection_name) WHERE resolved = false;

-- Subscription tracking for event consumers (alerts, integrations, AI agents)
CREATE TABLE event_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_name TEXT NOT NULL,                  -- e.g., 'variance_alert_agent', 'erp_sync_outbound', 'ai_narrative_generator'
    stream_type_filter TEXT,                        -- NULL = all streams
    event_type_filter TEXT,                         -- NULL = all event types
    last_processed_event_id UUID,
    last_processed_at TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Temporal Query: Budget State at a Specific Date

```sql
-- "What was the Police Department's General Fund budget on March 15, 2026?"
-- Replay events up to the specified date

SELECT
    (e.payload->>'account_code') AS account_code,
    (e.payload->>'adopted_amount')::NUMERIC AS amount,
    e.event_type,
    e.occurred_at
FROM budget_events e
WHERE e.stream_type = 'budget'
  AND e.org_id = :org_id
  AND e.fiscal_year = 'FY2026'
  AND e.occurred_at <= '2026-03-15T23:59:59Z'
  AND e.payload->>'dept_code' = '4100'             -- Police Department
  AND e.payload->>'fund_code' = '100'              -- General Fund
ORDER BY e.stream_id, e.event_version;

-- The application layer replays these events to reconstruct
-- the budget state as of March 15.
```

### Appropriation Balance with Full History

```sql
-- Current appropriation balance from the read projection
SELECT
    fund_code,
    dept_code,
    account_code,
    original_appropriation,
    amendments_net,
    current_appropriation,
    encumbrances,
    expenditures,
    available_balance,
    pct_consumed
FROM proj_appropriation_balances
WHERE org_id = :org_id
  AND fiscal_year = 'FY2026'
  AND pct_consumed > 80                            -- highlight high-consumption lines
ORDER BY pct_consumed DESC;
```

### Event Stream for a Single Appropriation

```sql
-- Full audit trail for a specific appropriation
SELECT
    event_type,
    event_version,
    payload,
    occurred_at,
    (SELECT display_name FROM ref_users WHERE id = e.caused_by) AS actor
FROM budget_events e
WHERE e.stream_id = :appropriation_stream_id
ORDER BY e.event_version;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | budget_events, event_snapshots, event_type_registry |
| Read Projections | 6 | proj_budget_current, proj_appropriation_balances, proj_revenue_tracking, proj_sefa_schedule, proj_cofog_expenditures, proj_workflow_status |
| Reference Data | 9 | ref_organisations, ref_funds, ref_departments, ref_chart_of_accounts, ref_programmes, ref_cofog_codes, ref_object_class_codes, ref_users, ref_roles |
| User-Role Mapping | 1 | ref_user_roles |
| Event Infrastructure | 3 | projection_status, event_dead_letters, event_subscriptions |
| **Total** | **22** | Core: 3 event tables + 6 projections + 9 reference + 4 infrastructure |

---

## Key Design Decisions

1. **Single event table with JSONB payloads** — rather than a separate table per event type, all events live in `budget_events` with typed JSONB payloads. This keeps the schema simple, makes new event types a zero-DDL operation, and enables powerful cross-event-type queries using GIN indexes.

2. **Stream-per-aggregate design** — each aggregate root (budget version, appropriation, encumbrance) has its own `stream_id` with monotonically increasing `event_version` numbers. This enables optimistic concurrency control (reject writes where `event_version` is not the expected next value) and efficient replay for individual aggregates.

3. **Projections are disposable** — every `proj_*` table can be dropped and rebuilt from the event store at any time. This means schema changes to read models are zero-risk: create the new projection table, replay events to populate it, swap the application to use it, drop the old one.

4. **Event snapshots for performance** — for aggregates with hundreds of events (e.g., a General Fund appropriation with thousands of expenditures), periodic snapshots avoid replaying the entire event history on every read. The application loads the most recent snapshot, then replays only events after the snapshot version.

5. **Correlation and causation IDs** — `correlation_id` links all events that resulted from a single user action (e.g., a budget adoption creates AppropriationAdopted events across all line items). `causation_id` traces the direct cause chain. Together they enable complete traceability for audit purposes.

6. **Reference data is CRUD, not event-sourced** — static reference data (funds, departments, chart of accounts, COFOG codes) uses standard relational tables because event sourcing adds complexity without proportional benefit for slowly-changing lookup data. Changes to reference data still produce events in the event store for audit purposes.

7. **Event type registry with JSON Schema** — the `event_type_registry` table documents every event type with its payload schema, enabling runtime validation, API documentation generation, and event schema evolution tracking.

8. **Projection status tracking** — the `projection_status` table tells operators and monitoring systems which projections are current and which have fallen behind, enabling automated alerts and self-healing rebuild triggers.

9. **Dead letter queue for resilience** — events that fail to process into projections are captured in `event_dead_letters` rather than blocking the pipeline, with retry and resolution tracking for operations teams.

10. **Fiscal year partitioning ready** — the `fiscal_year` column on `budget_events` is denormalised specifically to support PostgreSQL declarative partitioning by fiscal year. Historical fiscal years can be moved to cheaper storage while current-year queries remain fast.
