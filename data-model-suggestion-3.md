# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Budget & Appropriations Management · Created: 2026-05-22

## Philosophy

This model uses a core relational backbone for the universal budget concepts (funds, departments, appropriations, expenditures) while pushing jurisdiction-specific, configurable, and rapidly-evolving data into JSONB columns. The relational core guarantees integrity for financial data that must add up correctly; the JSONB extensions provide flexibility for the enormous variation across government types, jurisdictions, and reporting requirements.

Government budgeting is a domain where the core financial logic is universal (appropriation minus encumbrances minus expenditures equals available balance), but the presentation, classification, workflow rules, and reporting requirements vary wildly between a small Texas city, a California county, a New York school district, and a federal agency. A purely relational model must either include columns that are irrelevant to most jurisdictions (COFOG codes are meaningless to a small-town budget officer; OMB object class codes are irrelevant to a city) or undergo constant schema migrations as new jurisdiction types are onboarded. The hybrid approach solves this by keeping universal financial fields relational and putting variable metadata in JSONB.

This pattern is used in production by modern multi-tenant SaaS platforms including Stripe (financial metadata), Shopify (product metafields), and Salesforce (custom fields stored in flexible schema columns). PostgreSQL's JSONB type with GIN indexing provides near-relational query performance for JSONB fields, making this a pragmatic middle ground.

**Best for:** An MVP or early-stage product that needs to support multiple jurisdiction types quickly; multi-tenant SaaS deployments where each tenant has different chart of accounts structures, reporting fields, and compliance requirements; teams that want rapid iteration without constant database migrations.

**Trade-offs:**
- (+) Supports jurisdiction-specific fields without DDL changes — add new metadata via configuration, not migration
- (+) Faster MVP development — core schema covers ~80% of needs; edge cases go in JSONB
- (+) Lower table count than fully normalised model — fewer JOINs for common operations
- (+) PostgreSQL GIN indexes on JSONB provide good query performance for most access patterns
- (+) Easy to extend for new government types (federal, special district, school district) without schema changes
- (-) JSONB fields lack foreign key constraints — referential integrity must be enforced by the application
- (-) Type safety for JSONB data depends on application-level JSON Schema validation, not database constraints
- (-) Complex JSONB queries can be slower than indexed relational columns for high-cardinality filtering
- (-) Schema documentation must explicitly describe JSONB structures — they are not self-documenting in the DDL
- (-) Risk of JSONB becoming a "junk drawer" if not governed by clear conventions

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GASB Fund Accounting | Fund types and categories are relational columns with CHECK constraints; fund-specific GASB reporting metadata is in JSONB `properties` |
| GFOA Chart of Accounts | Core account segments (fund, dept, object) are relational; additional custom segments are in the `custom_segments` JSONB column on the chart of accounts |
| COFOG | COFOG codes are stored in the `classifications` JSONB column on chart of accounts entries — present only for jurisdictions that use international classification |
| DAIMS / GSDM | Federal-specific fields (agency identifier, main account code, sub-function, programme activity) are in the `federal_metadata` JSONB column — absent for non-federal entities |
| OMB Circular A-11 | A-11 budget data elements (object class, programme activity, budget function/sub-function) stored in JSONB for federal agency tenants |
| ISO 8601 | All timestamps use `TIMESTAMPTZ`; fiscal year boundaries stored as ISO 8601 date pairs |
| XBRL / FDTA | `xbrl_mapping` JSONB column on chart of accounts entries supports future GASB taxonomy alignment without schema changes |
| SEFA | Federal award metadata on expenditures uses a `grant_info` JSONB column, avoiding a mandatory relationship for non-grant governments |

---

## Core Organisation & Configuration

