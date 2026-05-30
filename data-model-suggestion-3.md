# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Tool & Calibration Management · Created: 2026-05-22

## Philosophy

This model uses a relational backbone for core entities and relationships, but stores variable, domain-specific, and extensible data in PostgreSQL JSONB columns. The insight driving this design is that calibration management spans dozens of instrument types (pressure gauges, torque wrenches, multimeters, thermocouples, CMMs, oscilloscopes), each with different measurement parameters, calibration point structures, and tolerance specifications — but they all share the same lifecycle (register, schedule, calibrate, certificate, review).

Rather than creating separate tables for each instrument type's measurement schema (which would produce an unmanageable table explosion) or forcing all instrument types into a lowest-common-denominator fixed schema (which would lose critical domain specificity), the hybrid approach keeps the universal lifecycle relational while letting instrument-specific parameters live in well-structured JSONB fields validated by JSON Schema.

This pattern is widely used in modern SaaS products that serve diverse customer segments. Salesforce (which underpins Qualityze EQMS) uses a metadata-driven flexible schema. GageList's REST API supports custom fields per account. The hybrid approach captures the same flexibility at the database level while maintaining the relational integrity needed for cross-cutting queries (due-date dashboards, audit reports, traceability chains).

**Best for:** Rapid MVP development; platforms serving diverse instrument types and industry sectors; multi-tenant SaaS where tenants need custom fields without schema migrations.

**Trade-offs:**
- Pro: Dramatically fewer tables (~13 vs ~24+ in normalized) — simpler schema, faster development
- Pro: New instrument types or calibration parameters added without schema migrations
- Pro: Tenant-specific custom fields supported natively via JSONB
- Pro: GIN indexes on JSONB columns enable fast queries on variable fields
- Pro: Relational core still provides referential integrity, JOIN capability, and constraint enforcement
- Pro: JSON Schema validation at the application layer ensures JSONB data quality
- Con: JSONB fields are opaque to the database's type system — no column-level constraints or foreign keys
- Con: Complex JSONB queries can be slower than equivalent flat-column queries
- Con: Reporting tools and BI connectors may struggle with nested JSONB structures
- Con: JSON Schema validation must be enforced in the application, not the database
- Con: Risk of "JSONB junk drawer" if schema discipline is not maintained

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 17025:2017 | Core certificate fields relational; instrument-specific measurement data in structured JSONB |
| ISO 10012:2003/2025 | Equipment classification and measurement management attributes in JSONB with JSON Schema templates |
| JCGM 100:2008 (GUM) | Uncertainty budget stored as structured JSONB array following GUM component schema |
| PTB DCC XML Schema | JSONB measurement payloads designed for direct DCC XML export mapping |
| D-SI Data Model | JSONB measurement values follow D-SI atomic quantity pattern: {value, unit, uncertainty} |
| ILAC G8 | Decision rule fields relational; guard-band parameters in calibration JSONB |
| IATF 16949 | OOT investigation fields extensible via JSONB for automotive-specific requirements |
| FDA 21 CFR Part 11 | Audit trail relational with JSONB changed_fields for field-level tracking |
| JSON Schema Draft 2020-12 | All JSONB columns validated against registered JSON Schemas for data quality |
| GS1 EPCIS 2.0 | Equipment identification supports GS1 barcodes and RFID via JSONB extended_ids |
| ISO 3166-1/2 | Site country_code relational; sub-jurisdiction details in JSONB |

---

