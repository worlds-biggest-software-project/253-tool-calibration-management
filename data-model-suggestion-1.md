# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Tool & Calibration Management · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design, giving every domain concept its own table with explicit foreign-key relationships. It is the approach most naturally aligned with the ISO/IEC 17025 and ISO 10012 standards, which define calibration data as a set of discrete, auditable records with well-defined relationships between laboratories, equipment, reference standards, measurement results, and certificates.

Normalized relational models are the backbone of regulated industries where data integrity, referential consistency, and audit readiness are non-negotiable. Products like Fluke MET/TEAM, IndySoft, and Blue Mountain RAM all use relational databases as their core storage engine. Regulatory frameworks (FDA 21 CFR Part 11, ISO 17025 Clause 7.5, IATF 16949 Clause 7.1.5.1) assume that records are discrete, immutable after approval, and cross-referenced — all properties that normalized relational design enforces naturally.

The trade-off is a higher table count and more complex JOIN queries, but this is offset by excellent tooling support (PostgreSQL, standard ORMs), straightforward schema migration, and the ability to enforce business rules at the database level through constraints and triggers.

**Best for:** Organisations prioritising ISO 17025 accreditation, regulatory compliance, and long-term data integrity over rapid schema evolution.

**Trade-offs:**
- Pro: Strong referential integrity enforced at the database level
- Pro: Directly maps to ISO 17025 certificate and record requirements
- Pro: Mature tooling, well-understood query patterns, excellent reporting support
- Pro: Natural fit for multi-site, multi-tenant deployments with row-level security
- Con: High table count (~40-50 tables) increases schema complexity
- Con: Adding new instrument types or jurisdiction-specific fields requires schema migrations
- Con: Complex queries involving deep joins (traceability chains) can be slow without careful indexing
- Con: Less flexible for rapid prototyping compared to document or hybrid approaches

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 17025:2017 | Calibration certificate fields (Clause 7.8), equipment records (Clause 6.4), technical records (Clause 7.5) directly map to table structures |
| ISO 10012:2003/2025 | Measurement management system framework informs the equipment-to-calibration-schedule relationship model |
| JCGM 100:2008 (GUM) | Uncertainty budget table structure follows GUM Type A/B component decomposition |
| JCGM 200:2012 (VIM) | Field naming uses VIM-compliant terminology (measurand, indication, measurement uncertainty) |
| PTB DCC XML Schema | Certificate data fields align with DCC administrative and measurement result elements for future DCC export |
| D-SI Data Model | Measurement result representation (value + unit + uncertainty) follows D-SI atomic quantity structure |
| ILAC P14/G8 | Decision rule and guard-band fields in calibration results support ILAC conformity assessment |
| FDA 21 CFR Part 11 | Audit trail table with electronic signature fields satisfies Part 11.10(e) requirements |
| IATF 16949 | MSA/gauge R&R study tables with AIAG methodology fields support automotive supplier compliance |
| GS1 EPCIS 2.0 | Asset identification fields support GS1 barcode and RFID identifier formats |
| ISO 3166-1/2 | Location and jurisdiction fields use ISO 3166 codes for multi-region deployments |
| iCalendar RFC 5545 | Calibration schedule recurrence fields align with iCalendar RRULE for calendar export |

---

## Core Infrastructure Tables

### Tenant and Organisation

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    address_line_1  VARCHAR(255),
    address_line_2  VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,        -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, slug)
);

CREATE INDEX idx_site_tenant ON site(tenant_id);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),            -- null when SSO-only
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,   -- e.g. 'calibration_technician', 'quality_manager', 'auditor'
    permissions     JSONB NOT NULL DEFAULT '[]',
    is_system_role  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    site_id         UUID REFERENCES site(id), -- null = all sites
    granted_by      UUID REFERENCES app_user(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, site_id)
);
```

---

## Equipment Management Tables

```sql
CREATE TABLE equipment_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,   -- e.g. 'Pressure Gauge', 'Torque Wrench', 'Multimeter'
    description     TEXT,
    parent_id       UUID REFERENCES equipment_category(id),  -- hierarchical categories
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name, parent_id)
);

