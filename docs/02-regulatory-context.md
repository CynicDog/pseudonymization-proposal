# 02 — Regulatory Context

## Korean PIPA (개인정보보호법)

The Personal Information Protection Act (PIPA) is Korea's primary data privacy law, enacted in 2011 and significantly amended in 2020. The 2020 amendment introduced a formal legal distinction between **pseudonymization (가명처리)** and **anonymization (익명처리)**, and created a legal basis for secondary use of pseudonymized data without additional consent — a critical enabler for data platform and ML use cases.

### Key Definitions

**가명처리 (Pseudonymization):** Processing personal information so it cannot be attributed to a specific individual without the use of additional information (the key or mapping). Pseudonymized data retains its original structure and can be re-identified with access to supplementary information.

**익명처리 (Anonymization):** Processing personal information so it can no longer be attributed to a specific individual at all, even with additional information. Anonymized data is entirely outside the scope of PIPA.

The practical implication: **pseudonymized data is still subject to PIPA** (the data controller remains accountable), but its use for statistics, scientific research, and public interest record-keeping is explicitly permitted without re-obtaining consent.

### 2020 Amendment — Permitted Secondary Uses

Under Article 28-2 of the amended PIPA, pseudonymized personal information may be processed without consent for:

1. Statistical purposes (including commercial statistics)
2. Scientific research (including industrial research)
3. Public interest record-keeping and archiving

Machine learning development and model training fall under category 2 (scientific/industrial research), making pseudonymized ML pipelines legally permissible under PIPA — provided proper pseudonymization safeguards are in place.


## PIPC Guidelines (개인정보보호위원회)

The Personal Information Protection Commission (PIPC) published the **"Comprehensive Guidelines on Processing Pseudonymized Data"** (가명정보 처리 가이드라인) in September 2020. These guidelines govern how organizations must implement pseudonymization in practice.

### Required Process

The PIPC guidelines mandate a structured process with four stages:

**Stage 1 — Preliminary Plan**
- Define the purpose of pseudonymization (e.g., ML model training, statistical analysis)
- Identify the scope: which data, which systems, which recipients
- Select the pseudonymization technique appropriate to the purpose and risk level
- Document the plan before processing begins

**Stage 2 — Risk Assessment**
- Evaluate re-identification risk: what is the probability that a motivated adversary could re-identify individuals using the pseudonymized dataset and publicly available information?
- Consider: dataset size, quasi-identifier combinations (e.g., age + region + job title), sensitivity of data categories
- Apply the minimum-use principle: pseudonymize only the fields strictly necessary for the stated purpose

**Stage 3 — Pseudonymization Execution**
- Apply the chosen technique under the documented plan
- Organizational accountability: designate a responsible party (equivalent to a DPO function) for the pseudonymization process
- Enforce technical controls: access restrictions on raw data, audit logging of all processing activities

**Stage 4 — Final Evaluation**
- Verify that the pseudonymization level achieved is adequate for the stated purpose
- Confirm that re-identification risk is within acceptable bounds
- Document the outcome and retain records for compliance audit

### Organizational Obligations

Organizations processing pseudonymized data must:
- Prevent re-identification: actively prohibit and technically prevent attempts to re-identify pseudonymized data
- Control third-party provision: if pseudonymized data is shared with a third party, additional safeguards and contractual obligations apply
- Maintain audit records: all stages of the pseudonymization process must be documented and retained


## KISA and K-ISMS

The Korea Internet & Security Agency (KISA, 한국인터넷진흥원) operates under the Ministry of Science and ICT. KISA:

- Sets and publishes technical standards for personal information security
- Administers the K-ISMS-P (Information Security Management System — Personal Information) certification framework
- Has published analysis reports on international de-identification practices (2016) and technical guidance on pseudonymization methods

K-ISMS-P certification is the leading Korean information security certification and is widely expected by regulators, enterprise clients, and partners. The pseudonymization controls described in this proposal align with K-ISMS-P control requirements for personal information processing.


## Compliance Posture

Unlike EU GDPR's prescriptive Recital 26 guidance, **PIPA does not mandate specific cryptographic algorithms**. Compliance is purpose-driven and risk-based:

- The organization selects a technique and documents why it is appropriate for the assessed risk level
- The PIPC evaluates whether the pseudonymization adequately prevents re-identification, not whether a specific algorithm was used
- This gives flexibility to adopt industry-standard techniques (FF1, HMAC-SHA-256) while the compliance obligation is met through documentation, risk assessment, and organizational controls

### What This Means for Implementation

| Compliance Requirement | Implementation Obligation |
|---|---|
| Preliminary plan | Document purpose, scope, technique before processing |
| Risk assessment | Quantify re-identification risk per data category; update when data changes |
| Minimum-use principle | Only pseudonymize fields actually needed downstream |
| Organizational accountability | Assign DPO-equivalent role; document chain of responsibility |
| Audit trail | Immutable logs: who triggered pseudonymization, when, on which dataset |
| Re-identification prevention | Raw zone break-glass access only; keys isolated in key management service |


## GDPR Alignment Note

While this company operates primarily under Korean law, international data exchanges (e.g., reinsurance data, cross-border cloud processing) may trigger GDPR applicability for data subjects in EU/EEA member states. GDPR Recital 26 and Article 4(5) define pseudonymization in terms compatible with PIPA's 가명처리 concept. The technical controls proposed in this document — FF1 with separate key management, HMAC-SHA-256, isolated key store — satisfy GDPR Article 32's requirement for "appropriate technical measures" and are consistent with EDPB guidance on pseudonymization.

No additional GDPR-specific architecture changes are required if the key management controls in [08-key-management.md](08-key-management.md) are implemented as specified.
