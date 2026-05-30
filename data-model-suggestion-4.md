# Data Model Suggestion 4: Graph-Relational (Traceability-First)

> Project: Tool & Calibration Management · Created: 2026-05-22

## Philosophy

This model combines a conventional relational database for operational CRUD with a graph layer specifically designed to model the traceability chains, reference standard hierarchies, and equipment relationship networks that are central to calibration management. The core insight is that metrological traceability — the unbroken chain of comparisons linking every measurement result back to national standards and ultimately to SI units — is fundamentally a graph problem.

In a calibration laboratory, a working instrument is calibrated against a reference standard, which was itself calibrated against a higher-tier reference, which traces back to a national metrology institute (NMI) like NIST, PTB, or NPL. When a reference standard goes out of tolerance, every instrument that was calibrated using that standard (and every product measured by those instruments) is potentially affected. This is a graph traversal problem: "find all downstream instruments and products affected by this out-of-tolerance reference standard." In a normalized relational model, this requires recursive CTEs that are difficult to write, maintain, and optimise. In a graph model, it is a simple traversal query.

The graph layer is implemented using PostgreSQL-native tables (graph_node and graph_edge) rather than a separate graph database like Neo4j. This keeps the operational simplicity of a single database while enabling graph traversal queries via recursive CTEs and the PostgreSQL ltree extension for hierarchy paths. For deployments that need deeper graph analytics, the same edge data can be exported to a dedicated graph engine.

**Best for:** Organisations with complex reference standard hierarchies, multi-tier traceability requirements, large instrument fleets where out-of-tolerance impact analysis must be fast and comprehensive, and environments where conflict-of-interest or equipment sharing relationships need to be modelled.

**Trade-offs:**
- Pro: Traceability chain queries are natural and performant — "find all instruments affected by this OOT standard" is a simple graph traversal
- Pro: Hierarchy navigation (equipment categories, organisational structure, SI unit traceability) is first-class
- Pro: Reference standard networks are explicitly modelled rather than implied through foreign keys
- Pro: Enables sophisticated analyses: shared calibration dependencies, single-points-of-failure in the traceability chain, calibration coverage gaps
- Pro: PostgreSQL-native implementation — no additional database infrastructure needed
- Con: Graph query patterns (recursive CTEs, ltree) are less familiar to many developers
- Con: Edge table maintenance adds write overhead compared to simple foreign keys
- Con: Dual representation (relational + graph) requires synchronisation discipline
- Con: Graph traversal performance degrades on very deep or very wide graphs without careful index tuning
- Con: More complex schema to understand and maintain than either pure relational or pure JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 17025:2017 Clause 6.5 | Metrological traceability requirements directly modelled as graph edges from DUT to reference standard to NMI |
| EURAMET / NIST Traceability | SI traceability chain represented as a directed acyclic graph (DAG) with typed edges |
| ISO 10012:2003/2025 | Measurement management system relationships between equipment, procedures, and standards modelled as graph |
| JCGM 100:2008 (GUM) | Uncertainty contribution hierarchy modelled as weighted edges in the traceability graph |
| JCGM 200:2012 (VIM) | VIM-compliant terminology used for graph node and edge type naming |
| ILAC P14 | Uncertainty propagation through traceability chain calculated via graph traversal |
| IATF 16949 | Out-of-tolerance impact analysis uses reverse graph traversal to identify affected instruments and products |
| PTB DCC XML Schema | Certificate-to-certificate traceability links export to DCC administrative data |
| FDA 21 CFR Part 11 | Graph edges carry audit metadata (who created the link, when, digital signature) |
| GS1 EPCIS 2.0 | Asset-to-product traceability edges support GS1 event-based tracking |
| ISO 3166-1/2 | Site and jurisdiction nodes carry ISO 3166 codes |

---

## Graph Layer Tables