CREATE TABLE manufacturer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    website         VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    category_id     UUID REFERENCES equipment_category(id),
    manufacturer_id UUID REFERENCES manufacturer(id),

    -- Identification
    asset_tag       VARCHAR(100) NOT NULL,    -- internal asset ID
    serial_number   VARCHAR(100),
    model_number    VARCHAR(100),
    barcode         VARCHAR(100),             -- GS1-128 / QR / DataMatrix value
    rfid_tag        VARCHAR(100),             -- EPCIS identifier if RFID-equipped
    description     TEXT,

    -- Classification
    equipment_type  VARCHAR(50) NOT NULL,     -- 'instrument', 'reference_standard', 'fixture', 'tool'
    accuracy_class  VARCHAR(50),              -- manufacturer-specified accuracy class
    measurement_range VARCHAR(100),           -- e.g. '0-100 PSI', '0-1000V'
    resolution      VARCHAR(50),             -- e.g. '0.01 mm'

    -- Ownership and location
    department      VARCHAR(100),
    custodian_id    UUID REFERENCES app_user(id),
    location_detail VARCHAR(255),             -- room, bench, toolbox etc.

    -- Lifecycle
    status          VARCHAR(50) NOT NULL DEFAULT 'in_service',
    -- Values: 'in_service', 'due_calibration', 'overdue', 'out_of_tolerance',
    --         'in_calibration', 'quarantined', 'retired', 'lost'
    purchase_date   DATE,
    warranty_expiry DATE,
    retirement_date DATE,
    cost_centre     VARCHAR(50),
    purchase_cost   NUMERIC(12,2),
    currency_code   CHAR(3),                  -- ISO 4217

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_equipment_asset_tag ON equipment(tenant_id, asset_tag);
CREATE INDEX idx_equipment_site ON equipment(site_id);
CREATE INDEX idx_equipment_status ON equipment(tenant_id, status);
CREATE INDEX idx_equipment_category ON equipment(category_id);
CREATE INDEX idx_equipment_barcode ON equipment(tenant_id, barcode) WHERE barcode IS NOT NULL;

CREATE TABLE equipment_document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id) ON DELETE CASCADE,
    document_type   VARCHAR(50) NOT NULL,     -- 'manual', 'datasheet', 'photo', 'sop'
    file_name       VARCHAR(255) NOT NULL,
    file_path       VARCHAR(1000) NOT NULL,
    mime_type       VARCHAR(100),
    file_size_bytes BIGINT,
    uploaded_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equipment_document_equip ON equipment_document(equipment_id);
```

---

## Calibration Scheduling Tables

```sql
CREATE TABLE calibration_procedure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    procedure_code  VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    version         INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- 'draft', 'active', 'superseded'
    method_reference VARCHAR(255),           -- e.g. 'ASTM E74', 'ISO 6789', lab SOP number
    measurement_points_template JSONB,       -- default set points for this procedure
    -- Example: [{"nominal": 10, "unit": "PSI", "tolerance_lower": -0.5, "tolerance_upper": 0.5}]
    environmental_requirements JSONB,        -- temperature, humidity ranges
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, procedure_code, version)
);

CREATE TABLE calibration_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    procedure_id    UUID REFERENCES calibration_procedure(id),
    interval_days   INTEGER NOT NULL,         -- calibration interval in days
    interval_source VARCHAR(50) NOT NULL DEFAULT 'fixed',
    -- Values: 'fixed', 'ncsl_adjusted', 'ai_predicted', 'manufacturer_recommended'
    last_calibration_date DATE,
    next_due_date   DATE NOT NULL,
    notification_days_before INTEGER NOT NULL DEFAULT 30,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cal_schedule_equipment ON calibration_schedule(equipment_id);