## Core Infrastructure

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    -- Tenant-level configuration: custom fields, label overrides, locale preferences
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Custom field definitions for this tenant
    custom_field_schemas JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "equipment": {
    --     "cost_centre": {"type": "string", "label": "Cost Centre", "required": false},
    --     "safety_class": {"type": "enum", "values": ["I","II","III"], "label": "Safety Class"}
    --   },
    --   "calibration_record": {
    --     "work_order_number": {"type": "string", "label": "SAP Work Order #"}
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2) NOT NULL,           -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    address         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"line1": "123 Industrial Rd", "city": "Detroit", "state": "MI", "postal": "48201"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_site_tenant ON site(tenant_id);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    roles           JSONB NOT NULL DEFAULT '[]',
    -- Example: ["calibration_technician", "quality_reviewer"]
    -- Permissions derived from role definitions in application config
    site_access     JSONB NOT NULL DEFAULT '[]',
    -- Example: ["site-uuid-1", "site-uuid-2"] or ["*"] for all sites
    is_active       BOOLEAN NOT NULL DEFAULT true,
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"certifications": ["ASQ CQT", "ISO 17025 Internal Auditor"], "phone": "+1-555-0123"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);
```

---

## Equipment Table (Hybrid)

```sql
CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),

    -- Core relational fields (universal across all equipment types)
    asset_tag       VARCHAR(100) NOT NULL,
    serial_number   VARCHAR(100),
    model_number    VARCHAR(100),
    manufacturer    VARCHAR(255),
    description     TEXT,
    equipment_type  VARCHAR(50) NOT NULL,      -- 'instrument', 'reference_standard', 'fixture', 'tool'
    category        VARCHAR(100) NOT NULL,      -- 'Pressure Gauge', 'Torque Wrench', 'Multimeter', etc.
    status          VARCHAR(50) NOT NULL DEFAULT 'in_service',

    -- JSONB: identification codes (barcode, RFID, customer tags)
    identifiers     JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "barcode": "GS1:01:09501101530003",
    --   "rfid": "urn:epc:id:sgtin:0614141.107346.2017",
    --   "customer_asset_id": "CUST-P-0042",
    --   "qr_code": "https://calibrate.example.com/eq/abc123"
    -- }

    -- JSONB: instrument-specific technical specifications
    specifications  JSONB NOT NULL DEFAULT '{}',
    -- Example for a pressure gauge:
    -- {
    --   "measurement_range": {"min": 0, "max": 100, "unit": "PSI"},
    --   "resolution": {"value": 0.01, "unit": "PSI"},
    --   "accuracy_class": "0.25",
    --   "connection_type": "1/4 NPT",
    --   "dial_size_mm": 100,
    --   "wetted_materials": ["316 SS"]
    -- }
    -- Example for a multimeter:
    -- {
    --   "dc_voltage_range": {"min": 0, "max": 1000, "unit": "V"},
    --   "ac_voltage_range": {"min": 0, "max": 1000, "unit": "V"},
    --   "resistance_range": {"min": 0, "max": 50, "unit": "MOhm"},
    --   "display_digits": 4.5,
    --   "true_rms": true,
    --   "cat_rating": "CAT IV 600V / CAT III 1000V"
    -- }

    -- JSONB: ownership, location, and administrative data
    location        JSONB NOT NULL DEFAULT '{}',
    -- Example: {"department": "Quality Lab", "building": "A", "room": "103", "bench": "2"}

    -- JSONB: financial and procurement data
    procurement     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"purchase_date": "2025-03-15", "purchase_cost": 450.00, "currency": "USD",
    --           "warranty_expiry": "2028-03-15", "vendor": "Instrumart", "po_number": "PO-2025-1234"}

    -- JSONB: tenant-defined custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"cost_centre": "CC-4500", "safety_class": "II"}

    -- Lifecycle
    custodian_id    UUID REFERENCES app_user(id),
    retired_at      TIMESTAMPTZ,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_equipment_asset_tag ON equipment(tenant_id, asset_tag);
CREATE INDEX idx_equipment_site ON equipment(site_id);
CREATE INDEX idx_equipment_status ON equipment(tenant_id, status);
CREATE INDEX idx_equipment_category ON equipment(tenant_id, category);
CREATE INDEX idx_equipment_type ON equipment(tenant_id, equipment_type);

-- GIN index on identifiers for fast barcode/RFID lookup
CREATE INDEX idx_equipment_identifiers ON equipment USING GIN (identifiers);

-- GIN index on specifications for filtering by instrument characteristics
CREATE INDEX idx_equipment_specs ON equipment USING GIN (specifications);

