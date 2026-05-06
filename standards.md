# Standards & API Reference

> Project: Medical Billing & Coding Assistant · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

No directly applicable ISO standards govern US medical billing and coding workflows in the way that domestic US regulatory frameworks do. The following ISO standards are tangentially relevant to any healthcare software product in this domain:

- **ISO 27001 — Information Security Management System (ISMS)**
  Official URL: https://www.iso.org/standard/27001
  Any platform handling Protected Health Information (PHI) should be ISO 27001 certified or aligned; demonstrates to health system customers that the vendor has formalised information security controls. Required by many enterprise procurement processes.

- **ISO 13606 — Electronic Health Record Communication**
  Official URL: https://www.iso.org/standard/40784.html
  An international standard for the structure and semantics of EHR extracts. Relevant as a reference model when designing FHIR-compatible clinical data ingestion pipelines for coding workflows.

---

### W3C & IETF Standards

- **HL7 FHIR R4 (HL7 Fast Healthcare Interoperability Resources, Release 4)**
  Official URL: https://hl7.org/fhir/
  The dominant standard for health data exchange in the US. FHIR defines RESTful APIs, resource types (Patient, Encounter, Claim, ExplanationOfBenefit), and JSON/XML data formats for exchanging clinical and administrative data. Required under CMS Interoperability and Prior Authorization Final Rule (CMS-0057-F) for payer APIs. EHR vendors (Epic, Oracle Health, Meditech) expose FHIR R4 endpoints that a coding assistant can consume to retrieve clinical notes and submit coded encounters.

- **SMART on FHIR (App Launch IG v2.2.0)**
  Official URL: https://build.fhir.org/ig/HL7/smart-app-launch/app-launch.html
  The de facto authorization standard for healthcare apps connecting to FHIR servers. Built on OAuth 2.0 with PKCE, it defines launch flows, scopes (e.g. `patient/Encounter.read`, `system/Claim.write`), and OpenID Connect identity tokens. A billing assistant embedded in or integrated with an EHR must implement SMART on FHIR to authenticate securely.

- **OAuth 2.0 (RFC 6749) and OpenID Connect**
  Official URL: https://www.rfc-editor.org/rfc/rfc6749
  The underlying authorization and identity framework for SMART on FHIR. All API-based integrations in this domain use OAuth 2.0 bearer tokens; OpenID Connect provides identity assertions about authenticated users (coders, billing staff, compliance officers).

- **RFC 7231 — HTTP/1.1 Semantics**
  Official URL: https://www.rfc-editor.org/rfc/rfc7231
  The foundational IETF standard for RESTful API communication. All FHIR and clearinghouse REST APIs are built on HTTP/1.1 or HTTP/2; RFC 7231 defines status codes, content negotiation, and caching headers used throughout.

---

### Healthcare-Specific Data Interchange Standards

- **ANSI ASC X12N 837P — Health Care Claim: Professional**
  Official URL: https://x12.org/flow/health-care | Developer reference: https://www.stedi.com/edi/x12/transaction-set/837
  The federally mandated HIPAA transaction for submitting professional (physician) claims from providers to payers. X12 Version 005010 is the current required version. Any coding assistant that generates or validates claims for submission must produce 837P-compliant output.

- **ANSI ASC X12N 837I — Health Care Claim: Institutional**
  Official URL: https://x12.org/flow/health-care | Developer reference: https://www.stedi.com/edi/x12/transaction-set/837
  The HIPAA transaction for hospital/institutional claims. Includes DRG and APC reimbursement groupings alongside ICD-10-CM/PCS and CPT codes.

- **ANSI ASC X12N 835 — Health Care Claim Payment/Advice (ERA)**
  Official URL: https://x12.org/examples | Developer reference: https://www.stedi.com/edi/x12/transaction-set/835
  Electronic Remittance Advice (ERA): payer response detailing how claims were adjudicated — paid amounts, denied codes (CARC/RARC), adjustments. Parsing 835 files is the foundation of denial management and payment reconciliation workflows.

- **ANSI ASC X12N 270/271 — Eligibility Inquiry and Response**
  Official URL: https://x12.org/flow/health-care
  Used to verify patient insurance eligibility and benefits before submitting claims. A coding and billing assistant should trigger 270 eligibility checks to confirm coverage prior to code assignment and claim generation.

- **ANSI ASC X12N 276/277 — Claim Status Request and Response**
  Official URL: https://x12.org/flow/health-care
  Allows providers to query the status of previously submitted claims and receive payer status responses. Used in denial tracking and follow-up workflows.

