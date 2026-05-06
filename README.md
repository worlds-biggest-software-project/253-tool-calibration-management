# Tool & Calibration Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native platform for tool tracking, calibration scheduling, and standards compliance across manufacturing, aerospace, defence, and life sciences.

Tool & Calibration Management is a candidate project to build a modern alternative to the legacy calibration management software market. It targets metrology managers, quality engineers, and calibration lab operators who need ISO/IEC 17025-traceable records, IATF 16949 and AS9100 compliance, and multi-site visibility — without the complexity, hardware lock-in, and opaque pricing of incumbent tools.

---

## Why Tool & Calibration Management?

- Incumbents such as Fluke MET/CAL and IndySoft deliver deep capability but require complex setup, specialist training, and tight coupling to specific instrument hardware ecosystems.
- Long-standing tools like GAGEpack and GAGEtrak are widely accepted by auditors but consistently described as having dated UIs and limited cloud or mobile support.
- CMMS-bundled calibration modules (eMaint, Qualityze) sacrifice metrology depth — quantitative multi-point measurements, uncertainty budgets, and MSA studies are basic or absent.
- Pricing ranges from ~$500–$1,500 perpetual licences for small shops to opaque enterprise SaaS contracts in the tens of thousands annually, with little middle ground for SMB-to-mid-market buyers wanting modern cloud features.
- Open-source or self-hosted alternatives are essentially absent from the market; the few that existed (e.g. Kalibro on SourceForge) are abandoned or unmaintained.

---

## Key Features

### Equipment Inventory & Identification

- Unique asset IDs with attributes, location, and status tracking across single or multiple sites
- Barcode and QR scanning for fast asset identification and check-in/check-out
- Role-based access control with user security groups
- Audit trail with immutable, timestamped change history

### Calibration Scheduling & Execution

- Configurable calibration intervals with automated due-date notifications
- Multi-point measurement capture with as-found and as-left readings, technician, date, and result
- Out-of-tolerance event logging with corrective action workflow
- Calibration certificate generation from recorded data with customisable templates

### Compliance & Quality Analysis

- Documentation aligned with ISO 9001 Clause 7.1.5, ISO/IEC 17025, IATF 16949, AS9100, and FDA 21 CFR Part 11
- MSA / gauge R&R study module covering variable and attribute studies
- Measurement uncertainty budget calculation aligned with GUM / JCGM 100
- Cross-site traceability dashboards for multi-site operations

### Integration & Mobility

- REST API for integration with CMMS, ERP, and QMS systems
- Bidirectional connectors for SAP PM, IBM Maximo, and Infor EAM
- Mobile app with offline calibration data capture and sync
- Webhook and no-code automation hooks for lightweight integrations

---

## AI-Native Advantage

AI capabilities planned for the platform target gaps that current incumbents do not address. Dynamic calibration interval prediction uses ML models trained on multi-cycle drift histories and actual instrument usage to replace fixed-interval scheduling, reducing unnecessary calibrations while preventing out-of-tolerance escapes. Automated ISO 17025-compliant certificate generation populates uncertainty budgets and traceability chains directly from raw measurement data. A natural language audit assistant cross-references calibration records against ISO 17025 and IATF 16949 clauses to surface gap items, and computer vision-assisted instrument identification streamlines high-volume toolroom workflows beyond what barcode scanning alone supports.

---

## Tech Stack & Deployment

The project is intended to support self-hosted and cloud deployment modes, with a mobile-first, offline-capable field calibration experience. A REST API is core to the architecture, enabling integration with CMMS, ERP, MES, and QMS systems. Implementation will derive from published standards (ISO/IEC 17025, ISO 10012, GUM/JCGM 100, AIAG MSA Reference Manual) rather than reverse-engineering proprietary tools.

---

## Market Context

The global calibration management software market was estimated at approximately USD 378 million in 2025, with a projected CAGR of roughly 3.9–5.4% through 2030–2032 (Technavio; Market Research Future; Metastat Insight). Pricing among incumbents spans perpetual licences from ~$500–$1,500 (Calibration Control, GAGEpack) to per-user SaaS from ~$30–$70/user/month (eMaint, GageList) and enterprise contracts in the tens of thousands annually (IndySoft, Fluke MET/CAL). Primary buyers are metrology and quality managers in manufacturing, calibration lab managers in aerospace, defence, and energy, automotive quality engineers under IATF 16949, and facilities managers in pharma and food production.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