-- GIN index on custom fields for tenant-specific queries
CREATE INDEX idx_equipment_custom ON equipment USING GIN (custom_fields);
```

---

## Calibration Procedure Templates

```sql
CREATE TABLE calibration_procedure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    procedure_code  VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    method_reference VARCHAR(255),

    -- JSONB: procedure template defining measurement points
    measurement_template JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "sequence": 1, "label": "0% FS",
    --     "nominal": 0, "unit": "PSI",
    --     "tolerance": {"lower": -0.25, "upper": 0.25},
    --     "directions": ["up"],
    --     "readings_required": 3
    --   },
    --   {
    --     "sequence": 2, "label": "25% FS",
    --     "nominal": 25, "unit": "PSI",
    --     "tolerance": {"lower": -0.25, "upper": 0.25},
    --     "directions": ["up", "down"],
    --     "readings_required": 3
    --   }
    -- ]

    -- JSONB: environmental requirements
    environmental_requirements JSONB NOT NULL DEFAULT '{}',
    -- Example: {"temperature": {"min": 20, "max": 25, "unit": "C"},
    --           "humidity": {"max": 60, "unit": "%RH"}}

    -- JSONB: required reference standard categories
    required_standards JSONB NOT NULL DEFAULT '[]',
    -- Example: ["Pressure Standard (4:1 TUR)", "Barometer"]

    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, procedure_code, version)
);
```

---

## Calibration Records (Hybrid)

```sql
CREATE TABLE calibration_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    procedure_id    UUID REFERENCES calibration_procedure(id),
    interval_days   INTEGER NOT NULL,
    interval_source VARCHAR(50) NOT NULL DEFAULT 'fixed',
    next_due_date   DATE NOT NULL,
    notification_days_before INTEGER NOT NULL DEFAULT 30,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: AI prediction metadata (when interval_source = 'ai_predicted')
    prediction_metadata JSONB,
    -- Example: {"model_version": "drift-v2.1", "confidence": 0.87,
    --           "predicted_interval_days": 180, "drift_rate_per_day": 0.0012,
    --           "last_training_date": "2026-04-01"}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cal_schedule_equipment ON calibration_schedule(equipment_id);
CREATE INDEX idx_cal_schedule_due ON calibration_schedule(next_due_date) WHERE is_active = true;

CREATE TABLE calibration_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    schedule_id     UUID REFERENCES calibration_schedule(id),
    procedure_id    UUID REFERENCES calibration_procedure(id),

    -- Core relational fields
    certificate_number VARCHAR(100) NOT NULL,
    calibration_type VARCHAR(50) NOT NULL DEFAULT 'scheduled',
    status          VARCHAR(50) NOT NULL DEFAULT 'in_progress',
    result          VARCHAR(20),
    performed_by    UUID NOT NULL REFERENCES app_user(id),
    reviewed_by     UUID REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    date_calibrated TIMESTAMPTZ NOT NULL,
    date_approved   TIMESTAMPTZ,

    -- JSONB: environmental conditions during calibration
    environment     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"temperature_c": 22.5, "humidity_pct": 45.0,
    --           "atmospheric_pressure_hpa": 1013.25, "altitude_m": 210}

    -- JSONB: reference standards used (traceability)
    reference_standards JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "equipment_id": "uuid-...",
    --     "asset_tag": "STD-P-001",
    --     "description": "Fluke 2271A Pressure Controller",
    --     "certificate_number": "NIST-CAL-2026-0042",
    --     "certificate_date": "2025-11-20",
    --     "accreditation_body": "NIST",
    --     "uncertainty": {"value": 0.005, "unit": "PSI", "k": 2}
    --   }
    -- ]

    -- JSONB: all measurement points and results
    measurements    JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "sequence": 1, "label": "0% FS",
    --     "nominal": 0, "unit": "PSI",
    --     "tolerance": {"lower": -0.25, "upper": 0.25},
    --     "as_found": 0.02, "as_found_deviation": 0.02,
    --     "as_left": 0.01, "as_left_deviation": 0.01,
    --     "uncertainty": {"expanded": 0.08, "k": 2.0, "probability": 0.9545},
    --     "result": "in_tolerance",
    --     "conformity": "conforming",
    --     "direction": "up"
    --   }
    -- ]

    -- JSONB: uncertainty budget (GUM-aligned)
    uncertainty_budget JSONB,
    -- Example:
    -- {
    --   "measurand": "Pressure indication error",
    --   "unit": "PSI",
    --   "components": [
    --     {"source": "Reference standard", "type": "B", "distribution": "normal",
    --      "value": 0.005, "divisor": 2.0, "u": 0.0025, "ci": 1.0, "dof": "inf"},
    --     {"source": "DUT resolution", "type": "B", "distribution": "rectangular",
    --      "value": 0.01, "divisor": 1.732, "u": 0.00577, "ci": 1.0, "dof": "inf"},
    --     {"source": "Repeatability", "type": "A", "distribution": "normal",
    --      "value": 0.008, "divisor": 1.0, "u": 0.008, "ci": 1.0, "dof": 9}
    --   ],
    --   "combined_u": 0.0102,
    --   "effective_dof": 42.7,
    --   "k": 2.0,
    --   "expanded_U": 0.0204
    -- }

    -- Decision rule (ILAC G8)
    decision_rule   VARCHAR(100),
    guard_band_factor NUMERIC(5,3),
    traceability_statement TEXT,

    -- JSONB: technician and reviewer notes
    notes           JSONB NOT NULL DEFAULT '{}',
    -- Example: {"technician": "Gauge adjusted at 50% FS", "reviewer": "Accepted per SOP-CAL-007"}

    -- JSONB: tenant-defined custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"sap_work_order": "WO-2026-0815", "customer_po": "PO-ACME-42"}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_cal_record_cert ON calibration_record(tenant_id, certificate_number);