- **ANSI ASC X12N 278 — Health Care Services Review (Prior Authorization)**
  Official URL: https://x12.org/flow/health-care | Availity developer docs: https://developer.availity.com/
  Used to submit and receive prior authorization requests and responses. Increasingly being replaced by FHIR-based workflows under CMS-0057-F, but X12 278 remains in active use across payers.

- **NCPDP SCRIPT Standard (Version 2023011)**
  Official URL: https://www.ncpdp.org/ | CMS requirements: https://www.cms.gov/medicare/regulations-guidance/electronic-prescribing/adopted-standard-and-transactions
  The national standard for electronic prescribing (e-prescribing) transactions between prescribers and pharmacies. Version 2023011 becomes the required CMS standard beginning January 1, 2028. Relevant to billing assistants that need to capture pharmacy-related HCPCS J-codes from drug administration encounters.

---

### Medical Terminology & Code Set Standards

- **ICD-10-CM (International Classification of Diseases, 10th Revision, Clinical Modification)**
  Official URL: https://www.cms.gov/medicare/coding-billing/icd-10-codes | Files: https://www.cdc.gov/nchs/icd/icd-10-cm/files.html
  US diagnosis coding standard mandated by HIPAA (45 CFR Part 162). Published annually by CMS/NCHS; FY 2026 release (October 1, 2025) contains 487 new codes, 28 deletions, 38 revisions. Distributed as publicly downloadable flat files and PDFs (no API; programmatic access via flat file processing). Free to use in US-market software.

- **ICD-10-PCS (Procedure Coding System)**
  Official URL: https://www.cms.gov/medicare/coding-billing/icd-10-codes | Guidelines: https://www.cms.gov/files/document/2026-official-icd-10-pcs-coding-guidelines.pdf
  US inpatient procedure coding standard for hospital facility billing. FY 2026 release contains 78,986 codes total (156 new, 27 deleted). Published and distributed by CMS as flat files. Free to use.

- **CPT® Code Set (Current Procedural Terminology)**
  Official URL: https://www.ama-assn.org/topics/cpt-codes | Developer Program: https://www.ama-assn.org/practice-management/cpt/cpt-developer-program
  AMA-proprietary procedure code set for physician and outpatient billing. CPT 2026 includes 288 new codes, 46 revisions, 84 deletions (418 total changes), including new codes for AI-related health services. A licence from AMA is mandatory for any software that displays, processes, or transmits CPT codes. The AMA CPT Developer Program provides API access via the AMA Intelligent Platform; licensing fees are usage-volume-based and can reach tens of thousands of dollars annually.

- **HCPCS Level II (Healthcare Common Procedure Coding System)**
  Official URL: https://www.cms.gov/medicare/coding-billing/healthcare-common-procedure-system
  CMS-maintained code set for services, supplies, and drugs not in CPT (DME, drugs, ambulance, etc.). Maintained quarterly by CMS; distributed as publicly downloadable files. Free to use.

