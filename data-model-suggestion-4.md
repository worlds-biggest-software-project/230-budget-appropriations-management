# Data Model Suggestion 4: Bi-Temporal Versioned Relational

> Project: Budget & Appropriations Management · Created: 2026-05-22

## Philosophy

This model applies bi-temporal data modeling to every core budget entity, tracking two independent time dimensions: **valid time** (when a fact was true in the real world) and **transaction time** (when the system recorded that fact). Every row in a budget table has four temporal columns: `valid_from`, `valid_to`, `recorded_at`, and `superseded_at`. No data is ever physically deleted or overwritten — updates create new versions while closing the previous version's `superseded_at` timestamp.

Government budgeting is inherently temporal. Budgets are proposed, amended, re-amended, and restated. An appropriation adopted in October may be amended in March and supplemented in June. Auditors need to answer questions like "What was the adopted budget for Public Safety on March 1?" and "When was that amendment first recorded in the system?" These are precisely the questions bi-temporal modeling was designed to answer. The valid time dimension captures the real-world effective dates of budget authority; the transaction time dimension captures when system records were created or corrected, supporting complete audit trails and late-arriving corrections.

This pattern is used in financial regulatory systems (Basel III liquidity reporting requires temporal views of positions), insurance policy management (ISO 17442 LEI temporal tracking), and data warehouse slowly-changing dimension (SCD) Type 2 implementations. PostgreSQL's range types and exclusion constraints provide native support for temporal data integrity.

**Best for:** Organisations that must answer complex temporal audit questions ("what did we report on date X?", "when was this correction entered?"), are subject to federal audit requirements (Single Audit Act, GASB Statement 34/54), need retroactive corrections without losing the original record, or plan to build time-series analytics on budget evolution patterns.

**Trade-offs:**
- (+) Complete temporal audit trail — every version of every record is preserved with both real-world and system timestamps
- (+) Point-in-time reconstruction — answer "what was the budget as of date X?" with a single WHERE clause
- (+) Supports retroactive corrections without data loss — corrections create new versions; original records remain
- (+) Natural fit for budget amendment tracking — each amendment is a new temporal version
- (+) Enables trend analysis and AI anomaly detection on budget evolution patterns
- (-) Higher storage requirements — every change creates a new row rather than updating in place
- (-) Queries require temporal predicates (WHERE valid_from <= X AND valid_to > X) on every table access
- (-) Write operations are more complex — updates must close the old version and insert the new one atomically
- (-) JOIN queries across temporal tables require temporal alignment logic
- (-) Schema is more complex to understand for developers unfamiliar with bi-temporal patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GASB Fund Accounting | Fund balance classifications are temporally versioned — GASB Statement 54 reclassifications are tracked as new temporal versions |
| GFOA Chart of Accounts | Account code changes over time (reclassifications, renumbering) are preserved as temporal versions rather than overwrites |
| COFOG | COFOG reclassifications during a fiscal year produce new temporal versions with updated classification codes |
| DAIMS / GSDM | Federal reporting corrections (File C resubmissions) map naturally to bi-temporal corrections: valid_time unchanged, new transaction_time |
| OMB Circular A-11 | Apportionment changes and rescissions are modelled as temporal amendments to appropriation records |
| ISO 8601 | All temporal columns use `TIMESTAMPTZ`; fiscal year boundaries are date ranges; valid_time uses date ranges |
| XBRL / FDTA | Temporal versioning supports XBRL corrections (amendment filings) with full traceability |
| SEFA | Award period validity modelled as valid_time ranges; expenditure corrections produce new temporal versions |

---

## Temporal Support Infrastructure

