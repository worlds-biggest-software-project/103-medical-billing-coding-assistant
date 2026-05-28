# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Medical Billing & Coding Assistant · Created: 2026-05-19

## Philosophy

This model treats every action in the billing and coding workflow as an immutable event. Instead of mutating rows in place, the system appends events to an append-only event store: "AI suggested ICD-10 code E11.65 for encounter X with confidence 0.94," "Coder confirmed code E11.65," "Claim C submitted to Medicare," "Denial received: CARC 97." The current state of any entity is derived by replaying its event stream. Materialised read models (projections) are maintained for efficient querying but are always rebuildable from the event store.

Event sourcing is used in financial systems (banking ledgers, payment processors), audit-critical platforms (regulatory compliance systems), and healthcare systems that need full temporal traceability. The pattern aligns naturally with HIPAA's requirement for comprehensive audit trails and the medical billing domain's need to answer temporal questions: "What codes were assigned to this encounter before the CDI query was answered?" "What was the claim state when the denial was issued?" "What did the AI suggest before the coder modified it?"

The architecture uses CQRS (Command Query Responsibility Segregation): write operations produce events, and separate read models are optimised for specific query patterns (denial dashboards, coder productivity, claim status). This decoupling means the write path can handle high-volume autonomous coding without blocking analytical queries.

**Best for:** Organisations that need complete, tamper-evident audit trails, temporal querying ("what was true at time T?"), and the ability to replay history for AI model retraining or compliance investigations.

**Trade-offs:**
- (+) Complete, immutable audit history by design — no separate audit_log table needed
- (+) Temporal queries ("show me the state of encounter X as of date Y") are native
- (+) AI model retraining can replay event streams to understand historical coding patterns
- (+) Denial learning loops can correlate coding events with denial events for pattern detection
- (+) Write path is append-only — no lock contention on hot tables
- (-) Higher storage requirements (every state change is a new row)
- (-) Eventual consistency between event store and read models requires careful handling
- (-) More complex application code (event handlers, projectors, snapshot management)
- (-) Simple queries like "get current claim status" require either a projection or event replay
- (-) Schema evolution of event types requires versioning and upcasting logic

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ICD-10-CM / ICD-10-PCS | Code values embedded in coding events; reference data in read models |
| CPT / HCPCS | Procedure codes in coding and claim events |
| NCCI Edits | Scrubbing events reference NCCI edit violations; edit rules in reference read model |
| X12 837P/I | Claim submission events carry serialised 837 segment data |
| X12 835 | Remittance events carry parsed 835 adjudication data |
| MS-DRG / APC | Grouper assignment events with full input/output captured |
| HIPAA Security Rule | Event store is the audit trail; immutability satisfies 6-year retention |
| FHIR R4 | FHIR resource IDs cross-referenced in events for EHR provenance |
| OCSF (Open Cybersecurity Schema Framework) | Event taxonomy inspired by OCSF structured event patterns |

---

## Event Store (Core)

```sql
-- The single source of truth: all domain events
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                    -- Aggregate root ID (encounter, claim, patient)
    stream_type     VARCHAR(50) NOT NULL,             -- 'encounter', 'claim', 'denial', 'patient', 'cdi_query'
    event_type      VARCHAR(100) NOT NULL,            -- e.g., 'CodeSuggestedByAI', 'CodeConfirmedByCoder', 'ClaimSubmitted'
    event_version   INTEGER NOT NULL,                 -- Monotonically increasing per stream
    organisation_id UUID NOT NULL,
    caused_by       UUID,                             -- User or system ID that caused this event
    correlation_id  UUID,                             -- Links related events across aggregates
    causation_id    UUID,                             -- The event that caused this event
    event_data      JSONB NOT NULL,                   -- Event-specific payload
    metadata        JSONB,                            -- Cross-cutting concerns (IP, user agent, AI model version)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, event_version)                  -- Optimistic concurrency control
);

-- Partitioned by created_at for retention management
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_org ON events(organisation_id);
CREATE INDEX idx_events_created ON events(created_at);
CREATE INDEX idx_events_correlation ON events(correlation_id);
CREATE INDEX idx_events_data ON events USING gin(event_data);
```

### Event Type Taxonomy

