# Standards & API Reference

> Project: Tool & Calibration Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 17025:2017 — General Requirements for the Competence of Testing and Calibration Laboratories**
- URL: https://www.iso.org/standard/66912.html
- The primary international accreditation standard for calibration laboratories. Clauses 6.4 (equipment), 7.6 (measurement uncertainty), and 7.5 (technical records) drive the core documentation and audit-trail requirements for any calibration management software. Compliance with this standard is the most universal software procurement criterion in the market.

**ISO 9001:2015 — Quality Management Systems (Clause 7.1.5 — Monitoring and Measuring Resources)**
- URL: https://www.iso.org/standard/62085.html
- Requires organisations to maintain calibrated monitoring and measuring equipment with documented intervals, traceability to national standards, and records of calibration status. This clause is the most common driver of basic calibration tracking requirements in general manufacturing QMS software.

**ISO 10012:2003 (revised 2025 as BS EN ISO 10012:2025) — Measurement Management Systems**
- URL: https://knowledge.bsigroup.com/articles/measurement-management-systems-bs-en-iso-10012-2025-is-available-to-pre-order
- Provides the framework for implementing measurement management systems within an organisation. Complements ISO 17025 with practical guidance on calibration periodicity, pass/fail criteria, uncertainty handling, and risk-based approaches. The 2025 revision aligns with ILAC G8 decision rules.

**IATF 16949:2016 — Automotive Quality Management System (Clause 7.1.5.1)**
- URL: https://www.iatfglobaloversight.org/
- Automotive-sector supplement to ISO 9001 requiring documented calibration plans, out-of-tolerance investigation records, gauge R&R studies, and mandatory use of accredited external calibration labs (ISO 17025). Drives the MSA/gauge R&R feature requirements in tools targeting automotive suppliers.

**AS9100 Rev D — Quality Management Systems for Aviation, Space, and Defence**
- URL: https://www.sae.org/standards/content/as9100d/
- Aerospace and defence quality management standard with stringent calibration, first-article inspection, and traceability requirements. Commonly cited alongside MIL-STD-45662A for calibration management in defence supply chains.

**ISO/IEC 17011:2017 — Conformity Assessment: Requirements for Accreditation Bodies**
- URL: https://www.iso.org/standard/67198.html
- Sets requirements for accreditation bodies that accredit calibration laboratories against ISO 17025. Relevant for software supporting accredited labs seeking or maintaining national accreditation body recognition.

---

### JCGM & Metrology Guides

**JCGM 100:2008 (GUM) — Guide to the Expression of Uncertainty in Measurement**
- URL: https://www.iso.org/sites/JCGM/GUM-introduction.htm
- The definitive international guide for evaluating and expressing measurement uncertainty. Calibration management software implementing uncertainty budgets must follow GUM methodology. The latest revision (GUM-1:2023) is available from BIPM: https://www.bipm.org/documents/20126/194484570/JCGM_GUM-1/

**JCGM 200:2012 (VIM) — International Vocabulary of Metrology**
- URL: https://www.bipm.org/en/committees/jc/jcgm/publications
- Defines all measurement and calibration terminology used across ISO 17025, ISO 10012, and GUM. Software data models, field labels, and documentation should use VIM-compliant terminology for audit acceptance.

**ILAC P14:09/2020 — ILAC Policy for Measurement Uncertainty in Calibration**
- URL: https://ilac.org/publications-and-resources/ilac-policy-series/
- Sets mandatory requirements for how accredited calibration laboratories estimate and state measurement uncertainty on calibration certificates. Calibration certificate generation features must produce uncertainty statements that satisfy this policy.

**ILAC G8:09/2019 — Guidelines on Decision Rules and Statements of Conformity**
- URL: https://ilac.org/publications-and-resources/ilac-guidance-series/
- Provides guidance on applying guard-banding and risk-based decision rules when stating conformity to a specification after calibration. Relevant for software implementing configurable pass/fail criteria with uncertainty-based guard bands (aligned with ISO 14253-1).

---

### Security & Compliance Standards

**FDA 21 CFR Part 11 — Electronic Records; Electronic Signatures**
- URL: https://www.ecfr.gov/current/title-21/chapter-I/subchapter-A/part-11
- US FDA regulation requiring secure, computer-generated, timestamped audit trails; electronic signature controls; and system validation for software managing regulated electronic records. Section 11.10(e) audit trail requirements are directly applicable to calibration record modification tracking. Mandatory for calibration software used in pharmaceutical and medical device manufacturing.