```sql
-- Helper function: check if a timestamp falls within a temporal window
CREATE OR REPLACE FUNCTION is_current(valid_from TIMESTAMPTZ, valid_to TIMESTAMPTZ, superseded_at TIMESTAMPTZ)
RETURNS BOOLEAN AS $$
    SELECT valid_to > now() AND superseded_at IS NULL;
$$ LANGUAGE sql IMMUTABLE;

-- Helper view predicate: "as of now" filter
-- Usage: WHERE is_valid_now(valid_from, valid_to, superseded_at)
CREATE OR REPLACE FUNCTION is_valid_now(vf TIMESTAMPTZ, vt TIMESTAMPTZ, sa TIMESTAMPTZ)
RETURNS BOOLEAN AS $$
    SELECT vf <= now() AND vt > now() AND sa IS NULL;
$$ LANGUAGE sql STABLE;

-- Helper: "as of a specific point in time" for both dimensions
CREATE OR REPLACE FUNCTION is_valid_at(
    vf TIMESTAMPTZ, vt TIMESTAMPTZ,
    sa TIMESTAMPTZ, ra TIMESTAMPTZ,
    as_of_valid TIMESTAMPTZ,
    as_of_transaction TIMESTAMPTZ
) RETURNS BOOLEAN AS $$
    SELECT vf <= as_of_valid AND vt > as_of_valid
       AND ra <= as_of_transaction AND (sa IS NULL OR sa > as_of_transaction);
$$ LANGUAGE sql IMMUTABLE;
```

## Organisation & Configuration (Temporal)

```sql
CREATE TABLE organisations (
    id              UUID NOT NULL,                  -- NOT a primary key (multiple versions share same ID)
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    org_type        TEXT NOT NULL CHECK (org_type IN ('state', 'county', 'city', 'town', 'special_district', 'school_district', 'authority', 'federal_agency')),
    jurisdiction    TEXT,
    fips_code       TEXT,
    uei             TEXT,
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,                    -- NULL = current transaction version
    changed_by      UUID,                           -- user who made the change
    change_reason   TEXT                            -- why the change was made
);

CREATE INDEX idx_org_id ON organisations (id);
CREATE INDEX idx_org_current ON organisations (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_org_valid ON organisations (id, valid_from, valid_to);

-- Current view of organisations
CREATE VIEW v_organisations AS
SELECT * FROM organisations
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();

-- Fiscal Years (temporal)
CREATE TABLE fiscal_years (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    label           TEXT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'planning' CHECK (status IN ('planning', 'proposed', 'adopted', 'amended', 'closed')),
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_fy_id ON fiscal_years (id);
CREATE INDEX idx_fy_current ON fiscal_years (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_fy_org ON fiscal_years (org_id);
```

## Funds & Structure (Temporal)

```sql
CREATE TABLE funds (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fund_code       TEXT NOT NULL,
    name            TEXT NOT NULL,
    fund_category   TEXT NOT NULL CHECK (fund_category IN ('governmental', 'proprietary', 'fiduciary')),
    fund_type       TEXT NOT NULL CHECK (fund_type IN (
        'general', 'special_revenue', 'capital_projects', 'debt_service', 'permanent',
        'enterprise', 'internal_service',
        'pension_trust', 'investment_trust', 'private_purpose_trust', 'custodial'
    )),
    is_major        BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_funds_id ON funds (id);
CREATE INDEX idx_funds_current ON funds (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_funds_org ON funds (org_id);

CREATE VIEW v_funds AS
SELECT * FROM funds
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();

CREATE TABLE departments (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    parent_id       UUID,
    dept_code       TEXT NOT NULL,
    name            TEXT NOT NULL,
    head_user_id    UUID,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_dept_id ON departments (id);
CREATE INDEX idx_dept_current ON departments (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_dept_org ON departments (org_id);

CREATE VIEW v_departments AS
SELECT * FROM departments
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();

CREATE TABLE programmes (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    dept_id         UUID,
    programme_code  TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_prog_id ON programmes (id);
CREATE INDEX idx_prog_current ON programmes (id) WHERE superseded_at IS NULL;
```

## Chart of Accounts (Temporal)

```sql
CREATE TABLE chart_of_accounts (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    account_code    TEXT NOT NULL,
    account_name    TEXT NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN ('revenue', 'expenditure', 'asset', 'liability', 'fund_balance')),
    fund_segment    TEXT,
    dept_segment    TEXT,
    programme_segment TEXT,
    object_segment  TEXT,
    sub_object_segment TEXT,
    object_class_code TEXT,
    cofog_division  TEXT,
    cofog_group     TEXT,
    cofog_class     TEXT,
    xbrl_element_id TEXT,
    parent_id       UUID,
    hierarchy_level INT NOT NULL DEFAULT 0,
    is_posting      BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_coa_id ON chart_of_accounts (id);
CREATE INDEX idx_coa_current ON chart_of_accounts (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_coa_org_type ON chart_of_accounts (org_id, account_type);
CREATE INDEX idx_coa_code ON chart_of_accounts (org_id, account_code);

CREATE VIEW v_chart_of_accounts AS
SELECT * FROM chart_of_accounts
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();
```

