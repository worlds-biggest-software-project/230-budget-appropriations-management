# Budget & Appropriations Management — Phased Development Plan

> Project: `230-budget-appropriations-management` · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.
> Research basis: README.md, research.md, features.md, standards.md, data-model-suggestion-1.md (primary), data-model-suggestion-2.md (audit pattern borrowed).

---

## Project Summary

An AI-native, open-source platform for government budget formulation, appropriations tracking, and public-facing budget reporting. Target users are state and local government finance directors, budget officers, legislative staff, and federal agency budget analysts who today rely on fragmented ERP modules, expensive specialist SaaS, or spreadsheets.

The MVP delivers: budget formulation with departmental submission and multi-level approval workflow; multi-fund, multi-year, and scenario (what-if) modelling; personnel cost planning; appropriations monitoring with encumbrance tracking; GFOA-aligned budget document export; role-based access, audit trail, and security controls.

The differentiating AI-native capabilities (narrative drafting, natural-language query, real-time variance alerting, automated SEFA mapping) ship in Phases 7–10, after the operational core is stable.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | **Python 3.12** | Chart-of-accounts modelling, GFOA narrative generation, XBRL/SEFA tooling, and LLM orchestration all have first-class Python libraries (Pydantic, openpyxl, lxml, openai, anthropic, instructor). Government finance integrators are comfortable reading Python. |
| API framework | **FastAPI 0.115+** | Native Pydantic v2 integration produces typed request/response models and auto-generates OpenAPI 3.1 docs (matches `standards.md` requirement). ASGI for async ERP calls and SSE streaming for AI features. |
| ORM | **SQLAlchemy 2.0 (async) + Alembic** | Mature, predictable migrations are essential for a financial schema that government IT teams will audit. SQLAlchemy 2.0's typed Core/ORM aligns with Pydantic v2 model conversion. |
| Database | **PostgreSQL 16** | Required for: `NUMERIC(18,2)` exact arithmetic on monetary amounts, `GENERATED ALWAYS AS … STORED` columns (used for encumbrance `outstanding_amount` per data-model-suggestion-1), `JSONB` audit payloads, declarative partitioning (audit log and expenditures by fiscal year), Row-Level Security for multi-tenant `org_id` isolation, and `tsvector` full-text search for budget narratives and document search. |
| Cache / queue broker | **Redis 7** | Backs RQ task queue, FastAPI response cache, websocket pub/sub for real-time variance alerts, and short-lived rate-limit counters for public transparency portal. |
| Task queue | **RQ (Redis Queue) 2.x** | Simpler ops than Celery; sufficient for periodic ERP sync, document generation jobs, AI narrative drafting, and projection rebuilds. Includes `rq-scheduler` for cron-style jobs (nightly SEFA refresh, daily appropriation snapshots). |
| Authentication | **Authlib + OIDC**; SAML 2.0 via `python3-saml` | OIDC for cloud SaaS deployments; SAML 2.0 for government enterprise SSO into Active Directory (per `standards.md`). Local password auth for dev/self-hosted. |
| Authorization | **Casbin (pycasbin)** | Government RBAC must scope by `(role, dept_id, fund_id)` triples; Casbin's ABAC-style policy DSL handles this declaratively and externally to code, making audit easier. |
| LLM provider abstraction | **`instructor` + Anthropic Claude (default), OpenAI fallback** | Structured outputs are essential for narrative generation that must match GFOA templates exactly. Configurable provider lets agencies choose FedRAMP-authorised endpoints. |
| Frontend | **Next.js 15 (App Router) + React 19 + TypeScript** | Two distinct UIs needed: internal Finance Console (forms, workflows, dashboards) and public Transparency Portal (interactive charts, SEO-indexable budget pages). Server Components produce accessible HTML for WCAG 2.1 AA compliance (Section 508). |
| UI components | **shadcn/ui + Tailwind CSS** | Accessible primitives based on Radix UI satisfy WCAG 2.1 AA; consistent visual system across both consoles. |
| Charting (transparency portal) | **Recharts + D3 for sunburst/treemap** | Recharts covers 90% of budget visualisations (bar, line, stacked area); D3 handles the GFOA-style fund-department-programme sunburst diagram. |
| PDF generation | **WeasyPrint 62+** | Pure-Python HTML→PDF that handles GFOA budget book layouts (cover pages, table of contents, multi-page tables, page headers/footers, signatures). Avoids native binary dependencies for self-hosted deployments. |
| XBRL output | **Arelle (CLI) wrapped via subprocess** | The de facto open-source XBRL processor; used by SEC and many government regulators. Produces FDTA-aligned XBRL instances using the GASB taxonomy (when finalised). |
| Containerisation | **Docker + docker-compose**, multi-stage builds | Self-hostable deployment for jurisdictions without cloud authority. The compose stack runs Postgres + Redis + API + worker + web. |
| Testing | **pytest 8 + pytest-asyncio + httpx**; **Playwright** for E2E browser tests | pytest for unit/integration; Playwright tests the full Finance Console workflows (departmental submission → approval → adoption). |
| Code quality | **Ruff** (lint+format), **mypy --strict**, **pre-commit hooks** | Strict typing on financial code is non-negotiable. Ruff replaces black+flake8+isort with a single tool. |
| Package management | **uv** (Python), **pnpm** (frontend) | uv is dramatically faster than pip for CI and produces deterministic lockfiles. pnpm for the Next.js workspace. |
| API client generation | **openapi-typescript** | Generates typed TS clients from FastAPI's OpenAPI 3.1 spec, eliminating drift between backend and frontends. |
| Observability | **OpenTelemetry** SDK; logs to JSON via `structlog` | Government agencies often standardise on OTel-compatible collectors. Structured logs enable SIEM ingestion. |
| Secret management | **`pydantic-settings` reading from env vars / `.env`**; AWS Secrets Manager / Azure Key Vault adapters for cloud | Twelve-factor config; cloud secret backends optional. |
| Key Python libraries | `pydantic`, `sqlalchemy[asyncio]`, `alembic`, `asyncpg`, `redis`, `rq`, `authlib`, `casbin`, `python3-saml`, `instructor`, `anthropic`, `openai`, `openpyxl`, `weasyprint`, `jinja2`, `httpx`, `structlog`, `opentelemetry-sdk`, `python-jose[cryptography]`, `passlib[bcrypt]` | Each is widely used, actively maintained, and licence-compatible (MIT/BSD/Apache 2). |

### Project Structure

