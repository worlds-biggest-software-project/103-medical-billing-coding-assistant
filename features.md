# Medical Billing & Coding Assistant — Feature & Functionality Survey

> Candidate #103 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| 3M / Solventum CAC (Computer-Assisted Coding) | NLP-based CAC for inpatient + outpatient | Proprietary SaaS | https://www.solventum.com |
| Optum360 EncoderPro / Codify | Coding reference + encoder + CAC | Proprietary SaaS | https://www.optumcoding.com |
| Nym Health | Autonomous AI medical coding (NLP) | Proprietary SaaS | https://nym.health |
| Fathom Health | Autonomous AI coding (LLM-based) | Proprietary SaaS | https://www.fathomhealth.com |
| Waystar (AltitudeAI) | Revenue cycle platform with AI denial management | Proprietary SaaS | https://www.waystar.com |
| Codify by AAPC | Coding reference tool for professional coders | Proprietary SaaS | https://www.aapc.com/codify |
| Change Healthcare (Optum) ClaimLogic | Claim scrubbing + clearinghouse | Proprietary SaaS | https://www.changehealthcare.com |
| nThrive | Revenue cycle management + coding analytics | Proprietary SaaS | https://www.nthrive.com |
| Medicomp Quippe / MEDCIN | Clinical documentation + AI coding at point of care | Proprietary SaaS | https://www.medicomp.com |
| AHRQ HCUP CCS | ICD-10 grouping for analytics (not billing) | Open Source / Free (US Gov) | https://hcup-us.ahrq.gov |

## Feature Analysis by Solution

### 3M / Solventum Computer-Assisted Coding (CAC)

**Core features**
- NLP engine reads clinical documentation (discharge summaries, operative notes, progress notes) and suggests ICD-10-CM/PCS and CPT codes
- Real-time code suggestion as clinicians dictate or type notes
- CDI (clinical documentation improvement) feedback: flags documentation gaps that would prevent code assignment
- DRG (MS-DRG) grouper integration for inpatient reimbursement validation
- APC grouper for outpatient facility coding validation
- NCCI edit checking before claim submission
- Coding quality auditing and coder productivity dashboards

**Differentiating features**
- Long-established CAC platform; deployed in large health systems and academic medical centers for 20+ years
- Pre-populated code suggestions improve first-pass rates compared to purely manual coding
- CDI workflow integration: CDI specialists and coders work in the same platform
- Hospitals report enhanced coding completeness and faster coding cycle times

**UX patterns**
- Coder workstation: clinical document displayed alongside suggested codes; coder confirms, modifies, or rejects suggestions
- CDI query workflow: system generates queries to physicians for documentation clarification
- Coder productivity metrics tracked per coder and per encounter type

**Integration points**
- HL7 v2.x ADT feeds for patient demographic and encounter data
- EHR integration via HL7 interfaces (Epic, Oracle, Meditech)
- X12 EDI 837 output to billing systems
- MS-DRG and APC grouper APIs for payer validation

**Known gaps**
- Human coders still required to review and confirm all code suggestions; not autonomous
- NLP accuracy diminishes with non-standard documentation styles, handwriting, or abbreviation-heavy notes
- Enterprise pricing ($50,000+/year) excludes small practices and physician groups
- AI model retraining requires vendor engagement; not self-optimizing per customer

**Licence / IP notes**
- Proprietary; 3M Health Information Systems (now Solventum following 3M spin-off, 2024)
- No open-source components; enterprise licensing only
- Solventum retains full IP in the NLP models and grouper logic

---

### Optum360 EncoderPro / Codify

**Core features**
- ICD-10-CM, ICD-10-PCS, CPT, and HCPCS code lookup with official guidelines and cross-references
- Encoder: guided code assignment based on diagnostic and procedure descriptions
- E/M (Evaluation and Management) complexity calculator
- NCCI (National Correct Coding Initiative) edit checking
- APC and DRG grouper tools for revenue impact calculation
- LCD/NCD (Local/National Coverage Determination) policy lookup
- Coding compliance audit tools

