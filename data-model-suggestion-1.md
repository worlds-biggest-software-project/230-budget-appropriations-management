# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Budget & Appropriations Management · Created: 2026-05-22

## Philosophy

This model follows the classical normalized relational approach (Third Normal Form), mapping each domain concept to a dedicated table with explicit foreign key relationships. The design mirrors the hierarchical structure of government fund accounting as defined by GASB and GFOA standards: organisations contain funds, funds contain departments, departments submit budgets against a chart of accounts, and appropriations are tracked at the fund-department-account intersection.

The strength of this approach is its alignment with how government finance officers already think about budgets. Every concept — fund, department, programme, object class, appropriation, encumbrance, expenditure — has a dedicated, well-typed table with referential integrity enforced at the database level. This makes the schema self-documenting and ensures that invalid data cannot be inserted (e.g., an encumbrance cannot reference a non-existent appropriation line).

Real-world systems using this pattern include Tyler Munis, Oracle Government Financials, and most legacy government ERP platforms. The US Treasury's DAIMS/GSDM data model and OMB's MAX budget data system both use normalised structures with explicit account identification codes, object classes, and programme activities as separate classified dimensions.

**Best for:** Organisations that value data integrity and standards compliance above all else, have well-defined budget structures that do not vary significantly across jurisdictions, and have teams experienced with relational database design.

**Trade-offs:**
- (+) Maximum data integrity through foreign keys and constraints
- (+) Clean alignment with GASB fund accounting model and GFOA chart of accounts conventions
- (+) Straightforward SQL reporting — standard JOINs produce standard government financial reports
- (+) Well-understood by government finance IT teams
- (-) High table count increases schema complexity and migration overhead
- (-) Adding jurisdiction-specific fields requires DDL changes (ALTER TABLE) rather than configuration
- (-) Rigid structure makes rapid prototyping and MVP iteration slower
- (-) Many-to-many junction tables proliferate, increasing query complexity for some report types

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GASB Fund Accounting | Fund types (governmental, proprietary, fiduciary) modelled as an enum on the `funds` table; fund balance classifications follow GASB Statement 54 |
| GFOA Chart of Accounts | Chart of accounts table structure follows GFOA's recommended segment hierarchy (fund, department, programme, object, sub-object) |
| COFOG | `cofog_division`, `cofog_group`, `cofog_class` columns on the `chart_of_accounts` table for international expenditure classification |
| DAIMS / GSDM | Federal account identification codes (`agency_id`, `main_account_code`) on `federal_accounts` table align with Treasury's data model |
| OMB Circular A-11 | Budget account structure, object class codes, and programme activity definitions follow A-11 Section 79 and 83 conventions |
| ISO 8601 | All date/time columns use `TIMESTAMPTZ`; fiscal year periods use ISO 8601 date ranges |
| XBRL / FDTA | `xbrl_element_id` on chart of accounts entries enables future mapping to GASB digital taxonomy elements |
| SEFA | `federal_award_id` linkage on expenditure records supports Schedule of Expenditures of Federal Awards preparation |

---

## Core Organisation & Configuration Tables

```sql
-- Tenant / Organisation
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    org_type        TEXT NOT NULL CHECK (org_type IN ('state', 'county', 'city', 'town', 'special_district', 'school_district', 'authority', 'federal_agency')),
    jurisdiction    TEXT,                           -- ISO 3166-2 code (e.g., 'US-CA', 'US-TX')
    fips_code       TEXT,                           -- FIPS code for US government entities
    duns_number     TEXT,                           -- SAM.gov unique entity identifier
    uei             TEXT,                           -- Unique Entity Identifier (replaced DUNS)
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisations_org_type ON organisations (org_type);
CREATE INDEX idx_organisations_jurisdiction ON organisations (jurisdiction);

-- Fiscal Year Periods
CREATE TABLE fiscal_years (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    label           TEXT NOT NULL,                  -- e.g., 'FY2026', 'FY2026-2027' (biennial)
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_biennial     BOOLEAN NOT NULL DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'planning' CHECK (status IN ('planning', 'proposed', 'adopted', 'amended', 'closed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, label)
);

CREATE INDEX idx_fiscal_years_org_status ON fiscal_years (org_id, status);

-- Fiscal Periods within a Fiscal Year (months, quarters)
CREATE TABLE fiscal_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    period_number   INT NOT NULL,                   -- 1-12 for monthly, 1-4 for quarterly
    period_type     TEXT NOT NULL CHECK (period_type IN ('month', 'quarter', 'year')),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    is_closed       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fiscal_year_id, period_number, period_type)
);
```

