# 09 — Implementation Roadmap

This document is designed to serve as a **sprint backlog** for the implementation repository. Each phase produces a concrete, testable deliverable. Tasks are ordered by dependency; items within a phase can be parallelized.


## Phase 1 — Foundation (Weeks 1–4)

**Deliverable:** A working local pseudonymization module that correctly applies FF1/HMAC to sample data, with Key Vault integration and Presidio Korean recognizers operational.

### 1.1 Data Classification Inventory
- [ ] Enumerate all source tables in scope (on-prem databases to be ingested via ADF)
- [ ] For each table, map each column to a classification tier (PI-class, SPII-class, or non-sensitive) using the company's PII/SPII definition taxonomy
- [ ] Identify entity join keys that must be consistently pseudonymized across tables (e.g., `customer_id`, `policy_number`)
- [ ] Produce a classification manifest (YAML or JSON): `{table}.{column} → {tier, technique, tweak, reversible}`
- [ ] Review manifest with data owners and legal/compliance stakeholder

### 1.2 Azure Key Vault Setup
- [ ] Provision Key Vault per environment (dev, staging, prod) with private endpoint
- [ ] Enable soft delete + purge protection on all Key Vaults
- [ ] Configure diagnostic settings: Key Vault audit logs → Log Analytics Workspace
- [ ] Generate initial key versions: `kv-ff1-general-v1`, `kv-ff1-sensitive-v1`, `kv-hmac-ref-v1`, `kv-aes-siv-v1`
- [ ] Configure RBAC: Managed Identity for ADF and pseudonymization compute; no human secret-value access in prod
- [ ] Set up quarterly rotation policy for all keys

### 1.3 Presidio Korean PII Recognizers
- [ ] Install Presidio Analyzer + Anonymizer + spaCy English model (baseline)
- [ ] Implement custom `PatternRecognizer` for 주민등록번호 (RRN): regex + checksum validator
- [ ] Implement custom `PatternRecognizer` for 사업자등록번호
- [ ] Implement custom `PatternRecognizer` for Korean mobile phone (010-XXXX-XXXX format)
- [ ] Implement column-name heuristic recognizer for Korean name fields (`*_이름`, `*_성명`)
- [ ] Integrate `kiwipiepy` as NLP engine for Korean free-text NER
- [ ] Unit test each recognizer against positive and negative examples
- [ ] Register custom recognizers in `RecognizerRegistry` and validate with sample Korean PII data

### 1.4 Base Polars Pseudonymization Module
- [ ] Implement FF1 from NIST SP 800-38G using `cryptography` library's AES ECB primitive
  - [ ] Feistel network: 10 rounds, configurable radix and tweak
  - [ ] Encrypt and decrypt operations
  - [ ] Validate implementation against all NIST SP 800-38G Appendix B test vectors before use
  - [ ] Handle: numeric-only strings, alphanumeric strings, format restoration (hyphens, separators) after encryption
- [ ] Implement HMAC-SHA-256 column transform using Python stdlib (`hmac` + `hashlib`)
  - [ ] Accept: Polars Series, HMAC key bytes
  - [ ] Return: Polars Series (hex-encoded HMAC digest, 64 chars)
- [ ] Implement SDC transforms for numerical fields (NumPy + Polars built-ins)
  - [ ] Top/bottom coding: `clip(lower, upper)` with configurable percentile thresholds per field
  - [ ] Rounding: integer-division rounding to configured unit (e.g., 10M KRW for income)
  - [ ] Micro-aggregation: sort → contiguous groups of k → replace with group mean (configurable k ≥ 5)
  - [ ] Noise addition (Laplace): for fields where DP noise is appropriate; parameterized by global sensitivity and ε
- [ ] Implement Presidio `StructuredEngine` column detection (regex + PatternRecognizer; no NER model)
  - [ ] Accept: Polars DataFrame
  - [ ] Return: per-column entity type annotations
  - [ ] No spaCy NER model required for 100% tabular structured data
