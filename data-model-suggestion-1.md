# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Medical Billing & Coding Assistant · Created: 2026-05-19

## Philosophy

This model follows a traditional normalized relational design where every domain concept — patients, encounters, diagnoses, procedures, claims, denials — gets its own table with strict foreign key relationships. The schema mirrors the structure of the medical billing domain itself: a patient has encounters, encounters have diagnoses and procedures, procedures generate claims, claims receive adjudication results, and denials feed back into analytics.

Normalized relational models are the backbone of healthcare IT. Epic's Chronicles database, Cerner's Millennium schema, and CMS's own claims processing systems all use heavily normalized relational designs. The approach provides maximum data integrity, supports complex cross-entity queries (e.g., "find all encounters for patient X where diagnosis Y was billed with procedure Z and denied by payer W"), and aligns naturally with the strict referential integrity requirements of medical billing compliance.

This design prioritises correctness, auditability, and standards alignment over write throughput or schema flexibility. Every code assignment, claim submission, and denial is a discrete, queryable record with full referential integrity back to the source encounter and patient.

**Best for:** Teams building a compliance-first platform where data integrity, complex cross-entity reporting, and audit trail completeness are the top priorities.

**Trade-offs:**
- (+) Maximum referential integrity — foreign keys prevent orphaned records and invalid references
- (+) Complex analytical queries are straightforward with standard SQL JOINs
- (+) Natural alignment with healthcare data standards (X12, FHIR resource types map cleanly to tables)
- (+) Easiest model for compliance auditors to understand and query directly
- (-) High table count increases schema migration complexity as the product evolves
- (-) Multi-jurisdiction or payer-specific field variations require either nullable columns or additional tables
- (-) Write-heavy workflows (high-volume autonomous coding) may encounter lock contention on hot tables
- (-) Adding new entity types or relationships requires DDL changes and migrations

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ICD-10-CM / ICD-10-PCS | `code_sets` reference table with `code_system = 'ICD10CM'` or `'ICD10PCS'`; annual version tracking |
| CPT (AMA) | `code_sets` reference table with `code_system = 'CPT'`; licence tracking in `code_set_licences` |
| HCPCS Level II | `code_sets` reference table with `code_system = 'HCPCS'` |
| NCCI Edits | `ncci_edits` table storing PTP edit pairs and MUE limits; quarterly refresh |
| X12 837P/I | `claims` table fields map to 837 loop/segment structure |
| X12 835 | `remittance_advice` and `remittance_lines` tables parse ERA data |
| MS-DRG | `drg_assignments` table linking encounters to DRG codes with weight and GMLOS |
| APC | `apc_assignments` table for outpatient groupings |
| FHIR R4 | Table structure mirrors FHIR resource types (Patient, Encounter, Claim, ExplanationOfBenefit) |
| HIPAA Security Rule | `audit_log` table with 6-year retention; all PHI access logged |
| SNOMED CT | `code_sets` reference table with cross-maps to ICD-10-CM |
| LCD/NCD | `coverage_determinations` table storing payer-specific coverage rules |

---

## Entity Management

### Organisations and Providers

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    npi             VARCHAR(10) UNIQUE,              -- National Provider Identifier (NPI-2 for orgs)
    tax_id          VARCHAR(11),                      -- EIN for billing
    org_type        VARCHAR(50) NOT NULL,             -- 'health_system', 'physician_group', 'billing_company'
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state           CHAR(2),                          -- ISO 3166-2:US
    zip_code        VARCHAR(10),
    phone           VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    npi             VARCHAR(10) UNIQUE NOT NULL,      -- NPI-1 for individual providers
    credential      VARCHAR(20),                      -- 'MD', 'DO', 'NP', 'PA'
    specialty_code  VARCHAR(10),                      -- CMS specialty code
    taxonomy_code   VARCHAR(20),                      -- NUCC Health Care Provider Taxonomy
    dea_number      VARCHAR(15),                      -- For prescribing providers
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_providers_organisation ON providers(organisation_id);
CREATE INDEX idx_providers_npi ON providers(npi);
CREATE INDEX idx_providers_specialty ON providers(specialty_code);
```

### Users and Access Control

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    role            VARCHAR(50) NOT NULL,             -- 'coder', 'cdi_specialist', 'billing_manager', 'compliance_officer', 'admin'
    provider_id     UUID REFERENCES providers(id),    -- Nullable; set if user is also a provider
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    permission      VARCHAR(100) NOT NULL,            -- 'code_encounters', 'submit_claims', 'view_denials', 'manage_users', 'audit_review'
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES users(id),
    UNIQUE(user_id, permission)
);

CREATE INDEX idx_users_organisation ON users(organisation_id);
CREATE INDEX idx_users_role ON users(role);
```