```sql
-- Extension for hierarchical path queries
CREATE EXTENSION IF NOT EXISTS ltree;

-- Graph nodes: any entity that participates in traceability or organisational relationships
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- Node types:
    -- 'equipment'          — instrument, gauge, tool
    -- 'reference_standard' — reference standard (also equipment, but distinguished by role)
    -- 'nmi'                — national metrology institute (NIST, PTB, NPL, etc.)
    -- 'certificate'        — calibration certificate (issued document)
    -- 'calibration_record' — calibration event
    -- 'site'               — physical location
    -- 'department'         — organisational unit
    -- 'product_lot'        — manufactured product lot (for OOT impact tracing)
    -- 'procedure'          — calibration procedure
    -- 'user'               — technician or reviewer
    entity_id       UUID NOT NULL,              -- FK to the corresponding relational table
    label           VARCHAR(500) NOT NULL,       -- human-readable label (e.g. "Fluke 87V SN:789012")
    properties      JSONB NOT NULL DEFAULT '{}', -- cached properties for graph queries without JOINs
    hierarchy_path  LTREE,                       -- materialised path for hierarchy queries
    -- Example: 'nist.pressure.primary_std_001.working_std_042.dut_0451'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_node_tenant ON graph_node(tenant_id);
CREATE INDEX idx_graph_node_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_graph_node_entity ON graph_node(entity_id);
CREATE INDEX idx_graph_node_hierarchy ON graph_node USING GIST (hierarchy_path);
CREATE INDEX idx_graph_node_props ON graph_node USING GIN (properties);

-- Graph edges: typed, directed relationships between nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id),
    target_node_id  UUID NOT NULL REFERENCES graph_node(id),
    edge_type       VARCHAR(100) NOT NULL,
    -- Edge types:
    -- 'calibrated_by'       — DUT → Reference Standard (traceability)
    -- 'traceable_to'        — Reference Standard → higher-tier standard or NMI
    -- 'calibrated_using'    — Calibration Record → Reference Standard
    -- 'produced_certificate'— Calibration Record → Certificate
    -- 'performed_by'        — Calibration Record → User (technician)
    -- 'reviewed_by'         — Calibration Record → User (reviewer)
    -- 'located_at'          — Equipment → Site
    -- 'belongs_to'          — Equipment → Department
    -- 'measured_product'    — Equipment → Product Lot
    -- 'supersedes'          — Certificate → previous Certificate
    -- 'uses_procedure'      — Calibration Record → Procedure
    -- 'parent_category'     — Equipment Category hierarchy
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for 'calibrated_by': {"certificate_number": "CAL-2026-0042",
    --   "calibration_date": "2026-03-15", "uncertainty": {"value": 0.005, "unit": "PSI", "k": 2}}
    -- Example for 'measured_product': {"measurement_date": "2026-04-01",
    --   "product_lot": "LOT-2026-Q1-0042", "measurement_type": "final_inspection"}
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                -- null = currently active
    created_by      UUID,                        -- user who created this relationship
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edge_source ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_graph_edge_target ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_graph_edge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_graph_edge_active ON graph_edge(source_node_id) WHERE valid_to IS NULL;
CREATE INDEX idx_graph_edge_temporal ON graph_edge(valid_from, valid_to);
CREATE INDEX idx_graph_edge_props ON graph_edge USING GIN (properties);

-- Unique constraint: prevent duplicate active edges of the same type between same nodes
CREATE UNIQUE INDEX idx_graph_edge_unique_active
    ON graph_edge(source_node_id, target_node_id, edge_type)
    WHERE valid_to IS NULL;
```

---

## Operational Relational Tables