**Differentiating features**
- Official AMA/CMS guidance integrated directly into code lookup (not just the codes themselves)
- Owned by Optum (UnitedHealth Group subsidiary) — deep payer-side intelligence informing the tool
- EncoderPro Professional CAC tier: NLP-assisted code suggestion from clinical text (upgrade tier)
- Widely adopted by hospital coding departments as the reference standard

**UX patterns**
- Search-first interface: coders type a diagnosis or procedure description and the encoder returns ranked code options with official guidelines
- Cross-reference navigation (e.g., ICD-10-CM → CPT crosswalk)
- Side-by-side document and code assignment view in the CAC tier

**Integration points**
- Export to billing systems via standard EDI formats
- Integration with Optum payment analytics tools
- Optum Intelligent EDI (claim scrubbing clearinghouse) for end-to-end RCM

**Known gaps**
- Reference / encoder tier ($300–$500/user/year) requires significant manual coder labor
- AI/NLP capability limited to higher-cost CAC tier; base tool is a manual reference only
- Optum is a major payer competitor — trust and data-sharing concerns among some provider organizations
- Coverage determination data (LCD/NCD) requires manual updates as policy changes

**Licence / IP notes**
- Proprietary; Optum / UnitedHealth Group subsidiary
- CPT code content licensed from AMA; ICD content from CMS
- No open-source components

---

### Nym Health

**Core features**
- Autonomous AI coding engine using specialized clinical linguistics NLP
- Processes full clinical charts and outputs complete ICD-10-CM, CPT, and HCPCS code sets without human intervention
- "Zero code touch" goal: charts move from clinical documentation directly to billing without human approval step
- Real-time compliance engine: codes validated against NCCI edits and payer LCD/NCD rules
- Charge capture optimization: identifies billable services in documentation that manual coders miss
- Supported specialties (as of 2026): emergency medicine, radiology, urgent care, outpatient surgery

**Differentiating features**
- Clinical linguistic engine trained to understand clinical context and narrative, not just keyword matching
- Recognized as Top AI Medical Coding Solution by Healthcare Tech Outlook Magazine (2026)
- Fully autonomous coding in supported specialties: no human review required for routine encounters
- Closed-loop performance reporting: accuracy metrics, appeal outcomes, and ROI dashboards provided to customers

**UX patterns**
- Operates in the background; clinical staff and coders see coded claims emerge from the EHR to the billing system
- Exception queue for low-confidence encounters requiring human coder review
- Dashboard showing automation rate, accuracy, denial rate, and financial impact

**Integration points**
- EHR integration via HL7 v2.x ADT and clinical document feeds
- FHIR API integration for modern EHR connectivity
- Output via X12 837 to billing systems and clearinghouses
- Payer rule library updated continuously by Nym's clinical informatics team

**Known gaps**
- Specialty coverage limited; complex inpatient DRG coding and multi-specialty academic medicine not yet fully supported
- Black-box concern: limited explanability for why specific codes were assigned (improving with clinical reasoning outputs)
- Customer must trust the autonomous engine; significant change-management effort for health systems with large coding departments
- Enterprise-only pricing; not accessible for small physician groups

**Licence / IP notes**
- Proprietary SaaS; Nym Health Inc.
- Raised $25M Series B (2022)
- Clinical NLP models are proprietary IP; no open-source components

---

### Fathom Health

**Core features**
- LLM-powered autonomous medical coding platform
- Reads full clinical charts and autonomously assigns ICD-10-CM, CPT, and HCPCS codes with reasoning chains
- Claims 90% average automation rate across full charts (highest in the marketplace per self-reported metrics)
- Performance SLAs on automation rate and claim turnaround time provided contractually
- Denial root-cause analysis: tracks which codes generate denials by payer and feeds back to coding model
- Supports high-volume specialties: radiology, emergency medicine, pathology, anesthesia

