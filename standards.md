# Standards & API Reference

> Project: Budget & Appropriations Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

No single ISO standard governs government budgeting directly; the relevant ISO standards cover data interchange, security, and interoperability applicable to any government financial system.

- **ISO 20022** — Universal financial industry message scheme (XML-based); increasingly adopted for government payment and financial messaging; relevant to ERP-to-treasury data flows. https://www.iso20022.org/
- **ISO/IEC 27001** — Information security management systems; required or strongly recommended for any system processing government financial data. https://www.iso.org/isoiec-27001-information-security.html
- **ISO 8601** — Date and time representation standard; critical for multi-year budget period definitions, fiscal year identifiers, and audit timestamps. https://www.iso.org/iso-8601-date-and-time-format.html

### UN & IMF Standards

- **COFOG (Classification of the Functions of Government)** — UN Statistics Division classification framework for government expenditure by function (10 divisions: General public services, Defence, Public order, Economic affairs, Environmental protection, Housing, Health, Recreation, Education, Social protection). Published by UNSD; foundational for cross-government and international expenditure comparison. https://unstats.un.org/unsd/classifications/Family/Detail/4
- **IMF Government Finance Statistics Manual (GFSM 2001/2014)** — International Monetary Fund framework for government financial statistics; incorporates COFOG and provides the conceptual basis for classifying government revenue, expenditure, and debt. https://www.imf.org/external/pubs/ft/gfs/manual/

### US Government Accounting Standards

- **GASB (Governmental Accounting Standards Board) Standards** — Authoritative accounting standards for US state and local governments; GASB is developing a voluntary digital financial reporting taxonomy (XBRL-based) aligned with the Financial Data Transparency Act. Core standards governing appropriations accounting, fund balance, and financial statement presentation. https://www.gasb.org/standards-and-guidance
- **FASAB (Federal Accounting Standards Advisory Board)** — Accounting standards for US federal agencies; governs appropriations tracking, budgetary resources, and financial reporting at the federal level. https://www.fasab.gov/
- **GAAFR (Governmental Accounting, Auditing, and Financial Reporting)** — GFOA's comprehensive reference aligned with GASB standards; defines the fund accounting model and chart of accounts conventions widely implemented in government ERP systems. https://www.gfoa.org/gaafr

### US Federal Budget Standards & Guidance

- **OMB Circular A-11 — Preparation, Submission, and Execution of the Budget** — The primary federal guidance governing how agencies formulate, submit, and execute the US federal budget. Defines standardised budget methods, terminology, and data elements required in agency submissions to OMB. Systems serving federal agencies must align with A-11 definitions. https://www.whitehouse.gov/omb/information-for-agencies/circulars/
- **GFOA Distinguished Budget Presentation Award Criteria (2026 revised)** — GFOA's publicly documented award criteria defining best practices for budget presentation across operating, capital, and transparency dimensions. In 2026 the criteria were expanded to cover all forms of budget communication (digital, dashboards, multimedia). Effectively a de facto quality standard for government budget documents in the US. https://www.gfoa.org/budget-award-2026-criteria
- **SEFA (Schedule of Expenditures of Federal Awards)** — Required annual schedule for any government receiving federal grants, mapping expenditures to federal award programs. Budget systems must produce or support SEFA preparation to pass annual single audits.

### W3C & IETF Standards

- **RFC 9110 — HTTP Semantics** — Core HTTP standard governing REST API design; all web-based budget software APIs should conform. https://www.rfc-editor.org/rfc/rfc9110
- **RFC 8288 — Web Linking** — Standard for expressing relationships between resources via HTTP Link headers; relevant to API hypermedia and navigation. https://www.rfc-editor.org/rfc/rfc8288
- **RFC 6749 — OAuth 2.0 Authorization Framework** — Standard for delegated authorisation used in government API integrations; required for connecting budget systems to ERP, identity providers, and third-party data sources. https://www.rfc-editor.org/rfc/rfc6749
- **OpenID Connect 1.0** — Identity layer on top of OAuth 2.0; standard for single sign-on integration with government identity systems (Active Directory, PIV/CAC). https://openid.net/connect/
- **W3C WCAG 2.1 (Web Content Accessibility Guidelines)** — Accessibility standard; Section 508 compliance for US government software mandates WCAG 2.1 AA conformance for all public-facing budget transparency portals. https://www.w3.org/TR/WCAG21/

### Data Model & API Specifications