## Fund Accounting Tables

```sql
-- Fund Types (per GASB)
-- Governmental: General, Special Revenue, Capital Projects, Debt Service, Permanent
-- Proprietary: Enterprise, Internal Service
-- Fiduciary: Pension Trust, Investment Trust, Private-Purpose Trust, Custodial

CREATE TABLE funds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fund_code       TEXT NOT NULL,                  -- e.g., '100' for General Fund
    name            TEXT NOT NULL,
    fund_category   TEXT NOT NULL CHECK (fund_category IN ('governmental', 'proprietary', 'fiduciary')),
    fund_type       TEXT NOT NULL CHECK (fund_type IN (
        'general', 'special_revenue', 'capital_projects', 'debt_service', 'permanent',
        'enterprise', 'internal_service',
        'pension_trust', 'investment_trust', 'private_purpose_trust', 'custodial'
    )),
    is_major        BOOLEAN NOT NULL DEFAULT false, -- GASB major fund determination
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, fund_code)
);

CREATE INDEX idx_funds_org_category ON funds (org_id, fund_category);
CREATE INDEX idx_funds_org_active ON funds (org_id) WHERE is_active = true;
```

## Organisational Structure Tables

```sql
-- Departments / Cost Centres
CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    parent_id       UUID REFERENCES departments(id),  -- supports departmental hierarchy
    dept_code       TEXT NOT NULL,
    name            TEXT NOT NULL,
    head_user_id    UUID,                              -- FK to users table
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, dept_code)
);

CREATE INDEX idx_departments_org_parent ON departments (org_id, parent_id);

-- Programmes / Activities
CREATE TABLE programmes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    dept_id         UUID REFERENCES departments(id),
    programme_code  TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, programme_code)
);

CREATE INDEX idx_programmes_org_dept ON programmes (org_id, dept_id);
```

## Chart of Accounts

```sql
-- Chart of Accounts (GFOA-aligned segment hierarchy)
CREATE TABLE chart_of_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    account_code    TEXT NOT NULL,                  -- full segmented code, e.g., '100-4100-51100'
    account_name    TEXT NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN ('revenue', 'expenditure', 'asset', 'liability', 'fund_balance')),
    -- GFOA segment breakdown
    fund_segment    TEXT,
    dept_segment    TEXT,
    programme_segment TEXT,
    object_segment  TEXT,                           -- object of expenditure (salaries, supplies, etc.)
    sub_object_segment TEXT,
    -- Classification codes
    object_class_code TEXT,                         -- OMB object class (A-11 Section 83)
    cofog_division  TEXT,                           -- COFOG 2-digit division code
    cofog_group     TEXT,                           -- COFOG 3-digit group code
    cofog_class     TEXT,                           -- COFOG 4-digit class code
    xbrl_element_id TEXT,                           -- future GASB digital taxonomy element
    -- Hierarchy
    parent_id       UUID REFERENCES chart_of_accounts(id),
    hierarchy_level INT NOT NULL DEFAULT 0,
    is_posting      BOOLEAN NOT NULL DEFAULT true,  -- can transactions post to this account?
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, account_code)
);

CREATE INDEX idx_coa_org_type ON chart_of_accounts (org_id, account_type);
CREATE INDEX idx_coa_org_object_class ON chart_of_accounts (org_id, object_class_code);
CREATE INDEX idx_coa_parent ON chart_of_accounts (parent_id);
CREATE INDEX idx_coa_cofog ON chart_of_accounts (cofog_division) WHERE cofog_division IS NOT NULL;
```

## Budget Formulation Tables