```
-- Encounter Lifecycle
EncounterCreated              -- Patient admitted/registered
ClinicalDocumentReceived      -- Note received from EHR
ClinicalDocumentUpdated       -- Note amended/addended

-- AI Coding Events
CodeSuggestedByAI             -- AI proposes a code with confidence and reasoning
CodeBatchSuggestedByAI        -- AI proposes complete code set for an encounter
AICodingModelVersionChanged   -- Tracks which AI model version produced suggestions

-- Coder Review Events
CodeConfirmedByCoder          -- Coder accepts AI suggestion
CodeModifiedByCoder           -- Coder changes AI suggestion to different code
CodeRejectedByCoder           -- Coder rejects AI suggestion entirely
CodeManuallyAssigned          -- Coder assigns code without AI suggestion
EMCalculationPerformed        -- E/M complexity calculated
DRGAssigned                   -- DRG grouper run, result captured

-- CDI Events
CDIQueryGenerated             -- AI or CDI specialist generates documentation query
CDIQuerySentToPhysician       -- Query dispatched to physician
CDIQueryResponseReceived      -- Physician responds to query
CDIQueryResolved              -- Documentation updated, coding impact assessed

-- Claim Lifecycle Events
ClaimDrafted                  -- Claim assembled from coded encounter
ClaimScrubbed                 -- Claim run through scrubbing engine
ScrubEditDetected             -- Specific scrub edit found (NCCI, LCD, payer rule)
ScrubEditResolved             -- Edit resolved by user or auto-fix
ClaimSubmitted                -- Claim sent to clearinghouse
ClaimAccepted                 -- Clearinghouse accepted claim
ClaimRejected                 -- Clearinghouse rejected claim (pre-adjudication)

-- Remittance Events
RemittanceReceived            -- 835 ERA parsed and ingested
ClaimPaid                     -- Claim fully paid
ClaimPartiallyPaid            -- Claim paid with adjustments
ClaimDenied                   -- Claim fully denied

-- Denial Management Events
DenialIdentified              -- Denial extracted from remittance
DenialCategorised             -- AI assigns denial category and root cause
DenialAssigned                -- Denial assigned to staff member
AppealInitiated               -- Appeal process started
AppealLetterGenerated         -- AI generates appeal letter
AppealSubmitted               -- Appeal sent to payer
AppealOutcomeReceived         -- Payer responds to appeal
DenialWrittenOff              -- Decision to write off denial

-- Payer Rule Events
PayerRuleLearned              -- AI detects new payer-specific rule from denial pattern
PayerRuleActivated            -- Learned rule put into production scrubbing
PayerRuleDeactivated          -- Rule retired
```

### Example Event Payloads

```sql
-- CodeSuggestedByAI event_data example:
-- {
--   "encounter_id": "550e8400-e29b-41d4-a716-446655440000",
--   "code_system": "ICD10CM",
--   "code": "E11.65",
--   "description": "Type 2 diabetes mellitus with hyperglycemia",
--   "sequence_number": 1,
--   "confidence_score": 0.9412,
--   "reasoning": "Patient presents with fasting glucose 287 mg/dL, documented type 2 DM on problem list, physician notes 'uncontrolled hyperglycemia'...",
--   "source_document_id": "doc-uuid-here",
--   "source_text_excerpt": "Assessment: Type 2 diabetes, uncontrolled. Fasting glucose 287.",
--   "ai_model_version": "medbill-llm-v3.2.1"
-- }

-- ClaimDenied event_data example:
-- {
--   "claim_id": "claim-uuid-here",
--   "payer_id": "payer-uuid-here",
--   "payer_claim_number": "2026MCAR0012345",
--   "carc_code": "97",
--   "rarc_code": "N30",
--   "denied_amount": 1245.00,
--   "denied_lines": [1, 3],
--   "group_code": "CO",
--   "remittance_line_id": "rem-uuid-here"
-- }

-- PayerRuleLearned event_data example:
-- {
--   "payer_id": "payer-uuid-here",
--   "rule_type": "bundling",
--   "description": "Payer denies CPT 36415 when billed with CPT 80053 on same date of service",
--   "detected_from_denials": ["denial-uuid-1", "denial-uuid-2", "denial-uuid-3"],
--   "confidence": 0.87,
--   "applicable_codes": ["36415", "80053"],
--   "recommendation": "Bundle 36415 into 80053 for this payer"
-- }
```