```
budget-appropriations/
├── README.md
├── LICENSE                              # Apache 2.0 (TBD per README)
├── pyproject.toml                       # uv-managed; pinned via uv.lock
├── uv.lock
├── Dockerfile                           # multi-stage: builder, api, worker
├── docker-compose.yml                   # postgres + redis + api + worker + web
├── docker-compose.dev.yml
├── .env.example
├── .pre-commit-config.yaml
├── .github/
│   └── workflows/
│       ├── ci.yml                       # lint, type-check, unit, integration
│       └── release.yml
├── docs/
│   ├── architecture.md
│   ├── deployment-self-hosted.md
│   ├── deployment-cloud.md
│   ├── gfoa-template-guide.md
│   └── api/                             # auto-generated OpenAPI HTML
├── alembic/
│   ├── env.py
│   └── versions/
├── src/
│   └── budget_app/
│       ├── __init__.py
│       ├── main.py                      # FastAPI app entrypoint
│       ├── settings.py                  # pydantic-settings
│       ├── db.py                        # async engine, session factory
│       ├── deps.py                      # FastAPI dependencies (get_session, get_current_user)
│       ├── domain/                      # SQLAlchemy ORM models, grouped by aggregate
│       │   ├── organisation.py          # organisations, fiscal_years, fiscal_periods
│       │   ├── fund.py                  # funds
│       │   ├── department.py            # departments, programmes
│       │   ├── chart_of_accounts.py
│       │   ├── budget.py                # budget_versions, budget_line_items, personnel_budget_items
│       │   ├── capital.py               # capital_projects, capital_project_phases
│       │   ├── appropriation.py         # appropriations, appropriation_amendments, encumbrances, expenditures, revenues
│       │   ├── federal_award.py
│       │   ├── workflow.py              # workflow_templates, _steps, _instances, _approvals
│       │   ├── rbac.py                  # users, roles, user_roles
│       │   ├── audit.py                 # audit_log
│       │   ├── document.py              # budget_documents
│       │   ├── performance.py           # performance_measures, performance_measure_results
│       │   └── erp.py                   # erp_connections, erp_sync_log
│       ├── schemas/                     # Pydantic v2 request/response models
│       │   ├── common.py                # Money, FiscalYearLabel, etc.
│       │   ├── budget.py
│       │   ├── appropriation.py
│       │   └── ...
│       ├── api/                         # FastAPI routers
│       │   ├── v1/
│       │   │   ├── __init__.py          # APIRouter aggregation
│       │   │   ├── auth.py
│       │   │   ├── organisations.py
│       │   │   ├── funds.py
│       │   │   ├── departments.py
│       │   │   ├── chart_of_accounts.py
│       │   │   ├── budgets.py
│       │   │   ├── appropriations.py
│       │   │   ├── encumbrances.py
│       │   │   ├── expenditures.py
│       │   │   ├── capital_projects.py
│       │   │   ├── workflows.py
│       │   │   ├── documents.py
│       │   │   ├── reports.py           # GFOA, SEFA, XBRL, CSV/JSON export
│       │   │   ├── transparency.py      # public read-only endpoints
│       │   │   ├── ai.py                # narrative, query, alerts
│       │   │   └── erp.py
│       ├── services/                    # business logic, transaction boundaries
│       │   ├── budgeting.py             # formulation rules, version branching
│       │   ├── appropriations.py        # balance calculation, amendment posting
│       │   ├── encumbrance.py
│       │   ├── workflow_engine.py
│       │   ├── scenario.py              # what-if model executor
│       │   ├── rbac.py                  # Casbin enforcer wrapper
│       │   ├── audit.py                 # write audit_log entries
│       │   ├── documents.py             # PDF / XBRL / CSV producers
│       │   ├── sefa.py
│       │   ├── ai/
│       │   │   ├── narrative.py         # GFOA narrative drafting
│       │   │   ├── query.py             # NL→SQL with safety
│       │   │   ├── variance_agent.py    # appropriations monitor
│       │   │   └── prompts/             # versioned prompt templates
│       │   └── erp/
│       │       ├── base.py              # ErpConnector ABC
│       │       ├── tyler_munis.py
│       │       ├── oracle_epm.py
│       │       ├── sap.py
│       │       ├── workday.py
│       │       └── csv_loader.py        # fallback file-based connector
│       ├── tasks/                       # RQ job functions
│       │   ├── erp_sync.py
│       │   ├── document_generation.py
│       │   ├── narrative_generation.py
│       │   ├── variance_scan.py
│       │   └── projection_refresh.py
│       ├── reports/
│       │   ├── templates/               # Jinja2 GFOA templates
│       │   │   ├── budget_book.html.j2
│       │   │   ├── executive_summary.html.j2
│       │   │   ├── department_detail.html.j2
│       │   │   ├── capital_plan.html.j2
│       │   │   └── sefa.html.j2
│       │   └── styles/
│       │       └── gfoa.css
│       ├── auth/
│       │   ├── oidc.py
│       │   ├── saml.py
│       │   └── jwt.py
│       ├── observability/
│       │   ├── logging.py               # structlog config
│       │   └── tracing.py               # OTel setup
│       └── seed/
│           ├── cofog_codes.py           # UN COFOG seed loader
│           ├── omb_object_classes.py
│           └── sample_jurisdiction.py   # demo city for development
├── tests/
│   ├── conftest.py                      # async db fixtures, factory-boy factories
│   ├── factories/
│   ├── unit/
│   │   ├── services/
│   │   ├── domain/
│   │   └── schemas/
│   ├── integration/
│   │   ├── api/
│   │   ├── workflows/
│   │   └── erp/
│   ├── e2e/                             # Playwright-driven
│   │   ├── playwright.config.ts
│   │   └── tests/
│   └── fixtures/
│       ├── chart_of_accounts/
│       ├── sample_budgets/
│       └── sefa/
├── web/
│   ├── package.json
│   ├── pnpm-workspace.yaml
│   ├── apps/
│   │   ├── console/                     # internal Finance Console (Next.js)
│   │   │   ├── app/
│   │   │   │   ├── (auth)/
│   │   │   │   ├── budgets/
│   │   │   │   ├── appropriations/
│   │   │   │   ├── capital/
│   │   │   │   ├── workflows/
│   │   │   │   ├── reports/
│   │   │   │   └── settings/
│   │   │   └── components/
│   │   └── portal/                      # public Transparency Portal (Next.js)
│   │       ├── app/
│   │       │   ├── budget/[year]/
│   │       │   ├── department/[code]/
│   │       │   ├── search/
│   │       │   └── ask/                 # NL query UI
│   │       └── components/
│   └── packages/
│       ├── api-client/                  # openapi-typescript-generated
│       └── ui/                          # shared shadcn/ui components
└── scripts/
    ├── load_cofog.py
    ├── load_omb_object_classes.py
    ├── load_sample_jurisdiction.py
    └── rebuild_projections.py           # used in Phase 6+
```

---

## Phase 1: Foundation, Tenancy, and Reference Data

### Purpose
Establish the project skeleton, deterministic dev environment, multi-tenant data isolation pattern, and the static reference data (organisations, fiscal years, fund types, COFOG codes, OMB object classes) that every later phase depends on. After this phase the codebase boots, runs migrations against a containerised Postgres, and exposes a health endpoint plus authenticated CRUD for organisations and fiscal years.

### Tasks

#### 1.1 — Repository skeleton, tooling, and CI

**What**: Create the project layout, dependency manifests, pre-commit hooks, and a GitHub Actions CI pipeline that runs on every push.

**Design**:
- `pyproject.toml` declares Python 3.12, runtime deps (FastAPI, SQLAlchemy[asyncio], asyncpg, alembic, pydantic, pydantic-settings, structlog, redis, rq, authlib, casbin), and dev deps (pytest, pytest-asyncio, httpx, ruff, mypy, factory-boy, pytest-cov).
- `uv.lock` checked in; CI uses `uv sync --frozen`.
- `pyproject.toml` configures Ruff (`line-length = 100`, full default rule set plus `B`, `I`, `UP`, `SIM`, `RUF`) and mypy (`strict = true`, `plugins = ["pydantic.mypy", "sqlalchemy.ext.mypy.plugin"]`).
- `.pre-commit-config.yaml` runs `ruff check --fix`, `ruff format`, `mypy src/`.
- `.github/workflows/ci.yml` jobs: `lint` (ruff), `typecheck` (mypy), `test-unit` (pytest -m "not integration"), `test-integration` (Postgres + Redis services in GHA, runs `-m integration`).
- Multi-stage `Dockerfile`: stage 1 builds wheels with `uv`; stage 2 (`api`) runs uvicorn; stage 3 (`worker`) runs `rq worker`.
- `docker-compose.yml` services: `postgres:16-alpine` (with healthcheck), `redis:7-alpine`, `api`, `worker`, `web-console`, `web-portal`.

**Testing**:
- Unit: `tests/unit/test_smoke.py::test_imports` — every module under `budget_app.*` imports without error.
- Unit: `tests/unit/test_settings.py::test_defaults` — `Settings()` with no env vars loads sensible defaults and raises `ValidationError` if `DATABASE_URL` is unset in non-test env.
- CI dry-run: `ruff check`, `mypy src/`, `pytest -q` all green on empty stubs.

#### 1.2 — Settings, database engine, and session lifecycle

**What**: Centralise configuration via `pydantic-settings`, expose an async SQLAlchemy engine, and provide a FastAPI dependency that yields a transactional session per request.

**Design**:
```python
# src/budget_app/settings.py
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="BUDGET_")
    database_url: PostgresDsn
    redis_url: RedisDsn
    secret_key: SecretStr
    jwt_alg: Literal["HS256", "RS256"] = "HS256"
    access_token_ttl_seconds: int = 3600
    environment: Literal["dev", "test", "staging", "prod"] = "dev"
    cors_origins: list[AnyHttpUrl] = []
    llm_provider: Literal["anthropic", "openai", "none"] = "none"
    anthropic_api_key: SecretStr | None = None
    openai_api_key: SecretStr | None = None
```