**Differentiating features**
- LLM-based architecture (vs. NLP rule-based): broader context window allows understanding of complex multi-document encounters
- Contractual performance guarantees on automation rate — differentiates from vendors who offer tools but not outcome commitments
- Claims to reduce costs, boost accuracy, cut denials, and accelerate cash (with customer case studies)
- Raised $46M Series B (2023) for autonomous coding expansion

**UX patterns**
- EHR-integrated workflow: frees coders from routine encounter assignment
- Human coder review queue for complex or low-confidence encounters
- Financial impact dashboard showing ROI vs. manual coding baseline

**Integration points**
- EHR integration via FHIR API and HL7 v2.x
- Billing system output via X12 837
- Clearinghouse integration for claim submission and denial tracking

**Known gaps**
- LLM approach creates explainability risk for compliance audits: tracing why a specific code was assigned requires model reasoning output, which is improving but not yet standard practice
- Specialty coverage expanding; complex inpatient DRG coding and surgical coding still require more human oversight
- As a newer entrant vs. 3M/Solventum, health system IT security teams subject it to higher scrutiny
- No self-serve or SME pricing; enterprise-focused

**Licence / IP notes**
- Proprietary SaaS; Fathom Health Inc.
- No open-source components; LLM models are proprietary

---

### Waystar (AltitudeAI)

**Core features**
- AI-powered claim scrubbing: 150+ trained AI models validating claims against payer rules before submission
- Eligibility verification: automated X12 270/271 queries at multiple touchpoints in the revenue cycle
- Denial management: AltitudeAI prevents $15.5B in denials reported; 90% reduction in time on denial appeals
- "Silent denials" management: AI solution that automatically identifies and enables appeal of unjustified payer payment take-backs
- Appeal package generation: AI accelerates appeal package creation by 90% vs. manual process
- Front-end patient access: AI-assisted insurance verification, coverage gap identification, and financial counseling
- Revenue leakage detection: validates patient assignments, supporting documentation, and coding accuracy pre-billing

**Differentiating features**
- Agentic AI vision: autonomous revenue cycle where AI acts within workflows end-to-end with minimal human intervention
- 99% clean claim and first-pass acceptance rates reported
- Publicly traded (Nasdaq IPO June 2024; ~$3.5B valuation) — financial stability and transparency as differentiator
- AltitudeAI ranked #1 in revenue cycle management by independent client surveys (2026)
- ~$3M net revenue recovered per 10,000 patient discharges from revenue leakage detection

**UX patterns**
- Revenue cycle command center: unified view of claims pipeline, denial queue, and financial performance
- Exception-based workflow: AI handles clean claims automatically; staff attention directed to exceptions and complex denials
- Automated worklist generation for denial appeals, prioritized by financial impact

**Integration points**
- EHR integration via HL7 and FHIR
- Clearinghouse: Waystar is its own clearinghouse, submitting to 5,000+ payers directly
- X12 EDI 837/835/270/271/276/277 (full EDI transaction suite)
- ERP and hospital finance system integrations

**Known gaps**
- Full-platform cost makes it most appropriate for hospital systems and large physician groups
- Coding (ICD-10/CPT assignment from clinical notes) is not Waystar's core product; it is a claims processing and RCM platform
- Integration complexity for practices already using an embedded EHR billing module
- "Silent denials" solution announced 2026; not yet broadly deployed with validated outcomes data

**Licence / IP notes**
- Proprietary SaaS; Waystar Holding Corp. (Nasdaq: WAY)
- No open-source components
- Acquired by Optum / Change Healthcare in a deal that was blocked by DOJ; remains independent (2022)

---

### Codify by AAPC

**Core features**
- ICD-10-CM, ICD-10-PCS, CPT, and HCPCS code lookup with official guidelines and annotations
- E/M complexity calculator (2021 guidelines)
- NCCI edit checker for code combination validation
- APC and DRG grouper tools
- LCD and NCD policy lookup linked to relevant CPT codes
- Coder training content and exam preparation resources integrated