```sql
-- Budget Versions (scenarios / what-if models)
CREATE TABLE budget_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    version_name    TEXT NOT NULL,                  -- e.g., 'Base Budget', 'Manager Request', 'City Manager Recommended', 'Council Adopted'
    version_type    TEXT NOT NULL CHECK (version_type IN ('base', 'departmental_request', 'executive_recommendation', 'legislative_adopted', 'amended', 'scenario')),
    is_adopted      BOOLEAN NOT NULL DEFAULT false,
    parent_version_id UUID REFERENCES budget_versions(id),  -- for scenario branching
    created_by      UUID,                           -- FK to users
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, fiscal_year_id, version_name)
);

CREATE INDEX idx_budget_versions_fy ON budget_versions (fiscal_year_id);

-- Budget Line Items (the core budget data)
CREATE TABLE budget_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_version_id UUID NOT NULL REFERENCES budget_versions(id),
    fund_id         UUID NOT NULL REFERENCES funds(id),
    dept_id         UUID NOT NULL REFERENCES departments(id),
    programme_id    UUID REFERENCES programmes(id),
    account_id      UUID NOT NULL REFERENCES chart_of_accounts(id),
    -- Amounts
    prior_year_actual   NUMERIC(18,2) DEFAULT 0,   -- prior year actuals for comparison
    current_year_budget NUMERIC(18,2) DEFAULT 0,   -- current year adopted budget
    requested_amount    NUMERIC(18,2) DEFAULT 0,   -- department request
    recommended_amount  NUMERIC(18,2) DEFAULT 0,   -- executive recommendation
    adopted_amount      NUMERIC(18,2) DEFAULT 0,   -- legislative adopted
    -- Narrative
    justification   TEXT,                           -- department justification text
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (budget_version_id, fund_id, dept_id, account_id, programme_id)
);

CREATE INDEX idx_bli_version ON budget_line_items (budget_version_id);
CREATE INDEX idx_bli_fund_dept ON budget_line_items (fund_id, dept_id);
CREATE INDEX idx_bli_account ON budget_line_items (account_id);

-- Personnel Budget Detail (position-based budgeting)
CREATE TABLE personnel_budget_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_version_id UUID NOT NULL REFERENCES budget_versions(id),
    dept_id         UUID NOT NULL REFERENCES departments(id),
    position_title  TEXT NOT NULL,
    position_number TEXT,                           -- HR position control number
    fte_count       NUMERIC(5,2) NOT NULL DEFAULT 1.0,
    pay_grade       TEXT,
    step            TEXT,
    base_salary     NUMERIC(18,2) NOT NULL,
    benefits_rate   NUMERIC(5,4),                   -- e.g., 0.3500 for 35%
    benefits_amount NUMERIC(18,2),
    total_compensation NUMERIC(18,2) NOT NULL,
    is_new_position BOOLEAN NOT NULL DEFAULT false,
    is_vacant       BOOLEAN NOT NULL DEFAULT false,
    fund_id         UUID NOT NULL REFERENCES funds(id),
    account_id      UUID NOT NULL REFERENCES chart_of_accounts(id),
    justification   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_personnel_version_dept ON personnel_budget_items (budget_version_id, dept_id);
```

## Capital Projects

```sql
-- Capital Projects
CREATE TABLE capital_projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    project_code    TEXT NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    dept_id         UUID REFERENCES departments(id),
    status          TEXT NOT NULL DEFAULT 'proposed' CHECK (status IN ('proposed', 'approved', 'in_progress', 'completed', 'cancelled')),
    priority_score  INT,                            -- 1-100 priority ranking
    total_estimated_cost NUMERIC(18,2),
    start_date      DATE,
    expected_completion DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, project_code)
);

-- Capital Project Funding Phases (multi-year)
CREATE TABLE capital_project_phases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES capital_projects(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    fund_id         UUID NOT NULL REFERENCES funds(id),
    phase_name      TEXT NOT NULL,                  -- e.g., 'Design', 'Construction', 'Equipment'
    planned_amount  NUMERIC(18,2) NOT NULL,
    funding_source  TEXT,                           -- e.g., 'General Fund', 'Bond Proceeds', 'Federal Grant'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cpp_project ON capital_project_phases (project_id);
CREATE INDEX idx_cpp_fy ON capital_project_phases (fiscal_year_id);
```

## Appropriations & Expenditure Tracking

