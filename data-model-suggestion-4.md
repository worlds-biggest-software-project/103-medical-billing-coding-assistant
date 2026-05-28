# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Medical Billing & Coding Assistant · Created: 2026-05-19

## Philosophy

This model combines a conventional relational layer for transactional billing workflows (claims, remittances, code assignments) with a property graph layer for relationship-rich queries that are awkward or slow in pure relational schemas. The graph layer models the dense web of relationships in medical billing: code-to-code relationships (NCCI edits, crosswalks, DRG groupings), provider-to-payer relationships (credentialing, enrollment, denial patterns), payer rule networks (which rules affect which codes for which payers in which regions), and temporal coding pattern chains (what codes tend to co-occur, what denial patterns cascade).

Graph databases are used in healthcare for drug interaction networks (FDA), clinical pathway analysis (Mayo Clinic research), and fraud detection in claims (CMS). The property graph model is particularly powerful for the medical billing domain because many critical questions are inherently graph traversals: "What is the shortest path from this denied code to a payable alternative?" "Which providers have the highest denial rates with which payers for which code families?" "Which NCCI edit chains create bundling cascades across a multi-procedure encounter?" "What payer rules are connected to this LCD, and which claims would be affected if the LCD changes?"

The implementation uses PostgreSQL with the Apache AGE extension (or a dedicated graph database like Neo4j alongside PostgreSQL) for the graph layer, keeping the transactional billing workflow in standard relational tables.

**Best for:** Organisations that need sophisticated code relationship analysis, denial pattern mining, payer rule network visualization, and AI-driven "similar encounter" retrieval for coding assistance.

**Trade-offs:**
- (+) Powerful multi-hop relationship queries that would require complex recursive CTEs in pure relational
- (+) Natural model for code-to-code relationships (crosswalks, edits, bundling chains)
- (+) Denial pattern mining across payer/code/provider dimensions becomes a graph traversal
- (+) AI can use graph embeddings for "similar encounter" retrieval and coding suggestion
- (+) Visual graph exploration tools for compliance officers investigating coding patterns
- (-) Additional infrastructure complexity (PostgreSQL + AGE extension or separate Neo4j instance)
- (-) Team must learn Cypher or openCypher query language alongside SQL
- (-) Graph databases have different consistency and transaction models than relational
- (-) Write throughput for graph updates can be lower than pure relational inserts
- (-) Less mature tooling ecosystem for ORMs, migrations, and monitoring compared to pure relational

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ICD-10-CM / ICD-10-PCS | Graph nodes for codes with edges for chapter/block hierarchy and cross-references |
| CPT / HCPCS | Graph nodes with edges to NCCI edit pairs, modifier requirements, and RVU values |
| NCCI Edits | Graph edges between code pairs representing edit relationships |
| SNOMED CT | SNOMED concepts as graph nodes with IS-A hierarchy edges; crosswalk edges to ICD-10 |
| MS-DRG / APC | Graph nodes for DRG/APC groups with edges to constituent ICD/CPT codes |
| LCD/NCD | Graph nodes for coverage determinations with edges to applicable codes and payers |
| X12 837/835 | Relational tables for transactional claim and remittance data |
| FHIR R4 | FHIR resource IDs as node properties for EHR provenance |
| HIPAA | Relational audit trail with graph-based access pattern analysis |

---

## Relational Layer (Transactional Billing)

### Core Operational Tables