**Differentiating features**
- Published by AAPC (American Academy of Professional Coders) — the professional association for medical coders
- Includes AAPC exam preparation and continuing education content alongside the reference tool
- Trusted by individual coders and small billing companies as a professional reference

**UX patterns**
- Code search first: type a disease or procedure name, receive ranked code list with guidelines and notes
- Side-by-side reference panes for ICD and CPT

**Integration points**
- Standalone reference tool; no direct EHR or billing system integration
- Export / copy codes to billing system manually

**Known gaps**
- Manual reference tool only; no NLP-assisted coding or AI code suggestion
- No claim scrubbing, denial management, or billing workflow features
- Requires human coder for all code assignment
- Not suitable as the sole coding tool for high-volume environments

**Licence / IP notes**
- Proprietary SaaS; AAPC (American Academy of Professional Coders)
- CPT content licensed from AMA; ICD content from CMS
- AAPC's annotations, explanations, and training content are proprietary

---

### Change Healthcare (Optum) ClaimLogic

**Core features**
- Claim scrubbing: checks claims against payer-specific rules, NCCI edits, and CMS policy before submission
- Electronic claim submission to 5,000+ payers via the Change Healthcare clearinghouse network
- ERA (835) posting and automated payment reconciliation
- Eligibility verification (270/271) at pre-registration and pre-service
- Claims status inquiry (276/277) and payer response management
- Denial and rejection tracking dashboard

**Differentiating features**
- One of the largest US clearinghouses by claim volume — extensive payer connectivity
- Payer rule library updated continuously from real-world claim adjudication data
- Note: Change Healthcare suffered a major ransomware attack in February 2024 that disrupted US healthcare billing for weeks — a significant enterprise risk event that has affected trust and customer retention

**UX patterns**
- Claims dashboard: pending, submitted, rejected, and paid queues
- Rule editor for practice-specific pre-submission edits (advanced tier)
- ERA auto-posting into practice management system

**Integration points**
- Native integration with most major EHR and PMS platforms
- X12 EDI full suite (837, 835, 270/271, 276/277)
- API for embedded integration in third-party RCM platforms

**Known gaps**
- The February 2024 ransomware attack exposed systemic infrastructure concentration risk; many providers accelerated plans to diversify clearinghouse vendors
- AI-powered claim optimization less advanced than Waystar AltitudeAI
- Coding assistance (ICD/CPT assignment) is not a core feature; clearinghouse only

**Licence / IP notes**
- Proprietary; Optum / UnitedHealth Group (acquired 2022, ~$13B; post-DOJ-cleared close)
- No open-source components

---

### nThrive

**Core features**
- Revenue cycle management suite: patient access, coding analytics, charge integrity, and denial management
- Coding accuracy and compliance monitoring: tracks coder performance, coding patterns, and audit findings
- CDI (clinical documentation improvement) workflow integration
- Denial management: root-cause analytics and automated worklist generation
- Patient financial services: financial counseling, payment estimation, and payment plans

**Differentiating features**
- End-to-end RCM platform covering front-end patient access through back-end denial recovery
- Strong coding compliance and audit analytics for HIM directors
- Advisory services layer: nThrive offers consulting alongside software for health system revenue cycle transformation

**UX patterns**
- Analytics-first interface: KPI dashboards for coding quality, denial rates, and collection performance
- Coder performance scorecards
- Denial categorization by reason code, payer, and service line

**Integration points**
- EHR integration via HL7 v2.x
- Clearinghouse connectivity for claim submission
- RPA (robotic process automation) for payer portal interactions

**Known gaps**
- Less AI-forward than Nym Health, Fathom Health, or Waystar in autonomous coding / claim scrubbing
- SME / physician group use cases underserved; primarily hospital and health system focus
- Product has undergone significant ownership changes affecting investment levels

**Licence / IP notes**
- Proprietary SaaS; nThrive (formerly Precyse + MedAssets revenue cycle assets)
- No open-source components

---

### Medicomp Quippe / MEDCIN