- [ ] Implement technique dispatcher: reads classification manifest, routes each column to correct transform
- [ ] Implement key store client wrapper: `get_key(key_name, version=None)` supporting both HashiCorp Vault and Azure Key Vault
- [ ] Write Parquet output with pseudonymization metadata in schema custom metadata
- [ ] Unit tests: round-trip FF1 against NIST vectors, HMAC determinism, SDC transforms preserve expected statistical properties
- [ ] Integration test with sample dataset from staging source

### 1.5 ADLS Zone Setup
- [ ] Provision ADLS Gen2 storage account with two containers: `raw` and `pseudonymized`
- [ ] Configure network: private endpoint, no public blob access
- [ ] Configure RBAC:
  - `raw`: ADF Managed Identity (write), pseudonymization compute MI (read); no other access
  - `pseudonymized`: data engineer AAD group (read/write), Databricks MI (read), ML endpoint MI (read)
- [ ] Enable diagnostic logging: ADLS access logs → Log Analytics Workspace
- [ ] Apply Microsoft Purview sensitivity labels to both containers


## Phase 2 — Pipeline Integration (Weeks 5–8)

**Deliverable:** End-to-end ADF pipeline running pseudonymization on real data with audit logging; ML feature engineering pipeline validated on pseudonymized data.

### 2.1 ADF Pipeline — Ingestion to Raw Zone
- [ ] Create ADF Linked Service for on-prem source DB via SHIR (TLS 1.2+; credential in Key Vault)
- [ ] Create ADF Dataset for each source table
- [ ] Create Copy Activity: source table → ADLS raw container (Parquet, snappy compression)
- [ ] Configure trigger (schedule or event-based)
- [ ] Add audit logging activity: log pipeline run metadata to Log Analytics on success and failure

### 2.2 ADF Pipeline — Pseudonymization Job Trigger
- [ ] Create ADF activity to invoke pseudonymization job after Copy Activity succeeds
  - Option A: Azure Functions (recommended for MB-scale; lightweight, fast cold start with Premium plan)
  - Option B: Databricks Notebook (acceptable; adds Databricks cluster startup latency)
- [ ] Pass parameters: source file path, target file path, classification manifest version
- [ ] Configure retry: 2 retries with 5-minute backoff on transient failures (Key Vault timeout, ADLS throttle)

### 2.3 Pseudonymization Job — Production Deployment
- [ ] Package pseudonymization module as Azure Functions app (Python 3.11, Premium EP1 plan)
- [ ] Configure Managed Identity on the Functions app; grant Key Vault secret reader role
- [ ] Configure VNet integration (private endpoint access to ADLS and Key Vault)
- [ ] Implement job entrypoint: read source path from trigger payload, run pseudonymization, write to pseudonymized path, emit audit log
- [ ] Load test: verify MB-scale files complete in < 60 seconds; GB-scale in < 10 minutes (Polars streaming mode)
- [ ] Smoke test with real staging data: verify pseudonymized output is non-identifiable to unaided human review

### 2.4 Audit Logging
- [ ] Implement `PseudonymizationAuditEvent` schema (see [08-key-management.md](08-key-management.md) for schema)
- [ ] Emit audit event to Azure Monitor via OpenTelemetry SDK on every job completion (success and failure)
- [ ] Configure Log Analytics alert: notify on job failure, on unexpected raw zone access
- [ ] Verify Key Vault audit logs are flowing to Log Analytics
- [ ] Verify ADLS access logs are flowing to Log Analytics
- [ ] Retention policy: 90-day active + archive for 1 year (PIPA audit requirement)

### 2.5 Referential Integrity Validation
- [ ] Write test harness: pseudonymize three related tables (e.g., customer, policy, claim) and verify that JOIN on pseudonymized key produces the same result as JOIN on raw key
- [ ] Verify determinism: run pseudonymization job twice on the same source file; confirm byte-identical Parquet output
- [ ] Verify cross-environment consistency: same source + same key → same pseudonym in dev and staging

### 2.6 ML Feature Engineering Pipeline Validation
- [ ] Load pseudonymized tables into Databricks
- [ ] Run existing feature engineering notebook against pseudonymized data (swap raw table references → pseudonymized table references)
- [ ] Verify all JOINs succeed
- [ ] Run a simple baseline ML model on pseudonymized features; compare AUC/F1 to baseline trained on raw data — expect zero meaningful difference (bijective transform preserves all statistical structure)
- [ ] Confirm: no raw PII column appears in the final feature DataFrame