```sql
-- Tenant / Organisation
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    org_type        TEXT NOT NULL CHECK (org_type IN ('state', 'county', 'city', 'town', 'special_district', 'school_district', 'authority', 'federal_agency')),
    jurisdiction    TEXT,                           -- ISO 3166-2 code
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    -- Jurisdiction-specific configuration in JSONB
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "fiscal_year_start_month": 10,             -- October for federal, July for many states
    --   "budget_basis": "modified_accrual",         -- or "full_accrual", "cash"
    --   "uses_biennial_budget": false,
    --   "uses_encumbrance_accounting": true,
    --   "gfoa_award_eligible": true,
    --   "fedramp_required": false,
    --   "chart_of_accounts_segments": ["fund", "dept", "programme", "object", "sub_object"],
    --   "custom_segments": ["project", "grant", "activity"],
    --   "budget_document_types": ["budget_book", "executive_summary", "capital_plan"],
    --   "approval_levels": ["department_head", "finance_director", "city_manager", "council"],
    --   "identifiers": {
    --     "fips_code": "06037",
    --     "uei": "ABC123DEF456",
    --     "duns": "123456789"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisations_type ON organisations (org_type);
CREATE INDEX idx_organisations_settings ON organisations USING gin (settings);

-- Fiscal Years
CREATE TABLE fiscal_years (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    label           TEXT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'planning' CHECK (status IN ('planning', 'proposed', 'adopted', 'amended', 'closed')),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties:
    -- {
    --   "is_biennial": false,
    --   "budget_message": "Fiscal responsibility with community investment",
    --   "key_assumptions": {
    --     "inflation_rate": 0.032,
    --     "property_tax_growth": 0.045,
    --     "population_growth": 0.012
    --   },
    --   "legislative_references": ["Ordinance 2026-001"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, label)
);
```

## Fund & Organisational Structure

```sql
-- Funds
CREATE TABLE funds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
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
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties:
    -- {
    --   "revenue_sources": ["property_tax", "sales_tax", "intergovernmental"],
    --   "restricted_purpose": "Parks and recreation capital improvements",
    --   "bond_info": {
    --     "issue_date": "2024-06-15",
    --     "maturity_date": "2044-06-15",
    --     "par_amount": 25000000
    --   },
    --   "gasb_54_classification": "committed"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, fund_code)
);

CREATE INDEX idx_funds_org ON funds (org_id);
CREATE INDEX idx_funds_properties ON funds USING gin (properties);

-- Departments (with hierarchy support)
CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    parent_id       UUID REFERENCES departments(id),
    dept_code       TEXT NOT NULL,
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties:
    -- {
    --   "head_name": "Jane Smith",
    --   "head_email": "jsmith@city.gov",
    --   "division": "Public Safety",
    --   "location": "City Hall, 3rd Floor",
    --   "fte_authorised": 45,
    --   "mission_statement": "..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, dept_code)
);

CREATE INDEX idx_departments_org ON departments (org_id);
CREATE INDEX idx_departments_parent ON departments (parent_id);
```

## Chart of Accounts (Flexible Segments)

```sql
-- Chart of Accounts with flexible segment structure
CREATE TABLE chart_of_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    account_code    TEXT NOT NULL,                  -- full segmented code
    account_name    TEXT NOT NULL,
    account_type    TEXT NOT NULL CHECK (account_type IN ('revenue', 'expenditure', 'asset', 'liability', 'fund_balance')),
    -- Standard segments (relational for fast filtering)
    fund_segment    TEXT,
    dept_segment    TEXT,
    object_segment  TEXT,
    -- Custom segments and classifications in JSONB
    custom_segments JSONB NOT NULL DEFAULT '{}',
    -- Example custom_segments:
    -- {
    --   "programme": "4110",
    --   "sub_object": "001",
    --   "project": "CIP-2026-003",
    --   "grant": "COPS-2025-042",
    --   "activity": "patrol"
    -- }
    classifications JSONB NOT NULL DEFAULT '{}',
    -- Example classifications:
    -- {
    --   "cofog_division": "03",
    --   "cofog_group": "031",
    --   "cofog_class": "0310",
    --   "object_class_code": "11.1",               -- OMB A-11 object class
    --   "budget_function": "750",                   -- federal budget function
    --   "budget_sub_function": "751",
    --   "xbrl_element_id": "gasb:PublicSafetyExpenditures",
    --   "sefa_category": "direct_federal"
    -- }
    parent_id       UUID REFERENCES chart_of_accounts(id),
    hierarchy_level INT NOT NULL DEFAULT 0,
    is_posting      BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, account_code)
);

CREATE INDEX idx_coa_org_type ON chart_of_accounts (org_id, account_type);
CREATE INDEX idx_coa_fund_dept ON chart_of_accounts (fund_segment, dept_segment);
CREATE INDEX idx_coa_object ON chart_of_accounts (object_segment);
CREATE INDEX idx_coa_custom ON chart_of_accounts USING gin (custom_segments);
CREATE INDEX idx_coa_classifications ON chart_of_accounts USING gin (classifications);
CREATE INDEX idx_coa_parent ON chart_of_accounts (parent_id);
```

