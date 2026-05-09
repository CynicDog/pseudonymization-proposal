# 06 — Technology Stack

## Primary Stack

| Component | Tool / Library | Version | Role |
|---|---|---|---|
| Transform engine | Polars | 1.x | Columnar DataFrame processing for MB-GB datasets |
| Columnar format | PyArrow | 17.x | Arrow IPC, Parquet I/O, zero-copy interop |
| FPE encryption | `ff3` (PyPI) | Latest | FF1 format-preserving encryption (NIST SP 800-38G) |
| Column-level encryption | `py4phi` | Latest | AES column encryption; key management helper |
| Authenticated encryption | `cryptography` (PyPI) | 43.x | AES-SIV (RFC 5297) for unstructured fields |
| Keyed hash | `hmac` + `hashlib` | stdlib | HMAC-SHA-256 for referential keys (no dependency) |
| PII detection | Microsoft Presidio | 2.x | NER + regex PII detection; anonymization operators |
| Korean NLP | `kiwipiepy` or `konlpy` | Latest | Korean tokenizer/NER for Presidio custom recognizer |
| Key management | HashiCorp Vault (on-prem) or Azure Key Vault via private link | `hvac` / `azure-keyvault-secrets` 4.x | On-prem key retrieval; no key material leaves on-prem boundary |
| Azure auth (if AKV) | `azure-identity` | 1.x | Service principal or Managed Identity for AKV access over private link |
| ADLS I/O | `adlfs` | Latest | `fsspec`-compatible Azure Data Lake Storage driver |
| Data catalog | Microsoft Purview | Azure-managed | Sensitivity labels, lineage |
| Orchestration | Azure Data Factory | Azure-managed | Pipeline scheduling, SHIR, activity chaining |
| Audit logging | Azure Monitor SDK | `azure-monitor-opentelemetry` | Pipeline + key access event logging |
| ML runtime | Databricks Runtime | 14.x LTS | Feature engineering and ML training on pseudonymized data |


## Why Polars Instead of PySpark

The current stack uses PySpark on Synapse/Databricks. For the actual data volumes in production, PySpark introduces costs that outweigh its benefits:

| Factor | PySpark | Polars |
|---|---|---|
| Cluster startup time | 3–8 minutes | < 1 second |
| MB-scale throughput | Bottlenecked by driver overhead | ~45x faster than Pandas; handles MB in milliseconds |
| GB-scale throughput | Competitive; best for true distributed data | Single-node streaming; handles up to ~100GB on 32GB RAM node |
| Dependency management | JVM + Python + cluster config | Pure Python package; no JVM |
| Cost | Full cluster (driver + workers) per run | Single Functions instance or single Databricks node |
| Arrow/Parquet native | Via conversion layer | Native Arrow backend; zero-copy Parquet I/O |
| Spark-specific patterns needed | Yes (RDDs, Catalyst optimizer) | No; standard Python data engineering |

**Decision:** Use Polars for the pseudonymization step, deployed on-prem as a standalone Python service or containerized process. This eliminates the need for cloud compute (Databricks, Azure Functions) in the pseudonymization path, keeping the service lightweight and independently deployable within the on-prem security boundary. Databricks remains as the ML training and feature engineering platform. If data volume growth reaches consistent multi-GB or TB scale, the FF1/HMAC pseudonymization logic is portable to PySpark UDFs without changing the technique, key management design, or classification manifest schema.


## Polars Pseudonymization Patterns

The following describes the architecture of the pseudonymization module (not implementation — see the implementation repository).

### Lazy Evaluation for Memory Efficiency

Polars supports lazy query execution (`LazyFrame`) where the computation plan is optimized before any data is loaded. For pseudonymization:

- Read raw Parquet lazily from ADLS (`pl.scan_parquet`)
- Apply column expressions as a declarative plan
- Execute with `.collect()` — Polars optimizes the execution plan, pushing column selection down before any transform

For files approaching GB scale, use `.sink_parquet()` (streaming sink) instead of `.collect()` to avoid materializing the full output in memory.

### Expression-Based Column Transforms

Pseudonymization transforms should be applied as Polars expressions using `map_elements` or a batched custom plugin (via the Polars plugin API in Rust for performance-critical paths). The implementation should:

- Fetch keys once per job run from Key Vault (not per-row)
- Apply FF1 or HMAC as a vectorized column operation, not row-by-row Python iteration
- Use `dtype=pl.Utf8` (String) for all pseudonymized outputs regardless of input type (pseudonyms are always string tokens)

### Parquet Schema Metadata

Write pseudonymization provenance into Parquet custom metadata (accessible via `pyarrow.parquet.read_schema`):