```sql
-- Appropriations (legislative authorisations)
CREATE TABLE appropriations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    fund_id         UUID NOT NULL REFERENCES funds(id),
    dept_id         UUID NOT NULL REFERENCES departments(id),
    account_id      UUID NOT NULL REFERENCES chart_of_accounts(id),
    programme_id    UUID REFERENCES programmes(id),
    original_amount NUMERIC(18,2) NOT NULL,         -- original adopted appropriation
    amended_amount  NUMERIC(18,2) NOT NULL,         -- current appropriation after amendments
    authority_type  TEXT NOT NULL DEFAULT 'annual' CHECK (authority_type IN ('annual', 'multi_year', 'no_year', 'continuing')),
    legislation_ref TEXT,                           -- reference to authorising ordinance/bill
    effective_date  DATE NOT NULL,
    expiration_date DATE,                           -- NULL for no-year appropriations
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fiscal_year_id, fund_id, dept_id, account_id, programme_id)
);

CREATE INDEX idx_appropriations_fy_fund ON appropriations (fiscal_year_id, fund_id);
CREATE INDEX idx_appropriations_dept ON appropriations (dept_id);

-- Appropriation Amendments / Transfers
CREATE TABLE appropriation_amendments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    appropriation_id UUID NOT NULL REFERENCES appropriations(id),
    amendment_type  TEXT NOT NULL CHECK (amendment_type IN ('increase', 'decrease', 'transfer_in', 'transfer_out')),
    amount          NUMERIC(18,2) NOT NULL,
    reason          TEXT NOT NULL,
    approved_by     UUID,                           -- FK to users
    approved_date   DATE,
    ordinance_ref   TEXT,                           -- reference to amending ordinance
    transfer_pair_id UUID REFERENCES appropriation_amendments(id),  -- links transfer in/out pairs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_amendments_appropriation ON appropriation_amendments (appropriation_id);

-- Encumbrances (committed obligations)
CREATE TABLE encumbrances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    fund_id         UUID NOT NULL REFERENCES funds(id),
    dept_id         UUID NOT NULL REFERENCES departments(id),
    account_id      UUID NOT NULL REFERENCES chart_of_accounts(id),
    appropriation_id UUID NOT NULL REFERENCES appropriations(id),
    encumbrance_type TEXT NOT NULL CHECK (encumbrance_type IN ('purchase_order', 'contract', 'requisition', 'travel_authorisation')),
    reference_number TEXT NOT NULL,                 -- PO number, contract number, etc.
    vendor_name     TEXT,
    original_amount NUMERIC(18,2) NOT NULL,
    liquidated_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    outstanding_amount NUMERIC(18,2) GENERATED ALWAYS AS (original_amount - liquidated_amount) STORED,
    status          TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'partially_liquidated', 'fully_liquidated', 'cancelled')),
    encumbered_date DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encumbrances_appropriation ON encumbrances (appropriation_id);
CREATE INDEX idx_encumbrances_fy_fund ON encumbrances (fiscal_year_id, fund_id);
CREATE INDEX idx_encumbrances_status ON encumbrances (status) WHERE status IN ('open', 'partially_liquidated');

-- Expenditures / Actuals (from ERP integration)
CREATE TABLE expenditures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_periods(id),
    fund_id         UUID NOT NULL REFERENCES funds(id),
    dept_id         UUID NOT NULL REFERENCES departments(id),
    account_id      UUID NOT NULL REFERENCES chart_of_accounts(id),
    programme_id    UUID REFERENCES programmes(id),
    appropriation_id UUID NOT NULL REFERENCES appropriations(id),
    encumbrance_id  UUID REFERENCES encumbrances(id),  -- NULL if direct expenditure
    amount          NUMERIC(18,2) NOT NULL,
    vendor_name     TEXT,
    description     TEXT,
    transaction_date DATE NOT NULL,
    erp_reference   TEXT,                           -- source system transaction ID
    federal_award_id UUID REFERENCES federal_awards(id),  -- for SEFA linkage
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_expenditures_appropriation ON expenditures (appropriation_id);
CREATE INDEX idx_expenditures_fy_period ON expenditures (fiscal_year_id, fiscal_period_id);
CREATE INDEX idx_expenditures_fund_dept ON expenditures (fund_id, dept_id);
CREATE INDEX idx_expenditures_date ON expenditures (transaction_date);
CREATE INDEX idx_expenditures_federal_award ON expenditures (federal_award_id) WHERE federal_award_id IS NOT NULL;

-- Revenue Estimates and Actuals
CREATE TABLE revenues (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    fiscal_period_id UUID REFERENCES fiscal_periods(id),  -- NULL for estimates
    fund_id         UUID NOT NULL REFERENCES funds(id),
    account_id      UUID NOT NULL REFERENCES chart_of_accounts(id),
    revenue_type    TEXT NOT NULL CHECK (revenue_type IN ('estimate', 'actual')),
    amount          NUMERIC(18,2) NOT NULL,
    source          TEXT,                           -- e.g., 'Property Tax', 'Sales Tax', 'Intergovernmental'
    description     TEXT,
    transaction_date DATE,
    erp_reference   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_revenues_fy_fund ON revenues (fiscal_year_id, fund_id);
CREATE INDEX idx_revenues_type ON revenues (revenue_type);
```

## Federal Awards & SEFA

