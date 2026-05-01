# Medical Billing & Coding Assistant

> Candidate #103 · Researched: 2026-05-01

## Existing Products and Software Packages

| Name | Description | Model | Pricing |
|------|-------------|-------|---------|
| 3M Encoder (Solventum) | Long-established computer-assisted coding (CAC) tool for ICD-10-CM/PCS and CPT; used in large health systems. NLP-based code suggestions from clinical notes. | Enterprise SaaS | Custom enterprise pricing |
| Optum360 Codify / Encoder Pro | Comprehensive coding reference and encoder from Optum (UnitedHealth subsidiary); includes ICD, CPT, HCPCS, and compliance tools. | SaaS | ~$300–$500/user/year (reference); encoder custom |
| CureMD AI Medical Coding | AI-assisted coding and billing with NLP-driven ICD/CPT suggestions, claim scrubbing, and denial management. | SaaS | Custom per-provider |
| Nym Health | Autonomous medical coding platform using clinical NLP; targets revenue cycle automation for health systems. | SaaS | Custom enterprise |
| Fathom Health | AI coding platform using LLMs to autonomously assign codes from clinical documentation; targets high-volume specialties. | SaaS | Custom enterprise; % of revenue cycle savings |
| XpertDox | AI medical coding with NLP-driven code suggestion, query management, and CDI (clinical documentation improvement). | SaaS | Custom |
| Codify by AAPC | Web-based coding reference with ICD-10, CPT, HCPCS lookup, E/M calculator, NCCI edits, and APC/DRG tools. Used primarily for manual coder reference. | SaaS | ~$49–$199/user/month |
| Find-A-Code | Online ICD-10, CPT, HCPCS encoder and reference tool; used by coders and billers. | SaaS | ~$20–$100/month |
| Change Healthcare (Optum) ClaimLogic | Claim scrubbing and editing clearinghouse; checks claims against payer rules and NCCI edits before submission. | SaaS / Clearinghouse | Per-claim transaction fees |
| Waystar | Revenue cycle platform with AI-powered claim scrubbing, eligibility, denial management, and analytics. | SaaS | % of collections or custom |
| OpenEvidence Coding Intelligence | AI coding feature for physicians to suggest and validate billing codes at point of care; launched 2024. | SaaS | Custom |
| AHRQ HCUP CCS (Clinical Classifications Software) | Free government tool for grouping ICD-10 diagnosis and procedure codes into clinically meaningful categories. Used for analytics, not billing. | Open Source / Free | Free |

## Relevant Industry Standards or Protocols

| Standard | Relevance |
|----------|-----------|
| ICD-10-CM (International Classification of Diseases, 10th Revision, Clinical Modification) | US diagnosis coding standard mandated by CMS; updated annually each October; core to all medical billing |
| ICD-10-PCS (Procedure Coding System) | US inpatient procedure coding standard used alongside ICD-10-CM for hospital facility billing |
| CPT (Current Procedural Terminology) | AMA-owned procedure code set used for physician and outpatient service billing; updated annually |
| HCPCS Level II (Healthcare Common Procedure Coding System) | CMS code set for durable medical equipment, drugs, ambulance services, and other items not in CPT |
| MS-DRG (Medicare Severity Diagnosis Related Groups) | CMS grouping system for inpatient payment based on ICD-10 codes; determines hospital reimbursement under IPPS |
| APC (Ambulatory Payment Classifications) | CMS outpatient prospective payment groupings based on CPT/HCPCS codes |
| NCCI (National Correct Coding Initiative) Edits | CMS rules preventing unbundling and inappropriate code combinations; enforced in claim scrubbing |
| X12 EDI 837P / 837I | HIPAA claim transaction formats for professional and institutional claims respectively |
| X12 EDI 835 | Electronic remittance advice for payment reconciliation and denial identification |
| NUBC UB-04 / CMS-1500 | Paper and electronic claim form standards for institutional and professional billing respectively |
| HIPAA Transactions and Code Set Rules (45 CFR Part 162) | Federal mandate for using standard code sets and transaction formats in all electronic healthcare billing |

## Available Research Materials