**MIL-STD-45662A — Calibration System Requirements (US DoD)**
- URL: https://www.dsp.dla.mil/
- US military standard defining requirements for calibration systems in defence supply chains. Superseded by ANSI/NCSL Z540-1 for most applications but still referenced in legacy defence contracts. Relevant for tools targeting DoD prime contractors and their subcontractors.

**ANSI/NCSL Z540-1 / ANSI/NCSL Z540.3 — Requirements for Calibration and Testing Laboratories**
- URL: https://blog.ansi.org/anab/ansi-z540-1/
- US national standard (supplemental to ISO 17025) widely required in US defence and aerospace procurement. Z540.3 adds requirements for measurement decision risk. Features for uncertainty calculation and risk-based calibration decisions must align with these requirements.

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URL (OAuth): https://datatracker.ietf.org/doc/html/rfc6749 | URL (OIDC): https://openid.net/connect/
- De-facto authentication and authorisation standards for modern SaaS applications. Calibration management SaaS must support OAuth 2.0 / OIDC for enterprise SSO integration (Microsoft Entra ID / Azure AD, Google Workspace, Okta). GageList's Microsoft Entra ID SSO support demonstrates market expectation.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Information security management framework increasingly required by enterprise customers when evaluating cloud calibration management SaaS. Relevant for data residency, access control, and audit logging architecture decisions.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (fka Swagger)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for documenting and designing REST APIs. Calibration management software offering developer APIs (GAGEtrak IIoT API, GageList REST API, Fluke MET/CONNECT) should publish OpenAPI 3.1 specifications to support code generation and third-party integration tooling.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Standard for describing and validating JSON data structures. Relevant for defining calibration record, asset, certificate, and measurement data schemas exchanged via REST APIs and webhook payloads.

**iCalendar (RFC 5545)**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Standard for calendar and scheduling data. Calibration due-date notifications and scheduling data exports may use iCalendar format for interoperability with enterprise calendar systems (Outlook, Google Calendar).

**EPCIS 2.0 / GS1 Standards (Barcode & RFID data)**
- URL: https://www.gs1.org/standards/epcis
- GS1 standards define barcode (GS1-128, QR, DataMatrix) and RFID data structures for asset identification. Relevant for calibration software implementing compliant asset labelling and RFID-based tracking.

---

### MCP Server Specifications

MCP (Model Context Protocol) is relevant to an AI-native calibration management tool for exposing calibration records, asset data, and audit trail queries as structured context to LLM-based audit preparation and decision-support features.

- **MCP Specification**: https://modelcontextprotocol.io/specification
- Recommended resources to expose via MCP server:
  - Calibration records (asset, date, result, technician, certificate reference)
  - Equipment inventory and status
  - Out-of-tolerance events and linked corrective actions
  - Calibration schedule and due-date data
  - Standards compliance checklist items (ISO 17025 clause mapping)

---

## Similar Products — Developer Documentation & APIs

### Fluke MET/CONNECT

- **Description:** RESTful API layer for Fluke MET/TEAM calibration management software, enabling third-party integration with asset and calibration result data. Described as the "modern default way to make systems talk to each other in a flexible way."
- **API Documentation:** https://media.fluke.com/444e8294-b7a4-4443-bafa-b2e6010730c4_original%20file.pdf (MET/CONNECT API Reference Overview PDF)
- **SDKs/Libraries:** No official SDK published; REST/JSON-based integration
- **Developer Guide:** https://www.fluke.com/en/product/fluke-software/fluke-calibration-software/met-connect-calibration-integration-software
- **Standards:** REST/JSON; endpoints for Asset, Point, and Result controllers
- **Authentication:** Per-seat API licence required; authentication details not publicly documented

---

### GAGEtrak REST API (IIoT)

- **Description:** REST API in GAGEtrak Pro and Lite enabling Industry 4.0 / IIoT integration for calibration and asset data. Supports custom field inclusion with standard gage and calibration requests.
- **API Documentation:** https://gagetrak.com/features/ (feature page mentions API; full docs require vendor contact)
- **SDKs/Libraries:** No published SDK; REST/JSON
- **Developer Guide:** Contact CyberMetrics via https://gagetrak.com/
- **Standards:** REST/JSON
- **Authentication:** Not publicly documented

---

### GageList REST API

- **Description:** Cloud calibration management REST API supporting gage and calibration data retrieval with custom field inclusion. Zapier and SSO integrations extend reach to no-code automation and enterprise identity providers.
- **API Documentation:** https://gagelist.com/features/ (API section; full documentation available to account holders)
- **SDKs/Libraries:** No official SDK; Zapier integration for no-code automation
- **Developer Guide:** https://gagelist.com/knowledge-base/changelog-webapp/
- **Standards:** REST/JSON; Zapier integration
- **Authentication:** API key (click-to-copy from account settings); SSO via Microsoft Entra ID (OAuth 2.0)