```sql
-- Federal Awards (for SEFA preparation)
CREATE TABLE federal_awards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    cfda_number     TEXT NOT NULL,                  -- Assistance Listing (formerly CFDA) number
    award_name      TEXT NOT NULL,
    federal_agency  TEXT NOT NULL,
    award_number    TEXT,
    award_amount    NUMERIC(18,2),
    pass_through_entity TEXT,                       -- if sub-recipient
    pass_through_number TEXT,
    is_major_programme BOOLEAN NOT NULL DEFAULT false,
    award_period_start DATE,
    award_period_end DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_federal_awards_org ON federal_awards (org_id);
CREATE INDEX idx_federal_awards_cfda ON federal_awards (cfda_number);
```

## Workflow & Approval Tables

```sql
-- Budget Workflow Definitions
CREATE TABLE workflow_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    workflow_type   TEXT NOT NULL CHECK (workflow_type IN ('budget_submission', 'amendment', 'transfer', 'capital_project')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Workflow Steps
CREATE TABLE workflow_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES workflow_templates(id),
    step_order      INT NOT NULL,
    step_name       TEXT NOT NULL,
    approver_role   TEXT NOT NULL,                  -- references role in RBAC
    is_required     BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Workflow Instances (active submissions)
CREATE TABLE workflow_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES workflow_templates(id),
    entity_type     TEXT NOT NULL,                  -- 'budget_version', 'amendment', 'transfer'
    entity_id       UUID NOT NULL,                  -- FK to the entity being approved
    current_step_id UUID REFERENCES workflow_steps(id),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'approved', 'rejected', 'withdrawn')),
    submitted_by    UUID NOT NULL,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wf_instances_status ON workflow_instances (status);

-- Workflow Approvals
CREATE TABLE workflow_approvals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instance_id     UUID NOT NULL REFERENCES workflow_instances(id),
    step_id         UUID NOT NULL REFERENCES workflow_steps(id),
    approver_id     UUID NOT NULL,                  -- FK to users
    decision        TEXT NOT NULL CHECK (decision IN ('approved', 'rejected', 'returned')),
    comments        TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wf_approvals_instance ON workflow_approvals (instance_id);
```

## RBAC & Audit Tables

```sql
-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Roles
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    role_name       TEXT NOT NULL,                  -- e.g., 'budget_officer', 'dept_head', 'finance_director', 'council_member', 'public_viewer'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, role_name)
);

-- User-Role Assignments (scoped to department)
CREATE TABLE user_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    role_id         UUID NOT NULL REFERENCES roles(id),
    dept_id         UUID REFERENCES departments(id),  -- NULL = org-wide
    fund_id         UUID REFERENCES funds(id),         -- NULL = all funds
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, role_id, dept_id, fund_id)
);

CREATE INDEX idx_user_roles_user ON user_roles (user_id);

-- Audit Log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    action          TEXT NOT NULL,                  -- 'create', 'update', 'delete', 'approve', 'reject', 'login'
    entity_type     TEXT NOT NULL,                  -- table name
    entity_id       UUID NOT NULL,
    old_values      JSONB,                          -- previous field values (for updates)
    new_values      JSONB,                          -- new field values
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log (user_id);
CREATE INDEX idx_audit_log_created ON audit_log (created_at);
CREATE INDEX idx_audit_log_org_action ON audit_log (org_id, action);
```

## Budget Document & Reporting Tables

```sql
-- Budget Documents (published budget books, reports)
CREATE TABLE budget_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    budget_version_id UUID REFERENCES budget_versions(id),
    document_type   TEXT NOT NULL CHECK (document_type IN ('budget_book', 'executive_summary', 'department_detail', 'capital_plan', 'sefa', 'acfr', 'transparency_report')),
    title           TEXT NOT NULL,
    format          TEXT NOT NULL CHECK (format IN ('pdf', 'html', 'xbrl', 'csv', 'json')),
    storage_path    TEXT,
    file_size_bytes BIGINT,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    gfoa_award_submitted BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_budget_docs_org_fy ON budget_documents (org_id, fiscal_year_id);

-- Performance Measures (linked to budget items)
CREATE TABLE performance_measures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    dept_id         UUID REFERENCES departments(id),
    programme_id    UUID REFERENCES programmes(id),
    measure_name    TEXT NOT NULL,
    measure_type    TEXT NOT NULL CHECK (measure_type IN ('output', 'outcome', 'efficiency', 'effectiveness')),
    unit_of_measure TEXT NOT NULL,                  -- e.g., 'count', 'percentage', 'dollars', 'days'
    target_value    NUMERIC(18,4),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Performance Measure Results (by fiscal year)
CREATE TABLE performance_measure_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    measure_id      UUID NOT NULL REFERENCES performance_measures(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    fiscal_period_id UUID REFERENCES fiscal_periods(id),
    target_value    NUMERIC(18,4),
    actual_value    NUMERIC(18,4),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (measure_id, fiscal_year_id, fiscal_period_id)
);

CREATE INDEX idx_pm_results_measure ON performance_measure_results (measure_id);
CREATE INDEX idx_pm_results_fy ON performance_measure_results (fiscal_year_id);
```