```python
# src/budget_app/db.py
engine = create_async_engine(str(settings.database_url), echo=False, pool_pre_ping=True)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with SessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

- Health endpoint `GET /healthz` returns `{"status": "ok", "db": "ok", "redis": "ok"}` with a 200 if all dependencies respond.

**Testing**:
- Unit: `test_settings_required_fields` — missing `BUDGET_DATABASE_URL` → `ValidationError`.
- Integration (real Postgres in container): `test_health_endpoint` — `httpx.AsyncClient` GET `/healthz` returns 200 and `db == "ok"`.
- Integration: `test_session_rollback_on_exception` — within a route that raises, no row is persisted.

#### 1.3 — Domain models for tenancy and fiscal calendar

**What**: Implement SQLAlchemy ORM models for `organisations`, `fiscal_years`, and `fiscal_periods` per data-model-suggestion-1 §"Core Organisation & Configuration Tables", plus the initial Alembic migration.

**Design**:
- Use SQLAlchemy 2.0 declarative `Mapped[]` syntax.
- `OrgType = Literal["state", "county", "city", "town", "special_district", "school_district", "authority", "federal_agency"]` enforced via `CheckConstraint` and Pydantic schema enum.
- Composite `UNIQUE(org_id, label)` on `fiscal_years`; `UNIQUE(fiscal_year_id, period_number, period_type)` on `fiscal_periods`.
- `Organisation.uei`: 12-character regex `^[A-HJ-NP-Z0-9]{12}$` (SAM.gov UEI format).
- `FiscalYear.status` state machine: `planning → proposed → adopted → amended → closed`; transitions validated in service layer (Phase 2).
- Helper method `FiscalYear.generate_periods(period_type: Literal["month", "quarter"])` that creates 12 or 4 `FiscalPeriod` rows.

**Testing**:
- Unit: `test_organisation_uei_format` — invalid UEI rejected by Pydantic schema; valid UEI accepted.
- Unit: `test_fiscal_year_overlap_detection` — creating two FY records with overlapping dates in same org returns validation error (custom validator).
- Integration: `test_alembic_upgrade_head_then_downgrade_base` — runs the migration up and back down cleanly on a fresh database.
- Integration: `test_generate_monthly_periods` — `fy = FiscalYear(start=2025-10-01, end=2026-09-30); fy.generate_periods("month")` produces 12 contiguous `FiscalPeriod` rows.

#### 1.4 — Row-Level Security and tenant context

**What**: Enable PostgreSQL RLS on every multi-tenant table so the active `org_id` (set as `SET LOCAL app.current_org = '<uuid>'`) gates all queries.

**Design**:
- Each tenant table gets `ALTER TABLE ... ENABLE ROW LEVEL SECURITY;` and `CREATE POLICY tenant_isolation ON <table> USING (org_id = current_setting('app.current_org', true)::uuid);`.
- `app/deps.py::get_session` does `await session.execute(text("SET LOCAL app.current_org = :oid"), {"oid": str(user.org_id)})` immediately after acquiring the session, when authenticated context is available.
- A `superuser` role used by background jobs sets `app.current_org` per task rather than relying on auth.
- Document the rule: **every** new table added in later phases must (a) include `org_id UUID NOT NULL REFERENCES organisations(id)` and (b) have an RLS policy. CI lint rule (custom mypy plugin or grep-based check) enforces this.

**Testing**:
- Integration: `test_rls_blocks_cross_tenant_read` — insert org A and org B fiscal_year rows; session set to org A returns only A's rows.
- Integration: `test_rls_blocks_cross_tenant_write` — attempt to UPDATE org B row while session is org A → zero rows affected.
- Integration: `test_superuser_bypass` — connecting as the migration role with `BYPASSRLS` sees all rows (needed for Alembic).

#### 1.5 — Seed reference data: COFOG, OMB object classes, GASB fund types

**What**: Provide bootstrap loaders for UN COFOG codes (10 divisions × ~70 groups × ~140 classes), OMB Circular A-11 Section 83 object classes, and the GASB fund taxonomy.

**Design**:
- `scripts/load_cofog.py`: parses bundled `data/cofog.json` (committed; derived from UNSD CSV) and upserts into `ref_cofog_codes(division TEXT, group TEXT NULL, class TEXT NULL, label TEXT, PRIMARY KEY(division, group, class))`.
- `scripts/load_omb_object_classes.py`: loads `data/omb_object_classes.json` (codes like `11.1` Full-time permanent, `25.3` Other purchases of goods and services from federal sources, etc.) into `ref_object_class_codes(code TEXT PRIMARY KEY, title TEXT, category TEXT)`.
- Loaders are idempotent (`ON CONFLICT DO UPDATE`).
- Make target `make seed-reference` runs both.

**Testing**:
- Unit: `test_cofog_loader_idempotent` — running loader twice leaves exactly one row per (division, group, class).
- Unit: `test_cofog_division_count` — exactly 10 division-level rows post-load.
- Fixture-based: `test_omb_object_class_categories` — known codes (`11.1` → category `personnel`) match expected mapping.

#### 1.6 — Sample jurisdiction seeder for development

**What**: One command that produces a realistic demo dataset: "City of Riverside" with 4 funds, 8 departments, ~80 chart-of-accounts rows, a prior-year adopted budget, and a current-year departmental request.

**Design**:
- `scripts/load_sample_jurisdiction.py` uses factory-boy factories from `tests/factories/` to create the org and child entities, then commits.
- Idempotent: detects existing demo org by name and short-circuits.
- Used by Playwright E2E tests in later phases.

**Testing**:
- Integration: `test_seed_sample_jurisdiction` — running the seeder twice produces the same row counts.

---

## Phase 2: Funds, Departments, Programmes, and Chart of Accounts

### Purpose
Implement the structural data that every budget line item references: funds (GASB taxonomy), departmental hierarchy, programmes, and the chart of accounts with COFOG/OMB classification metadata. Provide REST CRUD for each and a CSV import path so a jurisdiction can onboard its existing chart in one operation.

### Tasks

#### 2.1 — Funds domain and API

**What**: Implement `funds` table, ORM, Pydantic schemas, and `/api/v1/funds` router with full CRUD.

**Design**:
- ORM matches data-model-suggestion-1 §"Fund Accounting Tables"; `fund_category × fund_type` enum pairs validated by a Python `@validates("fund_type")` ensuring valid combinations (e.g., `general` only allowed when `category == governmental`).
- `GET /api/v1/funds?category=governmental&is_active=true` supports filter query params via FastAPI `Query()` parameters with typed enums.
- `POST /api/v1/funds` requires `budget_officer` or `finance_director` role; enforced via Casbin policy `p, finance_director, funds, write`.
- Soft delete via `is_active = false` rather than DELETE (financial data is never truly deleted).

**Testing**:
- Unit: `test_fund_type_category_mismatch` — `fund_category=governmental, fund_type=enterprise` → `ValidationError`.
- Integration: `test_post_fund_creates_audit_entry` — creating a fund writes a row to `audit_log` (audit hook wired in Phase 5; stubbed here).
- Integration: `test_list_funds_filtered_by_category` — created funds returned only when matching filter.

#### 2.2 — Departments, hierarchy, and programmes

**What**: Implement `departments` (self-referencing parent_id for hierarchy) and `programmes`; API for CRUD and a tree endpoint.

**Design**:
- `GET /api/v1/departments/tree` returns nested JSON `{ id, code, name, children: [...] }` built via single recursive CTE query for efficiency at scale.
- Tree depth capped at 5 (validated on `parent_id` assignment via service-layer query that walks ancestors).
- `Department.head_user_id` is nullable FK; head user receives workflow notifications by default.
- `Programme.dept_id` optional (programmes may span departments).

**Testing**:
- Unit: `test_department_max_depth` — attempting to nest a 6th level → service raises `DepthExceededError`.
- Unit: `test_cycle_detection` — setting `parent_id` such that a cycle would form raises `CycleError`.
- Integration: `test_tree_endpoint_returns_nested` — 3-level hierarchy seeded; response matches expected nested shape.

#### 2.3 — Chart of accounts with classification codes

**What**: Implement `chart_of_accounts` with GFOA segment columns, COFOG codes, OMB object class, and the future `xbrl_element_id` hook.

**Design**:
- ORM per data-model-suggestion-1 §"Chart of Accounts".
- Validators: `cofog_division` must exist in `ref_cofog_codes` if provided; `object_class_code` must exist in `ref_object_class_codes` if provided. Enforced via SQL FK to the reference tables.
- `account_code` regex configurable per organisation in `Organisation.coa_format_regex` (default `^\d{3}-\d{4}-\d{5}$` matching the `fund-dept-object` example).
- `hierarchy_level INT NOT NULL DEFAULT 0` — leaves where `is_posting = true` are the only valid targets for budget line items and expenditures; enforced at service layer.

**Testing**:
- Unit: `test_account_code_regex_validation` — invalid format rejected.
- Unit: `test_non_posting_account_rejected_for_budget_line` — service raises `AccountNotPostableError`.
- Integration: `test_cofog_fk_enforcement` — inserting a chart row with `cofog_division='99'` (unseeded) → IntegrityError.

#### 2.4 — Bulk CSV import for chart of accounts

**What**: A `POST /api/v1/chart-of-accounts/import` endpoint accepting `multipart/form-data` CSV upload and producing a per-row success/failure report.

**Design**:
- CSV columns: `account_code, account_name, account_type, fund_segment, dept_segment, programme_segment, object_segment, sub_object_segment, object_class_code, cofog_division, cofog_group, cofog_class, parent_account_code, is_posting`.
- Two-pass importer (Pandas DataFrame): pass 1 inserts all rows with `parent_id = NULL`; pass 2 resolves parents by `parent_account_code`.
- Returns `ImportReport(total: int, succeeded: int, failed: int, errors: list[RowError])` where `RowError` includes line number, column, and message.
- Wrapped in a single transaction unless `?strict=false`, in which case successful rows are committed and failures returned.

**Testing**:
- Fixture-based: `test_import_sample_chart_csv` — imports `tests/fixtures/chart_of_accounts/riverside_chart.csv` (≈ 80 rows) successfully.
- Unit: `test_import_resolves_parents_in_second_pass` — child rows ordered before parents in CSV still link correctly.
- Unit: `test_import_strict_mode_rolls_back_on_error` — any row failure aborts the entire import.

---

## Phase 3: Budget Formulation Engine

### Purpose
Implement the core value proposition: budget formulation. Departments enter requests against the chart of accounts; finance officers create scenario versions; personnel costs are modelled position-by-position; capital projects span multiple fiscal years. After this phase, a finance team can fully construct a proposed budget without touching a spreadsheet.

### Tasks

#### 3.1 — Budget versions and line items

**What**: Implement `budget_versions` (with parent/child branching for scenarios) and `budget_line_items`; expose `/api/v1/budgets`, `/api/v1/budgets/{id}/line-items`.

**Design**:
- `BudgetVersion.version_type` enum: `base, departmental_request, executive_recommendation, legislative_adopted, amended, scenario`.
- `BudgetVersion.parent_version_id` (self-FK) enables "fork from existing version" — used by scenarios and by promoting a recommendation into an adopted version.
- `POST /api/v1/budgets/{id}/clone` body `{new_name: str, version_type: VersionType}` → server-side bulk insert copying all `budget_line_items` to the new version with amounts carried forward; runs in a single transaction.
- `BudgetLineItem` columns and unique constraint match data-model-suggestion-1 §"Budget Formulation Tables".
- Computed property `BudgetLineItem.variance_pct = (requested - current_year_budget) / current_year_budget * 100`.

**Testing**:
- Unit: `test_clone_version_copies_all_line_items` — 50-line source version cloned → target has 50 lines with identical amounts.
- Unit: `test_clone_only_carries_requested_to_new_version` — `recommended_amount` and `adopted_amount` reset to 0 on cloned version.
- Integration: `test_unique_constraint_blocks_duplicate_line` — two POSTs with identical `(version, fund, dept, account, programme)` → second returns 409.
- Integration: `test_bulk_create_line_items` — `POST /budgets/{id}/line-items/bulk` with 1000 rows completes in <2s and produces correct row count.

#### 3.2 — Personnel budget items (position-based budgeting)

**What**: Implement `personnel_budget_items` for FTE-level salary and benefits modelling tied to a budget version.

**Design**:
- ORM per data-model-suggestion-1 §"Personnel Budget Items".
- `total_compensation` computed at write time: `base_salary * fte_count + benefits_amount` (server-calculated; clients send `base_salary, fte_count, benefits_rate` and the service computes the rest).
- `is_new_position` and `is_vacant` flags drive narrative templates ("X new positions requested for FY2026") in Phase 7.
- `POST /api/v1/budgets/{id}/personnel/rollup` aggregates personnel costs by `(fund, dept, account)` and either creates or updates the matching `budget_line_items` rows so the operating budget reflects the personnel detail without double-entry.

**Testing**:
- Unit: `test_total_compensation_calculation` — `base=50000, fte=1.5, benefits_rate=0.35` → `total_compensation = 50000*1.5*(1+0.35) = 101250`.
- Unit: `test_zero_fte_rejected` — `fte_count = 0` → `ValidationError`.
- Integration: `test_personnel_rollup_updates_line_items` — 3 personnel rows in dept 4100 with total comp 300k → matching account 51100 line item updated to 300k.

#### 3.3 — Capital projects and multi-year phasing

**What**: Implement `capital_projects` and `capital_project_phases`; provide priority scoring and a multi-year projection view.

**Design**:
- ORM per data-model-suggestion-1 §"Capital Projects".
- `CapitalProject.priority_score INT CHECK (priority_score BETWEEN 1 AND 100)`.
- `GET /api/v1/capital-projects/projection?start_fy=FY2026&years=5` returns 5-year matrix: `[{project_code, project_name, totals_by_year: {FY2026: 1.2M, FY2027: 4.5M, ...}}]`.
- Capital project funding sources free-text now; constrained to enum in Phase 6 when bonds tracking lands.

**Testing**:
- Unit: `test_priority_score_bounds` — `priority_score=150` → `ValidationError`.
- Integration: `test_5_year_projection_matrix` — project with phases in FY2026/FY2027/FY2028 → matrix columns include those years with correct amounts.

#### 3.4 — Scenario modelling (what-if engine)

**What**: A `POST /api/v1/budgets/{id}/scenarios` endpoint that creates a child `budget_version` with line items adjusted by a declarative `ScenarioInput`.

**Design**:
```python
class ScenarioAdjustment(BaseModel):
    target: Literal["fund", "department", "account_type", "all"]
    target_value: str | None         # e.g., "100" for General Fund
    operation: Literal["multiply", "add", "set"]
    amount_or_factor: Decimal        # e.g., 0.90 for 10% cut, or -50000 for $50k reduction