## Budget Formulation (Temporal)

```sql
CREATE TABLE budget_versions (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year_id  UUID NOT NULL,
    version_name    TEXT NOT NULL,
    version_type    TEXT NOT NULL CHECK (version_type IN ('base', 'departmental_request', 'executive_recommendation', 'legislative_adopted', 'amended', 'scenario')),
    is_adopted      BOOLEAN NOT NULL DEFAULT false,
    parent_version_id UUID,
    created_by      UUID,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_bv_id ON budget_versions (id);
CREATE INDEX idx_bv_current ON budget_versions (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_bv_fy ON budget_versions (fiscal_year_id);

CREATE VIEW v_budget_versions AS
SELECT * FROM budget_versions
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();

-- Budget Line Items (temporal — every edit creates a new version)
CREATE TABLE budget_line_items (
    id              UUID NOT NULL,                  -- stable identity across versions
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_version_id UUID NOT NULL,
    fund_id         UUID NOT NULL,
    dept_id         UUID NOT NULL,
    programme_id    UUID,
    account_id      UUID NOT NULL,
    -- Amounts
    prior_year_actual    NUMERIC(18,2) DEFAULT 0,
    current_year_budget  NUMERIC(18,2) DEFAULT 0,
    requested_amount     NUMERIC(18,2) DEFAULT 0,
    recommended_amount   NUMERIC(18,2) DEFAULT 0,
    adopted_amount       NUMERIC(18,2) DEFAULT 0,
    justification   TEXT,
    notes           TEXT,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_bli_id ON budget_line_items (id);
CREATE INDEX idx_bli_current ON budget_line_items (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_bli_bv ON budget_line_items (budget_version_id);
CREATE INDEX idx_bli_fund_dept ON budget_line_items (fund_id, dept_id);

CREATE VIEW v_budget_line_items AS
SELECT * FROM budget_line_items
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();

-- Personnel Budget Items (temporal)
CREATE TABLE personnel_budget_items (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_version_id UUID NOT NULL,
    dept_id         UUID NOT NULL,
    position_title  TEXT NOT NULL,
    position_number TEXT,
    fte_count       NUMERIC(5,2) NOT NULL DEFAULT 1.0,
    pay_grade       TEXT,
    step            TEXT,
    base_salary     NUMERIC(18,2) NOT NULL,
    benefits_rate   NUMERIC(5,4),
    benefits_amount NUMERIC(18,2),
    total_compensation NUMERIC(18,2) NOT NULL,
    is_new_position BOOLEAN NOT NULL DEFAULT false,
    is_vacant       BOOLEAN NOT NULL DEFAULT false,
    fund_id         UUID NOT NULL,
    account_id      UUID NOT NULL,
    justification   TEXT,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_pbi_id ON personnel_budget_items (id);
CREATE INDEX idx_pbi_current ON personnel_budget_items (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_pbi_bv_dept ON personnel_budget_items (budget_version_id, dept_id);
```

## Appropriations (Temporal — Core Strength of This Model)