CREATE INDEX idx_cal_record_equipment ON calibration_record(equipment_id);
CREATE INDEX idx_cal_record_date ON calibration_record(tenant_id, date_calibrated);
CREATE INDEX idx_cal_record_status ON calibration_record(tenant_id, status);
CREATE INDEX idx_cal_record_result ON calibration_record(tenant_id, result);

-- GIN index for querying inside measurements JSONB
CREATE INDEX idx_cal_record_measurements ON calibration_record USING GIN (measurements);
```

---

## Out-of-Tolerance and Corrective Actions

```sql
CREATE TABLE oot_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    calibration_record_id UUID NOT NULL REFERENCES calibration_record(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),

    severity        VARCHAR(20) NOT NULL,
    disposition     VARCHAR(50) NOT NULL DEFAULT 'pending',

    -- JSONB: the specific measurement(s) that triggered the OOT
    triggering_measurements JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"sequence": 3, "label": "50% FS", "nominal": 50, "as_found": 50.95,
    --            "tolerance_upper": 0.25, "deviation": 0.95}]

    -- JSONB: impact assessment details
    impact          JSONB NOT NULL DEFAULT '{}',
    -- Example: {"products_affected": "Lot 2026-Q1-0042 through 2026-Q1-0055",
    --           "retroactive_required": true, "retroactive_start": "2025-11-20",
    --           "retroactive_end": "2026-05-22", "risk_level": "high"}

    -- JSONB: corrective actions
    corrective_actions JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"id": "uuid-...", "type": "correction", "description": "Adjusted and recalibrated",
    --    "assigned_to": "uuid-...", "due_date": "2026-05-30", "status": "completed",
    --    "completed_at": "2026-05-25"},
    --   {"id": "uuid-...", "type": "corrective_action", "description": "Review product lots",
    --    "assigned_to": "uuid-...", "due_date": "2026-06-15", "status": "open"}
    -- ]

    reported_by     UUID REFERENCES app_user(id),
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_by     UUID REFERENCES app_user(id),
    resolved_at     TIMESTAMPTZ,
    resolution_notes TEXT,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oot_event_tenant ON oot_event(tenant_id);
CREATE INDEX idx_oot_event_equipment ON oot_event(equipment_id);
CREATE INDEX idx_oot_event_open ON oot_event(tenant_id, disposition) WHERE disposition = 'pending';
```

---

## MSA / Gauge R&R (Hybrid)

```sql
CREATE TABLE msa_study (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    study_type      VARCHAR(50) NOT NULL,
    study_method    VARCHAR(50) NOT NULL,
    methodology_ref VARCHAR(100),

    -- JSONB: study configuration
    study_config    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"num_operators": 3, "num_parts": 10, "num_trials": 3,
    --           "process_tolerance": 1.0, "tolerance_unit": "mm"}

    -- JSONB: all raw measurements (operator x part x trial matrix)
    raw_measurements JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"operator_id": "uuid-1", "operator_name": "Alice", "part": 1, "trial": 1, "value": 25.012, "unit": "mm"},
    --   {"operator_id": "uuid-1", "operator_name": "Alice", "part": 1, "trial": 2, "value": 25.015, "unit": "mm"},
    --   ...
    -- ]

    -- JSONB: computed results
    results         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "repeatability_ev": 0.0123,
    --   "reproducibility_av": 0.0089,
    --   "gage_rr": 0.0152,
    --   "part_variation_pv": 0.4521,
    --   "total_variation_tv": 0.4524,
    --   "pct_tolerance_grr": 1.52,
    --   "pct_study_grr": 3.36,
    --   "ndc": 42,
    --   "result": "acceptable"
    -- }

    performed_by    UUID REFERENCES app_user(id),
    study_date      DATE NOT NULL,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_msa_study_equipment ON msa_study(equipment_id);