The graph layer coexists with conventional relational tables for CRUD operations. The relational tables are the primary write target; graph nodes and edges are maintained in sync (via triggers or application logic).

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    address         JSONB NOT NULL DEFAULT '{}',
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
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE equipment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    asset_tag       VARCHAR(100) NOT NULL,
    serial_number   VARCHAR(100),
    model_number    VARCHAR(100),
    manufacturer    VARCHAR(255),
    description     TEXT,
    equipment_type  VARCHAR(50) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'in_service',
    specifications  JSONB NOT NULL DEFAULT '{}',
    identifiers     JSONB NOT NULL DEFAULT '{}',
    custodian_id    UUID REFERENCES app_user(id),
    location_detail VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_equipment_asset_tag ON equipment(tenant_id, asset_tag);
CREATE INDEX idx_equipment_site ON equipment(site_id);
CREATE INDEX idx_equipment_status ON equipment(tenant_id, status);

-- National Metrology Institutes (reference data)
CREATE TABLE national_metrology_institute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) NOT NULL UNIQUE,  -- 'NIST', 'PTB', 'NPL', 'NRC', etc.
    name            VARCHAR(255) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    website         VARCHAR(500),
    accreditation_scope JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed NMIs:
-- INSERT INTO national_metrology_institute (code, name, country_code) VALUES
-- ('NIST', 'National Institute of Standards and Technology', 'US'),
-- ('PTB', 'Physikalisch-Technische Bundesanstalt', 'DE'),
-- ('NPL', 'National Physical Laboratory', 'GB'),
-- ('NRC', 'National Research Council Canada', 'CA'),
-- ('NMIJ', 'National Metrology Institute of Japan', 'JP');

CREATE TABLE calibration_procedure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    procedure_code  VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    method_reference VARCHAR(255),
    measurement_template JSONB NOT NULL DEFAULT '[]',
    environmental_requirements JSONB NOT NULL DEFAULT '{}',
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
    interval_days   INTEGER NOT NULL,
    interval_source VARCHAR(50) NOT NULL DEFAULT 'fixed',
    next_due_date   DATE NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    prediction_metadata JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cal_schedule_due ON calibration_schedule(next_due_date) WHERE is_active = true;

CREATE TABLE calibration_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    schedule_id     UUID REFERENCES calibration_schedule(id),
    procedure_id    UUID REFERENCES calibration_procedure(id),
    certificate_number VARCHAR(100) NOT NULL,
    calibration_type VARCHAR(50) NOT NULL DEFAULT 'scheduled',
    status          VARCHAR(50) NOT NULL DEFAULT 'in_progress',
    result          VARCHAR(20),
    performed_by    UUID NOT NULL REFERENCES app_user(id),
    reviewed_by     UUID REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    date_calibrated TIMESTAMPTZ NOT NULL,
    date_approved   TIMESTAMPTZ,
    environment     JSONB NOT NULL DEFAULT '{}',
    measurements    JSONB NOT NULL DEFAULT '[]',
    uncertainty_budget JSONB,
    decision_rule   VARCHAR(100),
    traceability_statement TEXT,
    notes           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_cal_record_cert ON calibration_record(tenant_id, certificate_number);
CREATE INDEX idx_cal_record_equipment ON calibration_record(equipment_id);
CREATE INDEX idx_cal_record_date ON calibration_record(tenant_id, date_calibrated);