## Budget Formulation

```sql
-- Budget Versions / Scenarios
CREATE TABLE budget_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    version_name    TEXT NOT NULL,
    version_type    TEXT NOT NULL CHECK (version_type IN ('base', 'departmental_request', 'executive_recommendation', 'legislative_adopted', 'amended', 'scenario')),
    is_adopted      BOOLEAN NOT NULL DEFAULT false,
    parent_version_id UUID REFERENCES budget_versions(id),
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties (for scenarios):
    -- {
    --   "scenario_assumptions": {
    --     "revenue_adjustment": -0.10,
    --     "affected_sources": ["property_tax"],
    --     "description": "10% property tax shortfall scenario"
    --   },
    --   "comparison_base_version_id": "uuid"
    -- }
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, fiscal_year_id, version_name)
);

CREATE INDEX idx_bv_fiscal_year ON budget_versions (fiscal_year_id);

-- Budget Line Items (core financial data relational; narrative and extras in JSONB)
CREATE TABLE budget_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_version_id UUID NOT NULL REFERENCES budget_versions(id),
    fund_id         UUID NOT NULL REFERENCES funds(id),
    dept_id         UUID NOT NULL REFERENCES departments(id),
    account_id      UUID NOT NULL REFERENCES chart_of_accounts(id),
    -- Core amounts (relational for aggregation and constraints)
    prior_year_actual    NUMERIC(18,2) DEFAULT 0,
    current_year_budget  NUMERIC(18,2) DEFAULT 0,
    requested_amount     NUMERIC(18,2) DEFAULT 0,
    recommended_amount   NUMERIC(18,2) DEFAULT 0,
    adopted_amount       NUMERIC(18,2) DEFAULT 0,
    -- Extended data in JSONB
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "justification": "Three new officers needed for community policing initiative",
    --   "programme_id": "uuid",
    --   "programme_code": "4110",
    --   "monthly_phasing": [12500, 12500, 12500, 12500, 12500, 12500, 12500, 12500, 12500, 12500, 12500, 12500],
    --   "funding_sources": [
    --     {"source": "General Fund", "amount": 120000},
    --     {"source": "COPS Grant", "amount": 30000}
    --   ],
    --   "performance_links": [
    --     {"measure": "Response Time", "target": "< 5 minutes", "unit": "minutes"}
    --   ],
    --   "personnel": [
    --     {
    --       "position_title": "Police Officer",
    --       "position_number": "PD-042",
    --       "fte": 1.0,
    --       "base_salary": 42000,
    --       "benefits_rate": 0.35,
    --       "total_compensation": 56700,
    --       "is_new": true,
    --       "pay_grade": "G12",
    --       "step": "1"
    --     }
    --   ],
    --   "capital_project": {
    --     "project_code": "CIP-2026-003",
    --     "project_name": "Fire Station #5 Renovation",
    --     "phase": "Construction",
    --     "total_project_cost": 8500000,
    --     "prior_year_spent": 1200000,
    --     "future_year_planned": 4800000
    --   },
    --   "notes": "Council priority item",
    --   "attachments": [
    --     {"name": "staffing_analysis.pdf", "storage_key": "s3://..."}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (budget_version_id, fund_id, dept_id, account_id)
);

CREATE INDEX idx_bli_version ON budget_line_items (budget_version_id);
CREATE INDEX idx_bli_fund_dept ON budget_line_items (fund_id, dept_id);
CREATE INDEX idx_bli_account ON budget_line_items (account_id);
CREATE INDEX idx_bli_details ON budget_line_items USING gin (details);
```