---

## Snapshots (Performance Optimisation)

```sql
-- Periodic snapshots to avoid replaying long event streams
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,                 -- Event version this snapshot reflects
    state           JSONB NOT NULL,                    -- Serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

---

## Read Models (Materialised Projections)

These tables are derived from the event store and can be rebuilt at any time by replaying events.

### Encounter Read Model

```sql
CREATE TABLE rm_encounters (
    encounter_id    UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    patient_id      UUID NOT NULL,
    provider_id     UUID NOT NULL,
    encounter_type  VARCHAR(30) NOT NULL,
    admit_date      TIMESTAMPTZ NOT NULL,
    discharge_date  TIMESTAMPTZ,
    place_of_service VARCHAR(2),
    status          VARCHAR(30) NOT NULL,             -- Derived from latest event
    coding_status   VARCHAR(30) NOT NULL,             -- 'pending', 'ai_suggested', 'coder_review', 'coded', 'billed'
    total_codes     SMALLINT DEFAULT 0,
    ai_automation_rate NUMERIC(5,4),                  -- % of codes auto-accepted
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_encounters_org ON rm_encounters(organisation_id);
CREATE INDEX idx_rm_encounters_status ON rm_encounters(coding_status);
CREATE INDEX idx_rm_encounters_admit ON rm_encounters(admit_date);
```

### Current Codes Read Model

```sql
CREATE TABLE rm_current_codes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL,
    code_system     VARCHAR(20) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    description     TEXT,
    sequence_number SMALLINT NOT NULL,
    status          VARCHAR(20) NOT NULL,             -- 'ai_suggested', 'confirmed', 'modified', 'rejected'
    confidence_score NUMERIC(5,4),
    reasoning       TEXT,
    modifiers       VARCHAR(2)[],
    units           SMALLINT DEFAULT 1,
    assigned_by_type VARCHAR(10),                     -- 'ai', 'coder'
    assigned_by_id  UUID,
    last_event_id   UUID NOT NULL,                    -- Event that produced this state
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_codes_encounter ON rm_current_codes(encounter_id);
CREATE INDEX idx_rm_codes_code ON rm_current_codes(code_system, code);
```

### Claims Dashboard Read Model

```sql
CREATE TABLE rm_claims (
    claim_id        UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    encounter_id    UUID NOT NULL,
    patient_id      UUID NOT NULL,
    payer_id        UUID,
    claim_type      VARCHAR(5),
    claim_number    VARCHAR(50),
    total_charge    NUMERIC(12,2),
    total_paid      NUMERIC(12,2),
    total_adjusted  NUMERIC(12,2),
    total_denied    NUMERIC(12,2),
    status          VARCHAR(30) NOT NULL,
    submitted_at    TIMESTAMPTZ,
    adjudicated_at  TIMESTAMPTZ,
    days_to_payment INTEGER,                          -- Derived: adjudicated_at - submitted_at
    denial_count    SMALLINT DEFAULT 0,
    appeal_count    SMALLINT DEFAULT 0,
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_claims_org ON rm_claims(organisation_id);
CREATE INDEX idx_rm_claims_status ON rm_claims(status);
CREATE INDEX idx_rm_claims_payer ON rm_claims(payer_id);
CREATE INDEX idx_rm_claims_submitted ON rm_claims(submitted_at);
```

### Denial Analytics Read Model

```sql
CREATE TABLE rm_denials (
    denial_id       UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    claim_id        UUID NOT NULL,
    payer_id        UUID NOT NULL,
    provider_id     UUID,
    denial_date     DATE NOT NULL,
    carc_code       VARCHAR(10) NOT NULL,
    rarc_code       VARCHAR(10),
    denial_category VARCHAR(50),
    denied_amount   NUMERIC(10,2),
    denied_codes    JSONB,                            -- Array of {code_system, code, line_number}
    root_cause_ai   TEXT,
    status          VARCHAR(20) NOT NULL,
    appeal_count    SMALLINT DEFAULT 0,
    recovered_amount NUMERIC(10,2) DEFAULT 0,
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_denials_org ON rm_denials(organisation_id);
CREATE INDEX idx_rm_denials_payer ON rm_denials(payer_id);
CREATE INDEX idx_rm_denials_carc ON rm_denials(carc_code);
CREATE INDEX idx_rm_denials_category ON rm_denials(denial_category);
CREATE INDEX idx_rm_denials_date ON rm_denials(denial_date);
```

### Coder Productivity Read Model

```sql
CREATE TABLE rm_coder_productivity (
    user_id         UUID NOT NULL,
    period_date     DATE NOT NULL,
    encounters_coded SMALLINT DEFAULT 0,
    codes_assigned  SMALLINT DEFAULT 0,
    ai_confirmed    SMALLINT DEFAULT 0,
    ai_modified     SMALLINT DEFAULT 0,
    ai_rejected     SMALLINT DEFAULT 0,
    manual_assigned SMALLINT DEFAULT 0,
    avg_review_seconds INTEGER,
    PRIMARY KEY (user_id, period_date)
);
```

### Payer Rules Read Model

```sql
CREATE TABLE rm_payer_rules (
    rule_id         UUID PRIMARY KEY,
    payer_id        UUID NOT NULL,
    rule_type       VARCHAR(30) NOT NULL,
    description     TEXT NOT NULL,
    applicable_codes VARCHAR(10)[],
    confidence      NUMERIC(5,4),
    denial_count    INTEGER DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    learned_at      TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_payer_rules_payer ON rm_payer_rules(payer_id);
CREATE INDEX idx_rm_payer_rules_codes ON rm_payer_rules USING gin(applicable_codes);
```

---

## Reference Data (Non-Event-Sourced)

```sql
-- These are static reference tables, not event-sourced
CREATE TABLE ref_organisations (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    npi             VARCHAR(10) UNIQUE,
    org_type        VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_providers (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL REFERENCES ref_organisations(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    npi             VARCHAR(10) UNIQUE NOT NULL,
    specialty_code  VARCHAR(10),
    taxonomy_code   VARCHAR(20),
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_patients (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL REFERENCES ref_organisations(id),
    mrn             VARCHAR(50) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    date_of_birth   DATE NOT NULL,
    sex             CHAR(1),
    UNIQUE(organisation_id, mrn)
);

CREATE TABLE ref_payers (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    payer_id_code   VARCHAR(20) NOT NULL,
    payer_type      VARCHAR(30) NOT NULL,
    clearinghouse   VARCHAR(50),
    is_active       BOOLEAN DEFAULT true
);

CREATE TABLE ref_code_sets (
    code_system     VARCHAR(20) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    description     TEXT NOT NULL,
    effective_date  DATE NOT NULL,
    termination_date DATE,
    is_active       BOOLEAN DEFAULT true,
    PRIMARY KEY (code_system, code, effective_date)
);

CREATE TABLE ref_ncci_edits (
    column1_code    VARCHAR(10) NOT NULL,
    column2_code    VARCHAR(10) NOT NULL,
    modifier_indicator CHAR(1),
    effective_date  DATE NOT NULL,
    termination_date DATE,
    mue_value       SMALLINT,
    PRIMARY KEY (column1_code, column2_code, effective_date)
);

CREATE TABLE ref_users (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    role            VARCHAR(50) NOT NULL,
    is_active       BOOLEAN DEFAULT true
);
```

---

## Example Queries

### Replay encounter coding history

```sql
-- Get the full coding timeline for an encounter
SELECT
    event_type,
    event_data->>'code_system' AS code_system,
    event_data->>'code' AS code,
    event_data->>'confidence_score' AS confidence,
    caused_by,
    created_at
FROM events
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND stream_type = 'encounter'
  AND event_type IN ('CodeSuggestedByAI', 'CodeConfirmedByCoder', 'CodeModifiedByCoder', 'CodeRejectedByCoder')
ORDER BY event_version;
```

### Point-in-time state reconstruction

```sql
-- What codes were assigned to this encounter as of a specific date?
SELECT
    event_data->>'code_system' AS code_system,
    event_data->>'code' AS code,
    event_data->>'sequence_number' AS seq,
    event_type AS last_action,
    created_at
FROM events
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND event_type IN ('CodeSuggestedByAI', 'CodeConfirmedByCoder', 'CodeModifiedByCoder')
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version;
```

### Denial pattern analysis across events

```sql
-- Find payers where the same CARC code appears 5+ times in the last 90 days
SELECT
    event_data->>'payer_id' AS payer_id,
    event_data->>'carc_code' AS carc_code,
    COUNT(*) AS denial_count,
    SUM((event_data->>'denied_amount')::numeric) AS total_denied
FROM events
WHERE event_type = 'ClaimDenied'
  AND created_at > now() - INTERVAL '90 days'
GROUP BY event_data->>'payer_id', event_data->>'carc_code'
HAVING COUNT(*) >= 5
ORDER BY total_denied DESC;
```

### AI model accuracy tracking over time

```sql
-- Track AI suggestion acceptance rate by model version and month
WITH suggestions AS (
    SELECT
        event_data->>'ai_model_version' AS model_version,
        stream_id,
        event_data->>'code' AS suggested_code,
        created_at
    FROM events
    WHERE event_type = 'CodeSuggestedByAI'
),
reviews AS (
    SELECT
        stream_id,
        event_type,
        event_data->>'code' AS reviewed_code,
        created_at
    FROM events
    WHERE event_type IN ('CodeConfirmedByCoder', 'CodeModifiedByCoder', 'CodeRejectedByCoder')
)
SELECT
    s.model_version,
    date_trunc('month', r.created_at) AS month,
    COUNT(*) AS total_reviews,
    COUNT(*) FILTER (WHERE r.event_type = 'CodeConfirmedByCoder') AS confirmed,
    ROUND(COUNT(*) FILTER (WHERE r.event_type = 'CodeConfirmedByCoder')::numeric / COUNT(*)::numeric, 4) AS acceptance_rate
FROM suggestions s
JOIN reviews r ON s.stream_id = r.stream_id
GROUP BY s.model_version, date_trunc('month', r.created_at)
ORDER BY month, model_version;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single events table (partitioned by date) |
| Snapshots | 1 | Aggregate state snapshots for replay performance |
| Read Models | 6 | Encounters, codes, claims, denials, productivity, payer rules |
| Reference Data | 7 | Organisations, providers, patients, payers, code sets, NCCI edits, users |
| **Total** | **15** | Plus event type taxonomy (application code, not tables) |

---

## Key Design Decisions

1. **Single event store table with JSONB payloads:** All domain events go into one table with `event_type` discrimination and `event_data` as JSONB. This simplifies the write path and allows new event types to be added without DDL changes. The GIN index on `event_data` supports queries into event payloads.

2. **Stream-based partitioning with optimistic concurrency:** The `UNIQUE(stream_id, event_version)` constraint provides optimistic concurrency control — two concurrent writes to the same aggregate will fail, preventing lost updates. This is critical for coding workflows where multiple coders might review the same encounter.

3. **Correlation and causation IDs:** `correlation_id` links related events across aggregates (e.g., a denial event on a claim is correlated with the coding events on the encounter). `causation_id` tracks direct causal chains (e.g., a `PayerRuleLearned` event was caused by a `ClaimDenied` event). These enable powerful debugging and AI training queries.

4. **Read models are disposable and rebuildable:** Every `rm_*` table can be dropped and rebuilt from the event store. This means read model schema changes are low-risk — add a column, replay, deploy. No data migration needed.

5. **Reference data is not event-sourced:** Entities like organisations, providers, patients, and code sets are slow-changing reference data that don't benefit from event sourcing. They're stored in conventional relational tables with standard CRUD operations.

6. **Snapshots for long-lived streams:** Encounters with many coding iterations could accumulate hundreds of events. Periodic snapshots store the materialised aggregate state at a point in time, so replay only needs to process events after the last snapshot.

7. **Event store as the HIPAA audit trail:** The event store inherently satisfies HIPAA audit requirements — every action is immutable, timestamped, and attributed to a user or system. No separate audit log table is needed; the event store IS the audit log.

8. **AI model version tracking in events:** Every AI coding event includes the model version that produced it. This enables precise accuracy tracking across model upgrades and rollback analysis if a new model version degrades performance.