---

### Beamex CMX — Business Bridge Integration

- **Description:** Beamex Business Bridge is a bidirectional integration connector between Beamex CMX calibration management software and CMMS/ERP systems. Work orders created in SAP or IBM Maximo are automatically sent to CMX; completed calibration results and acknowledgements are returned to close the work order.
- **API Documentation:** https://www.beamex.com/us/calibration-software/cmx/business-bridge/
- **SDKs/Libraries:** Configured integration product; not a developer SDK
- **Developer Guide:** https://www.beamex.com/us/calibration-software/cmx/business-bridge/
- **Standards:** Connector-based; integrates with SAP PM, IBM Maximo, Infor EAM
- **Authentication:** Enterprise system credentials; configuration managed by Beamex implementation team

---

### IBM Maximo Calibration (Application Suite)

- **Description:** Calibration management module fully integrated within IBM Maximo Asset Management / Maximo Application Suite (MAS). Provides traceability, reverse traceability, calibration history, data sheets, and reporting as base CMMS functionality.
- **API Documentation:** https://www.ibm.com/docs/en/maximo-sap-con/8.1.0?topic=product-overview
- **SDKs/Libraries:** Maximo REST API; OSLC (Open Services for Lifecycle Collaboration) APIs
- **Developer Guide:** https://www.ibm.com/docs/en/mas-cd/continuous-delivery
- **Standards:** REST/JSON; OSLC; SAP Connector via IBM Maximo Connector for SAP
- **Authentication:** IBM IAM; OAuth 2.0; SAML 2.0

---

### Qualityze EQMS — Calibration Module (Salesforce Platform)

- **Description:** Calibration management delivered as a module within the Qualityze EQMS, built natively on Salesforce. All APIs are Salesforce platform APIs, giving access to calibration asset, criteria, tolerance assessment, and NC data.
- **API Documentation:** https://developer.salesforce.com/docs/apis (Salesforce REST and Bulk API documentation)
- **SDKs/Libraries:** Salesforce SDK (JavaScript, Python, Java, .NET, mobile); AppExchange ecosystem
- **Developer Guide:** https://trailhead.salesforce.com/
- **Standards:** Salesforce REST API; Salesforce Connect; OpenAPI-compatible via Salesforce API
- **Authentication:** OAuth 2.0 (Salesforce identity); SAML 2.0; MFA

---

### Ideagen Quality Management (formerly Gael Quality)

- **Description:** Cloud QHSE platform with calibration and equipment management module. Integrates calibration tracking within broader audit, non-conformance, and corrective action workflows.
- **API Documentation:** https://www.ideagen.com/standards/iso-17025
- **SDKs/Libraries:** REST API; integration available via vendor; no publicly published SDK
- **Developer Guide:** Contact Ideagen via https://www.ideagen.com/
- **Standards:** REST/JSON; ISO 17025 and IATF 16949 compliance framework
- **Authentication:** OAuth 2.0 / SAML SSO

---

### QT9 QMS — ISO 17025 / IATF 16949 Module

- **Description:** Cloud QMS with dedicated ISO 17025 and IATF 16949 compliance modules, including calibration equipment tracking, gage management, and ERP integration (SAP, Oracle, others) via built-in connectors and BI tools.
- **API Documentation:** https://qt9software.com/iso-17025
- **SDKs/Libraries:** REST API; ERP integration connectors
- **Developer Guide:** Contact QT9 Software via https://qt9software.com/
- **Standards:** REST/JSON; OpenAPI; ERP connector framework
- **Authentication:** OAuth 2.0; SSO

---

## Notes

- **No dominant open API standard exists** for calibration management data exchange. Unlike domains such as healthcare (HL7 FHIR) or financial services (ISO 20022), calibration management lacks a published interoperability schema. This is a significant opportunity for an open-source project to define a standard JSON schema for calibration records, certificates, and measurement data.
- **Measurement uncertainty data models** in existing APIs are largely proprietary. The GUM/JCGM 100 framework is the universally accepted calculation methodology, but no vendor has published a standard API schema for uncertainty budget data.
- **EURAMET and BIPM traceability chain documentation** is not programmatically accessible in any reviewed product; traceability is managed as document attachments rather than structured, queryable data.
- **MCP server exposure** of calibration data is an emerging opportunity not yet implemented by any commercial product, representing a clear differentiation path for an AI-native open-source tool.
