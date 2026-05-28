# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Medical Billing & Coding Assistant · Created: 2026-05-19

## Philosophy

This model uses a hybrid approach: core structural fields (patient demographics, encounter dates, claim amounts, code assignments) are stored in typed relational columns with indexes and foreign keys, while variable, payer-specific, specialty-specific, or rapidly-evolving fields are stored in JSONB columns. The relational skeleton ensures data integrity and query performance for the 80% of queries that follow standard patterns, while JSONB provides schema flexibility for the 20% that vary by payer, specialty, jurisdiction, or product evolution stage.

This pattern is widely used in modern healthcare SaaS platforms. Athenahealth's claims processing stores payer-specific adjudication rules as structured JSON. Epic's FHIR API returns resources with `extension` arrays for non-standard fields — a JSON-in-relational pattern. The approach recognises that medical billing has both universal structure (every claim has a patient, a payer, and procedure codes) and enormous variability (each payer has different LCD/NCD rules, modifier requirements, and bundling policies that change quarterly).

For a startup or MVP, this design offers a critical advantage: new payer rules, specialty-specific fields, or AI model outputs can be added to JSONB columns without database migrations, enabling rapid iteration during the product's formative period while maintaining relational integrity for the core billing workflow.

**Best for:** Teams building an MVP or early-stage product that needs to iterate rapidly on payer-specific rules, specialty configurations, and AI model outputs without constant schema migrations.

**Trade-offs:**
- (+) Rapid iteration — new fields added to JSONB without DDL changes or migrations
- (+) Payer-specific and specialty-specific variations handled naturally in JSONB
- (+) Core billing workflow has full relational integrity (foreign keys, constraints, typed columns)
- (+) JSONB indexing (GIN) supports efficient containment queries for payer rules
- (+) Fewer tables than fully normalised model — simpler to understand and deploy
- (-) JSONB fields lack database-level type enforcement — application must validate
- (-) Schema documentation for JSONB fields requires discipline (easy to accumulate undocumented fields)
- (-) Complex JSONB queries can be slower than equivalent joins on normalised tables
- (-) Reporting tools and BI connectors may struggle with JSONB structures
- (-) Risk of "JSONB creep" — too much data moving into JSONB defeats the purpose of relational columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ICD-10-CM / ICD-10-PCS | Relational `code` and `code_system` columns; JSONB `guidelines` field for code-specific notes |
| CPT / HCPCS | Relational code columns; JSONB for modifier rules and RVU data |
| NCCI Edits | Relational edit pair table with JSONB `rationale` and `exceptions` |
| X12 837P/I | Claims table with relational core fields; JSONB `edi_segments` for raw segment data |
| X12 835 | Remittance with relational amounts; JSONB `adjustments` for variable CARC/RARC detail |
| MS-DRG / APC | Relational DRG code and weight; JSONB for grouper input/output details |
| LCD/NCD | JSONB `rule_definition` allows arbitrary rule structures per payer |
| FHIR R4 | JSONB `fhir_extensions` mirrors FHIR's extension mechanism for non-standard fields |
| HIPAA | Audit trail in relational + JSONB for flexible event detail |

---

## Core Tables