```sql
-- Appropriations: the bi-temporal model shines here because appropriations
-- are amended throughout the fiscal year, and auditors need to know
-- both the current authorised amount AND the history of amendments.
CREATE TABLE appropriations (
    id              UUID NOT NULL,                  -- stable identity
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year_id  UUID NOT NULL,
    fund_id         UUID NOT NULL,
    dept_id         UUID NOT NULL,
    account_id      UUID NOT NULL,
    programme_id    UUID,
    -- The appropriated amount AT THIS POINT IN TIME
    appropriated_amount NUMERIC(18,2) NOT NULL,
    authority_type  TEXT NOT NULL DEFAULT 'annual' CHECK (authority_type IN ('annual', 'multi_year', 'no_year', 'continuing')),
    legislation_ref TEXT,
    amendment_ref   TEXT,                           -- NULL for original; ordinance ref for amendments
    amendment_type  TEXT CHECK (amendment_type IN ('original', 'increase', 'decrease', 'transfer_in', 'transfer_out', 'supplemental', 'rescission')),
    amendment_amount NUMERIC(18,2),                 -- the change amount (NULL for original)
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when this appropriation level took effect in reality
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when this record was entered in the system
    superseded_at   TIMESTAMPTZ,                        -- when a correction superseded this record
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_approp_id ON appropriations (id);
CREATE INDEX idx_approp_current ON appropriations (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_approp_fy_fund ON appropriations (fiscal_year_id, fund_id);
CREATE INDEX idx_approp_dept ON appropriations (dept_id);
CREATE INDEX idx_approp_valid ON appropriations (id, valid_from, valid_to);

-- Current appropriations view
CREATE VIEW v_appropriations AS
SELECT * FROM appropriations
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();

-- Full amendment history for an appropriation (all temporal versions)
-- Usage: SELECT * FROM v_appropriation_history WHERE id = :appropriation_id ORDER BY valid_from;
CREATE VIEW v_appropriation_history AS
SELECT
    id,
    version_id,
    appropriated_amount,
    amendment_type,
    amendment_amount,
    amendment_ref,
    legislation_ref,
    valid_from,
    valid_to,
    recorded_at,
    superseded_at,
    changed_by,
    change_reason
FROM appropriations
WHERE superseded_at IS NULL                         -- exclude corrected records
ORDER BY id, valid_from;
```

## Encumbrances & Expenditures (Temporal)

```sql
CREATE TABLE encumbrances (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year_id  UUID NOT NULL,
    fund_id         UUID NOT NULL,
    dept_id         UUID NOT NULL,
    account_id      UUID NOT NULL,
    appropriation_id UUID NOT NULL,
    encumbrance_type TEXT NOT NULL CHECK (encumbrance_type IN ('purchase_order', 'contract', 'requisition', 'travel_authorisation')),
    reference_number TEXT NOT NULL,
    vendor_name     TEXT,
    original_amount NUMERIC(18,2) NOT NULL,
    liquidated_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'partially_liquidated', 'fully_liquidated', 'cancelled')),
    encumbered_date DATE NOT NULL,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_enc_id ON encumbrances (id);
CREATE INDEX idx_enc_current ON encumbrances (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_enc_approp ON encumbrances (appropriation_id);
CREATE INDEX idx_enc_status ON encumbrances (status) WHERE superseded_at IS NULL;

CREATE VIEW v_encumbrances AS
SELECT * FROM encumbrances
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();

-- Expenditures (append-only with temporal correction support)
-- Expenditures are generally append-only, but corrections (voids, adjustments)
-- are modelled as new temporal versions rather than updates.
CREATE TABLE expenditures (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year_id  UUID NOT NULL,
    fiscal_period   INT NOT NULL,
    fund_id         UUID NOT NULL,
    dept_id         UUID NOT NULL,
    account_id      UUID NOT NULL,
    programme_id    UUID,
    appropriation_id UUID NOT NULL,
    encumbrance_id  UUID,
    amount          NUMERIC(18,2) NOT NULL,
    vendor_name     TEXT,
    description     TEXT,
    transaction_date DATE NOT NULL,
    erp_reference   TEXT,
    -- Federal award linkage
    federal_award_id UUID,
    cfda_number     TEXT,
    -- Correction tracking
    correction_type TEXT CHECK (correction_type IN ('original', 'void', 'adjustment', 'reclassification')),
    corrects_expenditure_id UUID,                   -- links to the original expenditure being corrected
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_exp_id ON expenditures (id);
CREATE INDEX idx_exp_current ON expenditures (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_exp_approp ON expenditures (appropriation_id);
CREATE INDEX idx_exp_fy_period ON expenditures (fiscal_year_id, fiscal_period);
CREATE INDEX idx_exp_date ON expenditures (transaction_date);
CREATE INDEX idx_exp_federal ON expenditures (federal_award_id) WHERE federal_award_id IS NOT NULL;

CREATE VIEW v_expenditures AS
SELECT * FROM expenditures
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();

-- Revenues (temporal)
CREATE TABLE revenues (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year_id  UUID NOT NULL,
    fund_id         UUID NOT NULL,
    account_id      UUID NOT NULL,
    revenue_type    TEXT NOT NULL CHECK (revenue_type IN ('estimate', 'actual')),
    amount          NUMERIC(18,2) NOT NULL,
    fiscal_period   INT,
    source          TEXT,
    transaction_date DATE,
    erp_reference   TEXT,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_rev_id ON revenues (id);
CREATE INDEX idx_rev_current ON revenues (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_rev_fy_fund ON revenues (fiscal_year_id, fund_id);
```