CREATE TABLE oot_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    calibration_record_id UUID NOT NULL REFERENCES calibration_record(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    severity        VARCHAR(20) NOT NULL,
    disposition     VARCHAR(50) NOT NULL DEFAULT 'pending',
    impact          JSONB NOT NULL DEFAULT '{}',
    corrective_actions JSONB NOT NULL DEFAULT '[]',
    reported_by     UUID REFERENCES app_user(id),
    reported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    resolution_notes TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oot_event_equipment ON oot_event(equipment_id);

CREATE TABLE msa_study (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    study_type      VARCHAR(50) NOT NULL,
    study_method    VARCHAR(50) NOT NULL,
    study_config    JSONB NOT NULL DEFAULT '{}',
    raw_measurements JSONB NOT NULL DEFAULT '[]',
    results         JSONB NOT NULL DEFAULT '{}',
    performed_by    UUID REFERENCES app_user(id),
    study_date      DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE equipment_checkout (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    equipment_id    UUID NOT NULL REFERENCES equipment(id),
    checked_out_by  UUID NOT NULL REFERENCES app_user(id),
    checked_out_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expected_return DATE,
    checked_in_at   TIMESTAMPTZ,
    checked_in_by   UUID REFERENCES app_user(id),
    return_notes    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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

## Graph Synchronisation Triggers

```sql
-- Automatically create/update graph nodes when equipment is inserted or updated
CREATE OR REPLACE FUNCTION sync_equipment_graph_node()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.tenant_id,
        CASE WHEN NEW.equipment_type = 'reference_standard' THEN 'reference_standard' ELSE 'equipment' END,
        NEW.id,
        COALESCE(NEW.manufacturer, '') || ' ' || COALESCE(NEW.model_number, '') || ' ' || NEW.asset_tag,
        jsonb_build_object(
            'asset_tag', NEW.asset_tag,
            'serial_number', NEW.serial_number,
            'category', NEW.category,
            'status', NEW.status,
            'site_id', NEW.site_id
        )
    )
    ON CONFLICT (entity_id) WHERE node_type IN ('equipment', 'reference_standard')
    DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        is_active = (NEW.status != 'retired'),
        updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_equipment_graph_sync
    AFTER INSERT OR UPDATE ON equipment
    FOR EACH ROW EXECUTE FUNCTION sync_equipment_graph_node();

-- Automatically create graph edges when a calibration record links to reference standards
CREATE OR REPLACE FUNCTION sync_calibration_graph_edges()
RETURNS TRIGGER AS $$
DECLARE
    ref_std JSONB;
    source_node_id UUID;
    target_node_id UUID;
BEGIN
    -- Get the graph node for the equipment being calibrated
    SELECT id INTO source_node_id FROM graph_node WHERE entity_id = NEW.equipment_id LIMIT 1;

    -- Get the graph node for the calibration record
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.tenant_id, 'calibration_record', NEW.id,
        'Cal ' || NEW.certificate_number,
        jsonb_build_object('certificate_number', NEW.certificate_number, 'date', NEW.date_calibrated,
                           'result', NEW.result, 'status', NEW.status)
    )
    ON CONFLICT DO NOTHING;

    -- Create edges for each reference standard used
    IF NEW.status = 'approved' THEN
        FOR ref_std IN SELECT * FROM jsonb_array_elements(COALESCE(NEW.measurements, '[]'::jsonb))
        LOOP
            -- Additional edge creation logic for reference standards
            -- would go here based on the reference_standards JSONB in the calibration record
            NULL;
        END LOOP;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_calibration_graph_sync
    AFTER INSERT OR UPDATE ON calibration_record
    FOR EACH ROW EXECUTE FUNCTION sync_calibration_graph_edges();
```

---

## Graph Traversal Queries

### Find all instruments affected by an out-of-tolerance reference standard

This is the critical query for IATF 16949 retroactive investigations: when a reference standard is found out of tolerance, which working instruments were calibrated using it, and what products did those instruments measure?

```sql
-- Downstream impact analysis: find all instruments calibrated (directly or indirectly)
-- using a reference standard that has gone out of tolerance
WITH RECURSIVE downstream AS (
    -- Start from the out-of-tolerance reference standard
    SELECT
        ge.target_node_id AS node_id,
        ge.edge_type,
        gn.label,
        gn.node_type,
        gn.entity_id,
        1 AS depth,
        ARRAY[ge.source_node_id] AS path
    FROM graph_edge ge
    JOIN graph_node gn ON gn.id = ge.target_node_id
    WHERE ge.source_node_id = :oot_reference_standard_node_id
      AND ge.edge_type = 'calibrated_by'
      AND ge.valid_to IS NULL
      -- Only consider calibrations within the OOT retroactive period
      AND (ge.properties->>'calibration_date')::date >= :retroactive_start_date

    UNION ALL

    -- Recursively find instruments calibrated by those reference standards
    SELECT
        ge.target_node_id,
        ge.edge_type,
        gn.label,
        gn.node_type,
        gn.entity_id,
        d.depth + 1,
        d.path || ge.source_node_id
    FROM downstream d
    JOIN graph_edge ge ON ge.source_node_id = d.node_id
    JOIN graph_node gn ON gn.id = ge.target_node_id
    WHERE ge.edge_type = 'calibrated_by'
      AND ge.valid_to IS NULL
      AND ge.target_node_id != ALL(d.path)  -- prevent cycles
      AND d.depth < 10                       -- limit traversal depth
)
SELECT
    node_type,
    label,
    entity_id,
    depth
FROM downstream
ORDER BY depth, label;
```

### Find the full traceability chain from a working instrument to SI units

```sql
-- Upstream traceability: trace from a DUT back to the national standard
WITH RECURSIVE traceability AS (
    SELECT
        gn.id AS node_id,
        gn.label,
        gn.node_type,
        gn.entity_id,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.entity_id = :equipment_id

    UNION ALL

    SELECT
        target_node.id,
        target_node.label,
        target_node.node_type,
        target_node.entity_id,
        t.depth + 1,
        t.path || target_node.id
    FROM traceability t
    JOIN graph_edge ge ON ge.source_node_id = t.node_id
    JOIN graph_node target_node ON target_node.id = ge.target_node_id
    WHERE ge.edge_type IN ('calibrated_by', 'traceable_to')
      AND ge.valid_to IS NULL
      AND target_node.id != ALL(t.path)
      AND t.depth < 15
)
SELECT
    depth,
    node_type,
    label,
    entity_id
FROM traceability
ORDER BY depth;

-- Example output:
-- 0 | equipment         | Fluke 87V SN:789012 (EQ-2026-00451)
-- 1 | reference_standard| Fluke 5520A Multifunction Calibrator STD-001
-- 2 | reference_standard| Fluke 732B DC Reference Standard STD-REF-001
-- 3 | nmi               | NIST - National Institute of Standards and Technology
```

### Using ltree for hierarchy queries

```sql
-- Find all equipment in the "Pressure" measurement hierarchy
SELECT id, label, entity_id, hierarchy_path
FROM graph_node
WHERE tenant_id = :tenant_id
  AND hierarchy_path <@ 'nist.pressure'  -- all descendants of the NIST pressure branch
  AND is_active = true;

-- Find common ancestor (shared reference standard) for two instruments
SELECT
    a.hierarchy_path,
    b.hierarchy_path,
    lca(a.hierarchy_path, b.hierarchy_path) AS common_ancestor
FROM graph_node a, graph_node b
WHERE a.entity_id = :equipment_id_1
  AND b.entity_id = :equipment_id_2;
```

### Identify single-points-of-failure in the traceability chain

```sql
-- Find reference standards that are the sole traceability source for many instruments
SELECT
    gn.label AS reference_standard,
    gn.entity_id,
    COUNT(DISTINCT ge.source_node_id) AS dependent_instrument_count
FROM graph_edge ge
JOIN graph_node gn ON gn.id = ge.target_node_id
WHERE ge.tenant_id = :tenant_id
  AND ge.edge_type = 'calibrated_by'
  AND ge.valid_to IS NULL
  AND gn.node_type = 'reference_standard'
GROUP BY gn.id, gn.label, gn.entity_id
HAVING COUNT(DISTINCT ge.source_node_id) > 10
ORDER BY dependent_instrument_count DESC;
```

### Find all calibrations performed by a specific technician on a date range

```sql
SELECT
    target_node.label AS calibration,
    target_node.properties->>'certificate_number' AS certificate,
    target_node.properties->>'result' AS result,
    ge.properties->>'role' AS role,
    ge.valid_from AS date
FROM graph_edge ge
JOIN graph_node source_node ON source_node.id = ge.source_node_id
JOIN graph_node target_node ON target_node.id = ge.target_node_id
WHERE source_node.entity_id = :user_id
  AND source_node.node_type = 'user'
  AND ge.edge_type IN ('performed_by', 'reviewed_by')
  AND ge.valid_from BETWEEN :start_date AND :end_date
ORDER BY ge.valid_from DESC;
```

---

## Graph Visualisation Data Export

The graph structure can be exported for visual network analysis tools:

```sql
-- Export nodes and edges for D3.js, Cytoscape, or Gephi visualisation
-- Nodes
SELECT
    id,
    node_type AS "group",
    label,
    properties
FROM graph_node
WHERE tenant_id = :tenant_id
  AND is_active = true;

-- Edges
SELECT
    source_node_id AS source,
    target_node_id AS target,
    edge_type AS label,
    properties
FROM graph_edge
WHERE tenant_id = :tenant_id
  AND valid_to IS NULL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge (the core graph structure) |
| Core Infrastructure | 3 | tenant, site, app_user |
| Reference Data | 1 | national_metrology_institute |
| Equipment Management | 1 | equipment (with JSONB specs) |
| Calibration | 4 | calibration_procedure, calibration_schedule, calibration_record, equipment_checkout |
| Quality Events | 2 | oot_event, msa_study |
| Audit & Compliance | 1 | audit_log |
| **Total** | **14** | Plus 2 graph tables that model all relationships |

---

## Key Design Decisions

1. **Graph layer in PostgreSQL, not a separate database** — using graph_node and graph_edge tables with recursive CTEs and ltree keeps the architecture simple (single database) while enabling graph traversal. If Neo4j-class performance is needed later, the edge data exports cleanly to a property graph engine.

2. **Temporal edges with valid_from/valid_to** — graph edges have time validity. When a reference standard is recertified, the old edge is closed (valid_to set) and a new edge created. This enables point-in-time traceability queries: "What was the traceability chain on March 15th?"

3. **Cached properties on graph nodes** — the properties JSONB on graph_node duplicates key attributes from the relational tables (asset_tag, status, category). This enables graph queries to return useful information without JOINing back to the relational tables, significantly improving traversal query performance.

4. **Unique active edge constraint** — the partial unique index on (source, target, edge_type) WHERE valid_to IS NULL prevents duplicate active relationships, enforcing graph integrity without application-level checks.

5. **ltree hierarchy paths** — the hierarchy_path field on graph_node uses PostgreSQL's ltree extension to enable fast subtree queries (finding all equipment under a specific branch of the traceability tree) without recursive CTEs. This is particularly useful for "find all instruments traceable to NIST pressure standards."

6. **Trigger-based graph synchronisation** — PostgreSQL triggers on the relational tables automatically maintain graph nodes and edges, ensuring the graph layer stays in sync without requiring application-level dual-write logic. This reduces the risk of graph/relational divergence.

7. **National Metrology Institute as a first-class entity** — NMIs (NIST, PTB, NPL, etc.) are modelled as both a reference table and graph nodes, making them the root of traceability trees. This enables queries like "show all our instruments traceable to PTB" which are common in audit preparation.

8. **Impact analysis as a graph traversal** — the critical regulatory query "what instruments and products are affected by this out-of-tolerance event" is a natural downstream traversal in the graph model, rather than a complex series of JOINs across multiple relational tables with nested subqueries.

9. **Separation of graph structure from operational CRUD** — the relational tables handle day-to-day operations (creating calibration records, updating equipment status), while the graph layer handles relationship-heavy queries (traceability, impact analysis, dependency mapping). This dual-model approach uses each storage pattern for what it does best.

10. **Graph as the foundation for AI features** — the graph structure provides natural training data for graph neural networks that can predict equipment drift patterns based on shared reference standards, identify calibration scheduling clusters, and detect anomalous traceability patterns that may indicate systematic measurement problems.