```sql
-- Standard relational tables for the transactional billing workflow
-- These handle the CRUD operations for day-to-day billing

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    npi             VARCHAR(10) UNIQUE,
    tax_id          VARCHAR(11),
    org_type        VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    npi             VARCHAR(10) UNIQUE NOT NULL,
    credential      VARCHAR(20),
    specialty_code  VARCHAR(10),
    taxonomy_code   VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    role            VARCHAR(50) NOT NULL,
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE patients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    mrn             VARCHAR(50) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    date_of_birth   DATE NOT NULL,
    sex             CHAR(1),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, mrn)
);

CREATE TABLE payers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    payer_id_code   VARCHAR(20) NOT NULL,
    payer_type      VARCHAR(30) NOT NULL,
    clearinghouse   VARCHAR(50),
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE encounters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    provider_id     UUID NOT NULL REFERENCES providers(id),
    encounter_type  VARCHAR(30) NOT NULL,
    admit_date      TIMESTAMPTZ NOT NULL,
    discharge_date  TIMESTAMPTZ,
    place_of_service VARCHAR(2) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    coding_status   VARCHAR(30) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounters_patient ON encounters(patient_id);
CREATE INDEX idx_encounters_provider ON encounters(provider_id);
CREATE INDEX idx_encounters_status ON encounters(coding_status);
CREATE INDEX idx_encounters_admit ON encounters(admit_date);

CREATE TABLE clinical_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    document_type   VARCHAR(50) NOT NULL,
    author_id       UUID NOT NULL REFERENCES providers(id),
    document_date   TIMESTAMPTZ NOT NULL,
    content_text    TEXT NOT NULL,
    is_final        BOOLEAN NOT NULL DEFAULT false,
    source_system   VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_docs_encounter ON clinical_documents(encounter_id);

CREATE TABLE code_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    code_system     VARCHAR(20) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    sequence_number SMALLINT NOT NULL,
    assignment_type VARCHAR(20) NOT NULL,
    confidence_score NUMERIC(5,4),
    reasoning_text  TEXT,
    modifiers       VARCHAR(2)[],
    units           SMALLINT DEFAULT 1,
    assigned_by     UUID REFERENCES users(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    review_action   VARCHAR(20),
    is_current      BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_code_assign_encounter ON code_assignments(encounter_id);
CREATE INDEX idx_code_assign_code ON code_assignments(code_system, code);

CREATE TABLE claims (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    payer_id        UUID NOT NULL REFERENCES payers(id),
    claim_type      VARCHAR(5) NOT NULL,
    claim_number    VARCHAR(50) NOT NULL UNIQUE,
    total_charge    NUMERIC(12,2) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    submitted_at    TIMESTAMPTZ,
    adjudicated_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claims_encounter ON claims(encounter_id);
CREATE INDEX idx_claims_status ON claims(status);
CREATE INDEX idx_claims_payer ON claims(payer_id);

CREATE TABLE claim_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claims(id),
    line_number     SMALLINT NOT NULL,
    procedure_code  VARCHAR(10) NOT NULL,
    modifiers       VARCHAR(2)[],
    diagnosis_pointers SMALLINT[],
    units           NUMERIC(7,2) NOT NULL DEFAULT 1,
    charge_amount   NUMERIC(10,2) NOT NULL,
    service_date    DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_lines_claim ON claim_lines(claim_id);

CREATE TABLE denials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claims(id),
    payer_id        UUID NOT NULL REFERENCES payers(id),
    denial_date     DATE NOT NULL,
    carc_code       VARCHAR(10) NOT NULL,
    rarc_code       VARCHAR(10),
    denial_category VARCHAR(50),
    denied_amount   NUMERIC(10,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    root_cause_ai   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_denials_claim ON denials(claim_id);
CREATE INDEX idx_denials_payer ON denials(payer_id);
CREATE INDEX idx_denials_carc ON denials(carc_code);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    details         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Graph Layer (Relationship Analysis)

The graph layer uses a property graph model. Below shows the schema using PostgreSQL with Apache AGE extension. The same model could be implemented in Neo4j with Cypher.

### Graph Node Types

```sql
-- Using Apache AGE extension for PostgreSQL
-- Load the extension
-- CREATE EXTENSION IF NOT EXISTS age;
-- SELECT * FROM ag_catalog.create_graph('medical_billing');

-- Node types (vertices) in the graph:

-- 1. MedicalCode — represents any code (ICD-10-CM, ICD-10-PCS, CPT, HCPCS, SNOMED, LOINC)
-- Properties: code_system, code, description, effective_date, termination_date, is_active,
--             chapter, block, rvu_total (CPT), hcc_category (ICD), raf_value (ICD)

-- 2. CodeGroup — represents a grouping (DRG, APC, CCS category, ICD chapter, CPT section)
-- Properties: group_type, group_code, description, weight, gmlos, fiscal_year

-- 3. Payer — mirrors the relational payer table
-- Properties: payer_id (UUID FK), name, payer_type, payer_id_code

-- 4. Provider — mirrors the relational provider table
-- Properties: provider_id (UUID FK), npi, name, specialty_code

-- 5. PayerRule — represents a payer-specific rule
-- Properties: rule_id, rule_type, description, confidence, is_active, source, effective_date

-- 6. CoverageDetermination — represents an LCD or NCD
-- Properties: determination_type, determination_id, title, effective_date, contractor_name