## Federal Awards (Temporal)

```sql
CREATE TABLE federal_awards (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    cfda_number     TEXT NOT NULL,
    award_name      TEXT NOT NULL,
    federal_agency  TEXT NOT NULL,
    award_number    TEXT,
    award_amount    NUMERIC(18,2),
    pass_through_entity TEXT,
    pass_through_number TEXT,
    is_major_programme BOOLEAN NOT NULL DEFAULT false,
    award_period_start DATE,
    award_period_end DATE,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_fa_id ON federal_awards (id);
CREATE INDEX idx_fa_current ON federal_awards (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_fa_org ON federal_awards (org_id);
CREATE INDEX idx_fa_cfda ON federal_awards (cfda_number);

CREATE VIEW v_federal_awards AS
SELECT * FROM federal_awards
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();
```

## Capital Projects (Temporal)

```sql
CREATE TABLE capital_projects (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    project_code    TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    dept_id         UUID,
    status          TEXT NOT NULL DEFAULT 'proposed',
    priority_score  INT,
    total_estimated_cost NUMERIC(18,2),
    start_date      DATE,
    expected_completion DATE,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_cp_id ON capital_projects (id);
CREATE INDEX idx_cp_current ON capital_projects (id) WHERE superseded_at IS NULL;

CREATE TABLE capital_project_phases (
    id              UUID NOT NULL,
    version_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL,
    fiscal_year_id  UUID NOT NULL,
    fund_id         UUID NOT NULL,
    phase_name      TEXT NOT NULL,
    planned_amount  NUMERIC(18,2) NOT NULL,
    funding_source  TEXT,
    -- Bi-temporal columns
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ NOT NULL DEFAULT 'infinity'::TIMESTAMPTZ,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_at   TIMESTAMPTZ,
    changed_by      UUID,
    change_reason   TEXT
);

CREATE INDEX idx_cpp_id ON capital_project_phases (id);
CREATE INDEX idx_cpp_current ON capital_project_phases (id) WHERE superseded_at IS NULL;
CREATE INDEX idx_cpp_project ON capital_project_phases (project_id);

CREATE VIEW v_capital_projects AS
SELECT * FROM capital_projects
WHERE superseded_at IS NULL AND valid_to > now() AND valid_from <= now();
```

## Workflow (Non-Temporal — Workflows Are Already Sequential)

```sql
-- Workflows are inherently sequential and do not benefit from bi-temporal
-- modeling. They use standard relational tables.

CREATE TABLE workflow_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    name            TEXT NOT NULL,
    workflow_type   TEXT NOT NULL CHECK (workflow_type IN ('budget_submission', 'amendment', 'transfer', 'capital_project')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workflow_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES workflow_templates(id),
    step_order      INT NOT NULL,
    step_name       TEXT NOT NULL,
    approver_role   TEXT NOT NULL,
    is_required     BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workflow_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES workflow_templates(id),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    current_step_id UUID REFERENCES workflow_steps(id),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'approved', 'rejected', 'withdrawn')),
    submitted_by    UUID NOT NULL,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wfi_status ON workflow_instances (status);

CREATE TABLE workflow_approvals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instance_id     UUID NOT NULL REFERENCES workflow_instances(id),
    step_id         UUID NOT NULL REFERENCES workflow_steps(id),
    approver_id     UUID NOT NULL,
    decision        TEXT NOT NULL CHECK (decision IN ('approved', 'rejected', 'returned')),
    comments        TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wfa_instance ON workflow_approvals (instance_id);
```

## Users & RBAC (Non-Temporal)

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    role_name       TEXT NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    role_id         UUID NOT NULL REFERENCES roles(id),
    dept_id         UUID,
    fund_id         UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, role_id, dept_id, fund_id)
);

CREATE INDEX idx_ur_user ON user_roles (user_id);
```

## Documents & Performance (Non-Temporal)

```sql
CREATE TABLE budget_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    fiscal_year_id  UUID NOT NULL,
    budget_version_id UUID,
    document_type   TEXT NOT NULL,
    title           TEXT NOT NULL,
    format          TEXT NOT NULL,
    storage_path    TEXT,
    file_size_bytes BIGINT,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    gfoa_award_submitted BOOLEAN NOT NULL DEFAULT false,
    -- Temporal snapshot: which versions of budget data were used to generate this document
    data_snapshot_timestamp TIMESTAMPTZ NOT NULL,   -- the "as-of" timestamp for temporal queries used in generation
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_docs_org_fy ON budget_documents (org_id, fiscal_year_id);