```

---

## Equipment Check-In/Check-Out

```sql
CREATE TABLE equipment_checkout (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    checked_out_by  UUID NOT NULL REFERENCES app_user(id),
    checked_out_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expected_return DATE,
    purpose         VARCHAR(255),
    checked_in_at   TIMESTAMPTZ,
    checked_in_by   UUID REFERENCES app_user(id),
    return_notes    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_checkout_equipment ON equipment_checkout(equipment_id);
CREATE INDEX idx_checkout_active ON equipment_checkout(equipment_id) WHERE checked_in_at IS NULL;
```

---

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changed_fields  JSONB,
    performed_by    UUID,
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    signature_meaning VARCHAR(100),
    ip_address      INET
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, performed_at);
CREATE INDEX idx_audit_log_record ON audit_log(table_name, record_id);
```

---

## Notification Rules

```sql
CREATE TABLE notification_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    event_type      VARCHAR(100) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example: {"days_before": 30, "channel": "email", "recipient_roles": ["quality_manager"],
    --           "template": "calibration_due_reminder"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## JSON Schema Validation Registry

```sql
-- JSON Schemas used to validate JSONB columns at the application layer
CREATE TABLE json_schema_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID,                       -- null = system-wide schema
    schema_name     VARCHAR(100) NOT NULL,       -- e.g. 'equipment.specifications.pressure_gauge'
    schema_version  INTEGER NOT NULL DEFAULT 1,
    json_schema     JSONB NOT NULL,              -- JSON Schema Draft 2020-12 document
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, schema_name, schema_version)
);

-- Example: registering a schema for pressure gauge specifications
-- INSERT INTO json_schema_registry (schema_name, json_schema, description) VALUES (
--   'equipment.specifications.pressure_gauge',
--   '{
--     "$schema": "https://json-schema.org/draft/2020-12/schema",
--     "type": "object",
--     "properties": {
--       "measurement_range": {
--         "type": "object",
--         "properties": {"min": {"type": "number"}, "max": {"type": "number"}, "unit": {"type": "string"}},
--         "required": ["min", "max", "unit"]
--       },
--       "resolution": {"type": "object", "properties": {"value": {"type": "number"}, "unit": {"type": "string"}}},
--       "accuracy_class": {"type": "string"},
--       "connection_type": {"type": "string"},
--       "dial_size_mm": {"type": "number"}
--     },
--     "required": ["measurement_range"]
--   }',
--   'Technical specification schema for pressure gauge equipment'
-- );
```

---

## JSONB Query Examples

### Find all pressure gauges with range above 500 PSI

```sql
SELECT id, asset_tag, specifications
FROM equipment
WHERE tenant_id = :tenant_id
  AND category = 'Pressure Gauge'
  AND (specifications->'measurement_range'->>'max')::numeric > 500
  AND specifications->'measurement_range'->>'unit' = 'PSI';
```

### Find all calibration records with out-of-tolerance measurement points

```sql
SELECT
    cr.id,
    cr.certificate_number,
    cr.date_calibrated,
    e.asset_tag,
    meas->>'label' AS point_label,
    meas->>'as_found' AS as_found,
    meas->>'nominal' AS nominal
FROM calibration_record cr
JOIN equipment e ON e.id = cr.equipment_id
CROSS JOIN LATERAL jsonb_array_elements(cr.measurements) AS meas
WHERE cr.tenant_id = :tenant_id
  AND meas->>'result' = 'out_of_tolerance'
  AND cr.date_calibrated >= '2026-01-01';