---

## Patient and Encounter Management

```sql
CREATE TABLE patients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    mrn             VARCHAR(50) NOT NULL,             -- Medical Record Number (org-specific)
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    date_of_birth   DATE NOT NULL,
    sex             CHAR(1),                          -- 'M', 'F', 'U' per X12 837 DMG segment
    ssn_last_four   CHAR(4),                          -- Last 4 of SSN for identity verification
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    state           CHAR(2),
    zip_code        VARCHAR(10),
    phone           VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, mrn)
);

CREATE TABLE patient_insurance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    payer_id        UUID NOT NULL REFERENCES payers(id),
    member_id       VARCHAR(80) NOT NULL,
    group_number    VARCHAR(50),
    plan_name       VARCHAR(255),
    subscriber_name VARCHAR(200),
    relationship    VARCHAR(20),                      -- 'self', 'spouse', 'child', 'other'
    priority        SMALLINT NOT NULL DEFAULT 1,      -- 1=primary, 2=secondary, 3=tertiary
    effective_date  DATE NOT NULL,
    termination_date DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE encounters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    provider_id     UUID NOT NULL REFERENCES providers(id),
    encounter_type  VARCHAR(30) NOT NULL,             -- 'inpatient', 'outpatient', 'emergency', 'observation'
    admit_date      TIMESTAMPTZ NOT NULL,
    discharge_date  TIMESTAMPTZ,
    place_of_service VARCHAR(2) NOT NULL,             -- CMS POS code (11=office, 21=inpatient hospital, 23=ER)
    admission_type  VARCHAR(2),                       -- UB-04 admission type code
    discharge_status VARCHAR(2),                      -- UB-04 patient discharge status code
    financial_class VARCHAR(50),                      -- 'medicare', 'medicaid', 'commercial', 'self_pay'
    status          VARCHAR(30) NOT NULL DEFAULT 'open', -- 'open', 'coded', 'billed', 'closed'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_encounters_patient ON encounters(patient_id);
CREATE INDEX idx_encounters_provider ON encounters(provider_id);
CREATE INDEX idx_encounters_status ON encounters(status);
CREATE INDEX idx_encounters_admit_date ON encounters(admit_date);
CREATE INDEX idx_encounters_organisation ON encounters(organisation_id);
```

---

## Clinical Documentation

```sql
CREATE TABLE clinical_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    document_type   VARCHAR(50) NOT NULL,             -- 'progress_note', 'discharge_summary', 'operative_note', 'h_and_p', 'consult'
    author_id       UUID NOT NULL REFERENCES providers(id),
    document_date   TIMESTAMPTZ NOT NULL,
    content_text    TEXT NOT NULL,                     -- Full clinical note text for NLP processing
    content_format  VARCHAR(20) NOT NULL DEFAULT 'plain_text', -- 'plain_text', 'rtf', 'cda_xml'
    fhir_resource_id VARCHAR(255),                    -- FHIR DocumentReference.id from source EHR
    source_system   VARCHAR(100),                     -- 'epic', 'cerner', 'meditech', 'manual_upload'
    is_final        BOOLEAN NOT NULL DEFAULT false,   -- True when note is signed/finalized
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_clinical_documents_encounter ON clinical_documents(encounter_id);
CREATE INDEX idx_clinical_documents_type ON clinical_documents(document_type);
```

---

## Code Sets and Reference Data