CREATE TABLE performance_measures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    dept_id         UUID,
    programme_id    UUID,
    measure_name    TEXT NOT NULL,
    measure_type    TEXT NOT NULL CHECK (measure_type IN ('output', 'outcome', 'efficiency', 'effectiveness')),
    unit_of_measure TEXT NOT NULL,
    target_value    NUMERIC(18,4),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE performance_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    measure_id      UUID NOT NULL REFERENCES performance_measures(id),
    fiscal_year_id  UUID NOT NULL,
    fiscal_period   INT,
    target_value    NUMERIC(18,4),
    actual_value    NUMERIC(18,4),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (measure_id, fiscal_year_id, fiscal_period)
);
```

## ERP Integration

```sql
CREATE TABLE erp_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    erp_type        TEXT NOT NULL,
    connection_name TEXT NOT NULL,
    endpoint_url    TEXT,
    auth_type       TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    sync_status     TEXT DEFAULT 'idle',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE erp_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES erp_connections(id),
    sync_type       TEXT NOT NULL,
    direction       TEXT NOT NULL,
    records_processed INT NOT NULL DEFAULT 0,
    records_failed  INT NOT NULL DEFAULT 0,
    error_details   TEXT,
    -- Temporal sync metadata
    sync_as_of      TIMESTAMPTZ NOT NULL,           -- the temporal "as-of" point for this sync
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_erp_sync_conn ON erp_sync_log (connection_id);
```

---

## Example Temporal Queries

### Point-in-Time Budget Reconstruction

```sql
-- "What was the adopted budget for the Police Department as of March 15, 2026?"
-- Uses the valid time dimension to find the budget state at that date.

SELECT
    bli.fund_id,
    bli.dept_id,
    bli.account_id,
    bli.adopted_amount,
    bli.justification,
    bli.valid_from,
    bli.valid_to
FROM budget_line_items bli
JOIN v_budget_versions bv ON bv.id = bli.budget_version_id
WHERE bli.dept_id = :police_dept_id
  AND bv.is_adopted = true
  AND bv.fiscal_year_id = :fy2026_id
  -- Temporal predicates: valid at March 15, current transaction version
  AND bli.valid_from <= '2026-03-15T23:59:59Z'
  AND bli.valid_to > '2026-03-15T23:59:59Z'
  AND bli.superseded_at IS NULL;
```

### Appropriation Amendment History

```sql
-- Full history of amendments to a specific appropriation line
-- Shows each temporal version with the change that produced it

SELECT
    appropriated_amount,
    amendment_type,
    amendment_amount,
    amendment_ref,
    valid_from AS effective_date,
    recorded_at AS entered_date,
    change_reason,
    (SELECT display_name FROM users WHERE id = a.changed_by) AS changed_by
FROM appropriations a
WHERE a.id = :appropriation_id
  AND a.superseded_at IS NULL                      -- exclude corrected versions
ORDER BY a.valid_from;
```

### Bi-Temporal Audit Query

```sql
-- "What did we THINK the Police appropriation was on March 15,
--  based on what we had recorded in the system as of March 15?"
-- vs.
-- "What do we NOW know the Police appropriation was on March 15?"
-- (accounts for retroactive corrections entered after March 15)

-- Query 1: As reported on March 15 (both valid and transaction time = March 15)
SELECT appropriated_amount, amendment_type, amendment_ref
FROM appropriations
WHERE id = :appropriation_id
  AND valid_from <= '2026-03-15T23:59:59Z'
  AND valid_to > '2026-03-15T23:59:59Z'
  AND recorded_at <= '2026-03-15T23:59:59Z'
  AND (superseded_at IS NULL OR superseded_at > '2026-03-15T23:59:59Z');

-- Query 2: Current knowledge of March 15 (valid time = March 15, transaction time = now)
SELECT appropriated_amount, amendment_type, amendment_ref
FROM appropriations
WHERE id = :appropriation_id
  AND valid_from <= '2026-03-15T23:59:59Z'
  AND valid_to > '2026-03-15T23:59:59Z'
  AND superseded_at IS NULL;
