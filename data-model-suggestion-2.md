# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Tool & Calibration Management · Created: 2026-05-22

## Philosophy

This model treats every state change in the calibration management system as an immutable event appended to an event store. The event store is the single source of truth; all queryable read models (equipment inventory, calibration status, certificate data, compliance dashboards) are materialised projections rebuilt from the event stream. This is the Command Query Responsibility Segregation (CQRS) pattern combined with event sourcing.

The approach is directly inspired by regulated financial ledger systems (double-entry bookkeeping is itself an event-sourced pattern), the Open Cap Format (OCF) which uses event-based transaction histories for securities lifecycle tracking, and the PTB Digital Calibration Certificate (DCC) initiative which emphasises machine-readable, traceable, and tamper-evident calibration data. In a calibration management context, every calibration performed, every out-of-tolerance finding, every equipment status change, and every certificate approval is a discrete, auditable event -- making event sourcing a natural architectural fit.

The central advantage for calibration management is that the system provides a complete, immutable, cryptographically verifiable audit trail by construction -- not as an afterthought. FDA 21 CFR Part 11 requires that electronic records include secure, timestamped audit trails; ISO/IEC 17025 Clause 7.5 requires that technical records be traceable and unalterable after approval. Event sourcing satisfies both requirements at the architectural level, because no event is ever modified or deleted. Temporal queries ("what was the calibration status of instrument X on March 15th?") are answered by replaying events to that timestamp, which is critical for retroactive out-of-tolerance investigations required by IATF 16949.

**Best for:** Organisations where regulatory audit trail integrity is the top priority, where temporal queries are frequent, and where AI-driven analytics on calibration drift patterns and change histories are planned.

**Trade-offs:**
- Pro: Complete, immutable audit trail by design -- every state change is recorded and recoverable
- Pro: Temporal queries are trivial -- replay events to any point in time to reconstruct historical state
- Pro: Natural fit for AI/ML analytics on calibration drift patterns, technician behaviour, and interval optimisation
- Pro: Supports FDA 21 CFR Part 11 and ISO 17025 Clause 7.5 architecturally rather than through application-layer logging
- Pro: New read models can be added without modifying the write path
- Pro: No separate audit_log table required -- the event store IS the audit trail
- Con: Higher implementation complexity -- requires event store, projection engine, and eventual consistency handling
- Con: Read model lag -- materialised views are eventually consistent, not immediately consistent after writes
- Con: Event schema evolution is non-trivial -- upcasting older event versions requires careful versioning strategy
- Con: Storage grows monotonically -- events are never deleted, requiring snapshot strategies for long-lived aggregates
- Con: Ad-hoc cross-entity SQL queries require pre-built projections; harder than in a normalised model
- Con: Developers must learn event sourcing patterns; less familiar than CRUD

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 17025:2017 Clause 7.5 | Technical records are immutable events -- the event store IS the record of all changes, satisfying traceability requirements |
| FDA 21 CFR Part 11.10(e) | Audit trail requirement satisfied by design -- every state change is a signed, timestamped event with cryptographic hash chain |
| ISO 10012:2003/2025 | Measurement management processes modelled as event-driven workflows with discrete lifecycle events |
| JCGM 100:2008 (GUM) | Uncertainty budget captured as structured event payload with Type A/B component decomposition |
| JCGM 200:2012 (VIM) | Event type names and payload field names use VIM-compliant metrology terminology |
| PTB DCC v3.3+ | Certificate-issued events carry DCC-compatible administrative and measurement result payloads for XML/JSON export |
| D-SI Data Model | Measurement quantity representation in event payloads follows D-SI atomic structure (value + unit + uncertainty) |
| ILAC P14/G8 | Decision rule and conformity assessment recorded as explicit fields in calibration-completed events |
| IATF 16949 Clause 7.1.5.1 | Retroactive out-of-tolerance investigation supported by event replay to any historical date |
| Open Cap Format (OCF) | Event-driven lifecycle tracking pattern adapted from OCF's transaction-based securities model |
| iCalendar RFC 5545 | Schedule-related events include recurrence metadata compatible with iCalendar export |

---

## Event Store Core