CREATE INDEX idx_cal_schedule_due ON calibration_schedule(next_due_date) WHERE is_active = true;
```

---

## Calibration Execution Tables

```sql
CREATE TABLE calibration_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    schedule_id     UUID REFERENCES calibration_schedule(id),
    procedure_id    UUID REFERENCES calibration_procedure(id),

    -- Certificate identification
    certificate_number VARCHAR(100) NOT NULL,
    certificate_revision INTEGER NOT NULL DEFAULT 0,

    -- Calibration metadata
    calibration_type VARCHAR(50) NOT NULL DEFAULT 'scheduled',
    -- Values: 'scheduled', 'unscheduled', 'initial', 'post_repair', 'reverification'
    status          VARCHAR(50) NOT NULL DEFAULT 'in_progress',
    -- Values: 'in_progress', 'pending_review', 'approved', 'rejected', 'voided'
    result          VARCHAR(20),              -- 'pass', 'fail', 'pass_with_adjustment'

    -- Personnel
    performed_by    UUID NOT NULL REFERENCES app_user(id),
    reviewed_by     UUID REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),

    -- Dates
    date_received   TIMESTAMPTZ,
    date_calibrated TIMESTAMPTZ NOT NULL,
    date_completed  TIMESTAMPTZ,
    date_approved   TIMESTAMPTZ,

    -- Location
    calibration_site_id UUID REFERENCES site(id),
    calibration_location VARCHAR(255),        -- specific lab, bench, or field location

    -- Environmental conditions during calibration
    ambient_temperature_c NUMERIC(5,1),
    ambient_humidity_pct  NUMERIC(5,1),
    atmospheric_pressure_hpa NUMERIC(7,2),

    -- Decision rule (ILAC G8)
    decision_rule   VARCHAR(100),             -- e.g. 'simple_acceptance', 'guard_banded_95'
    guard_band_factor NUMERIC(5,3),

    -- Traceability
    traceability_statement TEXT,              -- SI traceability chain description

    -- Notes
    technician_notes TEXT,
    reviewer_notes  TEXT,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_cal_record_cert ON calibration_record(tenant_id, certificate_number, certificate_revision);
CREATE INDEX idx_cal_record_equipment ON calibration_record(equipment_id);
CREATE INDEX idx_cal_record_date ON calibration_record(tenant_id, date_calibrated);
CREATE INDEX idx_cal_record_status ON calibration_record(tenant_id, status);