```

### Expenditure Correction Trail

```sql
-- Show original expenditure and all corrections
SELECT
    e.id,
    e.amount,
    e.correction_type,
    e.corrects_expenditure_id,
    e.valid_from,
    e.recorded_at,
    e.change_reason
FROM expenditures e
WHERE (e.id = :original_expenditure_id OR e.corrects_expenditure_id = :original_expenditure_id)
  AND e.superseded_at IS NULL
ORDER BY e.recorded_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Configuration | 2 | organisations, fiscal_years (both temporal) |
| Fund & Structure | 3 | funds, departments, programmes (all temporal) |
| Chart of Accounts | 1 | chart_of_accounts (temporal) |
| Budget Formulation | 3 | budget_versions, budget_line_items, personnel_budget_items (all temporal) |
| Capital Projects | 2 | capital_projects, capital_project_phases (both temporal) |
| Appropriations & Tracking | 4 | appropriations, encumbrances, expenditures, revenues (all temporal) |
| Federal Awards | 1 | federal_awards (temporal) |
| Workflow | 4 | workflow_templates, workflow_steps, workflow_instances, workflow_approvals (standard CRUD) |
| Users & RBAC | 3 | users, roles, user_roles (standard CRUD) |
| Documents & Performance | 3 | budget_documents, performance_measures, performance_results (standard CRUD) |
| ERP Integration | 2 | erp_connections, erp_sync_log (standard CRUD) |
| Views | 10 | v_organisations, v_funds, v_departments, v_chart_of_accounts, v_budget_versions, v_budget_line_items, v_appropriations, v_appropriation_history, v_encumbrances, v_expenditures, v_federal_awards, v_capital_projects |
| **Total** | **28 tables + 10+ views** | 16 temporal tables + 12 standard tables |

---

## Key Design Decisions

1. **Bi-temporal on financial entities, standard CRUD on operational entities** — funds, accounts, budgets, appropriations, encumbrances, and expenditures are bi-temporal because they are subject to amendments and corrections that auditors must trace. Workflows, users, and roles use standard CRUD because their change history is adequately served by the audit log.

2. **`id` + `version_id` dual-key pattern** — every temporal table has a stable `id` (the entity's identity across all versions) and a `version_id` (the primary key for each specific temporal row). This enables queries like "give me all versions of appropriation X" (`WHERE id = X`) while maintaining a unique PK per row.

3. **Partial indexes on current records** — `WHERE superseded_at IS NULL` partial indexes ensure that queries for current data (the most common access pattern) perform as fast as a non-temporal table, because the index only covers live rows.

4. **Current-state views** — `v_*` views abstract away the temporal predicates for application code that only needs current data. Application queries use views by default; audit and reporting queries access base tables with explicit temporal predicates.

5. **Appropriation amendments as temporal versions** — rather than a separate amendments table (as in Model 1), each amendment produces a new temporal version of the appropriation row with updated `appropriated_amount`, `amendment_type`, and `amendment_amount`. The full amendment history is the sequence of temporal versions ordered by `valid_from`.

6. **Expenditure corrections via `correction_type`** — voids, adjustments, and reclassifications are new temporal versions with `correction_type` indicating the nature of the correction and `corrects_expenditure_id` linking to the original. This preserves both the original record and the correction for audit purposes.

7. **Document snapshot timestamp** — `budget_documents.data_snapshot_timestamp` records the temporal "as-of" point used when generating the document. This enables exact reproduction of any published budget document by re-querying the temporal data at the same timestamp.

8. **No physical deletes** — the model never deletes data. "Deletion" is modelled by setting `valid_to` to the current timestamp (the entity is no longer valid) while keeping `superseded_at` NULL (the record of its validity period is not being corrected).

9. **`change_reason` on every temporal row** — every version includes a human-readable reason for the change, creating a self-documenting audit trail. For appropriation amendments, this captures the legislative rationale; for corrections, it explains why the correction was needed.

10. **Helper functions for temporal queries** — `is_valid_now()`, `is_valid_at()`, and `is_current()` functions encapsulate temporal predicates, reducing boilerplate in application queries and ensuring consistent temporal logic across the codebase.
