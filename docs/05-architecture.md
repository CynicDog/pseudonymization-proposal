# 05 — Architecture

## Design Principles

1. **Pseudonymize once, use everywhere.** Pseudonymization happens exactly once, at the ingestion boundary. All downstream systems consume only pseudonymized data.
2. **Keys never travel with data.** Encryption keys live exclusively in Azure Key Vault, accessed at runtime via Managed Identity. No key material appears in pipeline configuration, environment variables, or data files.
3. **Raw data is break-glass only.** The Raw Zone is accessible only via audited, approved break-glass procedures. Normal data engineering and ML workflows operate exclusively on the Pseudonymized Zone.
4. **Determinism by default.** All production pseudonymization is deterministic (same input + same key → same output) to preserve referential integrity for ML JOIN operations.
5. **Audit everything.** Every access to the Raw Zone and every Key Vault key retrieval is logged immutably in Azure Monitor.


## Three-Zone Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  ZONE 1 — SOURCE LAYER (On-Premises)                            │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │  Operational │    │  Relational  │    │  Other       │       │
│  │  DB          │    │  DB (MSSQL,  │    │  Sources     │       │
│  │              │    │  Oracle, etc)│    │              │       │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘       │
│         └────────────────────┴────────────────────┘              │
│                              │                                   │
│                    ADF Self-Hosted Integration                   │
│                    Runtime (SHIR)                                │
│                    TLS 1.2+ encrypted transit                    │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  ZONE 2 — PSEUDONYMIZATION LAYER (Azure)                        │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Azure Data Lake Storage — Raw Zone                        │  │
│  │  • Encrypted at rest (Azure Storage Service Encryption)    │  │
│  │  • Network: private endpoint only                          │  │
│  │  • Access: break-glass + ADF Managed Identity only         │  │
│  │  • All access logged to Azure Monitor                      │  │
│  └────────────────────────┬───────────────────────────────────┘  │
│                           │                                      │
│                    ADF Pipeline triggers                         │
│                    pseudonymization job                          │
│                           │                                      │
│  ┌────────────────────────▼───────────────────────────────────┐  │
│  │  Pseudonymization Job (Azure Functions or Databricks NB)   │  │
│  │                                                            │  │
│  │  1. Presidio Analyzer                                      │  │
│  │     └─ Scan columns → identify PII entities + types        │  │
│  │                                                            │  │
│  │  2. Technique Dispatcher                                   │  │
│  │     ├─ General PII fields    → FF1 (general key NS)        │  │
│  │     ├─ Sensitive PII fields  → FF1 (sensitive key NS)      │  │
│  │     ├─ Referential keys      → HMAC-SHA-256                │  │
│  │     └─ Free-text fields      → Presidio redact / masking   │  │
│  │                                                            │  │
│  │  3. Key Retrieval                                          │  │
│  │     └─ Azure Key Vault (Managed Identity, no hardcoding)   │  │
│  │                                                            │  │
│  │  4. Polars / PyArrow Transform                             │  │
│  │     └─ Columnar in-memory processing; Parquet output       │  │
│  └────────────────────────┬───────────────────────────────────┘  │
│                           │                                      │
│  ┌────────────────────────▼───────────────────────────────────┐  │
│  │  Azure Data Lake Storage — Pseudonymized Zone              │  │
│  │  • Parquet / Delta format                                  │  │
│  │  • Encrypted at rest                                       │  │
│  │  • Access: data engineers, ML engineers (RBAC)             │  │
│  │  • Sensitivity labels via Microsoft Purview                │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  ZONE 3 — CONSUMPTION LAYER (Azure)                             │
│                                                                  │
│  ┌──────────────────┐  ┌───────────────────┐  ┌─────────────┐  │
│  │  Databricks      │  │  ML Inference     │  │  Analytics  │  │
│  │  (Feature Eng.   │  │  Endpoint         │  │  / BI       │  │
│  │   + ML Training) │  │  (Pseudonymized   │  │  (Aggregate │  │
│  │                  │  │   features in/out)│  │   views)    │  │
│  └──────────────────┘  └───────────────────┘  └─────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Application Layer (Authorized re-identification only)     │  │
│  │  • Separate RBAC scope                                     │  │
│  │  • Pseudonym → identity translation for authorized outputs │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```


## Supporting Services

```
┌──────────────────────────────────────────────────────────┐
│  Azure Key Vault                                          │
│  • FF1 keys (general tier, sensitive tier)               │
│  • HMAC signing key                                      │
│  • AES-SIV key (unstructured fields)                     │
│  • Access: Managed Identity only (ADF, Databricks, Fn)   │
│  • Audit: all key access → Azure Monitor                 │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  Microsoft Purview                                        │
│  • Sensitivity labels on all ADLS assets                 │
│  • Data lineage: on-prem → Raw → Pseudo → ML             │
│  • Column-level classification metadata                  │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  Azure Monitor + Log Analytics                            │
│  • Raw Zone access log (immutable)                       │
│  • Key Vault access log (immutable)                      │
│  • Pipeline execution log (success/failure, row counts)  │
│  • Alert rules: unexpected raw zone access, key errors   │
└──────────────────────────────────────────────────────────┘
```


## Data Flow Detail

### Ingestion Flow (On-Prem → Raw Zone)

1. ADF pipeline scheduled or event-triggered
2. SHIR connects to on-prem DB using encrypted connection (TLS 1.2+, credential in ADF Linked Service / Key Vault reference)
3. Data written to Raw Zone in ADLS as Parquet (snappy compression)
4. ADF activity completion triggers pseudonymization job

### Pseudonymization Flow (Raw Zone → Pseudonymized Zone)

1. Pseudonymization job reads raw Parquet from ADLS using Managed Identity
2. Presidio Analyzer scans a sample of each column (or uses the pre-built classification inventory) to confirm entity types
3. Technique Dispatcher applies per-column technique based on classification:
   - FF1 with appropriate key namespace (fetched from Key Vault)
   - HMAC-SHA-256 with signing key (fetched from Key Vault)
   - Presidio redaction for free-text
4. Polars transform executes column-by-column in memory
5. Output written as Parquet to Pseudonymized Zone; column metadata (classification tier, technique applied, key version) embedded in Parquet schema metadata
6. Pipeline completion event logged to Azure Monitor; Purview lineage updated

### Consumption Flow (Pseudonymized Zone → ML)

1. Databricks reads pseudonymized Parquet from ADLS
2. Feature engineering uses pseudonymized entity IDs as JOIN keys (deterministic pseudonyms enable cross-table joins)
3. ML model trained on pseudonymized features; no PII in model inputs, weights, or logs
4. Inference endpoint receives pseudonymized feature vector; returns prediction attached to pseudonym
5. Application layer (if authorized re-identification is required for output) performs pseudonym → identity translation with separate RBAC and audit logging


## Network and Access Control

| Zone / Service | Network Boundary | Access Principal | Notes |
|---|---|---|---|
| On-prem DB | Private network | ADF SHIR service account | Firewall rules: SHIR IP only |
| ADLS Raw Zone | Private endpoint | ADF Managed Identity | No public endpoint |
| ADLS Pseudonymized Zone | Private endpoint | ADF MI, Databricks MI, ML Endpoint MI | No public endpoint |
| Azure Key Vault | Private endpoint | ADF MI, Databricks MI, Functions MI | No human access in prod |
| Databricks | VNet injection | Data engineers, ML engineers | IP allowlist + AAD auth |
| ML Inference Endpoint | Private or public (TLS) | Application services | JWT/OAuth2 auth |


## Scalability Considerations

For the current data scale (MB, occasionally GB):

- The pseudonymization job runs on a single Azure Functions Premium instance or a Databricks single-node cluster (not a full Spark cluster). Polars handles GB-scale data on a single node efficiently.
- If data volume grows toward 10+ GB per batch, Polars streaming mode (`scan_parquet` + lazy evaluation) processes data without loading the full file into memory.
- If data volume reaches consistent multi-GB or TB scale, the pseudonymization logic can be ported to Databricks PySpark with the same Presidio + FF1/HMAC approach — the technique design does not change, only the execution engine.

The proposed architecture is designed to scale out without architectural change. The pseudonymization layer is stateless (no vault), so horizontal scaling requires only adding compute, not distributed vault coordination.