```sql
CREATE TABLE code_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_system     VARCHAR(20) NOT NULL,             -- 'ICD10CM', 'ICD10PCS', 'CPT', 'HCPCS', 'SNOMED', 'LOINC'
    code            VARCHAR(20) NOT NULL,             -- e.g., 'E11.65', '99214', 'J0585'
    description     TEXT NOT NULL,
    long_description TEXT,
    effective_date  DATE NOT NULL,                    -- When this code became active
    termination_date DATE,                            -- When this code was retired (null if active)
    fiscal_year     SMALLINT,                         -- CMS fiscal year for ICD codes
    chapter         VARCHAR(100),                     -- ICD chapter or CPT section
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(code_system, code, effective_date)
);

CREATE INDEX idx_code_sets_system_code ON code_sets(code_system, code);
CREATE INDEX idx_code_sets_description ON code_sets USING gin(to_tsvector('english', description));

CREATE TABLE code_crosswalks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_system   VARCHAR(20) NOT NULL,             -- e.g., 'SNOMED'
    source_code     VARCHAR(20) NOT NULL,
    target_system   VARCHAR(20) NOT NULL,             -- e.g., 'ICD10CM'
    target_code     VARCHAR(20) NOT NULL,
    map_type        VARCHAR(20) NOT NULL,             -- 'exact', 'broader', 'narrower', 'approximate'
    effective_date  DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crosswalks_source ON code_crosswalks(source_system, source_code);
CREATE INDEX idx_crosswalks_target ON code_crosswalks(target_system, target_code);

CREATE TABLE ncci_edits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    column1_code    VARCHAR(10) NOT NULL,             -- CPT/HCPCS code (the comprehensive code)
    column2_code    VARCHAR(10) NOT NULL,             -- CPT/HCPCS code (the component code)
    edit_type       VARCHAR(10) NOT NULL,             -- 'PTP' (procedure-to-procedure), 'MUE'
    modifier_indicator CHAR(1),                       -- '0'=not allowed, '1'=modifier allowed, '9'=not applicable
    effective_date  DATE NOT NULL,
    termination_date DATE,
    ptp_edit_rationale VARCHAR(255),
    mue_value       SMALLINT,                         -- Max units for MUE edits
    mue_adjudication_indicator CHAR(1),               -- '1'=claim line, '2'=day, '3'=beneficiary
    quarter         VARCHAR(6),                       -- e.g., '2026Q2'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ncci_column1 ON ncci_edits(column1_code);
CREATE INDEX idx_ncci_column2 ON ncci_edits(column2_code);
CREATE INDEX idx_ncci_pair ON ncci_edits(column1_code, column2_code);
```

---

## Code Assignment (AI Coding Engine)

```sql
CREATE TABLE code_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    code_system     VARCHAR(20) NOT NULL,             -- 'ICD10CM', 'ICD10PCS', 'CPT', 'HCPCS'
    code            VARCHAR(20) NOT NULL,
    code_description TEXT,
    sequence_number SMALLINT NOT NULL,                -- 1=primary diagnosis, 2+=secondary
    assignment_type VARCHAR(20) NOT NULL,             -- 'ai_suggested', 'coder_assigned', 'coder_confirmed', 'coder_rejected'
    confidence_score NUMERIC(5,4),                    -- AI confidence 0.0000-1.0000
    reasoning_text  TEXT,                             -- LLM reasoning chain explaining code selection
    source_document_id UUID REFERENCES clinical_documents(id),
    source_text_excerpt TEXT,                         -- Relevant excerpt from clinical note supporting this code
    assigned_by     UUID REFERENCES users(id),        -- Null for AI assignments
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    reviewed_by     UUID REFERENCES users(id),        -- Coder who reviewed AI suggestion
    reviewed_at     TIMESTAMPTZ,
    review_action   VARCHAR(20),                      -- 'confirmed', 'modified', 'rejected'
    modifiers       VARCHAR(20)[],                    -- CPT modifiers: e.g., ARRAY['25', '59']
    units           SMALLINT DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_code_assignments_encounter ON code_assignments(encounter_id);
CREATE INDEX idx_code_assignments_code ON code_assignments(code_system, code);
CREATE INDEX idx_code_assignments_type ON code_assignments(assignment_type);
CREATE INDEX idx_code_assignments_confidence ON code_assignments(confidence_score) WHERE assignment_type = 'ai_suggested';

-- E/M level assignments (specialised for evaluation and management codes)
CREATE TABLE em_calculations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    code_assignment_id UUID NOT NULL REFERENCES code_assignments(id),
    mdm_level       VARCHAR(20) NOT NULL,             -- 'straightforward', 'low', 'moderate', 'high'
    num_diagnoses   SMALLINT,
    data_reviewed   TEXT,                             -- Summary of data reviewed (labs, imaging, records)
    risk_level      VARCHAR(20),                      -- 'minimal', 'low', 'moderate', 'high'
    time_based      BOOLEAN DEFAULT false,
    total_time_minutes SMALLINT,
    calculated_cpt  VARCHAR(5) NOT NULL,              -- Resulting E/M CPT code (e.g., '99214')
    guideline_year  SMALLINT NOT NULL DEFAULT 2021,   -- 2021 E/M guidelines
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- DRG assignments for inpatient encounters
CREATE TABLE drg_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    drg_type        VARCHAR(10) NOT NULL,             -- 'MS-DRG', 'APR-DRG', 'AP-DRG'
    drg_code        VARCHAR(5) NOT NULL,
    drg_description TEXT,
    relative_weight NUMERIC(8,4),
    gmlos           NUMERIC(5,2),                     -- Geometric mean length of stay
    alos            NUMERIC(5,2),                     -- Arithmetic mean length of stay
    mdc             VARCHAR(2),                       -- Major Diagnostic Category
    fiscal_year     SMALLINT NOT NULL,
    grouper_version VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drg_assignments_encounter ON drg_assignments(encounter_id);
```