**Core features**
- MEDCIN clinical knowledge base: structured clinical terminology covering 300,000+ clinical findings, symptoms, diagnoses, and procedures
- Quippe: physician point-of-care documentation tool using MEDCIN terminology for structured note entry
- Real-time billing code suggestion (ICD-10-CM, CPT, HCPCS, E/M) as physician documents the encounter
- CDI feedback at the point of note entry: alerts physician when documentation is insufficient for intended code
- Risk adjustment / HCC coding support integrated into documentation workflow

**Differentiating features**
- Operates at the point of care — AI coding embedded in physician documentation, not applied post-hoc
- MEDCIN knowledge base is one of the most comprehensive structured clinical terminology systems outside SNOMED CT
- Prevents downstream coding failures by correcting documentation before the note is signed
- Distinct from post-visit CAC tools: designed to eliminate coding errors at source

**UX patterns**
- Point-of-care documentation interface: physician selects findings from MEDCIN hierarchy or types free text; system structures it automatically
- Real-time E/M complexity calculator updates as physician documents
- Integrated into EHR workflow via SDK / API

**Integration points**
- EHR integration via SDK embedded in physician documentation workflow
- Output to billing system for code submission
- HL7 messaging for patient data

**Known gaps**
- Requires EHR vendor integration or significant implementation effort to embed MEDCIN in clinical workflow
- Physician adoption challenge: requires behavior change in how physicians document
- Less known than 3M/Solventum or Optum in the health system market
- AI reasoning transparency: MEDCIN is rule-based; LLM-level flexibility not available

**Licence / IP notes**
- Proprietary; Medicomp Systems Inc.
- MEDCIN knowledge base is proprietary IP; licence required for any use
- No open-source components

---

### AHRQ HCUP CCS (Clinical Classifications Software)

**Core features**
- Groups ICD-10-CM diagnosis codes into 285 clinically meaningful categories
- Groups ICD-10-PCS and CPT procedure codes into clinically meaningful categories
- Used for research, epidemiological analysis, and population health analytics
- Freely available from AHRQ (Agency for Healthcare Research and Quality)

**Differentiating features**
- Only free / open-access tool in this comparison
- US government-published; appropriate for academic research, public health surveillance, and benchmarking
- Well-documented and widely cited in peer-reviewed literature

**UX patterns**
- Crosswalk table distributed as downloadable flat files (CSV / SAS)
- No interactive UI; requires integration into analytics pipeline

**Integration points**
- Import into analytics tools (SAS, R, Python, SQL databases)
- Can augment any coding or analytics platform as a classification layer

**Known gaps**
- Not a billing tool; does not produce claims or drive reimbursement
- Not updated at the same frequency as ICD-10-CM; slight lag in incorporating new codes
- No NLP or AI features

**Licence / IP notes**
- **Public domain / open government data** (US government publication; AHRQ)
- No licence required for use in any context, including commercial products
- No copyleft; safe to embed in any software

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- ICD-10-CM and CPT code lookup with official guidelines and annotations
- NCCI edit checking before claim submission
- E/M complexity calculator (2021 guidelines)
- LCD / NCD policy lookup linked to procedure codes
- X12 EDI 837 claim output to clearinghouse or billing system
- Audit trail of code assignments for compliance review

### Differentiating Features
- Autonomous coding without human review step (Nym Health, Fathom Health — leading edge)
- LLM-based full-chart understanding producing coded encounters with reasoning chains (Fathom Health)
- Agentic AI revenue cycle with "silent denial" detection and automated appeal package generation (Waystar)
- Point-of-care coding embedded in physician documentation workflow (Medicomp Quippe — unique)
- Cross-customer payer denial intelligence shared across all clients for real-time scrubbing improvement (Waystar, Athenahealth analogy)
- CDI feedback at the moment of note entry to prevent documentation failures before signing (3M, Medicomp)