| Citation | Type |
|----------|------|
| American Medical Association. (2025). *CPT® 2025 Professional Edition*. AMA Press. | Code Set / Reference Book |
| CMS. (2025). *ICD-10-CM Official Guidelines for Coding and Reporting FY2025*. Centers for Medicare & Medicaid Services. https://www.cms.gov/medicare/coding-billing/icd-10-codes | Official Guidelines |
| Climent-Ballester, S., et al. (2025). Artificial intelligence to improve clinical coding practice in Scandinavia: Crossover randomized controlled trial. *Journal of Medical Internet Research*, 27, e71904. https://www.jmir.org/2025/1/e71904 | Randomized Controlled Trial |
| Patel, J., et al. (2024). Current applications of artificial intelligence in billing practices and clinical plastic surgery. *PMC / Aesthetic Surgery Journal*. https://pmc.ncbi.nlm.nih.gov/articles/PMC11216662/ | Peer-Reviewed Article |
| HIMSS. (2023). Reshaping the healthcare industry with AI-driven deep learning model in medical coding. HIMSS Resources. https://www.himss.org/resources/reshaping-healthcare-industry-ai-driven-deep-learning-model-medical-coding/ | Industry Report |
| Precedence Research. (2025). *AI in Medical Billing Market – Size, Share & Forecast to 2035*. https://www.precedenceresearch.com/press-release/ai-in-medical-billing-market | Market Report |
| AHIMA. (2023). *Clinical Documentation Integrity and Coding Compliance Study*. American Health Information Management Association. [Referenced widely; full report available to AHIMA members] | Industry Survey |
| AKASA / HFMA. (2023). *AI in Revenue Cycle Management Survey*. Healthcare Financial Management Association. [Referenced in industry sources; report available via HFMA] | Industry Survey |

## Market Research

**Market Size:** The global AI in medical billing market was valued at USD 4.70 billion in 2025 and is projected to reach approximately USD 45.38 billion by 2035, growing at a CAGR of 25.44% (Precedence Research, 2025). The broader medical coding and billing software market (including non-AI tools) is significantly larger.

**Pricing Landscape:**

| Segment | Typical Price |
|---------|--------------|
| Coding reference tools (Codify, Find-A-Code) | $20–$200/user/month |
| Computer-assisted coding (3M, Optum Encoder) | $5,000–$50,000+/yr enterprise |
| Autonomous AI coding platforms (Nym, Fathom) | Custom; often % of recovered revenue or FTE savings |
| Clearinghouse claim scrubbing (Change Healthcare, Waystar) | Per-claim ($0.05–$0.30/claim) or % of claims volume |

**Key Buyer Personas:**
- Revenue cycle directors at hospitals and health systems seeking to reduce coding backlogs and FTE costs
- Health information management (HIM) directors and coding supervisors managing accuracy and compliance
- Physician group billing managers dealing with high denial rates from payers
- Clinical documentation improvement (CDI) specialists focused on code capture completeness
- Payers and managed care organizations auditing claims for over/undercoding

**Notable Funding / Acquisitions:**
- Optum (UnitedHealth Group) acquired Change Healthcare for approximately USD 13 billion (2022; completed after DOJ antitrust review).
- Waystar went public (IPO) on Nasdaq in June 2024, raising ~USD 968 million, valuing the company at ~USD 3.5 billion.
- Fathom Health raised USD 46 million Series B (2023) for autonomous AI coding expansion.
- Nym Health raised USD 25 million Series B (2022) for NLP-based autonomous coding.

## AI-Native Opportunity

- **End-to-end autonomous coding:** Modern LLMs can read full clinical encounter notes and output complete, compliant ICD-10-CM/PCS and CPT code sets with reasoning chains — enabling near-fully automated coding workflows for straightforward encounter types, with human review reserved for complex cases only.
- **Denial root-cause analysis and learning loops:** AI can analyze patterns in denied claims across payer, code, provider, and diagnosis dimensions to identify systemic documentation or process failures, then feed those insights back into coder training and front-end charge capture rules.
- **Payer-specific rule modeling:** Each payer has proprietary LCD/NCD (local/national coverage determination) rules that go beyond standard NCCI edits. AI can model and continuously update these payer-specific rules, predicting denials that rule-based claim scrubbers miss.
- **Clinical documentation improvement (CDI) at point of care:** AI embedded in the EHR can alert physicians during note-writing when documentation is insufficient to support the intended diagnosis or procedure code, preventing downstream coding failures before the claim is generated.
- **Natural language query for compliance auditing:** Compliance officers can ask questions like "show me all encounters where a high-complexity E/M was billed but the documentation supports only moderate complexity" in plain language, replacing manual audit sampling with comprehensive AI-driven review.