---

## Claims and Billing

```sql
CREATE TABLE payers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    payer_id_code   VARCHAR(20) NOT NULL,             -- Clearinghouse payer ID
    payer_type      VARCHAR(30) NOT NULL,             -- 'medicare', 'medicaid', 'commercial', 'workers_comp', 'tricare'
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    state           CHAR(2),
    zip_code        VARCHAR(10),
    edi_receiver_id VARCHAR(20),                      -- X12 ISA08 receiver ID
    clearinghouse   VARCHAR(50),                      -- 'optum', 'stedi', 'availity', 'waystar'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE claims (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    patient_id      UUID NOT NULL REFERENCES patients(id),
    insurance_id    UUID NOT NULL REFERENCES patient_insurance(id),
    claim_type      VARCHAR(5) NOT NULL,              -- '837P' (professional) or '837I' (institutional)
    claim_number    VARCHAR(50) NOT NULL UNIQUE,       -- Internal claim tracking number
    payer_claim_number VARCHAR(50),                   -- Payer-assigned claim number (from 277/835)
    billing_provider_id UUID NOT NULL REFERENCES providers(id),
    rendering_provider_id UUID REFERENCES providers(id),
    referring_provider_id UUID REFERENCES providers(id),
    service_date_from DATE NOT NULL,
    service_date_to   DATE,
    place_of_service VARCHAR(2) NOT NULL,
    total_charge    NUMERIC(12,2) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft', -- 'draft','scrubbed','submitted','accepted','rejected','paid','denied','appealed'
    submitted_at    TIMESTAMPTZ,
    adjudicated_at  TIMESTAMPTZ,
    clearinghouse   VARCHAR(50),
    edi_control_number VARCHAR(20),                   -- X12 ISA13 interchange control number
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claims_encounter ON claims(encounter_id);
CREATE INDEX idx_claims_patient ON claims(patient_id);
CREATE INDEX idx_claims_status ON claims(status);
CREATE INDEX idx_claims_submitted ON claims(submitted_at);
CREATE INDEX idx_claims_organisation ON claims(organisation_id);

CREATE TABLE claim_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claims(id),
    line_number     SMALLINT NOT NULL,
    code_assignment_id UUID NOT NULL REFERENCES code_assignments(id),
    procedure_code  VARCHAR(10) NOT NULL,             -- CPT or HCPCS code
    modifiers       VARCHAR(2)[],                     -- Up to 4 modifiers
    diagnosis_pointers SMALLINT[] NOT NULL,            -- References to claim diagnosis sequence numbers
    units           NUMERIC(7,2) NOT NULL DEFAULT 1,
    charge_amount   NUMERIC(10,2) NOT NULL,
    revenue_code    VARCHAR(4),                       -- UB-04 revenue code (institutional claims only)
    service_date    DATE NOT NULL,
    ndc_code        VARCHAR(13),                      -- National Drug Code for drug administration
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_claim_lines_claim ON claim_lines(claim_id);

CREATE TABLE claim_diagnoses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claims(id),
    sequence_number SMALLINT NOT NULL,                -- 1=principal, 2+=secondary
    diagnosis_code  VARCHAR(10) NOT NULL,             -- ICD-10-CM code
    poa_indicator   CHAR(1),                          -- Present on admission: 'Y','N','U','W' (institutional)
    code_assignment_id UUID REFERENCES code_assignments(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(claim_id, sequence_number)
);

CREATE INDEX idx_claim_diagnoses_claim ON claim_diagnoses(claim_id);
CREATE INDEX idx_claim_diagnoses_code ON claim_diagnoses(diagnosis_code);
```