class ScenarioInput(BaseModel):
    name: str
    description: str
    adjustments: list[ScenarioAdjustment]
    base_amount_field: Literal["requested_amount", "recommended_amount"] = "requested_amount"
```
- Service iterates the source version's line items, applies adjustments in order, writes a new `budget_version` (`version_type="scenario"`, `parent_version_id=<source>`) and matching `budget_line_items`.
- Read-only after creation; users create a new scenario rather than mutate.

**Testing**:
- Unit: `test_scenario_multiply_general_fund_by_0_9` — base 100 lines, GF lines reduced by 10%, others unchanged.
- Unit: `test_scenario_set_overrides_existing` — `operation=set` replaces base amount exactly.
- Integration: `test_scenario_creation_immutable` — `PUT /line-items/{id}` on scenario line returns 403.

---

## Phase 4: Appropriations, Encumbrances, and Expenditures

### Purpose
Bridge from formulation (proposed budget) to execution (legally-adopted spending authority and actual transactions). This phase implements the heart of government fiscal control: an appropriation is a legal authority; encumbrances reserve part of that authority; expenditures consume it; the balance equation must always hold.

### Tasks

#### 4.1 — Appropriations and amendments

**What**: Implement `appropriations`, `appropriation_amendments` (per data-model-suggestion-1 §"Appropriations & Expenditure Tracking"), and the service that posts amendments while preserving original amounts.

**Design**:
- `Appropriation.original_amount` immutable after creation; `amended_amount` mutated only via `appropriation_amendments` rows (enforced by the service — direct UPDATE blocked via DB role permissions or SQLAlchemy event hook).
- `POST /api/v1/appropriations/{id}/amendments` body `{amendment_type, amount, reason, ordinance_ref, transfer_to_id?}`; transfer pairs link to each other via `transfer_pair_id`.
- The service runs the amendment and the `amended_amount` update atomically in one transaction.
- `GET /api/v1/appropriations?fund_id=...&dept_id=...` supports the segment filters every government user needs.
- Adopting a `budget_version` (`version_type="legislative_adopted"`) auto-creates an `appropriation` per matching line item via `POST /api/v1/budgets/{id}/adopt` — see 4.3.

**Testing**:
- Unit: `test_amendment_updates_amended_amount` — original 100k, increase 25k → `amended_amount=125k`.
- Unit: `test_transfer_pair_balances` — transfer 10k from A to B → A net `-10k`, B net `+10k`; pair links bidirectionally.
- Integration: `test_direct_update_to_original_amount_rejected` — SQLAlchemy event hook raises `OriginalAmountImmutableError`.

#### 4.2 — Encumbrances with computed outstanding amount

**What**: Implement `encumbrances` with the `outstanding_amount` generated column and lifecycle status machine.

**Design**:
- DDL exactly per data-model-suggestion-1: `outstanding_amount NUMERIC(18,2) GENERATED ALWAYS AS (original_amount - liquidated_amount) STORED`.
- Status machine: `open → partially_liquidated → fully_liquidated`; `open → cancelled`. Transitions enforced by service.
- `POST /api/v1/encumbrances/{id}/liquidate` body `{amount, expenditure_id?}` updates `liquidated_amount`; if equal to original, status flips to `fully_liquidated`.
- Validation: appropriation `available_balance` must cover the encumbrance amount; balance computed as `appropriation.amended_amount - (sum(open encumbrances) + sum(expenditures))`. Reject with `InsufficientAuthorityError` otherwise.

**Testing**:
- Unit: `test_outstanding_amount_computed` — encumbrance with original=10k, liquidated=3k → outstanding=7k via DB read.
- Unit: `test_partial_liquidation_status_change` — liquidate 4k of 10k → status `partially_liquidated`.
- Integration: `test_overbudget_encumbrance_rejected` — appropriation 100k, existing 95k encumbered, new 10k request → `InsufficientAuthorityError`.

#### 4.3 — Budget adoption: bridge from formulation to authority

**What**: `POST /api/v1/budgets/{version_id}/adopt` that converts a `legislative_adopted` budget version into `appropriations` rows.

**Design**:
- Idempotent: re-adopting the same version returns existing appropriations rather than duplicating.
- For each `budget_line_item` with `adopted_amount > 0`, creates a corresponding `appropriations` row with matching `(fund, dept, account, programme)` and `original_amount = amended_amount = adopted_amount`.
- `BudgetVersion.is_adopted = true`; FiscalYear status transitions to `adopted`.
- Body accepts `{effective_date, legislation_ref}` applied to every appropriation.

**Testing**:
- Integration: `test_adopt_creates_appropriations` — 50-line adopted version → 50 appropriation rows.
- Integration: `test_adoption_is_idempotent` — calling adopt twice produces 50 rows (not 100).
- Integration: `test_zero_amount_lines_not_appropriated` — line with `adopted_amount=0` does not create an appropriation.

#### 4.4 — Expenditures and revenues

**What**: Implement `expenditures` (with optional encumbrance linkage and `federal_award_id` for SEFA) and `revenues`.

**Design**:
- ORM per data-model-suggestion-1 §"Appropriations & Expenditure Tracking".
- `POST /api/v1/expenditures` body validates that, if `encumbrance_id` is set, the amount does not exceed remaining outstanding on that encumbrance; in that case it also calls `encumbrances.liquidate` in the same transaction.
- `expenditures.transaction_date` indexed; range queries for period reports are common.
- Revenues split between `estimate` (`fiscal_period_id` nullable) and `actual` (period required).

**Testing**:
- Unit: `test_expenditure_against_encumbrance_liquidates` — expenditure 4k against encumbrance with 7k outstanding → outstanding becomes 3k.
- Unit: `test_direct_expenditure_without_encumbrance_allowed` — `encumbrance_id=None` accepted.
- Integration: `test_expenditure_exceeds_encumbrance_rejected` — amount 10k against 7k outstanding → 422.

#### 4.5 — Appropriation balance API

**What**: Real-time `GET /api/v1/appropriations/{id}/balance` returning the full balance breakdown.

**Design**:
- Returns `AppropriationBalance(original, amended, total_encumbered, total_expended, available, pct_consumed)`.
- Implemented as a single SQL query that joins encumbrances and expenditures with sum aggregates (no materialised view yet; Phase 6 may add caching).
- `GET /api/v1/funds/{id}/balance-summary?fy=FY2026` returns rolled-up totals across the fund's appropriations.

**Testing**:
- Integration: `test_balance_with_no_activity` — fresh 100k appropriation → available=100k, pct_consumed=0.
- Integration: `test_balance_with_mixed_activity` — 100k appropriation, 30k encumbered, 20k expended → available=50k, pct_consumed=50.

---

## Phase 5: RBAC, Workflow Engine, and Audit Trail

### Purpose
Wire authentication, fine-grained authorisation, the budget-submission workflow engine, and a comprehensive audit log. Government finance systems are auditable by construction; this phase ensures every state change has an attributed actor and reason.

### Tasks

#### 5.1 — User, role, and policy model

**What**: Implement `users`, `roles`, `user_roles` (scoped by dept/fund) and a Casbin enforcer wrapping all authorisation checks.

**Design**:
- ORM per data-model-suggestion-1 §"RBAC & Audit Tables".
- Built-in roles seeded per organisation: `budget_officer`, `dept_head`, `finance_director`, `analyst`, `council_member`, `auditor`, `public_viewer`.
- Casbin model:
  ```
  [request_definition]
  r = sub, dom, obj, act
  [policy_definition]
  p = sub, dom, obj, act, eft
  [role_definition]
  g = _, _, _
  [matchers]
  m = g(r.sub, p.sub, r.dom) && r.obj == p.obj && r.act == p.act
  ```
  where `dom = (org_id, dept_id?, fund_id?)` and `obj/act` are resource/action strings.
- Policies stored in `casbin_rule` table (managed by Casbin's `SqlalchemyAdapter`).

**Testing**:
- Unit: `test_dept_head_can_only_submit_own_dept` — dept_head scoped to dept A → `enforce(user, "city-x", "budget:dept-A", "write") == True`; `enforce(user, "city-x", "budget:dept-B", "write") == False`.
- Unit: `test_finance_director_can_recommend_all` — finance_director can write recommended amounts on any dept's line items.
- Integration: `test_public_endpoint_no_auth` — `GET /api/v1/transparency/*` returns 200 without bearer token.

#### 5.2 — Authentication: OIDC, SAML, local

**What**: Provide three authentication adapters and JWT issuance.

**Design**:
- `POST /api/v1/auth/login` (local password — bcrypt-hashed) returns `{access_token, refresh_token, expires_in}`.
- `GET /api/v1/auth/oidc/login` and `/callback` — Authlib flow with PKCE.
- `POST /api/v1/auth/saml/acs` — SAML 2.0 Assertion Consumer; `python3-saml` validates the assertion against the IdP metadata.
- JWT claims include `sub` (user_id), `org_id`, `roles[]`, `dept_scope[]`, `fund_scope[]`, `exp`.
- Middleware `auth_required` decodes the JWT, sets `request.state.user`, and `SET LOCAL app.current_org` on the session.

**Testing**:
- Unit: `test_password_hash_roundtrip` — `verify(plain, hash(plain)) == True`.
- Unit: `test_jwt_expiry` — token issued with `expires_in=1` is rejected after sleeping 2s.
- Integration (mocked IdP): `test_oidc_callback_creates_user_if_missing` — IdP returns claims for new email → user provisioned with default `public_viewer` role.

#### 5.3 — Workflow engine

**What**: Implement `workflow_templates`, `workflow_steps`, `workflow_instances`, `workflow_approvals` as a generic engine and wire it to budget version submission.

**Design**:
- Template + steps configured per organisation; default template `Budget Submission` has steps: `dept_head_review → finance_director_review → cm_recommendation`.
- `POST /api/v1/workflows/instances` body `{template_id, entity_type, entity_id}` starts an instance; status `pending` → `in_progress` at first approval.
- `POST /api/v1/workflows/instances/{id}/decide` body `{decision: approved|rejected|returned, comments}` writes a `workflow_approvals` row, advances `current_step_id`, and marks complete when last step approved.
- On full approval of a `budget_version` workflow: version transitions from `departmental_request` → `executive_recommendation` (or whatever the template's final transition specifies).
- Email notifications dispatched via Phase 5.5 to each step's `approver_role` holders.

**Testing**:
- Unit: `test_workflow_advances_to_next_step` — 3-step workflow; first approver decides "approved" → `current_step_id` points to step 2.
- Unit: `test_rejection_terminates_workflow` — rejection at any step sets status to `rejected`, no further actions.
- Integration: `test_budget_submission_e2e` — dept_head submits → finance director approves → CM approves → version status `executive_recommendation`.

#### 5.4 — Audit log (immutable)

**What**: Centralised audit logging that records every mutating action with old/new JSONB payloads.

**Design**:
- Implemented as a SQLAlchemy event listener on every domain class registered in `domain.AUDITED_MODELS`. On `after_insert`/`after_update`/`after_delete` writes an `audit_log` row.
- `old_values` and `new_values` use `sqlalchemy.inspect(instance).attrs` to capture only changed columns.
- `audit_log` table has `DELETE` privilege REVOKED at the database role level; only the migration superuser may purge (retention policy).
- Partitioned by month for performance and cheap historical archiving (`PARTITION BY RANGE (created_at)`).
- `GET /api/v1/audit?entity_type=appropriations&entity_id=...` returns paginated history; restricted to `auditor` and `finance_director` roles.

**Testing**:
- Integration: `test_audit_records_appropriation_amendment` — POST amendment → audit_log row with action=update, old/new values populated.
- Integration: `test_audit_log_immutable` — non-superuser DELETE attempt returns permission error.
- Unit: `test_old_values_excludes_unchanged_fields` — updating only `notes` → `old_values` contains only `notes`, not the entire row.

#### 5.5 — Notifications (email + in-app)

**What**: Pluggable notifier that delivers workflow events to in-app feed and email.

**Design**:
- `NotificationChannel(Protocol)` with implementations: `EmailChannel` (SMTP via `aiosmtplib`), `InAppChannel` (writes to `notifications` table polled by the console UI).
- Templates in `src/budget_app/notifications/templates/` (Jinja2): `workflow_step_assigned.html`, `variance_alert.html`, `document_published.html`.
- Notification dispatch queued via RQ to avoid blocking the HTTP request.

**Testing**:
- Unit: `test_notification_dispatch_enqueues_job` — workflow approval → RQ `notifications.dispatch` job enqueued.
- Integration (fake SMTP via `aiosmtpd`): `test_email_rendered_and_sent` — captured message subject matches "Workflow step assigned: Finance Director Review".

---

## Phase 6: ERP/General Ledger Connector Framework

### Purpose
Provide a pluggable framework for syncing chart-of-accounts and actual transactions between this platform and the agency's existing ERP. Includes connectors for Tyler Munis, Oracle EPM Cloud, SAP, Workday, and a CSV/SFTP fallback for jurisdictions without API-capable ERPs.

### Tasks

#### 6.1 — ErpConnector ABC and connection management

**What**: Define the connector interface and `erp_connections`/`erp_sync_log` tables; implement the dispatcher that loads the right connector based on `erp_type`.

**Design**:
```python
# src/budget_app/services/erp/base.py
class ErpConnector(ABC):
    @abstractmethod
    async def test_connection(self) -> ConnectionTestResult: ...
    @abstractmethod
    async def fetch_chart_of_accounts(self, since: datetime | None) -> AsyncIterator[CoAReadModel]: ...
    @abstractmethod
    async def fetch_expenditures(self, fy: str, since: datetime | None) -> AsyncIterator[ExpenditureReadModel]: ...
    @abstractmethod
    async def fetch_revenues(self, fy: str, since: datetime | None) -> AsyncIterator[RevenueReadModel]: ...
    @abstractmethod
    async def push_adopted_appropriations(self, fy: str) -> PushResult: ...
```
- `ErpConnectionRegistry` is a simple dict `{"tyler_munis": TylerMunisConnector, ...}`.
- Credentials encrypted at rest using Fernet keys derived from `BUDGET_SECRET_KEY`; never returned in API responses.

**Testing**:
- Unit: `test_credentials_encrypted_in_db` — connection inserted with `api_key="secret"`; raw row in DB shows ciphertext.
- Integration: `test_connection_dispatch` — `erp_type="tyler_munis"` returns `TylerMunisConnector` instance.

#### 6.2 — CSV/SFTP fallback connector

**What**: A `csv_loader` connector that reads expenditure and revenue files from a configured SFTP path or local upload.

**Design**:
- Config: `{"sftp_host", "sftp_user", "sftp_path", "expenditure_glob", "revenue_glob"}`.
- File format documented in `docs/erp-csv-spec.md`; columns: `transaction_date, fund_code, dept_code, account_code, programme_code, amount, vendor_name, description, erp_reference`.
- Idempotent ingestion keyed on `(erp_reference, transaction_date)`.

**Testing**:
- Fixture-based: `test_csv_loader_ingests_sample_file` — `tests/fixtures/erp/expenditures_FY2026_Q1.csv` (200 rows) → 200 expenditures created.
- Unit: `test_csv_loader_skips_duplicates` — re-running ingestion on same file produces 0 new rows.
- Unit: `test_csv_loader_unknown_account_fails_row` — row with non-existent `account_code` written to `erp_sync_log.error_details` with row number.

#### 6.3 — Tyler Munis connector (reference implementation)

**What**: REST-based connector against Tyler Nexus integration platform.

**Design**:
- OAuth 2.0 client-credentials flow; `client_id` + `client_secret` stored encrypted.
- Endpoint mapping table in code: Tyler resource → internal entity. Configurable per Tyler version.
- Rate limiting respects Tyler's documented quotas via `aiolimiter`.
- Bidirectional sync: outbound `push_adopted_appropriations` posts to Tyler's budget-load endpoint.

**Testing**:
- Unit (mocked HTTPX): `test_tyler_auth_token_refresh` — when 401 received, connector refreshes token and retries once.
- Unit (mocked HTTPX): `test_tyler_chart_pagination` — multi-page response correctly yields all items.
- Integration (real Tyler sandbox; marked `@pytest.mark.real_erp`): runs against the publicly available Tyler demo tenant if `BUDGET_TYLER_TEST_URL` is set.

#### 6.4 — Oracle EPM, SAP, Workday connectors (skeletons)

**What**: Implement the same `ErpConnector` interface with thin wrappers around each vendor's REST API; full implementations completed iteratively post-MVP.

**Design**:
- Each connector follows Tyler's pattern: OAuth 2.0, rate-limited, paginated, with vendor-specific endpoint maps.
- Initial release implements `test_connection` and `fetch_chart_of_accounts` for each; remaining methods stubbed with `NotImplementedError` and tracked as backlog.

**Testing**:
- Unit (mocked): `test_oracle_test_connection_success` — mocked 200 from `/aif/rest/V1/ping` → `ConnectionTestResult(ok=True)`.
- Unit (mocked): `test_sap_oauth_flow_steps` — verifies the auth request sequence.

#### 6.5 — Scheduled sync jobs

**What**: Background jobs that run scheduled syncs per `erp_connections` row.

**Design**:
- `rq-scheduler` cron expression per connection; default nightly at 02:00 local.
- `erp_sync_log` written at start and completion of each sync; failures retry with exponential backoff.
- Admin endpoint `POST /api/v1/erp/connections/{id}/sync` triggers an ad-hoc sync.

**Testing**:
- Integration (using `fakeredis` and frozen time): `test_scheduled_sync_runs_at_cron_time` — advancing time triggers the job.
- Integration: `test_sync_failure_logged_with_retry_count` — connector raises → log row shows `error_details`, retry count increments on next attempt.

---

## Phase 7: AI — Narrative Generation, Variance Alerts, NL Query

### Purpose
Deliver the AI-native differentiators: GFOA-compliant narrative drafting from structured data, real-time variance alerting via an autonomous agent, and a natural-language query interface for non-technical stakeholders. These features are positioned as core architecture, not bolt-ons.

### Tasks

#### 7.1 — LLM provider abstraction

**What**: A single `LlmClient` interface backed by configurable providers; structured output via `instructor`.

**Design**:
```python
class LlmClient(Protocol):
    async def generate(
        self, system: str, user: str, *, response_model: type[BaseModel] | None = None,
        max_tokens: int = 2000, temperature: float = 0.2
    ) -> Any: ...

class AnthropicClient(LlmClient): ...
class OpenAIClient(LlmClient): ...
class NullClient(LlmClient):  # used when llm_provider=none for offline tests
    async def generate(self, *args, **kwargs): raise LLMDisabledError()
```
- Provider chosen by `settings.llm_provider`; allows agencies to point at FedRAMP-authorised endpoints (Azure OpenAI on Gov Cloud, Anthropic Bedrock).
- All LLM calls logged to `llm_call_log` table: `prompt_hash, response_hash, prompt_tokens, completion_tokens, latency_ms, model`. Enables cost reporting and audit.

**Testing**:
- Unit: `test_provider_factory_returns_correct_implementation` — settings switch returns matching class.
- Unit: `test_null_client_raises_when_disabled` — `settings.llm_provider="none"` → calls raise `LLMDisabledError`.

#### 7.2 — GFOA narrative generation

**What**: `POST /api/v1/ai/narratives/department/{dept_id}?fy=FY2026` produces a draft department budget narrative aligned to GFOA Distinguished Budget Presentation Award criteria.

**Design**:
- Pipeline: gather structured inputs (prior year actuals, current year budget, requested amount, % changes, new positions count, capital projects) → render via Jinja2 prompt template → LLM call with `instructor` returning `DepartmentNarrative` Pydantic model with required GFOA sections (Mission, Goals, Major Initiatives, Budget Highlights, Performance Measures).
- Prompt template:
  ```
  SYSTEM: You are a government budget officer preparing a GFOA Distinguished Budget
  Presentation Award-compliant narrative. Use formal, neutral tone. Cite all numeric
  figures explicitly. Never invent data not present in <budget_data>.

  USER: <budget_data>{json}</budget_data>
        <dept_context>{prior_narratives}</dept_context>
        <gfoa_criteria_excerpt>{criteria}</gfoa_criteria_excerpt>
        Produce DepartmentNarrative.
  ```
- Output stored as `budget_documents(document_type="department_detail", format="html")` for review and editing before publication.
- Drafted narratives are *suggestions*; user must accept and may edit. Tracked via `narrative_drafts` table with `status: draft|accepted|rejected`.

**Testing**:
- Unit (mocked LLM): `test_narrative_input_assembly` — given a dept with known budget data, prompt contains the expected numeric facts.
- Unit: `test_narrative_validates_no_hallucinated_numbers` — post-processing scans the narrative for numeric tokens not in `budget_data` → flagged as warnings.
- Integration (real LLM, marked `@pytest.mark.llm`): `test_narrative_end_to_end_sample_dept` — generates narrative for the demo Police Department; asserts all required GFOA sections present.

#### 7.3 — Variance alerting agent

**What**: A scheduled agent that scans all open appropriations and emits alerts when consumption trajectories indicate imminent overrun.

**Design**:
- RQ scheduled job `variance_scan` runs hourly.
- For each appropriation: compute current `pct_consumed`, days elapsed in fiscal year, projected end-of-year consumption (`pct_consumed * (365 / days_elapsed)`).
- Trigger conditions (configurable per organisation):
  - `pct_consumed > 80` and `month < 9` (in 12-month FY) → severity `warning`.
  - `projected_eoy > 100` → severity `critical`.
  - `recent 30-day burn rate > 2x trailing average` → severity `anomaly`.
- For each trigger, ask LLM (if enabled) to draft a one-paragraph explanation and recommended action; store as `variance_alerts(severity, message, recommended_action, llm_drafted: bool)`.
- Alerts surfaced in console dashboard and pushed via notifications.

**Testing**:
- Unit: `test_projection_math` — pct=50%, 3 months in → projected EOY = 200% → severity `critical`.
- Unit: `test_alert_dedupe_within_window` — same appropriation, same severity within 24h → no duplicate alert.
- Integration (mocked LLM): `test_variance_scan_creates_alerts_for_demo_data` — seeded over-spent appropriation produces a critical alert with LLM-drafted message.

#### 7.4 — Natural-language query (NL → structured)

**What**: `POST /api/v1/ai/query` accepting `{"question": "How much did Public Safety spend on overtime last quarter?"}` and returning structured budget data with sources.

**Design**:
- Two-stage architecture for safety:
  1. **Plan stage**: LLM produces a `QueryPlan(entity, filters, time_range, aggregations)` Pydantic model — NEVER raw SQL.
  2. **Execute stage**: `QueryPlan` translated into a parameterised SQLAlchemy query against pre-approved entity surface.
- Allowed entities: `expenditures`, `revenues`, `appropriations`, `budget_line_items`, `encumbrances`, `capital_projects`. No `users`, `audit_log`, or `personnel_budget_items` (PII).
- All query plans logged to `query_log` for audit.
- Response includes both the answer (rendered as a short natural-language summary) and a citation block listing the data sources (`appropriations.id`s, fiscal periods, etc.).
- Result rendered via second LLM call summarising the structured data; cited numbers must appear verbatim.

**Testing**:
- Unit: `test_query_plan_rejects_disallowed_entity` — plan referencing `users` → `UnauthorizedEntityError`.
- Unit (mocked LLM): `test_plan_to_sql_translation` — plan `{entity=expenditures, filters={dept=4100, account_object=overtime}, time_range=last_quarter, aggregations=[sum]}` → expected SQL fragment.
- Integration (real LLM): `test_e2e_nl_query_demo` — known question against seed data returns correct figure with citations.

---

## Phase 8: GFOA Budget Books, SEFA, XBRL Output

### Purpose
Produce the structured and human-readable documents that justify a government's budget for legislative bodies, the public, federal auditors, and (eventually) FDTA-aligned regulators.

### Tasks

#### 8.1 — GFOA budget book PDF generation

**What**: `POST /api/v1/documents/budget-book?fy=FY2026&version_id=...&award_template=true` generates a GFOA Distinguished Budget Presentation Award-aligned PDF.

**Design**:
- Document composed in HTML via Jinja2 templates: `cover.html.j2`, `toc.html.j2`, `transmittal_letter.html.j2`, `org_chart.html.j2`, `fund_summary.html.j2`, `department_detail.html.j2`, `capital_plan.html.j2`, `appendices.html.j2`.
- Each template receives a typed Pydantic context model assembled by `services.documents.assemble_budget_book(fy, version_id)`.
- WeasyPrint renders to PDF with the GFOA CSS stylesheet (`reports/styles/gfoa.css`).
- Output stored in S3-compatible blob storage (configurable via `BUDGET_BLOB_STORAGE_URL`); local filesystem fallback for self-hosted.
- Generation runs as RQ job; client polls `GET /api/v1/documents/{id}` for `status="ready"` and `download_url`.

**Testing**:
- Unit: `test_budget_book_context_assembly` — known seed data → context model has expected fund/dept counts.
- Integration: `test_budget_book_generation_produces_valid_pdf` — generated PDF parsed by `pypdf`, has ≥ 30 pages, contains "Budget Summary" string in extracted text.
- Fixture-based: `test_award_template_includes_all_required_sections` — generated TOC contains GFOA's required sections per the 2026 award criteria.

#### 8.2 — Structured exports (CSV, JSON, OpenAPI-conformant)

**What**: Export endpoints for raw budget/appropriation data as CSV and JSON.

**Design**:
- `GET /api/v1/exports/budget?fy=FY2026&format=csv` streams CSV via `StreamingResponse`.
- `GET /api/v1/exports/budget?fy=FY2026&format=json` returns canonical schema documented in OpenAPI.
- Schemas frozen in `schemas/exports.py` with versioning (`v1` in URL path) so downstream consumers can rely on stability.

**Testing**:
- Unit: `test_csv_export_column_order` — first row matches documented column order.
- Integration: `test_json_export_validates_against_schema` — output validated against the published JSON Schema.

#### 8.3 — SEFA schedule generation

**What**: `POST /api/v1/reports/sefa?fy=FY2026` produces the Schedule of Expenditures of Federal Awards required for annual single audits.

**Design**:
- Aggregates `expenditures.amount` grouped by `federal_awards.cfda_number, award_name, federal_agency, pass_through_entity`.
- Marks major programmes per the GAGAS risk-based criteria (configurable threshold).
- Output formats: PDF (via WeasyPrint), XLSX (via `openpyxl`), JSON.
- Sub-recipient flag handled per CFDA: pass-through expenditures separated from direct expenditures.

**Testing**:
- Fixture-based: `test_sefa_with_known_awards` — seeded expenditures totalling 1.2M across 3 federal awards → SEFA output groups correctly with matching totals.
- Unit: `test_major_programme_determination` — award with expenditures > 3% of total federal expenditures flagged as major.

#### 8.4 — XBRL / FDTA-ready output

**What**: Export budget summary data as an XBRL instance document conformant to the GASB digital taxonomy (when finalised) or a placeholder schema in the interim.

**Design**:
- `POST /api/v1/reports/xbrl?fy=FY2026&taxonomy=gasb-2026` produces an XBRL instance file.
- Implementation calls `Arelle` via subprocess with prepared instance JSON; Arelle is the de facto open-source XBRL processor.
- Until GASB's taxonomy is final, fall back to a published US Treasury sample taxonomy with placeholder element IDs sourced from `chart_of_accounts.xbrl_element_id`.
- Validation step runs Arelle in validation mode; export blocked unless instance validates clean.

**Testing**:
- Fixture-based: `test_xbrl_export_validates` — output passes Arelle's `--validate` mode against bundled taxonomy.
- Unit: `test_unmapped_account_warning` — chart account without `xbrl_element_id` causes a warning in the export report (not a failure).

---

## Phase 9: Public Transparency Portal

### Purpose
Deliver the public-facing site that elected officials, journalists, and residents use to explore the budget without authentication. WCAG 2.1 AA compliant, SEO-indexed, and powered by the same APIs as the internal console.

### Tasks

#### 9.1 — Public read-only API

**What**: `/api/v1/transparency/*` endpoints exposing aggregated budget data without authentication, rate-limited and cached.

**Design**:
- Endpoints: `GET /transparency/funds`, `/transparency/departments`, `/transparency/budgets/{fy}/summary`, `/transparency/spending/by-category?fy=...`, `/transparency/search?q=...`.
- Only data from `adopted` budget versions exposed; in-progress requests stay private.
- Rate limited to 60 requests/min per IP via Redis counter; bursts allowed up to 100.
- Response cached for 5 minutes via FastAPI middleware + Redis (`fastapi-cache2`).
- All responses include a `Cache-Control: public, max-age=300` header.

**Testing**:
- Integration: `test_transparency_endpoint_no_auth_required` — request without auth returns 200.
- Integration: `test_rate_limit_enforced` — 61st request in a minute returns 429.
- Integration: `test_unadopted_data_not_exposed` — `version_type="departmental_request"` data absent from transparency response.

#### 9.2 — Transparency Portal Next.js app

**What**: A standalone Next.js 15 app under `web/apps/portal/` consuming the transparency API and rendering accessible budget pages.

**Design**:
- Routes:
  - `/` — landing page with fiscal-year selector and high-level totals.
  - `/budget/[year]` — fund-level breakdown, multi-year revenue/expenditure chart.
  - `/department/[code]` — department detail with programmes and personnel summary.
  - `/spending/by-category` — interactive sunburst (D3) of fund → dept → object class.
  - `/search` — full-text search across departments, programmes, capital projects.
  - `/ask` — NL query interface (Phase 7.4).
- Server Components produce static HTML for SEO; cached via Next.js `cache: 'force-cache'` with on-demand revalidation when budget data changes.
- All charts have accessible tabular fallbacks (`<table>` with same data, `aria-describedby` linking chart and table).
- Colour palette tested with `pa11y-ci` for WCAG 2.1 AA contrast.
- Language: defaults to English; site copy externalised via `next-intl` for future translations.

**Testing**:
- E2E (Playwright): `test_landing_renders_current_fiscal_year` — navigate to `/`, verify FY2026 totals visible.
- E2E (Playwright + axe-core): `test_landing_accessibility_aa` — `axe.run()` returns zero AA-level violations.
- E2E: `test_budget_year_page_renders_chart_and_table` — `/budget/FY2026` shows both sunburst and tabular data; tab-key navigation traverses all interactive elements.

#### 9.3 — Embeddable widgets

**What**: Provide `<iframe>` embeddable widgets so jurisdictions can include charts on their existing CMS pages.

**Design**:
- Routes under `/embed/<widget>?fy=...&width=...&height=...` rendering a single chart with no chrome.
- `X-Frame-Options: ALLOWALL` for these routes only; CSP `frame-ancestors *` (configurable to lock down to specific origins).
- Widgets: `fund-summary`, `top-departments`, `spending-trend`, `capital-projects-map`.

**Testing**:
- E2E: `test_embed_widget_loads_in_iframe` — host page embeds widget, Playwright verifies render.
- Unit: `test_embed_security_headers` — only embed routes carry `X-Frame-Options: ALLOWALL`; main pages carry `DENY`.

---

## Phase 10: Hardening, Observability, and Release

### Purpose
Convert the working prototype into production-ready software: comprehensive observability, performance benchmarks, security review against OWASP ASVS, deployment artefacts (Helm chart, Terraform module), documentation, and the v1.0 release process.

### Tasks

#### 10.1 — Observability: structured logs, metrics, traces

**What**: Wire `structlog`, OpenTelemetry, and Prometheus metrics throughout the application.

**Design**:
- All log lines JSON-formatted with mandatory fields `timestamp, level, msg, trace_id, span_id, org_id, user_id, request_id`.
- OTel instrumentation: FastAPI, SQLAlchemy, RQ, HTTPX auto-instrumented; custom spans around LLM calls and ERP syncs.
- Prometheus endpoint `/metrics` exposes: HTTP request duration histograms, RQ job duration, LLM token counts, appropriation balance computation latency.
- Default OTel exporter: OTLP/gRPC; configurable via `OTEL_EXPORTER_OTLP_ENDPOINT`.

**Testing**:
- Unit: `test_structlog_emits_required_fields` — captured log line has all mandatory fields.
- Integration: `test_otel_span_emitted_per_request` — in-memory span collector captures one span per API call.

#### 10.2 — Performance baselines and load tests

**What**: Establish baseline performance for key operations and a Locust load test suite.

**Design**:
- Baselines (single-node, 4 vCPU, 8 GB RAM, Postgres on same host):
  - `GET /appropriations/{id}/balance`: p95 < 50ms.
  - `POST /budgets/{id}/line-items/bulk` with 1000 rows: < 3s.
  - Budget book PDF generation for 5-fund, 12-dept jurisdiction: < 30s.
  - `POST /ai/query`: p95 < 8s (LLM-dominated).
- `tests/perf/locustfile.py` simulates 50 finance staff + 500 public users.

**Testing**:
- Perf: `test_appropriation_balance_p95_under_50ms` — gated in CI as advisory only.
- Perf: `test_bulk_line_item_insert_under_3s` — fails CI if exceeded.

#### 10.3 — Security review

**What**: Audit against OWASP ASVS Level 2 and NIST SP 800-53 Moderate baseline subset.

**Design**:
- Static analysis: `bandit -r src/` in CI; zero high-severity findings required to merge.
- Dependency scanning: `pip-audit` and `npm audit` in CI; high-severity CVEs block release.
- Secret scanning: `gitleaks` pre-commit hook.
- TLS-required at all boundaries; HSTS header `max-age=63072000; includeSubDomains; preload`.
- CSP headers strict on internal console: `default-src 'self'; script-src 'self' 'unsafe-inline'` (Next.js needs inline; nonce-based in v1.1).
- Penetration test checklist in `docs/security-checklist.md` covering OWASP Top 10.

**Testing**:
- CI: bandit, pip-audit, npm audit, gitleaks all green.
- Manual: penetration test results documented in `docs/security/pentest-v1.0.md`.

#### 10.4 — Deployment artefacts

**What**: Production deployment artefacts: docker-compose for self-hosted; Helm chart for Kubernetes; Terraform module for AWS reference deployment.

**Design**:
- `deploy/helm/budget-app/` chart with values for replicas, resources, ingress, secrets.
- `deploy/terraform/aws/` module provisioning: VPC, RDS Postgres (Multi-AZ), ElastiCache Redis, ECS Fargate cluster, ALB, S3 bucket for documents, CloudFront for portal.
- `docker-compose.prod.yml` includes Caddy reverse proxy with automatic Let's Encrypt.

**Testing**:
- Integration (in CI): `helm lint deploy/helm/budget-app` clean; `terraform validate` clean.
- Integration (in nightly): `docker-compose -f docker-compose.prod.yml up` boots and `/healthz` returns 200.

#### 10.5 — Documentation and release

**What**: Complete user and operator documentation; cut v1.0.0 release.

**Design**:
- `docs/` site built with MkDocs Material:
  - User Guide: budget formulation workflow, capital planning, document generation.
  - Admin Guide: user/role management, ERP connector setup.
  - Operator Guide: self-hosted deployment, backups, monitoring, upgrade.
  - API Reference: auto-generated from OpenAPI 3.1 spec.
  - Standards Reference: how the system implements GFOA, GASB, COFOG (drawn from `standards.md`).
- `CHANGELOG.md` follows Keep a Changelog conventions.
- Release process: tag `v1.0.0`, GitHub Actions builds and pushes Docker images, publishes Helm chart to OCI registry, attaches release notes.

**Testing**:
- Docs build: `mkdocs build --strict` (warnings fail).
- Release dry-run: `release-please --dry-run` validates the release config.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Tenancy           ─── required by everything
    │
Phase 2: Funds, Depts, Chart of Accounts ─── requires Phase 1
    │
Phase 3: Budget Formulation             ─── requires Phase 2
    │
Phase 4: Appropriations & Expenditures  ─── requires Phase 3
    │
Phase 5: RBAC, Workflow, Audit          ─── requires Phase 4 (also retrofits earlier phases with audit hooks)
    ├── Phase 6: ERP Connectors         ─── requires Phase 4 (parallel with 7 and 8)
    ├── Phase 7: AI Features            ─── requires Phase 4 (parallel with 6 and 8)
    └── Phase 8: GFOA / SEFA / XBRL     ─── requires Phase 4 (parallel with 6 and 7)
                                                │
                                            Phase 9: Public Transparency Portal ─── requires Phase 8 (read-only API)
                                                │
                                            Phase 10: Hardening & Release        ─── requires all preceding phases
```

**Parallelism opportunities**: After Phase 5 ships, Phases 6, 7, and 8 can be developed by three independent contributors/teams since they touch disjoint code paths (ERP services vs. AI services vs. document generation). Phase 9 (Portal UI) can also begin partial work in parallel with Phase 8 by mocking the API responses.

**Estimated scope**: Large. A reasonable team of 4 engineers + 1 frontend + 1 designer over 9–12 months for v1.0; aggressive solo-with-AI scope of 4–5 months focusing only on MVP (Phases 1–5 + minimal 8) is achievable.

---

## Definition of Done (per phase)

Every phase must satisfy this checklist before being considered complete:

1. All tasks in the phase are implemented and merged to `main`.
2. All unit tests pass (`pytest -m "not integration and not real_erp and not llm"`).
3. All integration tests pass (`pytest -m integration`).
4. E2E Playwright tests for the phase pass (where the phase introduces UI).
5. `ruff check` and `ruff format --check` clean across `src/`, `tests/`, `web/`.
6. `mypy --strict src/` reports zero errors.
7. Test coverage on `src/budget_app/` ≥ 80 % for the phase's new modules.
8. `docker build` for `api`, `worker`, `console`, and `portal` images succeeds.
9. `docker-compose up` starts the full stack; `/healthz` returns 200.
10. New API endpoints appear in the auto-generated OpenAPI 3.1 spec; the TS client regenerates without errors.
11. New tables have Alembic migrations that `upgrade` and `downgrade` cleanly on an empty database.
12. Every new tenant table has Row-Level Security policy on `org_id` (verified by the CI script in 1.4).
13. Every new mutating endpoint writes to `audit_log` (verified by integration test).
14. New configuration options documented in `docs/configuration.md` and `.env.example` updated.
15. CHANGELOG entry added under "Unreleased".
16. Phase demo: a 10-minute walk-through video or recorded session demonstrating the new functionality end-to-end against the seeded sample jurisdiction.