## Appropriations & Expenditure Tracking

```sql
-- Appropriations
CREATE TABLE appropriations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    fund_id         UUID NOT NULL REFERENCES funds(id),
    dept_id         UUID NOT NULL REFERENCES departments(id),
    account_id      UUID NOT NULL REFERENCES chart_of_accounts(id),
    -- Core financial data (relational)
    original_amount NUMERIC(18,2) NOT NULL,
    amended_amount  NUMERIC(18,2) NOT NULL,
    -- Extended metadata in JSONB
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "authority_type": "annual",
    --   "legislation_ref": "Ordinance 2026-042",
    --   "effective_date": "2025-10-01",
    --   "expiration_date": null,
    --   "programme_id": "uuid",
    --   "programme_code": "4110",
    --   "federal_account": {
    --     "agency_id": "015",
    --     "main_account_code": "1000",
    --     "sub_account_code": "001"
    --   },
    --   "amendments": [
    --     {
    --       "id": "uuid",
    --       "type": "increase",
    --       "amount": 25000,
    --       "reason": "Mid-year supplemental",
    --       "ordinance_ref": "Ordinance 2026-078",
    --       "approved_by": "uuid",
    --       "approved_date": "2026-03-15"
    --     }
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (fiscal_year_id, fund_id, dept_id, account_id)
);

CREATE INDEX idx_approp_fy_fund ON appropriations (fiscal_year_id, fund_id);
CREATE INDEX idx_approp_dept ON appropriations (dept_id);
CREATE INDEX idx_approp_metadata ON appropriations USING gin (metadata);

-- Encumbrances
CREATE TABLE encumbrances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    appropriation_id UUID NOT NULL REFERENCES appropriations(id),
    encumbrance_type TEXT NOT NULL CHECK (encumbrance_type IN ('purchase_order', 'contract', 'requisition', 'travel_authorisation')),
    reference_number TEXT NOT NULL,
    original_amount NUMERIC(18,2) NOT NULL,
    liquidated_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    outstanding_amount NUMERIC(18,2) GENERATED ALWAYS AS (original_amount - liquidated_amount) STORED,
    status          TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'partially_liquidated', 'fully_liquidated', 'cancelled')),
    encumbered_date DATE NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "vendor_name": "Acme Office Supplies",
    --   "vendor_id": "V-10042",
    --   "contract_number": "C-2026-0015",
    --   "description": "Annual office supply contract",
    --   "delivery_date": "2026-04-01",
    --   "erp_reference": "MUNIS-PO-44521"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_enc_appropriation ON encumbrances (appropriation_id);
CREATE INDEX idx_enc_status ON encumbrances (status) WHERE status IN ('open', 'partially_liquidated');
CREATE INDEX idx_enc_details ON encumbrances USING gin (details);

-- Expenditures (actuals from ERP)
CREATE TABLE expenditures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    appropriation_id UUID NOT NULL REFERENCES appropriations(id),
    encumbrance_id  UUID REFERENCES encumbrances(id),
    amount          NUMERIC(18,2) NOT NULL,
    transaction_date DATE NOT NULL,
    fiscal_period   INT NOT NULL,                   -- 1-12 (or 1-13 for closing period)
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "vendor_name": "Acme Office Supplies",
    --   "description": "Q1 office supplies delivery",
    --   "invoice_number": "INV-2026-03344",
    --   "check_number": "CHK-88921",
    --   "erp_reference": "MUNIS-TX-889921",
    --   "object_class_code": "25.3",
    --   "cofog_division": "01",
    --   "grant_info": {
    --     "cfda_number": "16.710",
    --     "award_name": "COPS Hiring Program",
    --     "award_number": "2025-CK-WX-0042",
    --     "federal_agency": "Department of Justice",
    --     "pass_through_entity": null,
    --     "is_major_programme": false
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_exp_appropriation ON expenditures (appropriation_id);
CREATE INDEX idx_exp_date ON expenditures (transaction_date);
CREATE INDEX idx_exp_period ON expenditures (fiscal_period);
CREATE INDEX idx_exp_encumbrance ON expenditures (encumbrance_id) WHERE encumbrance_id IS NOT NULL;
CREATE INDEX idx_exp_details ON expenditures USING gin (details);

-- Revenues
CREATE TABLE revenues (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    fund_id         UUID NOT NULL REFERENCES funds(id),
    account_id      UUID NOT NULL REFERENCES chart_of_accounts(id),
    revenue_type    TEXT NOT NULL CHECK (revenue_type IN ('estimate', 'actual')),
    amount          NUMERIC(18,2) NOT NULL,
    fiscal_period   INT,                           -- NULL for estimates
    transaction_date DATE,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "source": "Property Tax",
    --   "description": "Q1 property tax collections",
    --   "erp_reference": "MUNIS-RV-112233"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rev_fy_fund ON revenues (fiscal_year_id, fund_id);
CREATE INDEX idx_rev_type ON revenues (revenue_type);
```