---

## Claim Scrubbing

```sql
CREATE TABLE scrub_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claims(id),
    scrub_run_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    overall_result  VARCHAR(10) NOT NULL,             -- 'pass', 'warn', 'fail'
    total_edits     SMALLINT NOT NULL DEFAULT 0,
    errors          SMALLINT NOT NULL DEFAULT 0,
    warnings        SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scrub_edit_details (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scrub_result_id UUID NOT NULL REFERENCES scrub_results(id),
    claim_line_id   UUID REFERENCES claim_lines(id),
    edit_type       VARCHAR(30) NOT NULL,             -- 'ncci_ptp', 'ncci_mue', 'lcd_ncd', 'em_level', 'modifier', 'payer_rule'
    severity        VARCHAR(10) NOT NULL,             -- 'error', 'warning', 'info'
    edit_code       VARCHAR(20),
    message         TEXT NOT NULL,
    suggestion      TEXT,                             -- AI-generated fix suggestion
    auto_resolved   BOOLEAN DEFAULT false,
    resolved_by     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scrub_results_claim ON scrub_results(claim_id);
CREATE INDEX idx_scrub_edit_details_result ON scrub_edit_details(scrub_result_id);
```

---

## Remittance and Denial Management

```sql
CREATE TABLE remittance_advice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    payer_id        UUID NOT NULL REFERENCES payers(id),
    era_date        DATE NOT NULL,
    check_number    VARCHAR(50),
    total_paid      NUMERIC(12,2),
    edi_control_number VARCHAR(20),                   -- X12 835 interchange control number
    raw_835_file_id UUID,                             -- Reference to stored 835 file
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE remittance_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    remittance_id   UUID NOT NULL REFERENCES remittance_advice(id),
    claim_id        UUID REFERENCES claims(id),
    claim_line_id   UUID REFERENCES claim_lines(id),
    payer_claim_number VARCHAR(50),
    procedure_code  VARCHAR(10),
    charged_amount  NUMERIC(10,2),
    paid_amount     NUMERIC(10,2),
    adjustment_amount NUMERIC(10,2),
    carc_code       VARCHAR(10),                      -- Claim Adjustment Reason Code
    rarc_code       VARCHAR(10),                      -- Remittance Advice Remark Code
    group_code      VARCHAR(2),                       -- 'CO' (contractual), 'PR' (patient resp), 'OA' (other adj)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_remittance_lines_claim ON remittance_lines(claim_id);
CREATE INDEX idx_remittance_lines_carc ON remittance_lines(carc_code);

CREATE TABLE denials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claims(id),
    claim_line_id   UUID REFERENCES claim_lines(id),
    remittance_line_id UUID REFERENCES remittance_lines(id),
    payer_id        UUID NOT NULL REFERENCES payers(id),
    denial_date     DATE NOT NULL,
    carc_code       VARCHAR(10) NOT NULL,
    rarc_code       VARCHAR(10),
    denial_category VARCHAR(50),                      -- 'medical_necessity', 'coding_error', 'timely_filing', 'auth_required', 'duplicate', 'bundling'
    denied_amount   NUMERIC(10,2) NOT NULL,
    root_cause_ai   TEXT,                             -- AI-generated root cause analysis
    status          VARCHAR(20) NOT NULL DEFAULT 'open', -- 'open', 'under_review', 'appealed', 'overturned', 'upheld', 'written_off'
    assigned_to     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_denials_claim ON denials(claim_id);
CREATE INDEX idx_denials_payer ON denials(payer_id);
CREATE INDEX idx_denials_status ON denials(status);
CREATE INDEX idx_denials_category ON denials(denial_category);
CREATE INDEX idx_denials_carc ON denials(carc_code);

CREATE TABLE denial_appeals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    denial_id       UUID NOT NULL REFERENCES denials(id),
    appeal_level    SMALLINT NOT NULL DEFAULT 1,      -- 1=first appeal, 2=second, 3=external review
    appeal_date     DATE NOT NULL,
    appeal_letter   TEXT,                             -- Generated or uploaded appeal letter
    supporting_docs UUID[],                           -- References to uploaded supporting documents
    submitted_to    VARCHAR(255),
    outcome         VARCHAR(20),                      -- 'pending', 'overturned', 'upheld', 'partial'
    outcome_date    DATE,
    recovered_amount NUMERIC(10,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appeals_denial ON denial_appeals(denial_id);
```