## Phase 3 — Hardening & Compliance (Weeks 9–12)

**Deliverable:** PIPA-compliant pseudonymization layer in production; risk assessment documented; security validated; stakeholder sign-off.

### 3.1 PIPA Risk Assessment Documentation
- [ ] Produce Preliminary Plan document (목적, 대상, 가명처리 방법, 위험 수준)
- [ ] Conduct re-identification risk assessment:
  - Entropy analysis: for each pseudonymized field, estimate bits of entropy in the pseudonym space
  - Quasi-identifier analysis: identify combinations of non-pseudonymized fields that could narrow re-identification (e.g., timestamp + region + gender)
  - Singling-out risk: can any individual be uniquely identified from the pseudonymized dataset alone?
- [ ] Document risk mitigation: FF1 with AES-256 key; key in Key Vault with MI access only; entropy ≫ attacker's computational budget
- [ ] Minimum-use principle review: confirm only columns flagged in the classification manifest are retained in the pseudonymized zone; others are dropped

### 3.2 Key Rotation Automation Test
- [ ] Execute a dry-run key rotation in staging:
  - Generate v2 key; update pipeline to use v2; run pseudonymization on staging data
  - Verify new output is different from v1 output (different key → different pseudonym)
  - Verify v1 data remains readable by specifying `version=v1` in Key Vault client
  - Run referential integrity test on v2-pseudonymized data

### 3.3 Access Control Audit
- [ ] Review RBAC on all Key Vaults: confirm no human has secret-value access in prod
- [ ] Review RBAC on ADLS raw container: confirm only ADF MI and pseudonymization compute MI have access
- [ ] Review network security: confirm all services accessed via private endpoints; no public exposure
- [ ] Simulate unauthorized access attempt: verify access is denied and alert fires in Log Analytics

### 3.4 Re-identification Risk Quantification
- [ ] Run `anonymeter` (CNIL-recognized privacy risk framework) against the pseudonymized dataset:
  - [ ] Singling-out attack: can an attacker uniquely identify an individual record?
  - [ ] Linkability attack: can records of the same individual be linked across two pseudonymized datasets?
  - [ ] Inference attack: can sensitive attribute values (e.g., claim amount, income tier) be inferred from non-sensitive fields?
- [ ] Document `anonymeter` output as part of PIPA risk assessment (re-identification probability per attack type)
- [ ] Optionally: engage internal security team or external third party for manual re-identification attempt using the pseudonymized dataset + public sources
- [ ] Remediate any identified weaknesses (typically: increase k for micro-aggregation, tighten top-code threshold, or add noise to residual high-entropy fields)

### 3.5 ML Inference Pipeline Validation
- [ ] Deploy a test inference endpoint consuming pseudonymized features
- [ ] Verify: inference logs contain no raw PII
- [ ] Verify: model weights (MLflow artifact) contain no recoverable PII (use adversarial ML membership inference attack as test — e.g., via `ml_privacy_meter` library)
- [ ] Verify: application layer re-identification translation is RBAC-gated and audit-logged

### 3.6 Stakeholder Review and Sign-off
- [ ] Present PIPA compliance documentation to legal / DPO function
- [ ] Present security architecture to information security team
- [ ] Data engineering sign-off: pipeline SLA and operational runbook accepted
- [ ] ML engineering sign-off: feature store and inference pipeline validated
- [ ] Production go-live approval


## Operational Runbook Outline (Post-Launch)

The implementation team should produce the following operational documents alongside the above roadmap:

- **Incident response playbook:** suspected key compromise, unexpected raw zone access, pipeline failure affecting pseudonymized data
- **Key rotation runbook:** step-by-step quarterly rotation procedure with verification checklist
- **Onboarding new source tables:** how to add a new on-prem table to the pseudonymization pipeline (classification manifest update → ADF pipeline update → test run → go-live)
- **Re-pseudonymization procedure:** how to re-pseudonymize historical data after a key rotation
- **PIPA audit response pack:** pre-assembled documents (risk assessment, classification manifest, audit logs) for a PIPC inquiry