## ERP Integration Tables

```sql
-- ERP Connector Configurations
CREATE TABLE erp_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    erp_type        TEXT NOT NULL CHECK (erp_type IN ('tyler_munis', 'sap', 'oracle', 'workday', 'infor', 'sage_intacct', 'custom')),
    connection_name TEXT NOT NULL,
    endpoint_url    TEXT,
    auth_type       TEXT NOT NULL CHECK (auth_type IN ('oauth2', 'api_key', 'basic', 'saml')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    sync_status     TEXT DEFAULT 'idle' CHECK (sync_status IN ('idle', 'syncing', 'error', 'completed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ERP Sync Log
CREATE TABLE erp_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES erp_connections(id),
    sync_type       TEXT NOT NULL CHECK (sync_type IN ('full', 'incremental')),
    direction       TEXT NOT NULL CHECK (direction IN ('inbound', 'outbound')),
    records_processed INT NOT NULL DEFAULT 0,
    records_failed  INT NOT NULL DEFAULT 0,
    error_details   TEXT,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_erp_sync_connection ON erp_sync_log (connection_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Configuration | 3 | organisations, fiscal_years, fiscal_periods |
| Fund Accounting | 1 | funds |
| Organisational Structure | 2 | departments, programmes |
| Chart of Accounts | 1 | chart_of_accounts |
| Budget Formulation | 3 | budget_versions, budget_line_items, personnel_budget_items |
| Capital Projects | 2 | capital_projects, capital_project_phases |
| Appropriations & Tracking | 5 | appropriations, appropriation_amendments, encumbrances, expenditures, revenues |
| Federal Awards | 1 | federal_awards |
| Workflow & Approval | 4 | workflow_templates, workflow_steps, workflow_instances, workflow_approvals |
| RBAC & Audit | 4 | users, roles, user_roles, audit_log |
| Documents & Reporting | 3 | budget_documents, performance_measures, performance_measure_results |
| ERP Integration | 2 | erp_connections, erp_sync_log |
| **Total** | **31** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed systems, ERP sync without key collisions, and future multi-tenant sharding without key coordination.

2. **Explicit fund-department-account intersection** — the `budget_line_items` and `appropriations` tables use composite unique constraints on (fund, dept, account, programme) to mirror the government chart of accounts segment structure exactly, preventing duplicate entries at the same budget intersection.

3. **Separate appropriation and budget tables** — `budget_versions` / `budget_line_items` track formulation (what is proposed), while `appropriations` track enacted authority (what was legislatively adopted). This separation reflects the real-world distinction between budget proposals and legal spending authority.

4. **Encumbrance tracking as a first-class entity** — encumbrances are not merely a status on an expenditure; they have their own lifecycle (open, partially liquidated, fully liquidated, cancelled) with computed outstanding amounts, reflecting how government accounting actually works.

5. **COFOG and XBRL codes on chart of accounts** — international classification codes and future GASB digital taxonomy element IDs are stored directly on the chart of accounts, avoiding a separate mapping table and making XBRL export a direct operation.

6. **Federal awards as a separate entity** — SEFA preparation requires linking expenditures to federal awards; the `federal_award_id` on expenditures enables direct SEFA schedule generation without complex mapping logic.

7. **Workflow engine is generic** — a single workflow template/step/instance/approval pattern serves budget submissions, amendments, and transfers, avoiding separate approval tables for each process type.

8. **Audit log uses JSONB for old/new values** — since audit entries span all entity types with varying column structures, JSONB is the pragmatic choice for change capture without creating a separate audit table per entity.

9. **Generated column for outstanding encumbrance amount** — `outstanding_amount` is computed by PostgreSQL as `original_amount - liquidated_amount`, ensuring it is always consistent without application-level calculation.

10. **Multi-tenant via org_id** — every table carries an `org_id` foreign key enabling row-level security for multi-tenant deployment, while keeping a single schema for simpler operations.