---

## Coverage Determinations and Payer Rules

```sql
CREATE TABLE coverage_determinations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payer_id        UUID REFERENCES payers(id),       -- Null for CMS national determinations
    determination_type VARCHAR(5) NOT NULL,           -- 'LCD', 'NCD'
    determination_id VARCHAR(20) NOT NULL,            -- CMS LCD/NCD ID (e.g., 'L33830')
    title           TEXT NOT NULL,
    applicable_codes VARCHAR(10)[] NOT NULL,           -- CPT/HCPCS codes this determination covers
    effective_date  DATE NOT NULL,
    termination_date DATE,
    contractor_name VARCHAR(255),                     -- MAC name for LCDs
    summary_text    TEXT,
    full_text       TEXT,
    source_url      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_coverage_det_codes ON coverage_determinations USING gin(applicable_codes);
CREATE INDEX idx_coverage_det_payer ON coverage_determinations(payer_id);

CREATE TABLE payer_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payer_id        UUID NOT NULL REFERENCES payers(id),
    rule_type       VARCHAR(30) NOT NULL,             -- 'bundling', 'modifier_requirement', 'auth_required', 'timely_filing', 'frequency_limit'
    rule_name       VARCHAR(255) NOT NULL,
    applicable_codes VARCHAR(10)[],
    rule_logic      TEXT NOT NULL,                     -- Human-readable rule description
    effective_date  DATE NOT NULL,
    termination_date DATE,
    source          VARCHAR(50),                      -- 'manual', 'ai_learned', 'lcd_ncd'
    confidence      NUMERIC(5,4),                     -- For AI-learned rules
    denial_count    INTEGER DEFAULT 0,                -- Number of denials this rule would have prevented
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payer_rules_payer ON payer_rules(payer_id);
CREATE INDEX idx_payer_rules_codes ON payer_rules USING gin(applicable_codes);
```

---

## CDI (Clinical Documentation Improvement)

```sql
CREATE TABLE cdi_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounters(id),
    document_id     UUID REFERENCES clinical_documents(id),
    query_type      VARCHAR(30) NOT NULL,             -- 'specificity', 'conflicting_dx', 'missing_linkage', 'poa_clarification'
    query_text      TEXT NOT NULL,                     -- Question posed to the physician
    ai_generated    BOOLEAN NOT NULL DEFAULT true,
    suggested_codes VARCHAR(10)[],                     -- Codes that could be assigned if documentation improves
    potential_impact TEXT,                             -- E.g., "Could change DRG from 193 to 192, +$2,400 reimbursement"
    status          VARCHAR(20) NOT NULL DEFAULT 'open', -- 'open', 'sent', 'responded', 'resolved', 'withdrawn'
    sent_to         UUID REFERENCES providers(id),
    sent_at         TIMESTAMPTZ,
    response_text   TEXT,
    responded_at    TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cdi_queries_encounter ON cdi_queries(encounter_id);
CREATE INDEX idx_cdi_queries_status ON cdi_queries(status);
```