```

### Look up equipment by barcode or RFID tag

```sql
SELECT id, asset_tag, category, status
FROM equipment
WHERE tenant_id = :tenant_id
  AND identifiers @> '{"barcode": "GS1:01:09501101530003"}'::jsonb;

-- Or by RFID:
SELECT id, asset_tag, category, status
FROM equipment
WHERE tenant_id = :tenant_id
  AND identifiers @> '{"rfid": "urn:epc:id:sgtin:0614141.107346.2017"}'::jsonb;
```

### Extract uncertainty budget components for reporting

```sql
SELECT
    cr.certificate_number,
    cr.uncertainty_budget->>'measurand' AS measurand,
    comp->>'source' AS source,
    comp->>'type' AS eval_type,
    comp->>'u' AS standard_uncertainty,
    cr.uncertainty_budget->>'expanded_U' AS expanded_uncertainty
FROM calibration_record cr
CROSS JOIN LATERAL jsonb_array_elements(cr.uncertainty_budget->'components') AS comp
WHERE cr.id = :calibration_record_id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Infrastructure | 3 | tenant, site, app_user |
| Equipment Management | 1 | equipment (with JSONB for specs, identifiers, custom fields) |
| Calibration Procedures | 1 | calibration_procedure (with JSONB measurement template) |
| Calibration Execution | 2 | calibration_schedule, calibration_record (with JSONB measurements, uncertainty) |
| Out-of-Tolerance / CAPA | 1 | oot_event (with JSONB impact, corrective actions) |
| MSA / Gauge R&R | 1 | msa_study (with JSONB raw measurements, results) |
| Tool Tracking | 1 | equipment_checkout |
| Audit & Compliance | 1 | audit_log |
| Notifications | 1 | notification_rule |
| Schema Validation | 1 | json_schema_registry |
| **Total** | **13** | Significantly fewer tables than normalized; JSONB carries the variability |

---

## Key Design Decisions

1. **Relational for lifecycle, JSONB for variability** — the clear separation between "what every piece of equipment shares" (asset_tag, status, site, custodian) and "what varies by instrument type" (specifications, measurement points, uncertainty budgets) drives the hybrid split. New instrument types never require ALTER TABLE.

2. **Measurements stored as JSONB array in calibration_record** — rather than a separate measurement_point table, all measurements for a calibration are stored together. This simplifies the common access pattern (load a complete calibration record with all its measurements) while still supporting cross-record queries via jsonb_array_elements and GIN indexes.

3. **Corrective actions embedded in oot_event JSONB** — for most deployments, an OOT event has 1-3 corrective actions. Embedding them avoids a JOIN for the most common read pattern. If a deployment needs complex CAPA management, this can be extracted to a separate table later.

4. **JSON Schema registry for validation** — since PostgreSQL does not natively validate JSONB against a schema, the json_schema_registry table stores JSON Schema documents that the application layer uses to validate data before writes. This provides the data quality guarantees that JSONB alone cannot.

5. **Custom fields via tenant-level schema definitions** — tenants can define custom fields (in custom_field_schemas on the tenant table) that appear in the custom_fields JSONB on equipment and calibration_record. No schema migration needed per tenant.

6. **GIN indexes on JSONB columns** — enables fast containment queries (@>) for barcode/RFID lookups and equipment specification filtering. The trade-off is increased index maintenance cost on writes, but reads are the dominant workload in calibration management.

7. **D-SI-compatible measurement representation** — JSONB measurement values follow the {value, unit, uncertainty} pattern from the PTB D-SI data model, making future DCC XML export straightforward.

8. **AI prediction metadata in schedule** — the prediction_metadata JSONB field on calibration_schedule captures the model version, confidence score, and drift rate used by the AI interval prediction engine, enabling reproducibility and audit of AI-driven scheduling decisions.

9. **Single equipment table for all types** — unlike the normalized model which might separate reference_standard from instrument, this model uses one equipment table with equipment_type as a discriminator. The specifications JSONB adapts to each type. This reduces JOINs and simplifies the API.

10. **Audit log remains relational** — even in a JSONB-heavy model, the audit log uses a simple relational structure because it needs to be append-only, immutable, and efficiently queryable by time range and user — all properties better served by flat relational columns with B-tree indexes.