-- 7. DenialPattern — represents a detected denial pattern
-- Properties: pattern_id, carc_code, denial_category, occurrence_count, first_seen, last_seen,
--             avg_denied_amount, ai_description
```

### Graph Edge Types

```sql
-- Edge types (relationships) in the graph:

-- Code-to-Code Relationships
-- (MedicalCode)-[:NCCI_PTP_EDIT {modifier_indicator, effective_date, termination_date}]->(MedicalCode)
-- (MedicalCode)-[:CROSSWALK {map_type, source}]->(MedicalCode)
-- (MedicalCode)-[:IS_CHILD_OF]->(MedicalCode)              -- ICD hierarchy
-- (MedicalCode)-[:FREQUENTLY_CO_OCCURS {frequency, confidence}]->(MedicalCode)
-- (MedicalCode)-[:ALTERNATIVE_CODE {context, confidence}]->(MedicalCode)

-- Code-to-Group Relationships
-- (MedicalCode)-[:MEMBER_OF]->(CodeGroup)                   -- ICD code -> DRG, CPT -> APC
-- (MedicalCode)-[:CLASSIFIED_AS]->(CodeGroup)               -- ICD -> CCS category

-- Payer Relationships
-- (Payer)-[:HAS_RULE]->(PayerRule)
-- (PayerRule)-[:APPLIES_TO]->(MedicalCode)
-- (PayerRule)-[:DERIVED_FROM]->(CoverageDetermination)
-- (Payer)-[:COVERS]->(MedicalCode)                          -- Code is covered by payer
-- (Payer)-[:DENIES_FREQUENTLY {count, rate}]->(MedicalCode) -- Denial pattern edge

-- Provider Relationships
-- (Provider)-[:CREDENTIALED_WITH]->(Payer)
-- (Provider)-[:DENIAL_PATTERN {carc_code, count, rate}]->(Payer)
-- (Provider)-[:CODES_FREQUENTLY {count, specialty}]->(MedicalCode)

-- Denial Pattern Relationships
-- (DenialPattern)-[:INVOLVES_CODE]->(MedicalCode)
-- (DenialPattern)-[:INVOLVES_PAYER]->(Payer)
-- (DenialPattern)-[:INVOLVES_PROVIDER]->(Provider)
-- (DenialPattern)-[:SIMILAR_TO {similarity_score}]->(DenialPattern)

-- Coverage Determination Relationships
-- (CoverageDetermination)-[:COVERS_CODE]->(MedicalCode)
-- (CoverageDetermination)-[:ISSUED_BY]->(Payer)
```

### Graph Creation Examples (Apache AGE / Cypher)

```sql
-- Create medical code nodes from reference data
SELECT * FROM cypher('medical_billing', $$
    CREATE (c:MedicalCode {
        code_system: 'ICD10CM',
        code: 'E11.65',
        description: 'Type 2 diabetes mellitus with hyperglycemia',
        effective_date: '2025-10-01',
        is_active: true,
        chapter: 'E00-E89',
        block: 'E08-E13',
        hcc_category: '19',
        raf_value: 0.104
    })
    RETURN c
$$) as (result agtype);

-- Create NCCI PTP edit edge between two CPT codes
SELECT * FROM cypher('medical_billing', $$
    MATCH (c1:MedicalCode {code_system: 'CPT', code: '99285'})
    MATCH (c2:MedicalCode {code_system: 'CPT', code: '93010'})
    CREATE (c1)-[:NCCI_PTP_EDIT {
        modifier_indicator: '1',
        effective_date: '2026-01-01',
        rationale: 'EKG interpretation component of high-level ED visit'
    }]->(c2)
$$) as (result agtype);