---

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,                             -- Null for system/AI actions
    action          VARCHAR(50) NOT NULL,             -- 'code_assigned', 'code_reviewed', 'claim_submitted', 'claim_scrubbed', 'denial_created', 'appeal_submitted'
    entity_type     VARCHAR(50) NOT NULL,             -- 'encounter', 'code_assignment', 'claim', 'denial'
    entity_id       UUID NOT NULL,
    old_values      JSONB,                            -- Previous state (for updates)
    new_values      JSONB,                            -- New state
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partitioned by month for performance and retention management
-- In production, this table should be range-partitioned on created_at
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_action ON audit_log(action);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);
CREATE INDEX idx_audit_log_org ON audit_log(organisation_id);
```

---

## Analytics and Reporting

```sql
CREATE TABLE coder_productivity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    period_date     DATE NOT NULL,                    -- Day-level granularity
    encounters_coded SMALLINT NOT NULL DEFAULT 0,
    codes_assigned  SMALLINT NOT NULL DEFAULT 0,
    ai_suggestions_confirmed SMALLINT NOT NULL DEFAULT 0,
    ai_suggestions_modified SMALLINT NOT NULL DEFAULT 0,
    ai_suggestions_rejected SMALLINT NOT NULL DEFAULT 0,
    avg_time_per_encounter_seconds INTEGER,
    accuracy_rate   NUMERIC(5,4),                     -- Based on audit results
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(user_id, period_date)
);

CREATE INDEX idx_coder_productivity_user_date ON coder_productivity(user_id, period_date);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Providers | 2 | Organisations, providers |
| Users & Access Control | 2 | Users, permissions |
| Patients & Encounters | 3 | Patients, insurance, encounters |
| Clinical Documentation | 1 | Clinical documents |
| Code Sets & Reference Data | 3 | Code sets, crosswalks, NCCI edits |
| Code Assignment & AI | 3 | Code assignments, E/M calculations, DRG assignments |
| Claims & Billing | 4 | Payers, claims, claim lines, claim diagnoses |
| Claim Scrubbing | 2 | Scrub results, scrub edit details |
| Remittance & Denials | 4 | Remittance advice, remittance lines, denials, appeals |
| Coverage & Payer Rules | 2 | Coverage determinations, payer rules |
| CDI | 1 | CDI queries |
| Audit & Analytics | 2 | Audit log, coder productivity |
| **Total** | **29** | |

---

## Key Design Decisions

1. **Separate `code_assignments` from `claim_lines`:** Code assignments track the AI/coder coding process (with confidence scores and reasoning chains), while claim lines represent the final billed output. This separation supports audit trails and allows multiple code assignment attempts per encounter before a claim is finalized.

2. **NCCI edits as a first-class table:** Rather than embedding edit rules in application code, NCCI PTP edits and MUEs are stored relationally, allowing quarterly CMS updates via bulk import and enabling SQL-based claim scrubbing queries.

3. **Denials tracked independently from remittance lines:** A denial is an actionable workflow item with its own lifecycle (open → appealed → overturned/upheld), while remittance lines are raw ERA parse results. The separation supports denial analytics across payer/code/provider dimensions.

4. **UUID primary keys throughout:** Standard for multi-tenant SaaS; avoids sequence-based ID conflicts during data migration or multi-region deployment.

5. **Provider NPI as unique constraint:** NPI is the national standard identifier for healthcare providers (NPI-1 for individuals, NPI-2 for organisations). Using it as a unique constraint ensures no duplicate provider records and aligns with X12 837 provider identification requirements.

6. **Code sets versioned by effective_date:** Medical codes change annually (ICD-10-CM in October, CPT in January). The `effective_date` / `termination_date` pattern allows the system to maintain historical code validity and query "what codes were active on date X?"

7. **AI reasoning stored in `reasoning_text`:** The LLM reasoning chain for each code assignment is stored as text in the code_assignments table, providing full explainability for compliance auditors without requiring a separate event store.

8. **Audit log designed for partitioning:** The audit_log table is designed for range partitioning by `created_at` to support HIPAA's 6-year retention requirement while maintaining query performance on recent data.