### Underserved Areas / Opportunities
- End-to-end autonomous coding with full explainability: LLM reasoning chains that auditors can follow for compliance review
- Natural language compliance query: "show all encounters where high-complexity E/M was billed but documentation supports only moderate complexity" — addressed by early AI platforms but not yet productized broadly
- Payer-specific LCD/NCD rule modeling beyond NCCI edits: AI modeling proprietary payer adjudication rules that rule-based scrubbers miss
- Denial learning loop: AI models that improve claim scrubbing rules automatically from each denial received, per payer
- CDI + coding in a single AI pass: simultaneously improve documentation and assign codes from a single ambient listening session during the encounter
- Coding audit at population scale: AI reviewing 100% of encounters for compliance rather than the current 1–5% manual sampling

### AI-Augmentation Candidates
- Full-encounter autonomous coding (ICD-10-CM/PCS + CPT) from clinical note with reasoning output
- Real-time CDI alerts during note-writing: "this documentation does not support the planned procedure code"
- Payer-specific rule modeling using LLMs trained on LCD/NCD texts and historical adjudication data
- Denial root-cause natural language query interface for HIM directors
- Revenue leakage AI: scan all charge capture events for missed billable items vs. documented services
- Autonomous appeal package generation from denial reason codes and supporting clinical documentation

## Legal & IP Summary

- **No GPL or AGPL tools** in this comparison that would create embedding concerns for a proprietary product.
- **AHRQ HCUP CCS**: Public domain US government data — free to embed with no IP restrictions.
- **CPT code set (AMA)**: The CPT code set is proprietary to the American Medical Association. Any software that displays, processes, or transmits CPT codes requires an AMA Data Use Licence. This is a significant and recurring IP cost. Annual licence fees are usage-based and can reach tens of thousands of dollars annually for high-volume platforms. This applies to every solution in this comparison.
- **ICD-10-CM / ICD-10-PCS**: Owned by CMS and NCHS (US government). Free to use in US-market products without a licence.
- **HCPCS Level II**: Maintained by CMS; publicly available and free to use.
- **MS-DRG and APC grouper logic**: CMS publishes the grouper logic annually; implementing it in software is permitted. Commercial grouper vendors (3M, Optum) sell high-performance pre-built implementations.
- **NCCI edits**: Published quarterly by CMS; free to use for billing compliance purposes.
- **Medicomp MEDCIN knowledge base**: Fully proprietary — requires a licence from Medicomp Systems to embed. Do not incorporate MEDCIN content without a signed agreement.
- Key strategic IP risk: any AI model trained on proprietary coding manuals, payer LCD/NCD documents, or commercial coding content (AAPC Codify, Optum360 annotations) must have appropriate content licences for training data use.

## Recommended Feature Scope

**Must-have (MVP)**:
- ICD-10-CM, CPT, and HCPCS code lookup with official CMS/AMA guidelines and cross-references
- NCCI edit validation and E/M complexity calculator
- AI NLP code suggestion from clinical note text: ICD-10-CM diagnosis and CPT procedure code recommendations with confidence scores
- X12 EDI 837 claim generation (professional and institutional) and ERA 835 parsing
- Denial tracking dashboard: denial reason code analysis by payer, code, and provider
- Audit trail with code assignment history for compliance review

**Should-have (v1.1)**:
- Autonomous coding for high-volume, lower-complexity specialties (emergency medicine, radiology, urgent care) with configurable human review threshold
- CDI feedback: real-time alerts when clinical documentation is insufficient to support the intended code set
- Payer-specific rule modeling beyond NCCI: AI model trained on LCD/NCD texts and historical denial data per payer
- Denial root-cause natural language query: HIM director asks in plain English, AI retrieves relevant encounters
- Appeal package auto-generation from denial reason code and supporting clinical documentation

**Nice-to-have (backlog)**:
- LLM reasoning chain output for every code assignment: full explainability for compliance auditors
- Learning loop: automatic claim scrubbing rule updates from each denial received, per payer, without manual rule configuration
- Population-scale coding compliance audit: AI review of 100% of encounters vs. current 1–5% manual sampling
- Risk adjustment / HCC optimization: identify suspected HCCs in unstructured notes across the patient population
- Natural language compliance query interface for compliance officers and HIM directors