```sql
-- The single source of truth: an append-only event log.
-- No row in this table is ever updated or deleted.
CREATE TABLE event_store (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    aggregate_type      VARCHAR(100) NOT NULL,
    -- Values: 'Equipment', 'CalibrationRecord', 'CalibrationSchedule',
    --         'OotInvestigation', 'MsaStudy', 'CalibrationProcedure'
    aggregate_id        UUID NOT NULL,
    sequence_number     BIGINT NOT NULL,          -- per-aggregate monotonic ordering
    event_type          VARCHAR(150) NOT NULL,     -- e.g. 'EquipmentRegistered', 'CalibrationCompleted'
    event_schema_version INTEGER NOT NULL DEFAULT 1, -- for payload upcasting across versions
    payload             JSONB NOT NULL,            -- event-specific data (see catalogue below)
    metadata            JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"correlation_id": "...", "causation_id": "...", "ip_address": "..."}

    -- Identity and signing
    performed_by        UUID,                      -- user who caused the event
    signature_meaning   VARCHAR(100),              -- 'created', 'reviewed', 'approved', 'modified'
    signature_method    VARCHAR(50),               -- 'password', 'mfa', 'biometric'

    -- Integrity: cryptographic hash chain for tamper detection
    payload_hash        VARCHAR(128) NOT NULL,     -- SHA-512 of canonical payload JSON
    previous_event_hash VARCHAR(128),              -- hash of prior event in same aggregate (null for first)

    -- Timestamps
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE(aggregate_id, sequence_number)
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions (example for 2026)
-- CREATE TABLE event_store_y2026m01 PARTITION OF event_store
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- CREATE TABLE event_store_y2026m02 PARTITION OF event_store
--     FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Primary query: load all events for an aggregate to rebuild state
CREATE INDEX idx_event_aggregate ON event_store(aggregate_id, sequence_number);

-- Tenant-scoped event stream for cross-aggregate projections
CREATE INDEX idx_event_tenant_time ON event_store(tenant_id, recorded_at);

-- Subscribe to specific event types (used by projection workers)
CREATE INDEX idx_event_type_time ON event_store(event_type, recorded_at);

-- User activity audit
CREATE INDEX idx_event_user ON event_store(performed_by, occurred_at);


-- Snapshots: periodically capture aggregate state to avoid replaying long histories
CREATE TABLE aggregate_snapshot (
    aggregate_id        UUID NOT NULL,
    aggregate_type      VARCHAR(100) NOT NULL,
    snapshot_at_sequence BIGINT NOT NULL,           -- event sequence_number at time of snapshot
    state               JSONB NOT NULL,            -- serialised aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, snapshot_at_sequence)
);


-- Event schema registry for versioning and payload validation
CREATE TABLE event_schema_registry (
    event_type          VARCHAR(150) NOT NULL,
    schema_version      INTEGER NOT NULL,
    json_schema         JSONB NOT NULL,            -- JSON Schema for payload validation
    upcaster_class      VARCHAR(255),              -- code reference for upcasting from prior version
    deprecated_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (event_type, schema_version)
);


-- Projection checkpoint: tracks how far each projector has consumed the event stream
CREATE TABLE projection_checkpoint (
    projection_name     VARCHAR(100) PRIMARY KEY,
    last_event_id       UUID NOT NULL,
    last_sequence       BIGINT NOT NULL,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Type Catalogue

### Equipment Lifecycle Events

```
EquipmentRegistered         -- new equipment added to inventory
EquipmentAttributesUpdated  -- model, serial, category, accuracy class changed
EquipmentRelocated          -- moved to different site or location
EquipmentStatusChanged      -- status transition (in_service -> quarantined, etc.)
EquipmentCheckedOut         -- issued to a user for field use
EquipmentCheckedIn          -- returned by a user
EquipmentCustodianChanged   -- ownership transferred between users
EquipmentRetired            -- permanently removed from service
EquipmentDocumentAttached   -- manual, datasheet, photo, SOP linked
```

### Calibration Lifecycle Events

```
CalibrationStarted                  -- work order opened, technician assigned
CalibrationEnvironmentRecorded      -- temperature, humidity, pressure at time of calibration
CalibrationReferenceStandardLinked  -- traceability reference standard attached
CalibrationMeasurementRecorded      -- single measurement point captured (as-found, as-left, uncertainty)
CalibrationUncertaintyBudgetComputed -- GUM-aligned uncertainty budget calculated
CalibrationCompleted                -- all measurements finished, overall result determined
CalibrationReviewRequested          -- submitted for quality review
CalibrationApproved                 -- reviewer approved the record
CalibrationRejected                 -- reviewer rejected; requires rework
CalibrationVoided                   -- record invalidated (procedure error discovered)
CertificateIssued                   -- ISO 17025 calibration certificate generated
CertificateRevised                  -- certificate updated (new revision)
```

### Schedule Events

```
CalibrationScheduleCreated      -- interval defined for equipment
CalibrationIntervalAdjusted     -- interval changed (manual, NCSL, or AI-predicted)
CalibrationDueNotificationSent  -- due-date notification dispatched
CalibrationScheduleSuspended    -- temporarily paused (equipment in storage)
CalibrationScheduleResumed      -- reactivated
```

### Out-of-Tolerance Investigation Events

```
OutOfToleranceDetected          -- OOT condition identified during calibration
ImpactAssessmentRecorded        -- retroactive assessment of affected products/measurements
OotDispositioned                -- decision made (adjust_recalibrate, repair, retire, use_as_is)
CorrectiveActionCreated         -- CAPA task linked to the OOT event
CorrectiveActionCompleted       -- CAPA task finished
CorrectiveActionVerified        -- effectiveness reviewed and accepted
OotInvestigationClosed          -- full investigation resolved
```

### MSA / Gauge R&R Events

```
MsaStudyCreated                 -- gauge R&R study initiated
MsaMeasurementRecorded          -- individual operator/part/trial measurement
MsaStudyAnalysisCompleted       -- R&R analysis results calculated
MsaStudyApproved                -- results reviewed and accepted
```

---

## Example Event Payloads

### EquipmentRegistered

```json
{
    "asset_tag": "EQ-2026-00451",
    "serial_number": "SN-789012",
    "model_number": "Fluke 87V",
    "manufacturer": "Fluke Corporation",
    "category": "Multimeter",
    "equipment_type": "instrument",
    "site_id": "a1b2c3d4-e5f6-...",
    "location_detail": "Building A, Lab 3, Bench 2",
    "measurement_range": "0-1000V AC/DC",
    "resolution": "0.01V",
    "accuracy_class": "True RMS",
    "barcode": "GS1:01:09501101530003",
    "purchase_date": "2026-01-15",
    "purchase_cost": {"amount": 450.00, "currency": "USD"},
    "custom_fields": {"department": "Metrology Lab", "cost_centre": "CC-400"}
}
```

### CalibrationMeasurementRecorded

Follows D-SI atomic quantity representation (value + unit + uncertainty):

```json
{
    "sequence_number": 3,
    "point_label": "50% FS",
    "nominal": {"value": 50.000, "unit": "PSI"},
    "tolerance": {"lower": -0.25, "upper": 0.25, "unit": "PSI"},
    "as_found": {"value": 50.12, "deviation": 0.12},
    "as_left": {"value": 50.02, "deviation": 0.02},
    "uncertainty": {
        "expanded": 0.08,
        "coverage_factor": 2.0,
        "coverage_probability": 0.9545,
        "unit": "PSI"
    },
    "point_result": "in_tolerance",
    "conformity_statement": "conforming",
    "direction": "up"
}
```

### CalibrationUncertaintyBudgetComputed

GUM-aligned uncertainty budget with Type A and Type B components:

```json
{
    "measurand": "Pressure indication error at 50 PSI",
    "measurand_unit": "PSI",
    "components": [
        {
            "source": "Reference standard uncertainty",
            "evaluation_type": "B",
            "distribution": "normal",
            "input_value": 0.02,
            "divisor": 2.0,
            "standard_uncertainty": 0.01,
            "sensitivity_coefficient": 1.0,
            "degrees_of_freedom": null
        },
        {
            "source": "DUT resolution",
            "evaluation_type": "B",
            "distribution": "rectangular",
            "input_value": 0.05,
            "divisor": 1.732,
            "standard_uncertainty": 0.0289,
            "sensitivity_coefficient": 1.0,
            "degrees_of_freedom": null
        },
        {
            "source": "Repeatability (10 readings)",
            "evaluation_type": "A",
            "distribution": "normal",
            "input_value": 0.03,
            "divisor": 1.0,
            "standard_uncertainty": 0.03,
            "sensitivity_coefficient": 1.0,
            "degrees_of_freedom": 9
        }
    ],
    "combined_standard_uncertainty": 0.0425,
    "effective_degrees_of_freedom": 47.2,
    "coverage_factor": 2.0,
    "expanded_uncertainty": 0.085
}
```

### OutOfToleranceDetected

```json
{
    "calibration_record_id": "cal-uuid-...",
    "measurement_point_sequence": 3,
    "severity": "major",
    "deviation": {"value": 1.2, "unit": "PSI"},
    "nominal_value": 75.0,
    "tolerance_upper": 0.5,
    "as_found_value": 76.2
}
```

### ImpactAssessmentRecorded (IATF 16949 retroactive investigation)

```json
{
    "impact_assessment": "Products measured between last calibration and this finding may be affected",
    "products_affected": "Lot 2026-Q1-batch-44 through 2026-Q1-batch-52",
    "retroactive_action_required": true,
    "retroactive_period_start": "2025-11-20",
    "retroactive_period_end": "2026-05-20"
}
```

---

## Materialised Read Models (Projections)

These tables are populated by event projection workers that consume the event stream. They are disposable -- they can be dropped and rebuilt from the event store at any time.

```sql
-- Projection: current equipment state (denormalised from equipment lifecycle events)
CREATE TABLE read_equipment (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    site_id             UUID,
    asset_tag           VARCHAR(100) NOT NULL,
    serial_number       VARCHAR(100),
    model_number        VARCHAR(100),
    manufacturer        VARCHAR(255),
    equipment_type      VARCHAR(50),
    category            VARCHAR(255),
    description         TEXT,
    status              VARCHAR(50) NOT NULL,
    custodian_id        UUID,
    custodian_name      VARCHAR(255),
    location_detail     VARCHAR(255),
    barcode             VARCHAR(100),
    rfid_tag            VARCHAR(100),
    measurement_range   VARCHAR(100),
    resolution          VARCHAR(50),
    accuracy_class      VARCHAR(50),
    purchase_date       DATE,
    purchase_cost       NUMERIC(12,2),
    currency_code       CHAR(3),
    custom_fields       JSONB NOT NULL DEFAULT '{}',

    -- Denormalised calibration status
    last_calibration_date   DATE,
    last_calibration_result VARCHAR(20),
    next_calibration_due    DATE,
    calibration_interval_days INTEGER,
    calibration_interval_source VARCHAR(50),

    -- Checkout status
    is_checked_out      BOOLEAN NOT NULL DEFAULT false,
    checked_out_by_name VARCHAR(255),
    checked_out_at      TIMESTAMPTZ,

    -- Projection metadata
    last_event_sequence BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_read_equip_tag ON read_equipment(tenant_id, asset_tag);
CREATE INDEX idx_read_equip_status ON read_equipment(tenant_id, status);
CREATE INDEX idx_read_equip_due ON read_equipment(tenant_id, next_calibration_due);
CREATE INDEX idx_read_equip_barcode ON read_equipment(tenant_id, barcode) WHERE barcode IS NOT NULL;


-- Projection: calibration record summary
CREATE TABLE read_calibration_record (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    equipment_id        UUID NOT NULL,
    equipment_asset_tag VARCHAR(100),
    certificate_number  VARCHAR(100) NOT NULL,
    certificate_revision INTEGER NOT NULL DEFAULT 0,
    calibration_type    VARCHAR(50),
    status              VARCHAR(50) NOT NULL,
    result              VARCHAR(20),
    performed_by_name   VARCHAR(255),
    reviewed_by_name    VARCHAR(255),
    approved_by_name    VARCHAR(255),
    date_calibrated     TIMESTAMPTZ,
    date_approved       TIMESTAMPTZ,
    procedure_code      VARCHAR(50),
    procedure_version   INTEGER,
    calibration_site    VARCHAR(255),
    decision_rule       VARCHAR(100),
    ambient_temperature_c   NUMERIC(5,1),
    ambient_humidity_pct    NUMERIC(5,1),
    measurement_count   INTEGER NOT NULL DEFAULT 0,
    oot_count           INTEGER NOT NULL DEFAULT 0,
    has_uncertainty_budget BOOLEAN NOT NULL DEFAULT false,
    reference_standards JSONB DEFAULT '[]',

    last_event_sequence BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_read_cal_cert ON read_calibration_record(tenant_id, certificate_number, certificate_revision);
CREATE INDEX idx_read_cal_equipment ON read_calibration_record(equipment_id);
CREATE INDEX idx_read_cal_date ON read_calibration_record(tenant_id, date_calibrated);
CREATE INDEX idx_read_cal_status ON read_calibration_record(tenant_id, status);


-- Projection: denormalised measurement points for reporting and analysis
CREATE TABLE read_measurement_point (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    calibration_record_id UUID NOT NULL,
    equipment_id        UUID NOT NULL,
    date_calibrated     TIMESTAMPTZ NOT NULL,
    sequence_number     INTEGER NOT NULL,
    point_label         VARCHAR(100),
    nominal_value       NUMERIC(20,8),
    unit                VARCHAR(50),
    tolerance_lower     NUMERIC(20,8),
    tolerance_upper     NUMERIC(20,8),
    as_found_value      NUMERIC(20,8),
    as_found_deviation  NUMERIC(20,8),
    as_left_value       NUMERIC(20,8),
    as_left_deviation   NUMERIC(20,8),
    expanded_uncertainty NUMERIC(20,8),
    coverage_factor     NUMERIC(5,2),
    point_result        VARCHAR(20),
    conformity_statement VARCHAR(50),
    direction           VARCHAR(10),
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_read_meas_record ON read_measurement_point(calibration_record_id);
CREATE INDEX idx_read_meas_equip_time ON read_measurement_point(equipment_id, date_calibrated);
CREATE INDEX idx_read_meas_oot ON read_measurement_point(tenant_id, point_result) WHERE point_result = 'out_of_tolerance';


-- Projection: out-of-tolerance investigation tracker
CREATE TABLE read_oot_investigation (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    calibration_id      UUID NOT NULL,
    equipment_id        UUID NOT NULL,
    equipment_asset_tag VARCHAR(100),
    severity            VARCHAR(20),
    disposition         VARCHAR(50),
    deviation_value     NUMERIC(20,8),
    deviation_unit      VARCHAR(50),
    impact_assessed     BOOLEAN NOT NULL DEFAULT false,
    retroactive_action_required BOOLEAN NOT NULL DEFAULT false,
    corrective_actions_total INTEGER NOT NULL DEFAULT 0,
    corrective_actions_open  INTEGER NOT NULL DEFAULT 0,
    reported_at         TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,

    last_event_sequence BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_read_oot_tenant ON read_oot_investigation(tenant_id);
CREATE INDEX idx_read_oot_open ON read_oot_investigation(tenant_id, disposition) WHERE resolved_at IS NULL;


-- Projection: site-level calibration dashboard (aggregated counts)
CREATE TABLE read_site_dashboard (
    tenant_id           UUID NOT NULL,
    site_id             UUID NOT NULL,
    site_name           VARCHAR(255),
    total_equipment     INTEGER NOT NULL DEFAULT 0,
    in_service          INTEGER NOT NULL DEFAULT 0,
    due_within_30_days  INTEGER NOT NULL DEFAULT 0,
    overdue             INTEGER NOT NULL DEFAULT 0,
    in_calibration      INTEGER NOT NULL DEFAULT 0,
    quarantined         INTEGER NOT NULL DEFAULT 0,
    open_oot_events     INTEGER NOT NULL DEFAULT 0,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, site_id)
);


-- Projection: measurement drift history for AI interval optimisation
CREATE TABLE read_drift_history (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    equipment_id        UUID NOT NULL,
    asset_tag           VARCHAR(100),
    calibration_id      UUID NOT NULL,
    date_calibrated     TIMESTAMPTZ NOT NULL,
    point_label         VARCHAR(100),
    nominal_value       NUMERIC(20,8),
    unit                VARCHAR(50),
    as_found_deviation  NUMERIC(20,8),
    expanded_uncertainty NUMERIC(20,8),
    point_result        VARCHAR(20),
    interval_days_at_time INTEGER,                 -- what interval was in effect when this was measured
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_read_drift_equip ON read_drift_history(equipment_id, date_calibrated);
CREATE INDEX idx_read_drift_tenant ON read_drift_history(tenant_id, date_calibrated);
```

---

## Temporal Query Examples

### "What was the status of instrument EQ-2026-00451 on March 15, 2026?"

```sql
-- Replay all equipment events up to the target date to reconstruct state:
SELECT event_type, payload, occurred_at, performed_by
FROM event_store
WHERE aggregate_type = 'Equipment'
  AND aggregate_id = :equipment_aggregate_id
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY sequence_number ASC;

-- Application code replays these events through the Equipment aggregate
-- to reconstruct the exact state as of that date.
-- For performance, start from the nearest snapshot:
SELECT state, snapshot_at_sequence
FROM aggregate_snapshot
WHERE aggregate_id = :equipment_aggregate_id
  AND snapshot_at_sequence <= (
      SELECT MAX(sequence_number) FROM event_store
      WHERE aggregate_id = :equipment_aggregate_id
        AND occurred_at <= '2026-03-15T23:59:59Z'
  )
ORDER BY snapshot_at_sequence DESC
LIMIT 1;
-- Then replay only events after the snapshot sequence.
```

### Find all out-of-tolerance measurement events this quarter (from event store)

```sql
SELECT
    e.aggregate_id AS calibration_record_id,
    e.payload->>'point_label' AS point_label,
    (e.payload->'nominal'->>'value')::NUMERIC AS nominal,
    (e.payload->'as_found'->>'value')::NUMERIC AS as_found,
    (e.payload->'as_found'->>'deviation')::NUMERIC AS deviation,
    (e.payload->'uncertainty'->>'expanded')::NUMERIC AS uncertainty,
    e.performed_by,
    e.occurred_at
FROM event_store e
WHERE e.tenant_id = :tenant_id
  AND e.event_type = 'CalibrationMeasurementRecorded'
  AND e.payload->>'point_result' = 'out_of_tolerance'
  AND e.occurred_at >= date_trunc('quarter', CURRENT_DATE)
ORDER BY e.occurred_at DESC;
```

### Extract drift time series for AI interval prediction

```sql
-- From the read_drift_history projection (fast, pre-computed):
SELECT date_calibrated, point_label, nominal_value, as_found_deviation,
       expanded_uncertainty, interval_days_at_time
FROM read_drift_history
WHERE equipment_id = :equipment_id
  AND point_label = '50% FS'
ORDER BY date_calibrated ASC;
-- Feed directly into drift rate ML model.
```

### Verify cryptographic audit chain integrity

```sql
WITH chain AS (
    SELECT
        event_id,
        sequence_number,
        payload_hash,
        previous_event_hash,
        LAG(payload_hash) OVER (ORDER BY sequence_number) AS expected_previous
    FROM event_store
    WHERE aggregate_id = :aggregate_id
    ORDER BY sequence_number
)
SELECT event_id, sequence_number
FROM chain
WHERE sequence_number > 1
  AND previous_event_hash IS DISTINCT FROM expected_previous;
-- Any rows returned indicate tampering.
```

---

## Architecture Flow

```
                Command Side                              Query Side
  ┌───────────┐    ┌──────────┐    ┌───────────┐    ┌──────────────────┐
  │  REST API │───>│ Command  │───>│ Aggregate │───>│   Event Store    │
  │  (write)  │    │ Handler  │    │ (validate │    │ (append-only)    │
  └───────────┘    └──────────┘    │  + emit)  │    └────────┬─────────┘
                                   └───────────┘             │
                                                     published events
                                                             │
                   ┌─────────────────────────────────────────┤
                   │                 │                │       │
                   ▼                 ▼                ▼       ▼
            ┌────────────┐  ┌─────────────┐  ┌────────┐  ┌────────┐
            │ Equipment  │  │ Calibration │  │  OOT   │  │ Drift  │
            │ Projector  │  │  Projector  │  │Projector│  │Projector│
            └─────┬──────┘  └──────┬──────┘  └───┬────┘  └───┬────┘
                  │                │              │            │
                  ▼                ▼              ▼            ▼
           ┌───────────┐  ┌────────────┐  ┌──────────┐  ┌──────────┐
           │   read_   │  │   read_    │  │  read_   │  │  read_   │
           │ equipment │  │calibration │  │  oot_    │  │  drift_  │
           │           │  │  _record   │  │investig. │  │ history  │
           └───────────┘  └────────────┘  └──────────┘  └──────────┘
                   │                │              │            │
                   └────────────────┴──────────────┴────────────┘
                                       │
                                       ▼
                               ┌───────────┐
                               │  REST API │
                               │  (read)   │
                               └───────────┘
```

---

## Infrastructure Tables

```sql
-- Multi-tenant identity (minimal -- these are not event-sourced)
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store Core | 4 | event_store (partitioned), aggregate_snapshot, event_schema_registry, projection_checkpoint |
| Infrastructure | 2 | tenant, app_user |
| Read Model: Equipment | 1 | read_equipment (disposable, rebuildable from events) |
| Read Model: Calibration | 2 | read_calibration_record, read_measurement_point |
| Read Model: OOT/CAPA | 1 | read_oot_investigation |
| Read Model: Dashboard | 1 | read_site_dashboard |
| Read Model: AI/Analytics | 1 | read_drift_history |
| **Total** | **12** | 6 core/infrastructure + 6 projections; projections are disposable |

---

## Key Design Decisions

1. **Single event_store table as the sole source of truth** -- all domain events for all aggregate types stored in one partitioned table. This simplifies backup, replication, and security while supporting per-aggregate replay via the composite index on (aggregate_id, sequence_number).

2. **Cryptographic hash chain per aggregate** -- each event records the SHA-512 hash of its payload and the hash of the previous event in the same aggregate, creating a tamper-evident chain. This satisfies FDA 21 CFR Part 11 integrity requirements and enables automated verification without relying solely on database access controls.

3. **No separate audit_log table** -- unlike the normalised relational model (Suggestion 1), the event store itself IS the audit trail. Every "who changed what when" query is answered by querying events directly, eliminating the dual-write problem and the risk of audit log divergence.

4. **Event schema registry with upcasting** -- as the system evolves, event payload schemas change. The event_schema_registry table records the JSON Schema for each event type version, and upcaster functions transform old events to new formats during replay.

5. **D-SI-aligned measurement payloads** -- measurement values in event payloads follow the D-SI atomic quantity structure (value + unit + uncertainty), ensuring consistency with the emerging international digital metrology data standard.

6. **Materialised read models are explicitly disposable** -- each read_ table includes last_event_sequence to track projection progress. If a read model becomes corrupt, needs a new column, or a new projection is required, it is dropped and rebuilt from the event store. No data migration required.

7. **Aggregate snapshots for performance** -- long-lived aggregates (equipment with 10+ years of calibration history, producing thousands of events) use periodic snapshots so state reconstruction starts from the nearest snapshot rather than event zero.

8. **Measurement drift history as a purpose-built AI feature store** -- the read_drift_history table is designed specifically to feed ML-driven calibration interval optimisation, extracting the time series of deviation values per equipment per measurement point with the interval in effect at each calibration.

9. **Separate write and read API paths** -- the CQRS architecture allows the write path (command handlers, aggregate validation, event emission) to be independently scaled from the read path (projection queries), which is critical for high-volume calibration labs processing hundreds of certificates daily.

10. **Domain validation in aggregate before event emission** -- business rules are enforced before events are emitted. For example, a CalibrationApproved event can only be emitted if the aggregate state is 'pending_review' and the approver is not the same person who performed the calibration, enforcing ISO 17025 independence requirements in the domain model.