## Workflow & Approvals

```sql
-- Workflow Definitions (configurable per org via JSONB)
CREATE TABLE workflows (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    workflow_type   TEXT NOT NULL CHECK (workflow_type IN ('budget_submission', 'amendment', 'transfer', 'capital_project')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    definition      JSONB NOT NULL,
    -- Example definition:
    -- {
    --   "steps": [
    --     {
    --       "order": 1,
    --       "name": "Department Head Review",
    --       "approver_role": "dept_head",
    --       "scope": "department",
    --       "required": true,
    --       "auto_approve_below": 5000,
    --       "sla_hours": 72
    --     },
    --     {
    --       "order": 2,
    --       "name": "Finance Director Review",
    --       "approver_role": "finance_director",
    --       "scope": "organisation",
    --       "required": true,
    --       "sla_hours": 120
    --     },
    --     {
    --       "order": 3,
    --       "name": "City Manager Approval",
    --       "approver_role": "city_manager",
    --       "scope": "organisation",
    --       "required": true,
    --       "sla_hours": 168
    --     }
    --   ],
    --   "notifications": {
    --     "on_submit": ["finance_director"],
    --     "on_approve": ["submitter", "next_approver"],
    --     "on_reject": ["submitter", "dept_head"]
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Workflow Instances (tracking active approvals)
CREATE TABLE workflow_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID NOT NULL REFERENCES workflows(id),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'approved', 'rejected', 'withdrawn')),
    submitted_by    UUID NOT NULL,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    history         JSONB NOT NULL DEFAULT '[]',
    -- Example history:
    -- [
    --   {
    --     "step": 1,
    --     "step_name": "Department Head Review",
    --     "approver_id": "uuid",
    --     "approver_name": "Jane Smith",
    --     "decision": "approved",
    --     "comments": "Staffing request aligns with strategic plan",
    --     "decided_at": "2026-02-10T14:30:00Z"
    --   },
    --   {
    --     "step": 2,
    --     "step_name": "Finance Director Review",
    --     "approver_id": "uuid",
    --     "approver_name": "Bob Jones",
    --     "decision": "approved",
    --     "comments": "Within departmental allocation",
    --     "decided_at": "2026-02-12T09:15:00Z"
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wfi_status ON workflow_instances (status);
CREATE INDEX idx_wfi_entity ON workflow_instances (entity_type, entity_id);
CREATE INDEX idx_wfi_history ON workflow_instances USING gin (history);
```

## Users, RBAC & Audit