-- Create payer denial pattern edge
SELECT * FROM cypher('medical_billing', $$
    MATCH (p:Payer {payer_id_code: 'BCBSMA'})
    MATCH (c:MedicalCode {code_system: 'CPT', code: '36415'})
    CREATE (p)-[:DENIES_FREQUENTLY {
        count: 47,
        rate: 0.34,
        primary_carc: '97',
        first_seen: '2025-11-01',
        last_seen: '2026-05-15',
        recommendation: 'Bundle with comprehensive panel codes'
    }]->(c)
$$) as (result agtype);
```

---

## Example Graph Queries

### Find all NCCI edit chains for a set of encounter codes

```sql
-- Given codes assigned to an encounter, find all NCCI PTP edit violations
SELECT * FROM cypher('medical_billing', $$
    MATCH (c1:MedicalCode)-[e:NCCI_PTP_EDIT]->(c2:MedicalCode)
    WHERE c1.code IN ['99285', '93010', '36415', '71046']
      AND c2.code IN ['99285', '93010', '36415', '71046']
      AND e.effective_date <= '2026-05-19'
      AND (e.termination_date IS NULL OR e.termination_date > '2026-05-19')
    RETURN c1.code AS comprehensive_code,
           c2.code AS component_code,
           e.modifier_indicator,
           e.rationale
$$) as (comprehensive_code agtype, component_code agtype, modifier_indicator agtype, rationale agtype);
```

### Find alternative codes when a denial occurs

```sql
-- When code X is denied by payer Y, find alternative codes that payer Y does pay
SELECT * FROM cypher('medical_billing', $$
    MATCH (denied:MedicalCode {code: '99285', code_system: 'CPT'})
    MATCH (denied)-[:ALTERNATIVE_CODE]->(alt:MedicalCode)
    MATCH (payer:Payer {payer_id_code: 'BCBSMA'})
    WHERE NOT EXISTS((payer)-[:DENIES_FREQUENTLY]->(alt))
    RETURN alt.code, alt.description, alt.rvu_total
    ORDER BY alt.rvu_total DESC
$$) as (code agtype, description agtype, rvu_total agtype);
```

### Discover provider-payer denial hotspots

```sql
-- Find providers with high denial rates for specific code families with specific payers
SELECT * FROM cypher('medical_billing', $$
    MATCH (prov:Provider)-[dp:DENIAL_PATTERN]->(payer:Payer)
    WHERE dp.rate > 0.15
    MATCH (prov)-[:CODES_FREQUENTLY]->(c:MedicalCode)
    MATCH (payer)-[:DENIES_FREQUENTLY]->(c)
    RETURN prov.name, prov.npi, payer.name, c.code, c.description,
           dp.carc_code, dp.rate AS provider_denial_rate
    ORDER BY dp.rate DESC
    LIMIT 20
$$) as (provider agtype, npi agtype, payer agtype, code agtype,
        description agtype, carc agtype, rate agtype);
```

### Trace LCD coverage chains

```sql
-- Find all codes affected by a specific LCD and the payer rules derived from it
SELECT * FROM cypher('medical_billing', $$
    MATCH (lcd:CoverageDetermination {determination_id: 'L33830'})
    MATCH (lcd)-[:COVERS_CODE]->(c:MedicalCode)
    OPTIONAL MATCH (rule:PayerRule)-[:DERIVED_FROM]->(lcd)
    OPTIONAL MATCH (payer:Payer)-[:HAS_RULE]->(rule)
    RETURN lcd.title,
           c.code, c.code_system, c.description,
           rule.rule_type, rule.description AS rule_desc,
           payer.name AS payer_name
    ORDER BY c.code
$$) as (lcd_title agtype, code agtype, code_system agtype, code_desc agtype,
        rule_type agtype, rule_desc agtype, payer_name agtype);
```

### Find similar encounters for AI coding assistance

```sql
-- Use graph-based encounter similarity: encounters that share the same code patterns
-- First, represent encounters as connected to their assigned codes in the graph
-- Then find encounters similar to the current one

SELECT * FROM cypher('medical_billing', $$
    MATCH (current_enc:Encounter {encounter_id: 'enc-uuid-here'})
    MATCH (current_enc)-[:CODED_WITH]->(c:MedicalCode)
    WITH current_enc, collect(c) AS current_codes
    MATCH (other_enc:Encounter)-[:CODED_WITH]->(oc:MedicalCode)
    WHERE other_enc <> current_enc
      AND oc IN current_codes
    WITH other_enc, count(oc) AS shared_codes, current_codes
    WHERE shared_codes >= 2
    RETURN other_enc.encounter_id, shared_codes,
           toFloat(shared_codes) / size(current_codes) AS similarity
    ORDER BY similarity DESC
    LIMIT 10
$$) as (encounter_id agtype, shared_codes agtype, similarity agtype);
```

---

## Graph Synchronisation

```sql
-- Trigger to sync relational data to graph on insert/update
-- This runs after new code assignments, denials, or payer rules are created