- **OpenAPI Specification 3.1** — De facto standard for REST API documentation; all major government budget platform APIs (OpenGov, Oracle EPM) use or are expected to conform to OpenAPI definitions. https://spec.openapis.org/oas/v3.1.0
- **JSON Schema (Draft 2020-12)** — Standard for describing and validating JSON data structures; used in DATA Act schema definitions and emerging GASB digital taxonomy tooling. https://json-schema.org/
- **XBRL (eXtensible Business Reporting Language)** — Structured financial reporting standard mandated by SEC and being adopted by GASB for government financial reporting under FDTA. XBRL taxonomies define machine-readable financial statement elements. https://xbrl.org/
- **DATA Act Information Model Schema (DAIMS)** — US Treasury data standard derived from the Digital Accountability and Transparency Act; defines data elements for federal agency financial reporting to USASpending.gov. Systems serving federal agencies must produce DAIMS-compliant output. https://fiscal.treasury.gov/data-transparency/DAIMS-current.html

### Financial Data Transparency Act (FDTA)

- **Financial Data Transparency Act of 2022 (P.L. 117-263)** — US law requiring federal financial regulators to adopt non-proprietary, machine-readable data standards. In 2024, seven federal agencies released proposed rules for implementation. The compliance date for the SEC Rule is expected in early 2027. The GASB digital taxonomy project is the state/local government implementation pathway. Budget and appropriations systems should plan for FDTA-aligned structured output. https://xbrl.us/home/priorities/efficiency/fdta/

### Security & Authentication Standards

- **FedRAMP (Federal Risk and Authorization Management Program)** — US government programme standardising security authorisation for cloud services; based on NIST SP 800-53. Required for cloud software used by federal agencies; OpenGov and OneStream are among the budget platforms pursuing or holding FedRAMP authorisation. Mandatory controls include SAML 2.0 SSO, AES-256 encryption, PIV/CAC authentication, and defined audit log retention. https://www.gsa.gov/technology/government-it-initiatives/fedramp
- **FISMA (Federal Information Security Modernization Act)** — US federal law requiring agencies to secure information systems; FedRAMP fulfils FISMA's cloud-specific requirements. State and local systems follow NIST SP 800-53 guidelines as a best practice. https://www.cisa.gov/topics/cyber-threats-and-advisories/federal-information-security-modernization-act
- **NIST SP 800-53 (Security and Privacy Controls)** — Comprehensive security control catalogue underpinning FedRAMP and FISMA; defines controls for access management, audit logging, configuration management, and incident response relevant to budget systems. https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- **SAML 2.0** — XML-based single sign-on standard; widely used for government enterprise SSO integration with Active Directory Federation Services. https://docs.oasis-open.org/security/saml/

### MCP Server Specifications

An MCP (Model Context Protocol) server for budget and appropriations management would expose tools enabling AI agents to query budget data, run scenario models, and monitor appropriations. Relevant MCP patterns:
- Budget data query tools (fund balances, appropriation amounts, encumbrances, actuals by department)
- Scenario modelling tools (create/run/compare budget scenarios)
- Variance alerting subscription tools
- Budget document generation tools (narrative drafting, GFOA-format output)

Reference MCP specification: https://modelcontextprotocol.io/specification

---

## Similar Products — Developer Documentation & APIs

### OpenGov Public Service Platform API

- **Description:** Cloud-based government platform offering budgeting, performance reporting, open data, and citizen engagement. Provides REST APIs for data access and integration.
- **API Documentation:** https://developer.opengov.com/docs/overview
- **API Portal (endpoints):** https://api.docs.opengov.com/
- **Developer Portal:** https://developer.opengov.com/catalog
- **SDK/Libraries:** Not publicly documented; API key-based authentication used
- **Developer Guide:** https://developer.opengov.com/docs/quickstart
- **Standards:** REST/JSON; OpenAPI-compatible
- **Authentication:** API Key (managed through developer portal app management)
- **Notes:** The broader OpenGov Platform API is documented; the Budgeting & Planning module's API availability is limited per some user reports.

### USASpending.gov API

- **Description:** US Treasury open API providing comprehensive federal spending data including contracts, grants, loans, and direct payments. Covers appropriations, obligations, and outlays at the federal account level.
- **API Documentation:** https://api.usaspending.gov/
- **GitHub (source):** https://github.com/fedspendingtransparency/usaspending-api
- **Data Dictionary:** https://api.usaspending.gov/docs/data-dictionary
- **Training Materials:** https://www.usaspending.gov/data/Basic-API-Training.pdf
- **Standards:** REST/JSON; versioned endpoints (v2); conforms to DAIMS schema
- **Authentication:** Public API — no authentication required for read access
- **Notes:** Supports filtering by federal account, agency, award type, and disaster spending. Some endpoints use Elasticsearch for performance.

### US Treasury Fiscal Data API

- **Description:** US Treasury open data platform providing machine-readable federal financial data including debt, interest rates, revenue, spending, and the Monthly Treasury Statement.
- **API Documentation:** https://fiscaldata.treasury.gov/api-documentation/
- **Portal:** https://fiscaldata.treasury.gov/
- **Standards:** RESTful, GET requests, JSON responses, standard HTTP response codes
- **Authentication:** Public API — no authentication required
- **Notes:** Covers the US Government Financial Report, Monthly Treasury Statement, and Daily Treasury Statement datasets. Machine-readable formats via file download and API.