```
schema_metadata = {
    "pseudo_version": "1.0",
    "pseudo_date": "<ISO8601 timestamp>",
    "columns": {
        "customer_id": {"technique": "FF1", "key_version": "kv-pseudo-general-v2"},
        "email": {"technique": "HMAC-SHA-256", "key_version": "kv-hmac-ref-v1"},
        ...
    }
}
```

This metadata enables downstream systems to know which columns are pseudonymized, which technique was applied, and which key version was used — essential for re-pseudonymization during key rotation.


## Korean PII Patterns for Presidio

Standard Presidio ships with recognizers for English and global PII (email, phone, IP, etc.) but requires custom recognizers for Korean-specific patterns. The implementation team should build and register the following:

### 주민등록번호 (Resident Registration Number, RRN)
- Format: `YYMMDD-NNNNNNN` (13 digits with hyphen)
- Regex: `\d{6}-[1-4]\d{6}`
- Validation: Luhn-style checksum on all 13 digits
- Classification tier: SPII (Government ID)

### 사업자등록번호 (Business Registration Number)
- Format: `NNN-NN-NNNNN`
- Regex: `\d{3}-\d{2}-\d{5}`
- Classification tier: PII (General Identifier, if individual proprietor)

### Korean Mobile Phone Number
- Formats: `010-NNNN-NNNN`, `01X-NNN-NNNN` (older formats)
- Regex: `01[0-9]-\d{3,4}-\d{4}`
- FF1 radix: 10 (numeric-only after stripping hyphens; restore hyphen structure after pseudonymization)
- Classification tier: PII (General Identifier)

### Korean Address
- Patterns vary; use regex for 도/시/구/군/읍/동 administrative unit suffixes
- For structured address fields, detect and mask the personal address components while preserving the administrative unit level (city/district) for regional statistics

### Korean Name (이름)
- Korean names are 2–4 characters; common family names (김, 이, 박, 최, 정...) complicate NER
- Use `kiwipiepy` POS tagger or `konlpy` for NER-based detection in free-text; use column-name heuristics (`*_name`, `*_성명`, `*_이름`) for structured field detection
- Classification tier: PII (General Identifier)


## PyArrow Integration

PyArrow is used as the I/O layer (Parquet read/write) and the in-memory data interchange format between Polars and Presidio.

- Polars DataFrames expose `.to_arrow()` → `pyarrow.Table` for interop with PyArrow-native tools
- `adlfs` provides an `fsspec`-compatible filesystem driver for ADLS Gen2; Polars and PyArrow both support `fsspec` for transparent cloud storage I/O
- Delta Lake format (via `deltalake` Python library) can be adopted in the Pseudonymized Zone if ACID transactions or time travel are needed; Delta tables are Arrow/Parquet native


## Key Management Integration (On-Prem Service)

The pseudonymization service runs on-prem and retrieves keys from one of two sources:

**Option 1 — HashiCorp Vault (on-prem, preferred)**

HashiCorp Vault deployed on-prem keeps all key material entirely within the on-prem network. The pseudonymization service authenticates via AppRole or a service token:

```
hvac.Client(url="https://vault.internal:8200", token=service_token)
    → client.secrets.kv.v2.read_secret_version(path="pseudo/ff1-general")
    → secret["data"]["data"]["key"]  # AES-256 key material
```

**Option 2 — Azure Key Vault over Private Link**

If Azure Key Vault is preferred for unified key lifecycle management (rotation policy, audit logs feeding into Azure Monitor), it is accessible from on-prem via ExpressRoute or VPN with a private endpoint. The service authenticates via a service principal credential:

```
azure-identity ClientSecretCredential(tenant_id, client_id, client_secret)
    → SecretClient(vault_url, credential)
    → client.get_secret("kv-pseudo-general-v1")
    → secret.value  # AES-256 key material
```

In both cases: keys are never written to disk, environment variables, logs, or pipeline configuration. They are held in memory for the duration of the job and discarded when the process exits. Key version management (retrieving a specific version for re-pseudonymization of historical data) is supported by both Vault and AKV.


## Dependency Summary (requirements.txt skeleton)

```
polars>=1.0.0
pyarrow>=17.0.0
ff3>=1.0.0
py4phi>=0.5.0
cryptography>=43.0.0
presidio-analyzer>=2.2.0
presidio-anonymizer>=2.2.0
spacy>=3.7.0
kiwipiepy>=0.17.0
adlfs>=2024.4.0
hvac>=2.3.0                        # HashiCorp Vault client (on-prem KMS)
azure-identity>=1.17.0             # only if using Azure Key Vault over private link
azure-keyvault-secrets>=4.8.0      # only if using Azure Key Vault over private link
azure-monitor-opentelemetry>=1.3.0
```