### Organisation, Provider, and User Management

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    npi             VARCHAR(10) UNIQUE,
    tax_id          VARCHAR(11),
    org_type        VARCHAR(50) NOT NULL,
    contact_info    JSONB NOT NULL DEFAULT '{}',
    -- contact_info example:
    -- {
    --   "address": {"line1": "100 Main St", "city": "Boston", "state": "MA", "zip": "02101"},
    --   "phone": "617-555-0100",
    --   "fax": "617-555-0101",
    --   "billing_contact": {"name": "Jane Smith", "email": "jsmith@example.com"}
    -- }
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_clearinghouse": "stedi",
    --   "auto_submit_clean_claims": true,
    --   "ai_coding_enabled": true,
    --   "ai_auto_confirm_threshold": 0.95,
    --   "specialties": ["emergency_medicine", "radiology", "urgent_care"],
    --   "edi_config": {"sender_id": "ACME001", "receiver_qualifier": "ZZ"}
    -- }
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
    provider_details JSONB NOT NULL DEFAULT '{}',
    -- provider_details example:
    -- {
    --   "dea_number": "AB1234563",
    --   "state_licences": [{"state": "MA", "number": "12345", "expiry": "2027-06-30"}],
    --   "enrolled_payers": ["medicare", "bcbs_ma", "aetna"],
    --   "coding_specialties": ["emergency_medicine"],
    --   "avg_encounters_per_day": 24
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_providers_org ON providers(organisation_id);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    role            VARCHAR(50) NOT NULL,
    provider_id     UUID REFERENCES providers(id),
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example:
    -- ["code_encounters", "submit_claims", "view_denials", "manage_payer_rules"]
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "default_code_system_view": "ICD10CM",
    --   "worklist_sort": "admit_date_desc",
    --   "confidence_display_threshold": 0.70
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(organisation_id);
```

### Patients and Insurance

```sql
CREATE TABLE patients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    mrn             VARCHAR(50) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    date_of_birth   DATE NOT NULL,
    sex             CHAR(1),
    demographics    JSONB NOT NULL DEFAULT '{}',
    -- demographics example:
    -- {
    --   "address": {"line1": "42 Elm St", "city": "Cambridge", "state": "MA", "zip": "02139"},
    --   "phone": "617-555-0200",
    --   "email": "patient@example.com",
    --   "preferred_language": "en",
    --   "race": "2106-3",
    --   "ethnicity": "2186-5"
    -- }
    insurance       JSONB NOT NULL DEFAULT '[]',
    -- insurance example:
    -- [
    --   {
    --     "priority": 1,
    --     "payer_id": "payer-uuid",
    --     "member_id": "ABC123456",
    --     "group_number": "GRP001",
    --     "plan_name": "Blue Cross PPO",
    --     "subscriber_name": "John Doe",
    --     "relationship": "self",
    --     "effective_date": "2025-01-01",
    --     "is_active": true
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, mrn)
);

CREATE INDEX idx_patients_org ON patients(organisation_id);
CREATE INDEX idx_patients_dob ON patients(date_of_birth);
CREATE INDEX idx_patients_insurance ON patients USING gin(insurance);
```

### Encounters and Clinical Documents

```sql
CREATE TABLE encounters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    provider_id     UUID NOT NULL REFERENCES providers(id),
    encounter_type  VARCHAR(30) NOT NULL,
    admit_date      TIMESTAMPTZ NOT NULL,
    discharge_date  TIMESTAMPTZ,
    place_of_service VARCHAR(2) NOT NULL,
    financial_class VARCHAR(50),
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    coding_status   VARCHAR(30) NOT NULL DEFAULT 'pending',  -- 'pending', 'ai_processing', 'ai_suggested', 'coder_review', 'coded', 'billed'
    encounter_details JSONB NOT NULL DEFAULT '{}',
    -- encounter_details example:
    -- {
    --   "admission_type": "1",
    --   "discharge_status": "01",
    --   "attending_provider_npi": "1234567890",
    --   "chief_complaint": "chest pain",
    --   "acuity": "3",
    --   "ehr_encounter_id": "ENC-2026-00123",
    --   "fhir_encounter_id": "Encounter/abc123"
    -- }
    ai_coding_result JSONB,
    -- ai_coding_result example:
    -- {
    --   "model_version": "medbill-llm-v3.2.1",
    --   "processed_at": "2026-05-19T10:30:00Z",
    --   "processing_time_ms": 2340,
    --   "overall_confidence": 0.91,
    --   "auto_confirmed": false,
    --   "documents_processed": 3,
    --   "suggested_codes": [
    --     {"system": "ICD10CM", "code": "I21.01", "desc": "STEMI of LAD", "seq": 1, "confidence": 0.96, "reasoning": "..."},
    --     {"system": "CPT", "code": "99285", "desc": "ED visit high complexity", "seq": 1, "confidence": 0.88, "reasoning": "..."}
    --   ],
    --   "em_calculation": {"mdm_level": "high", "calculated_cpt": "99285"},
    --   "drg_suggestion": {"drg": "280", "description": "AMI discharged alive", "weight": 1.4028}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounters_patient ON encounters(patient_id);
CREATE INDEX idx_encounters_provider ON encounters(provider_id);
CREATE INDEX idx_encounters_status ON encounters(coding_status);
CREATE INDEX idx_encounters_admit ON encounters(admit_date);
CREATE INDEX idx_encounters_org ON encounters(organisation_id);
CREATE INDEX idx_encounters_ai_result ON encounters USING gin(ai_coding_result);

CREATE TABLE clinical_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    document_type   VARCHAR(50) NOT NULL,
    author_id       UUID NOT NULL REFERENCES providers(id),
    document_date   TIMESTAMPTZ NOT NULL,
    content_text    TEXT NOT NULL,
    is_final        BOOLEAN NOT NULL DEFAULT false,
    source_info     JSONB NOT NULL DEFAULT '{}',
    -- source_info example:
    -- {
    --   "source_system": "epic",
    --   "fhir_resource_id": "DocumentReference/doc123",
    --   "content_format": "plain_text",
    --   "note_template": "ED_NOTE_V2",
    --   "word_count": 1250,
    --   "nlp_entities_extracted": 47
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_clinical_docs_encounter ON clinical_documents(encounter_id);
```

---

## Code Assignment

```sql
CREATE TABLE code_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    code_system     VARCHAR(20) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    sequence_number SMALLINT NOT NULL,
    assignment_type VARCHAR(20) NOT NULL,             -- 'ai_suggested', 'coder_confirmed', 'coder_modified', 'coder_rejected', 'manual'
    confidence_score NUMERIC(5,4),
    modifiers       VARCHAR(2)[],
    units           SMALLINT DEFAULT 1,
    is_current      BOOLEAN NOT NULL DEFAULT true,    -- False for superseded assignments
    assigned_by     UUID REFERENCES users(id),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    ai_details      JSONB,
    -- ai_details example:
    -- {
    --   "model_version": "medbill-llm-v3.2.1",
    --   "reasoning": "Patient presents with acute STEMI of LAD artery confirmed by EKG and troponin elevation...",
    --   "source_document_id": "doc-uuid",
    --   "source_text_excerpt": "Assessment: Acute STEMI, LAD territory",
    --   "alternative_codes": [
    --     {"code": "I21.09", "confidence": 0.12, "reason": "Other STEMI sites considered but LAD most specific"},
    --     {"code": "I21.3", "confidence": 0.04, "reason": "STEMI unspecified - less specific, lower confidence"}
    --   ],
    --   "supporting_evidence": [
    --     {"type": "lab", "finding": "Troponin I 8.4 ng/mL (elevated)"},
    --     {"type": "ekg", "finding": "ST elevation in V1-V4"}
    --   ]
    -- }
    review_details  JSONB,
    -- review_details example (populated when coder reviews):
    -- {
    --   "reviewed_by": "user-uuid",
    --   "reviewed_at": "2026-05-19T11:00:00Z",
    --   "action": "confirmed",
    --   "original_code": "I21.01",
    --   "modified_to": null,
    --   "review_notes": "Agree with AI assessment, clear LAD STEMI documentation"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_code_assign_encounter ON code_assignments(encounter_id);
CREATE INDEX idx_code_assign_code ON code_assignments(code_system, code);
CREATE INDEX idx_code_assign_current ON code_assignments(encounter_id) WHERE is_current = true;
CREATE INDEX idx_code_assign_confidence ON code_assignments(confidence_score) WHERE assignment_type = 'ai_suggested';
```

---

## Claims and Billing

```sql
CREATE TABLE payers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    payer_id_code   VARCHAR(20) NOT NULL,
    payer_type      VARCHAR(30) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    payer_config    JSONB NOT NULL DEFAULT '{}',
    -- payer_config example:
    -- {
    --   "clearinghouse": "stedi",
    --   "edi_receiver_id": "BCBSMA001",
    --   "timely_filing_days": 90,
    --   "requires_prior_auth_codes": ["27447", "63030", "22551"],
    --   "modifier_requirements": {
    --     "25": {"required_with": ["99213", "99214", "99215"], "documentation_note": "Must document separately identifiable E/M"}
    --   },
    --   "bundling_overrides": [
    --     {"code1": "36415", "code2": "80053", "action": "bundle", "note": "Venipuncture included in comprehensive panel"}
    --   ],
    --   "contact": {"phone": "800-555-0300", "provider_portal": "https://portal.bcbs.com"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    billing_details JSONB NOT NULL DEFAULT '{}',
    -- billing_details example:
    -- {
    --   "billing_provider_npi": "1234567890",
    --   "rendering_provider_npi": "0987654321",
    --   "referring_provider_npi": null,
    --   "place_of_service": "23",
    --   "frequency_code": "1",
    --   "service_date_from": "2026-05-15",
    --   "service_date_to": "2026-05-15",
    --   "edi_control_number": "000012345",
    --   "clearinghouse_tracking_id": "STD-2026-00456",
    --   "payer_claim_number": "2026BCBS789012"
    -- }
    claim_lines     JSONB NOT NULL DEFAULT '[]',
    -- claim_lines example:
    -- [
    --   {
    --     "line_number": 1,
    --     "procedure_code": "99285",
    --     "code_system": "CPT",
    --     "modifiers": ["25"],
    --     "diagnosis_pointers": [1, 2],
    --     "units": 1,
    --     "charge_amount": 850.00,
    --     "service_date": "2026-05-15",
    --     "revenue_code": "0450",
    --     "code_assignment_id": "assign-uuid"
    --   },
    --   {
    --     "line_number": 2,
    --     "procedure_code": "93010",
    --     "code_system": "CPT",
    --     "modifiers": [],
    --     "diagnosis_pointers": [1],
    --     "units": 1,
    --     "charge_amount": 125.00,
    --     "service_date": "2026-05-15"
    --   }
    -- ]
    claim_diagnoses JSONB NOT NULL DEFAULT '[]',
    -- claim_diagnoses example:
    -- [
    --   {"sequence": 1, "code": "I21.01", "poa": "Y"},
    --   {"sequence": 2, "code": "I25.10", "poa": "Y"},
    --   {"sequence": 3, "code": "E11.65", "poa": "Y"}
    -- ]
    scrub_results   JSONB,
    -- scrub_results example:
    -- {
    --   "scrubbed_at": "2026-05-19T10:45:00Z",
    --   "overall_result": "pass",
    --   "edits": [
    --     {"type": "ncci_ptp", "severity": "warning", "message": "Modifier 25 required for 99285 with 93010", "auto_resolved": true}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claims_encounter ON claims(encounter_id);
CREATE INDEX idx_claims_patient ON claims(patient_id);
CREATE INDEX idx_claims_status ON claims(status);
CREATE INDEX idx_claims_org ON claims(organisation_id);
CREATE INDEX idx_claims_payer ON claims(payer_id);
CREATE INDEX idx_claims_submitted ON claims(submitted_at);
CREATE INDEX idx_claims_lines ON claims USING gin(claim_lines);
CREATE INDEX idx_claims_diagnoses ON claims USING gin(claim_diagnoses);
```

---

## Remittance, Denials, and Appeals

```sql
CREATE TABLE remittances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    payer_id        UUID NOT NULL REFERENCES payers(id),
    era_date        DATE NOT NULL,
    check_number    VARCHAR(50),
    total_paid      NUMERIC(12,2),
    edi_control_number VARCHAR(20),
    line_items      JSONB NOT NULL DEFAULT '[]',
    -- line_items example:
    -- [
    --   {
    --     "claim_id": "claim-uuid",
    --     "payer_claim_number": "2026BCBS789012",
    --     "procedure_code": "99285",
    --     "charged": 850.00,
    --     "paid": 612.40,
    --     "adjustments": [
    --       {"group_code": "CO", "carc": "45", "amount": 237.60, "rarc": "N362"}
    --     ]
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_remittances_org ON remittances(organisation_id);
CREATE INDEX idx_remittances_payer ON remittances(payer_id);
CREATE INDEX idx_remittances_date ON remittances(era_date);
CREATE INDEX idx_remittances_items ON remittances USING gin(line_items);

CREATE TABLE denials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    claim_id        UUID NOT NULL REFERENCES claims(id),
    payer_id        UUID NOT NULL REFERENCES payers(id),
    denial_date     DATE NOT NULL,
    carc_code       VARCHAR(10) NOT NULL,
    rarc_code       VARCHAR(10),
    denial_category VARCHAR(50),
    denied_amount   NUMERIC(10,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    assigned_to     UUID REFERENCES users(id),
    denial_details  JSONB NOT NULL DEFAULT '{}',
    -- denial_details example:
    -- {
    --   "denied_lines": [1, 3],
    --   "denied_codes": ["99285", "93010"],
    --   "root_cause_ai": "Payer requires modifier 59 for 93010 when billed with 99285 on same date",
    --   "similar_denials_count": 7,
    --   "suggested_action": "Resubmit with modifier 59 on line 2",
    --   "financial_impact": {
    --     "denied_amount": 125.00,
    --     "estimated_recovery_if_appealed": 112.50,
    --     "appeal_cost_estimate": 15.00
    --   }
    -- }
    appeals         JSONB NOT NULL DEFAULT '[]',
    -- appeals example:
    -- [
    --   {
    --     "appeal_level": 1,
    --     "appeal_date": "2026-06-01",
    --     "letter_generated": true,
    --     "submitted_to": "BCBS MA Appeals Dept",
    --     "outcome": "overturned",
    --     "outcome_date": "2026-06-15",
    --     "recovered_amount": 112.50
    --   }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_denials_claim ON denials(claim_id);
CREATE INDEX idx_denials_payer ON denials(payer_id);
CREATE INDEX idx_denials_status ON denials(status);
CREATE INDEX idx_denials_carc ON denials(carc_code);
CREATE INDEX idx_denials_category ON denials(denial_category);
CREATE INDEX idx_denials_org ON denials(organisation_id);
```

---

## Payer Rules (AI-Learned and Manual)

```sql
CREATE TABLE payer_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payer_id        UUID NOT NULL REFERENCES payers(id),
    rule_type       VARCHAR(30) NOT NULL,
    rule_name       VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    source          VARCHAR(20) NOT NULL,             -- 'manual', 'ai_learned', 'lcd', 'ncd'
    confidence      NUMERIC(5,4),
    effective_date  DATE NOT NULL,
    termination_date DATE,
    rule_definition JSONB NOT NULL,
    -- rule_definition example (bundling rule):
    -- {
    --   "condition": {
    --     "codes_present": ["36415", "80053"],
    --     "same_date_of_service": true,
    --     "same_provider": true
    --   },
    --   "action": "deny_column2",
    --   "affected_code": "36415",
    --   "explanation": "Venipuncture (36415) bundled into comprehensive metabolic panel (80053)",
    --   "override_modifier": null,
    --   "learned_from_denials": ["denial-uuid-1", "denial-uuid-2"],
    --   "denial_pattern_count": 12,
    --   "last_denial_date": "2026-05-10"
    -- }
    -- rule_definition example (auth required):
    -- {
    --   "condition": {"codes": ["27447"], "place_of_service": ["21", "22"]},
    --   "action": "require_prior_auth",
    --   "lcd_id": "L33830",
    --   "documentation_requirements": ["operative report", "failed conservative treatment"],
    --   "auth_submission_method": "fhir_pas"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payer_rules_payer ON payer_rules(payer_id);
CREATE INDEX idx_payer_rules_type ON payer_rules(rule_type);
CREATE INDEX idx_payer_rules_definition ON payer_rules USING gin(rule_definition);
```

---

## CDI Queries

```sql
CREATE TABLE cdi_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    query_type      VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    sent_to         UUID REFERENCES providers(id),
    created_by      UUID REFERENCES users(id),
    query_details   JSONB NOT NULL,
    -- query_details example:
    -- {
    --   "question": "Can you clarify whether the patient's chest pain is consistent with acute coronary syndrome or non-cardiac chest pain?",
    --   "ai_generated": true,
    --   "document_id": "doc-uuid",
    --   "current_codes": ["R07.9"],
    --   "potential_codes": ["I21.01", "I20.0"],
    --   "potential_drg_impact": {"from": "313", "to": "280", "revenue_delta": 4200.00},
    --   "sent_at": "2026-05-19T09:00:00Z",
    --   "response": "Confirmed acute STEMI based on troponin and EKG findings. Updated note accordingly.",
    --   "responded_at": "2026-05-19T14:30:00Z",
    --   "resolution": "Code updated from R07.9 to I21.01. DRG changed from 313 to 280."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cdi_queries_encounter ON cdi_queries(encounter_id);
CREATE INDEX idx_cdi_queries_status ON cdi_queries(status);
```

---

## Reference Data

```sql
CREATE TABLE code_sets (
    code_system     VARCHAR(20) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    description     TEXT NOT NULL,
    effective_date  DATE NOT NULL,
    termination_date DATE,
    is_active       BOOLEAN DEFAULT true,
    code_details    JSONB NOT NULL DEFAULT '{}',
    -- code_details example (ICD-10-CM):
    -- {
    --   "long_description": "Type 2 diabetes mellitus with hyperglycemia",
    --   "chapter": "E00-E89",
    --   "block": "E08-E13",
    --   "hcc_category": "19",
    --   "raf_value": 0.104,
    --   "guidelines": "Use additional code to identify control status",
    --   "includes_notes": ["hyperglycemia NOS"],
    --   "excludes1": ["E10.-"],
    --   "crosswalk_snomed": ["44054006"]
    -- }
    -- code_details example (CPT):
    -- {
    --   "long_description": "Office or other outpatient visit, established patient, moderate MDM",
    --   "section": "Evaluation and Management",
    --   "rvu_work": 1.30,
    --   "rvu_pe": 1.73,
    --   "rvu_mp": 0.10,
    --   "rvu_total": 3.13,
    --   "global_period": "XXX",
    --   "modifier_notes": "Use modifier 25 for separately identifiable E/M"
    -- }
    PRIMARY KEY (code_system, code, effective_date)
);

CREATE INDEX idx_code_sets_desc ON code_sets USING gin(to_tsvector('english', description));

CREATE TABLE ncci_edits (
    column1_code    VARCHAR(10) NOT NULL,
    column2_code    VARCHAR(10) NOT NULL,
    effective_date  DATE NOT NULL,
    termination_date DATE,
    modifier_indicator CHAR(1),
    edit_details    JSONB NOT NULL DEFAULT '{}',
    -- edit_details example:
    -- {
    --   "rationale": "Column 2 code is a component of column 1 code",
    --   "mue_value": null,
    --   "mue_adjudication": null,
    --   "exceptions": [],
    --   "quarter": "2026Q2"
    -- }
    PRIMARY KEY (column1_code, column2_code, effective_date)
);

CREATE INDEX idx_ncci_col1 ON ncci_edits(column1_code);
CREATE INDEX idx_ncci_col2 ON ncci_edits(column2_code);
```

---

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB NOT NULL,
    -- changes example:
    -- {
    --   "field": "status",
    --   "old": "ai_suggested",
    --   "new": "coder_review",
    --   "code_system": "ICD10CM",
    --   "code": "I21.01",
    --   "context": {"ip": "10.0.0.1", "user_agent": "Mozilla/5.0..."}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_action ON audit_log(action);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Example Queries

### Find payer-specific bundling rules that apply to a claim

```sql
SELECT
    pr.rule_name,
    pr.rule_definition->>'explanation' AS explanation,
    pr.confidence,
    (pr.rule_definition->>'denial_pattern_count')::int AS denial_count
FROM payer_rules pr
WHERE pr.payer_id = 'payer-uuid'
  AND pr.is_active = true
  AND pr.rule_definition->'condition' @> '{"codes_present": ["36415"]}'::jsonb
ORDER BY pr.confidence DESC;
```

### Query denials with financial impact analysis from JSONB

```sql
SELECT
    d.carc_code,
    d.denial_category,
    COUNT(*) AS denial_count,
    SUM(d.denied_amount) AS total_denied,
    SUM((d.denial_details->'financial_impact'->>'estimated_recovery_if_appealed')::numeric) AS estimated_recovery,
    AVG(d.denied_amount) AS avg_denial
FROM denials d
WHERE d.payer_id = 'payer-uuid'
  AND d.denial_date > now() - INTERVAL '90 days'
GROUP BY d.carc_code, d.denial_category
ORDER BY total_denied DESC;
```

### Find encounters where AI coding confidence was low

```sql
SELECT
    e.id AS encounter_id,
    e.admit_date,
    e.ai_coding_result->>'overall_confidence' AS confidence,
    e.ai_coding_result->>'model_version' AS model,
    jsonb_array_length(e.ai_coding_result->'suggested_codes') AS code_count
FROM encounters e
WHERE e.organisation_id = 'org-uuid'
  AND e.coding_status = 'ai_suggested'
  AND (e.ai_coding_result->>'overall_confidence')::numeric < 0.80
ORDER BY e.admit_date DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 3 | Organisations, providers, users |
| Patients | 1 | Patients (insurance embedded as JSONB) |
| Encounters & Docs | 2 | Encounters, clinical documents |
| Code Assignment | 1 | Code assignments with JSONB AI details |
| Claims & Billing | 2 | Payers, claims (lines/diagnoses as JSONB) |
| Remittance & Denials | 2 | Remittances (lines as JSONB), denials (appeals as JSONB) |
| Payer Rules | 1 | Payer rules with JSONB rule definitions |
| CDI | 1 | CDI queries with JSONB details |
| Reference Data | 2 | Code sets, NCCI edits |
| Audit | 1 | Audit log |
| **Total** | **16** | ~45% fewer tables than normalised model |

---

## Key Design Decisions

1. **Claim lines and diagnoses as JSONB arrays within the claims table:** A claim typically has 1-12 lines and 1-12 diagnoses. Storing these as JSONB arrays in the claim row eliminates 2 join tables and reflects the atomic nature of claim submission (the entire claim is submitted as one unit). GIN indexes support queries into claim line contents.

2. **Patient insurance as JSONB array:** Patients typically have 1-3 insurance coverages. The JSONB array eliminates a join table and supports the common query pattern of "get patient with all their insurance" without a join.

3. **Payer configuration as JSONB:** Each payer has unique requirements (timely filing windows, modifier rules, bundling overrides, prior auth codes) that vary widely. JSONB allows adding new payer-specific fields without migration, which is critical during MVP when payer integration is being built iteratively.

4. **AI coding result embedded in encounter:** The complete AI coding output (model version, all suggested codes with reasoning, E/M calculation, DRG suggestion) is stored as JSONB on the encounter. This keeps the full AI output together as a single unit and avoids requiring a separate table for what is conceptually one AI processing result.

5. **Denial appeals as JSONB array within denials:** Appeals are a sequential lifecycle on a denial (level 1, level 2, external review). Storing them as a JSONB array keeps the full denial lifecycle in one row, simplifying the common query pattern of "get denial with all its appeals."

6. **Payer rule definitions as JSONB:** Payer rules vary enormously in structure — bundling rules have code pairs, auth rules have documentation requirements, frequency rules have time windows. JSONB allows each rule type to have its own structure without a one-size-fits-all relational schema.

7. **Code assignment with separate relational and JSONB layers:** The code assignment table uses relational columns for queryable fields (code, system, sequence, confidence) and JSONB for AI-specific detail (reasoning chains, alternative codes, supporting evidence). This balances query performance with flexibility.

8. **Fewer tables, faster development:** 16 tables vs. 29 in the normalised model means fewer migrations, simpler ORM mappings, and faster development velocity during MVP. The trade-off is that some reporting queries need JSONB operators instead of simple joins.