```sql
-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "title": "Finance Director",
    --   "phone": "555-0142",
    --   "notification_preferences": {
    --     "email": true,
    --     "in_app": true,
    --     "digest_frequency": "daily"
    --   },
    --   "auth_provider": "saml",
    --   "last_login_at": "2026-05-20T08:30:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Roles (with permissions in JSONB)
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    role_name       TEXT NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions:
    -- [
    --   {"resource": "budget_line_items", "actions": ["read", "create", "update"], "scope": "department"},
    --   {"resource": "appropriations", "actions": ["read"], "scope": "organisation"},
    --   {"resource": "reports", "actions": ["read", "export"], "scope": "organisation"},
    --   {"resource": "workflows", "actions": ["approve"], "scope": "department"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, role_name)
);

-- User-Role Assignments
CREATE TABLE user_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    role_id         UUID NOT NULL REFERENCES roles(id),
    dept_id         UUID REFERENCES departments(id),
    fund_id         UUID REFERENCES funds(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, role_id, dept_id, fund_id)
);

CREATE INDEX idx_user_roles_user ON user_roles (user_id);

-- Audit Log (structured events with flexible payload)
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    user_id         UUID,
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,                          -- {field: {old: ..., new: ...}} for updates
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example context:
    -- {
    --   "ip_address": "10.0.1.42",
    --   "user_agent": "Mozilla/5.0 ...",
    --   "session_id": "uuid",
    --   "workflow_instance_id": "uuid"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log (user_id);
CREATE INDEX idx_audit_time ON audit_log (created_at);
CREATE INDEX idx_audit_changes ON audit_log USING gin (changes);
```

## Documents & Reports

```sql
-- Budget Documents
CREATE TABLE budget_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    budget_version_id UUID REFERENCES budget_versions(id),
    document_type   TEXT NOT NULL,
    title           TEXT NOT NULL,
    format          TEXT NOT NULL,
    storage_path    TEXT,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "file_size_bytes": 2456789,
    --   "page_count": 142,
    --   "gfoa_award_submitted": true,
    --   "gfoa_sections_covered": ["policy_document", "financial_plan", "operations_guide", "communications_device"],
    --   "generated_by_ai": true,
    --   "generation_prompt_id": "uuid",
    --   "xbrl_output_included": false
    -- }
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_docs_org_fy ON budget_documents (org_id, fiscal_year_id);
```

## Capital Projects

```sql
-- Capital Projects (heavy use of JSONB for variable project data)
CREATE TABLE capital_projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    project_code    TEXT NOT NULL,
    name            TEXT NOT NULL,
    dept_id         UUID REFERENCES departments(id),
    status          TEXT NOT NULL DEFAULT 'proposed' CHECK (status IN ('proposed', 'approved', 'in_progress', 'completed', 'cancelled')),
    total_estimated_cost NUMERIC(18,2),
    -- All variable project data in JSONB
    project_data    JSONB NOT NULL DEFAULT '{}',
    -- Example project_data:
    -- {
    --   "description": "Complete renovation of Fire Station #5 including seismic retrofit",
    --   "priority_score": 85,
    --   "priority_criteria": {
    --     "public_safety": 25,
    --     "deferred_maintenance": 20,
    --     "strategic_plan_alignment": 20,
    --     "grant_leverage": 10,
    --     "community_impact": 10
    --   },
    --   "location": {"address": "123 Main St", "lat": 34.0522, "lng": -118.2437},
    --   "start_date": "2026-07-01",
    --   "expected_completion": "2028-06-30",
    --   "phases": [
    --     {"name": "Design", "fiscal_year": "FY2026", "fund_code": "400", "amount": 850000, "funding_source": "Capital Projects Fund"},
    --     {"name": "Construction", "fiscal_year": "FY2027", "fund_code": "400", "amount": 5200000, "funding_source": "Bond Proceeds"},
    --     {"name": "Equipment", "fiscal_year": "FY2028", "fund_code": "400", "amount": 2450000, "funding_source": "Capital Projects Fund + FEMA Grant"}
    --   ],
    --   "funding_breakdown": [
    --     {"source": "Capital Projects Fund", "amount": 4500000},
    --     {"source": "GO Bond Series 2025", "amount": 3000000},
    --     {"source": "FEMA BRIC Grant", "amount": 1000000}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, project_code)
);

CREATE INDEX idx_cp_org ON capital_projects (org_id);
CREATE INDEX idx_cp_status ON capital_projects (status);
CREATE INDEX idx_cp_data ON capital_projects USING gin (project_data);
```