- **SNOMED CT (Systematized Nomenclature of Medicine — Clinical Terms)**
  Official URL: https://www.snomed.org/ | Terminology Services API: https://www.implementation.snomed.org/terminology-services
  Comprehensive clinical terminology covering diagnoses, procedures, findings, and observable entities. Required in US EHR systems (ONC certification). Maps to ICD-10 and LOINC. Accessed via FHIR Terminology Server APIs (e.g., Snowstorm — https://github.com/IHTSDO/snowstorm — an open-source HL7 FHIR Terminology Server that hosts SNOMED CT, LOINC, and ICD-10-CM). NLP pipelines for clinical coding should incorporate SNOMED CT concept normalization.

- **LOINC (Logical Observation Identifiers Names and Codes)**
  Official URL: https://loinc.org/
  International terminology for laboratory and clinical observations (lab tests, vital signs, imaging panels). Free to use; no licence required. Relevant for coding assistants that need to identify billable laboratory observations or map clinical findings to CPT codes. Integrated with SNOMED CT via the LOINC Extension to SNOMED CT.

- **RxNorm**
  Official URL: https://www.nlm.nih.gov/research/umls/rxnorm/index.html | API: https://lhncbc.nlm.nih.gov/RxNav/APIs/RxNormAPIs.html
  NIH National Library of Medicine normalised drug terminology linking brand and generic drug names to standardised identifiers (RXCUIs). Free RESTful API (no licence required). Relevant for identifying HCPCS J-code drug administration claims from medication data in clinical notes.

- **NCCI (National Correct Coding Initiative) Edits**
  Official URL: https://www.cms.gov/national-correct-coding-initiative-ncci | Quarterly files: https://www.cms.gov/medicare/coding-billing/national-correct-coding-initiative-ncci-edits/medicare-ncci-procedure-procedure-ptp-edits
  CMS rules preventing unbundling and inappropriate code combinations. Published quarterly as downloadable text/Excel files (no real-time API). Claim scrubbing logic must ingest and apply NCCI Procedure-to-Procedure (PTP) edits, Medically Unlikely Edits (MUEs), and Add-On Code Edits. Free to use.

- **MS-DRG Grouper (Medicare Severity Diagnosis Related Groups)**
  Official URL: https://www.cms.gov/medicare/payment/prospective-payment-systems/acute-inpatient-pps/ms-drg-classifications-and-software
  CMS grouping algorithm determining inpatient reimbursement under IPPS based on ICD-10-CM/PCS codes. CMS distributes official Java source code free of charge under FOIA. Open-source Python implementations also exist (e.g., `drgpy` on GitHub: https://github.com/yubin-park/drgpy). A billing assistant performing inpatient facility coding must validate DRG assignments.

---

### Regulatory & Compliance Frameworks

- **HIPAA Transactions and Code Set Rules (45 CFR Part 162)**
  Official URL: https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-162
  Federal mandate requiring all HIPAA-covered entities to use standard X12 code sets and transaction formats for electronic healthcare billing. Governs adoption timelines (24 months for covered entities, 36 months for small health plans). Any billing/coding software serving HIPAA-covered entities must produce standards-compliant outputs.

- **HIPAA Security Rule (45 CFR Part 164)**
  Official URL: https://www.hhs.gov/hipaa/for-professionals/privacy/laws-regulations/combined-regulation-text/index.html
  Requires administrative, physical, and technical safeguards for PHI. API-based billing assistants must implement encryption in transit (TLS 1.2+), access controls, audit logging (retained ≥ 6 years), and Business Associate Agreements (BAAs) with all subprocessors.

- **ONC 21st Century Cures Act Final Rule (Interoperability & Information Blocking)**
  Official URL: https://healthit.gov/regulations/cures-act-final-rule/
  Prohibits "information blocking" by providers, payers, and health IT developers. Mandates FHIR R4 APIs (US Core Implementation Guide) for patient and population data access. EHR vendors must expose certified APIs; a coding assistant that reads clinical notes from EHRs should use these mandated FHIR endpoints.

- **CMS Interoperability and Prior Authorization Final Rule (CMS-0057-F)**
  Official URL: https://www.cms.gov/newsroom/fact-sheets/cms-interoperability-prior-authorization-final-rule-cms-0057-f
  Requires Medicare Advantage, Medicaid, and CHIP plans to implement FHIR APIs for prior authorization (operational requirements from January 1, 2026; full API compliance from January 1, 2027). Relevant for coding assistants that need to check whether a procedure code will require prior authorization before claim submission.

---

### FHIR Implementation Guides

- **HL7 Da Vinci Prior Authorization Support (PAS) IG v2.1.0**
  Official URL: https://hl7.org/fhir/us/davinci-pas/
  Defines FHIR-based prior authorization submission and response workflows, mapping between FHIR resources and X12 278 transactions. Required for integration with payer PA systems under CMS-0057-F.

- **HL7 Da Vinci Coverage Requirements Discovery (CRD) IG**
  Official URL: https://build.fhir.org/ig/HL7/davinci-crd/
  Enables real-time discovery of payer coverage requirements (LCD/NCD rules, prior auth requirements) within the EHR workflow using CDS Hooks. Highly relevant for a coding assistant that should alert clinicians when a planned procedure code is not covered or requires authorization.

- **CARIN Consumer Directed Payer Data Exchange (CARIN IG for Blue Button) v2.1.0**
  Official URL: https://www.hl7.org/fhir/us/carin-bb/
  Defines FHIR-based payer APIs exposing adjudicated claims (ExplanationOfBenefit resources) with detailed coding information (ICD, CPT, HCPCS codes, deny/pay amounts). Mandated by CMS-0057-F for payer Patient Access APIs. A denial management module can consume CARIN BB EOB resources to import adjudication results directly.

- **HL7 Da Vinci Payer Data Exchange (PDex) IG v2.1.0**
  Official URL: https://build.fhir.org/ig/HL7/davinci-epdx/
  Governs payer-to-payer and payer-to-provider clinical and claims data exchange via FHIR. Relevant for scenarios where the billing assistant needs to retrieve prior claims history from a payer for coding context.

---

## Similar Products — Developer Documentation & APIs

### CMS Blue Button 2.0 API
- **Description:** Government FHIR API providing Medicare beneficiaries' Part A, B, and D claims data for 60+ million enrollees. Returns adjudicated claims as FHIR ExplanationOfBenefit resources with ICD-10, CPT, HCPCS, and NPI codes. Refreshed weekly.
- **API Documentation:** https://bluebutton.cms.gov/api-documentation/
- **Developer Portal:** https://developer.cms.gov/bluebutton-api/
- **SDKs / Libraries:** No official SDK; community Python and JavaScript libraries available. Sandbox environment with synthetic beneficiary data provided.
- **Standards:** FHIR R4, CARIN IG for Blue Button, OAuth 2.0, JSON
- **Authentication:** OAuth 2.0 (beneficiary-authorized access)
- **Notes:** Read-only claims history data. Useful for training denial prediction models and building patient claim history views.

### Optum (Change Healthcare) Developer Portal
- **Description:** Formerly Change Healthcare; now Optum Intelligent EDI clearinghouse. Processes $1.5 trillion in claims annually, connecting to 2,400+ payers. Offers RESTful and batch APIs for eligibility, claims submission, remittance, attachments, prior authorization, and FHIR interoperability.
- **API Documentation:** https://developer.optum.com/eligibilityandclaims/reference/api-overview
- **Developer Portal:** https://developer.optum.com/
- **Getting Started:** https://developer.optum.com/eligibilityandclaims/docs/get-started-with-optum-api
- **API Endpoints:** X12 837P/I claim submission, X12 270/271 eligibility, X12 835 ERA, X12 276/277 claim status, X12 275 attachments, FHIR interoperability services
- **Standards:** X12 005010, FHIR R4, JSON (REST wrapper over EDI), OpenAPI
- **Authentication:** OAuth 2.0 (API key + bearer token)
- **Notes:** Sandbox environment available. A key integration point for claim submission, ERA retrieval, and denial tracking in any billing assistant.

### Stedi Healthcare Clearinghouse API
- **Description:** API-first healthcare clearinghouse built on AWS. Connects to 3,400+ payers. Allows developers to submit claims and eligibility checks in JSON (automatically converted to X12 EDI) or raw X12 format. Public OpenAPI spec available. Developer-friendly alternative to legacy clearinghouse integrations.
- **API Documentation:** https://www.stedi.com/docs/healthcare/api-reference
- **Developer Portal / Docs:** https://www.stedi.com/docs/healthcare
- **SDKs / Libraries:** REST/JSON API; webhooks for event-driven processing; EDI reference site at https://www.stedi.com/edi/x12
- **Standards:** X12 005010 (837P, 837I, 270/271, 276/277, 835), JSON, REST, OpenAPI, webhook delivery
- **Authentication:** API key (HTTPS)
- **Notes:** Particularly suitable for health tech startups and embedded billing products seeking developer-friendly APIs without legacy EDI infrastructure. CMS Medicare eligibility attestation required as of May 2026.

### Availity Developer Portal
- **Description:** Major healthcare information network connecting 2+ million providers to health plans. Provides REST APIs for HIPAA-standard X12 transactions (270/271, 276/277, 278) and FHIR-based services, plus value-add APIs (IsAuthRequired, Auth Attachments, Enhanced Claim Status).
- **API Documentation:** https://developer.availity.com/blog/2025/3/25/availity-api-guide
- **Developer Portal:** https://developer.availity.com/
- **Key APIs:** Eligibility & Benefits (270/271), Enhanced Claim Status (276/277+), Service Authorization (278), Claim Attachments (275), HIPAA Transactions, FHIR APIs
- **Standards:** X12 005010, FHIR R4, REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** OAuth 2.0 access token
- **Notes:** Strong payer network coverage for prior authorization workflows; relevant for coding assistants that need to check auth requirements before claim submission.

### Fathom Health — Autonomous Medical Coding Platform
- **Description:** LLM-powered autonomous coding platform for professional and facility coding (ICD-10-CM, CPT/HCPCS, E/M, modifiers). Integrates with Epic, Oracle Cerner, and Meditech. Claims ~90% automation rate across full charts with contractual SLAs.
- **API Documentation:** https://developers.fathom.ai/
- **Standards:** FHIR R4 (EHR integration), HL7 v2.x, X12 837 output
- **Authentication:** API key + OAuth 2.0 (per documentation)
- **Notes:** The developer API (`developers.fathom.ai`) enables embedding Fathom's autonomous coding engine into third-party platforms. The LLM reasoning chain output (code-level explanations) is accessible via the API for compliance audit trails.

### Waystar Revenue Cycle Platform
- **Description:** Publicly traded (Nasdaq: WAY) revenue cycle platform with AI-powered claim scrubbing (AltitudeAI), eligibility verification, denial management, prior authorization, and patient payments. Connects to 5,000+ payers as its own clearinghouse.
- **API Documentation:** https://developer.patientco.com/ui/ (Waystar developer portal)
- **Integration Reference:** https://supergood.ai/docs/waystar-api/
- **Key APIs:** X12 837/835/270/271/276/277/278 (full EDI suite), HL7 v2.x, FHIR R4 connectivity, RESTful batch and real-time APIs, webhook event delivery
- **Standards:** X12 005010, FHIR R4, REST/JSON, HL7 v2.x
- **Authentication:** OAuth 2.0
- **Notes:** Suitable for integration as the downstream clearinghouse and denial management layer. Waystar's AltitudeAI APIs provide programmatic access to AI claim scrubbing results and denial risk scoring.

### NLM RxNorm API
- **Description:** NIH National Library of Medicine free RESTful API for normalized drug terminology. Resolves brand and generic drug names to standardised RXCUIs and links to HCPCS J-codes, NDC numbers, and drug class taxonomies. Supports JSON and XML responses.
- **API Documentation:** https://lhncbc.nlm.nih.gov/RxNav/APIs/RxNormAPIs.html
- **API Explorer:** https://lhncbc.nlm.nih.gov/RxNav/
- **Standards:** REST, JSON/XML; integrates with FHIR MedicationRequest resources
- **Authentication:** None (open public API)
- **Notes:** Free to use without licence. Key integration for coding assistants that need to map administered drugs in clinical notes to HCPCS J-codes for drug administration billing.

### SNOMED International Snowstorm FHIR Terminology Server
- **Description:** Open-source HL7 FHIR Terminology Server capable of hosting SNOMED CT, LOINC, ICD-10-CM, and other code systems. Supports FHIR terminology operations (`$lookup`, `$validate-code`, `$translate`, `$expand`) for concept normalization in NLP coding pipelines.
- **GitHub (Snowstorm):** https://github.com/IHTSDO/snowstorm
- **SNOMED Terminology Services documentation:** https://www.implementation.snomed.org/terminology-services
- **SNOMED Browser:** https://browser.ihtsdotools.org/
- **Standards:** FHIR R4 Terminology Services, REST, JSON
- **Authentication:** None in open-source deployment; configurable per deployment
- **Notes:** Deploy locally for NLP pipeline integration. SNOMED CT requires a licence for most commercial uses (check snomed.org for affiliate licences by country). LOINC and ICD-10-CM require no licence.

---

## Notes

### Key Integration Architecture

A medical billing and coding assistant will typically integrate standards and APIs at four layers:

1. **Clinical data ingestion**: EHR FHIR R4 APIs (SMART on FHIR + OAuth 2.0) to retrieve clinical encounters, notes, and orders.
2. **Code assignment**: AI/NLP engine mapping clinical text to ICD-10-CM, CPT, HCPCS codes; validated against NCCI edits (quarterly CMS files) and E/M calculators.
3. **Claim generation and submission**: X12 837P/I claim generation → clearinghouse API (Stedi, Optum, Availity, Waystar) → payer adjudication.
4. **Denial management**: X12 835 ERA parsing → CARIN IG for Blue Button EOB ingestion → denial analytics → appeal workflow.

### Emerging Standards to Watch

- **CMS-0062-P (2026 proposed rule)**: Extends CMS-0057-F prior authorization FHIR API requirements to cover drugs; if finalized, will require coding assistants to check drug prior authorization requirements in real time.
- **HL7 Da Vinci Clinical Data Exchange (CDex) IG**: Enables structured clinical document exchange between payers and providers in support of prior authorization and claim audits; increasingly adopted alongside PAS.
- **NCPDP SCRIPT 2023011**: Becomes the federally required e-prescribing standard in 2028; relevant for billing assistants handling specialty pharmacy and drug administration encounters.
- **CPT AI Codes (2026)**: AMA added new CPT codes specifically for AI-based clinical decision support services in CPT 2026; an AI coding assistant may itself generate billable AI-service CPT codes for client documentation.