### Oracle EPM Cloud REST API

- **Description:** Oracle Planning and Budgeting Cloud (part of Oracle EPM Cloud) provides REST APIs for administering planning applications, loading data, running rules, and extracting reports.
- **API Documentation:** https://docs.oracle.com/en/cloud/saas/planning-budgeting-cloud/admin-only-hdi.html
- **Developer Guide:** https://docs.oracle.com/en/cloud/saas/enterprise-performance-management-common/cgsad/
- **SDK/Libraries:** Oracle EPM Automate (CLI + scripted automation); Smart View (Excel add-in)
- **Standards:** REST/JSON; OpenAPI-compatible
- **Authentication:** OAuth 2.0; supports SSO via Identity Cloud Service
- **Notes:** FedRAMP and IL-authorised for federal use cases. Supports FDMEE for bulk data loading from ERP systems.

### Tyler Technologies Munis (Tyler Enterprise ERP) API

- **Description:** Tyler's government ERP platform exposes APIs via the Tyler Nexus integration platform for connecting Munis budget and financial data to third-party systems.
- **API Documentation:** Available to Tyler clients and partners through the Tyler Community portal
- **Integration Platform:** Tyler Nexus — https://www.tylertech.com/products/nexus
- **Standards:** REST/JSON; partner integrations use Tyler-defined API contracts
- **Authentication:** OAuth 2.0 / API Key (partner-managed)
- **Notes:** Not publicly documented; access requires Tyler client relationship. Dominant in US local government; Munis Budget Entry documentation is available through government client portals (e.g., Sarpy County Nebraska).

### Euna Budget (Questica) Integration Framework

- **Description:** Euna Budget integrates with ERP systems for data import/export; specific API documentation is not publicly available but integration with Tyler Munis, SAP, Infor, Oracle, and Workday is documented as a product capability.
- **API Documentation:** Available to clients through Euna Solutions support portal
- **Partner Integrations:** Infor, SAP, Workday Financial Management, Oracle, Tyler Munis, Carahsoft, Sage Intacct
- **Standards:** ERP-specific connectors; REST-based where ERP supports it
- **Authentication:** Client-managed credentials per integration
- **Notes:** Euna Budget manages over $538 billion in public funds. Questica's Advanced Calculation Engine supports driver-based scenario modelling via the platform UI rather than API.

### Bloomberg Government API

- **Description:** Bloomberg Government's Federal Funding Flow tool provides federal budget, appropriations, and spending intelligence. Enterprise API access is available to subscribers.
- **API Documentation:** Enterprise clients only — not publicly documented
- **Developer Guide:** Available through Bloomberg enterprise agreements
- **Standards:** Bloomberg data standards; REST/JSON for enterprise integrations
- **Authentication:** Bloomberg Terminal / enterprise API key
- **Notes:** Launched an AI-powered budget-to-spend analysis tool in 2025. Best-in-class federal appropriations tracking from bill to contract-level expenditure.

### XBRL US Taxonomy APIs

- **Description:** XBRL US maintains US GAAP and government financial reporting taxonomies used for structured financial statement submission.
- **Taxonomy Downloads:** https://xbrl.us/xbrl-taxonomy/2025-us-gaap/
- **FDTA Resources:** https://xbrl.us/home/priorities/efficiency/fdta/
- **Standards:** XBRL (ISO 17625); JSON Schema; XSD
- **Authentication:** Public download; no API authentication required
- **Notes:** GASB is building a voluntary state/local government taxonomy aligned with FDTA. Budget systems should plan to consume and produce XBRL-compatible structured output as FDTA compliance dates approach.

---

## Notes

- **GASB Digital Taxonomy (in progress):** GASB's voluntary digital financial reporting taxonomy is under active development (Phase One expected in 2025-2026). Budget and appropriations systems should monitor GASB's progress and plan for XBRL output capability to align with Financial Data Transparency Act requirements. See: https://www.gasb.org/
- **FDTA Compliance Timeline:** The first FDTA proposed rules were published in August 2024 by seven federal agencies. SEC Rule compliance is expected in early 2027. State and local government requirements will follow as GASB taxonomy development matures.
- **OMB Circular A-11 Updates:** OMB updates A-11 annually. The 2025 edition is available at https://www.whitehouse.gov/wp-content/uploads/2025/08/a11.pdf. Systems serving federal agencies should track annual revisions.
- **Open Standards Gap:** No open standard governs the budget formulation workflow itself (submission, approval, consolidation). This is an area where vendor-defined workflow models dominate. An AI-native open-source tool has an opportunity to define an open workflow schema that could become a community standard.