## ERP Integration

```sql
-- ERP Connections
CREATE TABLE erp_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    erp_type        TEXT NOT NULL,
    connection_name TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "endpoint_url": "https://erp.city.gov/api/v2",
    --   "auth_type": "oauth2",
    --   "sync_schedule": "0 */4 * * *",
    --   "sync_direction": "bidirectional",
    --   "field_mappings": {
    --     "fund_code": "GL_FUND",
    --     "dept_code": "GL_ORG",
    --     "account_code": "GL_ACCOUNT",
    --     "amount": "AMOUNT",
    --     "transaction_date": "POST_DATE"
    --   },
    --   "last_sync_at": "2026-05-20T04:00:00Z",
    --   "last_sync_status": "completed",
    --   "last_sync_records": 1542
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Performance Measures

```sql
-- Performance Measures (flexible structure for varied measurement types)
CREATE TABLE performance_measures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    dept_id         UUID REFERENCES departments(id),
    measure_name    TEXT NOT NULL,
    measure_type    TEXT NOT NULL CHECK (measure_type IN ('output', 'outcome', 'efficiency', 'effectiveness')),
    unit_of_measure TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    definition      JSONB NOT NULL DEFAULT '{}',
    -- Example definition:
    -- {
    --   "description": "Average emergency response time from dispatch to arrival",
    --   "data_source": "CAD system",
    --   "calculation_method": "Mean of all Priority 1 response times",
    --   "reporting_frequency": "monthly",
    --   "strategic_goal": "Community Safety",
    --   "programme_code": "4110"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Performance Results (by fiscal year and period)
CREATE TABLE performance_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    measure_id      UUID NOT NULL REFERENCES performance_measures(id),
    fiscal_year_id  UUID NOT NULL REFERENCES fiscal_years(id),
    fiscal_period   INT,
    target_value    NUMERIC(18,4),
    actual_value    NUMERIC(18,4),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (measure_id, fiscal_year_id, fiscal_period)
);

CREATE INDEX idx_pr_measure ON performance_results (measure_id);
CREATE INDEX idx_pr_fy ON performance_results (fiscal_year_id);
```

---

## Example JSONB Queries

### Find All Budget Items with Personnel Detail

```sql
SELECT
    bli.id,
    f.fund_code,
    d.dept_code,
    coa.account_code,
    bli.requested_amount,
    bli.details->'personnel' AS personnel_detail,
    jsonb_array_length(bli.details->'personnel') AS position_count
FROM budget_line_items bli
JOIN funds f ON f.id = bli.fund_id
JOIN departments d ON d.id = bli.dept_id
JOIN chart_of_accounts coa ON coa.id = bli.account_id
WHERE bli.budget_version_id = :version_id
  AND bli.details ? 'personnel'                    -- has personnel key
  AND jsonb_array_length(bli.details->'personnel') > 0;
```

### Filter Expenditures by Federal Award (SEFA Preparation)

```sql
SELECT
    e.amount,
    e.transaction_date,
    e.details->'grant_info'->>'cfda_number' AS cfda_number,
    e.details->'grant_info'->>'award_name' AS award_name,
    e.details->'grant_info'->>'federal_agency' AS federal_agency
FROM expenditures e
WHERE e.org_id = :org_id
  AND e.details @> '{"grant_info": {}}'            -- has grant_info key
  AND e.details->'grant_info' ? 'cfda_number'      -- with CFDA number