CREATE OR REPLACE FUNCTION sync_code_assignment_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    -- Add CODED_WITH edge between encounter and code in graph
    PERFORM ag_catalog.cypher('medical_billing',
        format(
            'MERGE (e:Encounter {encounter_id: %L})
             MERGE (c:MedicalCode {code_system: %L, code: %L})
             MERGE (e)-[:CODED_WITH {
                 assignment_id: %L,
                 confidence: %s,
                 assigned_at: %L
             }]->(c)',
            NEW.encounter_id, NEW.code_system, NEW.code,
            NEW.id, COALESCE(NEW.confidence_score, 0), NEW.assigned_at
        )
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_code_to_graph
    AFTER INSERT ON code_assignments
    FOR EACH ROW
    WHEN (NEW.is_current = true)
    EXECUTE FUNCTION sync_code_assignment_to_graph();

-- Similar triggers for denials, payer rules, etc.
CREATE OR REPLACE FUNCTION sync_denial_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    -- Create/update denial pattern edge between payer and denied codes
    -- Look up the claim's codes and create DENIES_FREQUENTLY edges
    PERFORM ag_catalog.cypher('medical_billing',
        format(
            'MERGE (p:Payer {payer_id: %L})
             MERGE (dp:DenialPattern {
                 denial_id: %L,
                 carc_code: %L,
                 denial_category: %L
             })
             MERGE (p)-[:HAS_DENIAL]->(dp)',
            NEW.payer_id, NEW.id, NEW.carc_code, COALESCE(NEW.denial_category, 'unknown')
        )
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_denial_to_graph
    AFTER INSERT ON denials
    FOR EACH ROW
    EXECUTE FUNCTION sync_denial_to_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Relational — Core Entities | 6 | Organisations, providers, users, patients, payers, encounters |
| Relational — Documents | 1 | Clinical documents |
| Relational — Coding | 1 | Code assignments |
| Relational — Claims | 2 | Claims, claim lines |
| Relational — Denials | 1 | Denials |
| Relational — Audit | 1 | Audit log |
| Graph — Node Types | 7 | MedicalCode, CodeGroup, Payer, Provider, PayerRule, CoverageDetermination, DenialPattern |
| Graph — Edge Types | 16 | NCCI edits, crosswalks, hierarchies, payer rules, denial patterns, etc. |
| **Total Relational** | **12** | |
| **Total Graph Elements** | **23** | 7 node types + 16 edge types |

---

## Key Design Decisions

1. **Relational for transactions, graph for relationships:** The billing workflow (create encounter, assign codes, submit claim, process remittance) is a linear pipeline well-suited to relational CRUD. The analytical questions (code relationships, denial patterns, payer rule networks) are relationship traversals well-suited to a graph. The hybrid avoids forcing either paradigm where it does not fit.

2. **Medical codes as first-class graph nodes:** Every ICD-10-CM, CPT, HCPCS, and SNOMED code is a node in the graph with edges to related codes. This makes the entire medical coding taxonomy navigable as a graph, enabling queries like "find all codes within 2 hops of E11.65" for coding assistance.

3. **NCCI edits as graph edges:** NCCI PTP edits are naturally a graph relationship (code A bundles code B). Representing them as edges enables multi-hop bundling chain detection that would require recursive CTEs in a relational-only model.

4. **Denial patterns as graph structures:** Denial patterns connecting payers, codes, and providers as a subgraph enable discovery queries: "Which payer-code-provider triangles have the highest denial rates?" This is a natural graph pattern match, not a practical SQL query.

5. **Trigger-based graph synchronisation:** Relational inserts trigger graph updates to keep the graph consistent. The graph is eventually consistent with the relational source, which is acceptable for analytical queries that don't need real-time transactional guarantees.

6. **Apache AGE for single-database deployment:** Using the Apache AGE PostgreSQL extension allows graph queries alongside relational queries in a single database, avoiding the operational complexity of running a separate Neo4j instance. For larger deployments, Neo4j can be swapped in with the same Cypher queries.

7. **Graph embeddings for AI coding:** The graph structure enables generation of graph embeddings (node2vec, GraphSAGE) that capture the structural relationships between codes, payers, and providers. These embeddings can enhance the AI coding model's ability to suggest related codes and predict denial likelihood.

8. **Lightweight relational layer:** The relational layer has only 12 tables (fewer than models 1 and 3) because relationship-heavy data (code crosswalks, NCCI edits, payer rule networks) lives in the graph rather than in relational junction tables.
