# Medical Billing & Coding Assistant

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for ICD-10/CPT code suggestion, claim scrubbing, and denial management across the full medical revenue cycle.

The Medical Billing & Coding Assistant brings autonomous and assistive AI coding to provider organisations of every size — from solo physician groups to academic health systems. It targets the persistent pain of manual coding labour, denial backlogs, and payer-specific claim rejections that incumbent enterprise platforms only partially address. The goal is a transparent, standards-aligned alternative to the proprietary CAC, autonomous coding, and clearinghouse stacks that dominate the market today.

---

## Why Medical Billing & Coding Assistant?

- Incumbent CAC platforms such as 3M / Solventum and Optum360 EncoderPro are priced for large enterprises ($5,000–$50,000+/year), excluding small practices and physician groups from modern coding automation.
- Autonomous coding leaders such as Nym Health and Fathom Health operate as enterprise-only black boxes with limited explainability, making compliance review and customer-side model tuning difficult.
- Major clearinghouses are concentrated and fragile — the February 2024 Change Healthcare ransomware attack disrupted US billing for weeks and accelerated demand for diversified, transparent infrastructure.
- Optum-owned tooling raises trust and data-sharing concerns for providers, since Optum is also a major payer competitor.
- Existing reference tools (Codify by AAPC, Find-A-Code) are manual-only and offer no NLP-assisted coding, leaving high-volume coders without modern automation at an accessible price point.

---

## Key Features

### Coding Reference and Validation

- ICD-10-CM, ICD-10-PCS, CPT, and HCPCS code lookup with official CMS/AMA guidelines and cross-references
- E/M complexity calculator aligned to 2021 guidelines
- NCCI edit checking to prevent unbundling and inappropriate code combinations
- LCD / NCD policy lookup linked to relevant procedure codes
- MS-DRG and APC grouper integration for inpatient and outpatient reimbursement validation

### AI-Assisted and Autonomous Coding

- NLP-driven code suggestion from clinical note text with confidence scores for ICD-10-CM diagnosis and CPT procedure codes
- Autonomous coding workflow for high-volume, lower-complexity specialties (emergency medicine, radiology, urgent care) with configurable human review thresholds
- LLM reasoning chain output for every code assignment to support compliance audit and explainability
- Exception queue routing low-confidence encounters to human coder review

### Clinical Documentation Improvement (CDI)

- Real-time CDI alerts during physician note-writing when documentation is insufficient to support the intended code set
- Query workflow for documentation clarification between coders and clinicians
- Risk adjustment / HCC coding identification across unstructured notes
- Charge capture optimisation surfacing billable services missed in manual coding

### Claim Scrubbing and Denial Management

- X12 EDI 837P / 837I claim generation and ERA 835 parsing for payment reconciliation
- Payer-specific rule modelling beyond NCCI, trained on LCD/NCD texts and historical denial data
- Denial root-cause analytics across payer, code, provider, and diagnosis dimensions
- Natural language compliance query interface (e.g. "show all encounters where high-complexity E/M was billed but documentation supports only moderate complexity")
- Automated appeal package generation from denial reason codes and supporting clinical documentation

### Compliance, Audit, and Analytics

- Audit trail of code assignments with full assignment history for compliance review
- Population-scale coding compliance audit covering 100% of encounters rather than 1–5% manual sampling
- Coder performance and productivity dashboards
- Closed-loop reporting on automation rate, accuracy, denial rate, and financial impact

---

## AI-Native Advantage

Modern LLMs can read full clinical encounter notes and emit complete, compliant ICD-10-CM/PCS and CPT code sets with reasoning chains, enabling near-fully automated coding for routine encounter types. AI also makes payer-specific rule modelling tractable by learning each payer's proprietary LCD/NCD adjudication patterns that rule-based scrubbers miss, and supports denial learning loops that improve scrubbing rules automatically from each denial received. Embedded in the EHR at the point of care, AI prevents downstream coding failures by alerting physicians to documentation gaps before notes are signed. Natural-language compliance querying replaces manual audit sampling with comprehensive, AI-driven review.

---

## Tech Stack & Deployment

The platform is designed around standard healthcare interoperability protocols: HL7 v2.x for ADT and clinical document feeds, FHIR APIs for modern EHR connectivity, and the full X12 EDI suite (837, 835, 270/271, 276/277) for claim, remittance, eligibility, and status transactions. Standard CMS code sets (ICD-10-CM, ICD-10-PCS, HCPCS Level II) and grouper logic (MS-DRG, APC) are free to incorporate; CPT integration requires an AMA Data Use Licence. AHRQ HCUP CCS, a public-domain US government classification, is safe to embed as an analytics layer. Deployment targets include self-hosted, cloud, and hybrid models so providers can balance data residency, security, and integration with existing EHR and clearinghouse vendors.

---

## Market Context

The global AI in medical billing market was valued at USD 4.70 billion in 2025 and is projected to reach approximately USD 45.38 billion by 2035, growing at a 25.44% CAGR (Precedence Research, 2025). Incumbent pricing spans $20–$200/user/month for reference tools, $5,000–$50,000+/year for enterprise CAC, per-claim fees ($0.05–$0.30) for clearinghouse scrubbing, and custom contracts (often a percentage of recovered revenue) for autonomous AI coding. Primary buyers are revenue cycle directors, HIM directors, physician group billing managers, CDI specialists, and payer audit teams.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