ORDER BY e.details->'grant_info'->>'cfda_number', e.transaction_date;
```

### Aggregate by COFOG Classification

```sql
SELECT
    coa.classifications->>'cofog_division' AS cofog_division,
    coa.classifications->>'cofog_group' AS cofog_group,
    SUM(e.amount) AS total_expenditures
FROM expenditures e
JOIN appropriations a ON a.id = e.appropriation_id
JOIN chart_of_accounts coa ON coa.id = a.account_id
WHERE e.org_id = :org_id
  AND coa.classifications ? 'cofog_division'
GROUP BY
    coa.classifications->>'cofog_division',
    coa.classifications->>'cofog_group'
ORDER BY cofog_division, cofog_group;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Configuration | 2 | organisations, fiscal_years |
| Fund & Structure | 2 | funds, departments |
| Chart of Accounts | 1 | chart_of_accounts (custom segments in JSONB) |
| Budget Formulation | 2 | budget_versions, budget_line_items (personnel detail in JSONB) |
| Appropriations & Tracking | 4 | appropriations, encumbrances, expenditures, revenues |
| Workflow | 2 | workflows (definition in JSONB), workflow_instances (history in JSONB) |
| Users & RBAC | 3 | users, roles (permissions in JSONB), user_roles |
| Audit | 1 | audit_log |
| Documents | 1 | budget_documents |
| Capital Projects | 1 | capital_projects (phases in JSONB) |
| ERP Integration | 1 | erp_connections (config in JSONB) |
| Performance | 2 | performance_measures, performance_results |
| **Total** | **22** | Approximately 30% fewer tables than normalized model |

---

## Key Design Decisions

1. **JSONB for jurisdiction-variable data, relational for financial arithmetic** — amounts that must be summed, subtracted, or compared (appropriation, encumbrance, expenditure) are always relational `NUMERIC(18,2)` columns. Metadata that varies by jurisdiction or record type goes in JSONB. This ensures financial integrity while providing flexibility.

2. **Personnel detail embedded in budget line item JSONB** — rather than a separate `personnel_budget_items` table, position-level detail is stored in the `details` JSONB column of `budget_line_items`. This avoids an empty table for governments that do not use position-based budgeting while preserving full detail for those that do.

3. **Capital project phases in JSONB** — multi-year capital project phasing varies enormously across governments. Some have 3 phases, others have 20. Storing phases as a JSONB array avoids the complexity of a separate phases table with varying column requirements.

4. **Workflow definition in JSONB** — workflow steps, notification rules, and SLA thresholds are configuration data that varies per organisation. Storing the workflow definition as a JSONB document makes adding new workflow types a configuration change, not a schema migration.

5. **Approval history in workflow instance JSONB** — the full approval history (who approved, when, with what comments) is stored as a JSONB array on the workflow instance rather than a separate approvals table. This keeps the approval chain as a single document for easy display and audit.

6. **Classifications as a separate JSONB column on chart of accounts** — COFOG codes, OMB object class codes, XBRL element IDs, and SEFA categories are in a dedicated `classifications` JSONB column. This separates "how this account is classified for reporting" from "what segments define this account", making it easy to add new classification systems without touching the core structure.

7. **GIN indexes on all JSONB columns** — every JSONB column has a GIN index, enabling containment queries (`@>`), existence checks (`?`), and path-based filtering at reasonable performance. The trade-off is slightly slower writes and larger index sizes.

8. **Federal metadata only when relevant** — fields like `federal_account`, `budget_function`, and `programme_activity` that are only relevant to federal agencies live in JSONB. A city government never sees these fields in their data, avoiding confusion and wasted storage.

9. **ERP field mappings in connection config** — rather than a separate field mapping table, the ERP connection's `config` JSONB includes a `field_mappings` object that maps local field names to ERP field names. This makes onboarding a new ERP type a configuration exercise.

10. **Permissions as JSONB arrays on roles** — rather than a separate permissions table with resource/action pairs, permissions are stored as JSONB arrays on the `roles` table. This makes permission sets self-contained and easily copyable between organisations.