-- Reference standards used during calibration (traceability)
CREATE TABLE calibration_reference_standard (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    calibration_record_id UUID NOT NULL REFERENCES calibration_record(id) ON DELETE CASCADE,
    reference_equipment_id UUID NOT NULL REFERENCES equipment(id),
    certificate_number  VARCHAR(100),         -- certificate of the reference standard itself
    certificate_date    DATE,
    accreditation_body  VARCHAR(100),         -- e.g. 'NIST', 'UKAS', 'DAkkS'
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cal_ref_std_record ON calibration_reference_standard(calibration_record_id);

-- Individual measurement points within a calibration
CREATE TABLE measurement_point (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    calibration_record_id UUID NOT NULL REFERENCES calibration_record(id) ON DELETE CASCADE,
    sequence_number     INTEGER NOT NULL,
    point_label         VARCHAR(100),         -- e.g. '50% FS', 'Set Point 3'

    -- Nominal / target value
    nominal_value       NUMERIC(20,8) NOT NULL,
    unit                VARCHAR(50) NOT NULL, -- SI unit or derived (e.g. 'Pa', 'V', 'mm', 'N·m')

    -- Tolerance
    tolerance_lower     NUMERIC(20,8),
    tolerance_upper     NUMERIC(20,8),
    tolerance_unit      VARCHAR(50),

    -- As-found reading (before adjustment)
    as_found_value      NUMERIC(20,8),
    as_found_deviation  NUMERIC(20,8),

    -- As-left reading (after adjustment, if any)
    as_left_value       NUMERIC(20,8),
    as_left_deviation   NUMERIC(20,8),

    -- Measurement uncertainty (GUM)
    expanded_uncertainty NUMERIC(20,8),
    coverage_factor     NUMERIC(5,2) DEFAULT 2.0,  -- k factor, typically 2 for 95%
    coverage_probability NUMERIC(5,4) DEFAULT 0.9545,
    uncertainty_unit    VARCHAR(50),

    -- Result
    point_result        VARCHAR(20) NOT NULL,  -- 'in_tolerance', 'out_of_tolerance', 'adjusted'
    conformity_statement VARCHAR(50),           -- 'conforming', 'non_conforming', 'not_stated' per ILAC G8

    -- Direction (for hysteresis checks)
    direction           VARCHAR(10),           -- 'up', 'down'

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_meas_point_record ON measurement_point(calibration_record_id);
```

---

## Uncertainty Budget Tables

```sql
-- GUM-aligned uncertainty budget for a calibration record
CREATE TABLE uncertainty_budget (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    calibration_record_id UUID NOT NULL REFERENCES calibration_record(id) ON DELETE CASCADE,
    measurement_point_id UUID REFERENCES measurement_point(id),
    budget_label        VARCHAR(255),         -- e.g. 'Pressure at 50 PSI'
    measurand           VARCHAR(255) NOT NULL,
    measurand_unit      VARCHAR(50) NOT NULL,
    combined_standard_uncertainty NUMERIC(20,10),
    expanded_uncertainty NUMERIC(20,10),
    coverage_factor     NUMERIC(5,2) DEFAULT 2.0,
    effective_degrees_of_freedom NUMERIC(10,1),  -- Welch-Satterthwaite
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_uncertainty_budget_record ON uncertainty_budget(calibration_record_id);

-- Individual uncertainty components (Type A and Type B)
CREATE TABLE uncertainty_component (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_id           UUID NOT NULL REFERENCES uncertainty_budget(id) ON DELETE CASCADE,
    sequence_number     INTEGER NOT NULL,
    source_description  VARCHAR(255) NOT NULL, -- e.g. 'Resolution of DUT', 'Reference standard uncertainty'
    evaluation_type     CHAR(1) NOT NULL,      -- 'A' (statistical) or 'B' (other means)
    distribution        VARCHAR(50),           -- 'normal', 'rectangular', 'triangular', 'u_shaped'
    input_value         NUMERIC(20,10),
    divisor             NUMERIC(10,4),         -- e.g. sqrt(3) for rectangular
    standard_uncertainty NUMERIC(20,10) NOT NULL,
    sensitivity_coefficient NUMERIC(20,10) DEFAULT 1.0,
    contribution        NUMERIC(20,10),        -- (ci * ui)^2
    degrees_of_freedom  NUMERIC(10,1),
    unit                VARCHAR(50),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_uncertainty_component_budget ON uncertainty_component(budget_id);
```

---

## Out-of-Tolerance and Corrective Action Tables

```sql
CREATE TABLE oot_event (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    calibration_record_id UUID NOT NULL REFERENCES calibration_record(id),
    equipment_id        UUID NOT NULL REFERENCES equipment(id),
    measurement_point_id UUID REFERENCES measurement_point(id),

    severity            VARCHAR(20) NOT NULL,  -- 'minor', 'major', 'critical'
    deviation_value     NUMERIC(20,8),
    deviation_unit      VARCHAR(50),

    -- Impact assessment (IATF 16949 requirement)
    impact_assessment   TEXT,
    products_affected   TEXT,
    retroactive_action_required BOOLEAN NOT NULL DEFAULT false,
    retroactive_period_start DATE,
    retroactive_period_end DATE,

    -- Disposition
    disposition         VARCHAR(50) NOT NULL DEFAULT 'pending',
    -- Values: 'pending', 'use_as_is', 'adjust_recalibrate', 'repair', 'retire', 'quarantine'

    reported_by         UUID REFERENCES app_user(id),
    reported_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_by         UUID REFERENCES app_user(id),
    resolved_at         TIMESTAMPTZ,
    resolution_notes    TEXT,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oot_event_tenant ON oot_event(tenant_id);
CREATE INDEX idx_oot_event_equipment ON oot_event(equipment_id);

CREATE TABLE corrective_action (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenant(id),
    oot_event_id        UUID REFERENCES oot_event(id),
    action_type         VARCHAR(50) NOT NULL,  -- 'correction', 'corrective_action', 'preventive_action'
    description         TEXT NOT NULL,
    assigned_to         UUID REFERENCES app_user(id),
    due_date            DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'open',
    -- Values: 'open', 'in_progress', 'completed', 'verified', 'closed'
    completed_at        TIMESTAMPTZ,
    verified_by         UUID REFERENCES app_user(id),
    verified_at         TIMESTAMPTZ,
    effectiveness_review TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_corrective_action_oot ON corrective_action(oot_event_id);
```

---

## MSA / Gauge R&R Tables

```sql
CREATE TABLE msa_study (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    study_type      VARCHAR(50) NOT NULL,     -- 'gage_rr_crossed', 'gage_rr_nested', 'linearity', 'bias', 'stability'
    study_method    VARCHAR(50) NOT NULL,     -- 'xbar_r', 'anova', 'range'
    methodology_ref VARCHAR(100),             -- 'AIAG MSA 4th Edition', 'ISO 22514-7'

    -- Study parameters
    num_operators   INTEGER NOT NULL DEFAULT 3,
    num_parts       INTEGER NOT NULL DEFAULT 10,
    num_trials      INTEGER NOT NULL DEFAULT 3,
    process_tolerance NUMERIC(20,8),
    tolerance_unit  VARCHAR(50),

    -- Results
    repeatability_ev NUMERIC(20,8),           -- Equipment Variation
    reproducibility_av NUMERIC(20,8),         -- Appraiser Variation
    gage_rr         NUMERIC(20,8),            -- Total Gage R&R
    part_variation_pv NUMERIC(20,8),
    total_variation_tv NUMERIC(20,8),
    pct_tolerance_gage_rr NUMERIC(8,4),       -- %GRR vs tolerance
    pct_study_gage_rr NUMERIC(8,4),           -- %GRR vs study variation
    ndc             INTEGER,                   -- Number of Distinct Categories

    -- Disposition
    result          VARCHAR(20),              -- 'acceptable', 'marginal', 'unacceptable'
    -- Acceptable: <10% GRR, Marginal: 10-30%, Unacceptable: >30%

    performed_by    UUID REFERENCES app_user(id),
    study_date      DATE NOT NULL,
    notes           TEXT,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_msa_study_equipment ON msa_study(equipment_id);

CREATE TABLE msa_measurement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    study_id        UUID NOT NULL REFERENCES msa_study(id) ON DELETE CASCADE,
    operator_id     UUID NOT NULL REFERENCES app_user(id),
    part_number     INTEGER NOT NULL,
    trial_number    INTEGER NOT NULL,
    measured_value  NUMERIC(20,8) NOT NULL,
    unit            VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_msa_measurement_study ON msa_measurement(study_id);
CREATE UNIQUE INDEX idx_msa_measurement_unique ON msa_measurement(study_id, operator_id, part_number, trial_number);
```

---

## Audit Trail Table

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,     -- 'INSERT', 'UPDATE', 'DELETE'
    changed_fields  JSONB,                    -- {"field": {"old": "x", "new": "y"}}
    performed_by    UUID REFERENCES app_user(id),
    performed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Electronic signature (FDA 21 CFR Part 11)
    signature_meaning VARCHAR(100),           -- 'created', 'reviewed', 'approved', 'modified'
    signature_method  VARCHAR(50),            -- 'password', 'mfa', 'biometric'
    ip_address      INET,
    user_agent      VARCHAR(500)
);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, performed_at);
CREATE INDEX idx_audit_log_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_log_user ON audit_log(performed_by, performed_at);

-- Partition by month for performance on large audit log tables
-- CREATE TABLE audit_log (...) PARTITION BY RANGE (performed_at);
```

---

## Notification and Scheduling Tables

```sql
CREATE TABLE notification_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    event_type      VARCHAR(100) NOT NULL,
    -- Values: 'calibration_due', 'calibration_overdue', 'oot_event', 'approval_required',
    --         'certificate_expiring', 'study_due'
    days_before     INTEGER,                  -- for due-date based notifications
    channel         VARCHAR(20) NOT NULL DEFAULT 'email',  -- 'email', 'sms', 'webhook', 'in_app'
    recipient_role  VARCHAR(100),
    recipient_user_id UUID REFERENCES app_user(id),
    template_name   VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notification_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    rule_id         UUID REFERENCES notification_rule(id),
    recipient_email VARCHAR(255),
    subject         VARCHAR(500),
    channel         VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL,     -- 'sent', 'failed', 'bounced'
    related_table   VARCHAR(100),
    related_id      UUID,
    sent_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    error_message   TEXT
);

CREATE INDEX idx_notification_log_tenant ON notification_log(tenant_id, sent_at);
```

---

## Equipment Check-In/Check-Out Tables

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
    condition_on_return VARCHAR(50),          -- 'good', 'damaged', 'needs_repair'
    return_notes    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_checkout_equipment ON equipment_checkout(equipment_id);
CREATE INDEX idx_checkout_active ON equipment_checkout(equipment_id) WHERE checked_in_at IS NULL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Infrastructure | 5 | tenant, site, app_user, role, user_role |
| Equipment Management | 4 | equipment, equipment_category, manufacturer, equipment_document |
| Calibration Scheduling | 2 | calibration_procedure, calibration_schedule |
| Calibration Execution | 3 | calibration_record, calibration_reference_standard, measurement_point |
| Uncertainty Budgets | 2 | uncertainty_budget, uncertainty_component |
| Out-of-Tolerance / CAPA | 2 | oot_event, corrective_action |
| MSA / Gauge R&R | 2 | msa_study, msa_measurement |
| Audit & Compliance | 1 | audit_log |
| Notifications | 2 | notification_rule, notification_log |
| Tool Tracking | 1 | equipment_checkout |
| **Total** | **24** | Core schema; grows with features |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables multi-site merge, API-first design, and avoids sequential ID leakage in a multi-tenant system.

2. **Tenant ID on every operational table** — supports PostgreSQL row-level security (RLS) policies for data isolation. Composite indexes always lead with tenant_id.

3. **Separate measurement_point table rather than JSONB array** — each measurement point is a first-class row, enabling SQL queries like "find all out-of-tolerance readings across all equipment this quarter" without JSONB extraction.

4. **GUM-aligned uncertainty budget decomposition** — the uncertainty_budget and uncertainty_component tables mirror the standard GUM uncertainty budget worksheet (source, type, distribution, divisor, sensitivity coefficient), enabling automated uncertainty calculation and reporting.

5. **DCC-compatible certificate fields** — calibration_record fields are designed to export directly to PTB Digital Calibration Certificate (DCC) XML format, future-proofing for Industry 4.0 metrology data exchange.

6. **ILAC G8 decision rule support** — the decision_rule and guard_band_factor fields on calibration_record, combined with conformity_statement on measurement_point, support auditable conformity assessment per ILAC G8.

7. **IATF 16949 out-of-tolerance investigation** — the oot_event table includes impact_assessment, products_affected, and retroactive_action fields required by automotive quality audits.

8. **Audit log as a single append-only table** — satisfies FDA 21 CFR Part 11 requirements with changed_fields JSONB for field-level change tracking, electronic signature fields, and partition-readiness for performance at scale.

9. **Reference standard traceability as an explicit junction table** — calibration_reference_standard links each calibration to the reference standards used, with their own certificate numbers and accreditation bodies, creating a queryable traceability chain.

10. **Calibration procedure versioning** — procedure_code + version allows procedure evolution while maintaining historical links to which version was used for each calibration record.
