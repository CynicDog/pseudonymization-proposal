# 05 — Architecture

## Design Principles

1. **Pseudonymize once, use everywhere.** Pseudonymization happens at a single designated boundary. All downstream systems consume only pseudonymized data.
2. **Keys never travel with data.** Encryption keys live in an isolated key store (on-prem KMS or Azure Key Vault via private link), accessed at runtime. No key material appears in pipeline configuration, environment variables, or data files.
3. **Pseudonymization runs on-prem.** The pseudonymization service is deployed on-premises, not on cloud compute. This keeps key material and the pseudonymization logic within the on-prem security boundary regardless of where the data boundary is drawn.
4. **Placement is a policy decision, not a technical one.** The pseudonymization module is designed identically regardless of where in the pipeline it is placed. Two integration points are viable; the choice depends on data governance policy and pipeline design.
5. **Determinism by default.** All production pseudonymization is deterministic (same input + same key → same output) to preserve referential integrity for downstream JOIN operations.
6. **Audit everything.** Every pseudonymization job execution and every key retrieval is logged immutably.

## The On-Prem Pseudonymization Service

The pseudonymization layer is a standalone Python service (Polars + ff3 + HMAC — see [06-technology-stack.md](06-technology-stack.md)) packaged for on-prem deployment. It can run as:

- A Docker container on an on-prem application server
- A Linux/Windows batch process triggered by a scheduler or file-arrival event
- A lightweight HTTP service (FastAPI) called by adjacent pipeline components

The service reads from a local or network-accessible data source, applies per-column pseudonymization per the classification manifest, writes the pseudonymized output, and emits an audit log event. It does not depend on Databricks, Azure Functions, or any cloud compute runtime. Cloud resources it may interact with are limited to storage (ADLS read/write) and key retrieval (Azure Key Vault via private link, or on-prem HashiCorp Vault).

## Integration Point Options

The placement of this service within the pipeline is not yet decided. Two viable options exist, each with distinct data governance tradeoffs. The module design is identical in both cases.

### Option A — Pre-Ingestion (Pseudonymize Before Cloud)

Data is pseudonymized on-prem before ADF ever touches it. The cloud receives only pseudonymized data from the start.

```
On-Prem Network
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Source DB(s)                                                │
│      │                                                       │
│      ▼                                                       │
│  On-Prem Pseudonymization Service                            │
│  ├── Reads raw data from source DB (or staging file area)    │
│  ├── Applies FF1 / HMAC per classification manifest          │
│  ├── Fetches keys from on-prem KMS (HashiCorp Vault or AKV   │
│  │   via private link)                                       │
│  └── Writes pseudonymized output to staging area             │
│      │                                                       │
│      ▼                                                       │
│  Pseudonymized Staging Area (on-prem)                        │
│      │                                                       │
└──────┼───────────────────────────────────────────────────────┘
       │  ADF SHIR (TLS 1.2+)
       │  Ingests pseudonymized data — no raw PII ever enters cloud
       ▼
Azure Cloud
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  ADLS — Ingested Zone (pseudonymized)                        │
│      │                                                       │
│      ▼                                                       │
│  Databricks (medallion: bronze → silver → gold)              │
│  Feature engineering, ML training                            │
│      │                                                       │
│      ▼                                                       │
│  ML Inference Endpoint / Analytics / BI                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- Raw PII is never transmitted to or stored in the cloud
- Cloud infrastructure requires no raw-zone access controls or break-glass procedures for PII
- Strongest regulatory posture: cloud provider processes no personal data at all
- Pseudonymization applied to source data as-is, before any enrichment or cross-table joins in the medallion
- If cross-table consistency is needed (e.g., `customer_id` pseudonymized identically across multiple source tables), the service must be given a unified view or run with a shared key and classification manifest across all source tables

### Option B — Post-Medallion Egress (Pseudonymize After Cloud Enrichment)

Raw data is ingested into the cloud, processed through the Databricks medallion (bronze → silver → gold), and the enriched output is egressed back to on-prem for pseudonymization before being re-ingested as the final, pseudonymized dataset.

```
On-Prem Network
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Source DB(s)                                                │
│      │                                                       │
└──────┼───────────────────────────────────────────────────────┘
       │  ADF SHIR — raw data ingestion (TLS 1.2+)
       ▼
Azure Cloud
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  ADLS — Raw Zone                                             │
│  (raw PII present; strict access controls required)          │
│      │                                                       │
│      ▼                                                       │
│  Databricks Medallion Pipeline                               │
│  Bronze: raw ingest                                          │
│  Silver: cleanse, join, conform                              │
│  Gold: curated domain entities (customer 360, unified claim) │
│      │                                                       │
│      │  Egress: ADF SHIR or ADLS → on-prem transfer         │
│      │  (gold-layer output exported to on-prem)              │
└──────┼───────────────────────────────────────────────────────┘
       │
       ▼
On-Prem Network
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  On-Prem Pseudonymization Service                            │
│  ├── Receives gold-layer export (enriched, joined entities)  │
│  ├── Applies FF1 / HMAC per classification manifest          │
│  └── Writes pseudonymized output to staging area             │
│      │                                                       │
└──────┼───────────────────────────────────────────────────────┘
       │  ADF SHIR — re-ingestion of pseudonymized data
       ▼
Azure Cloud
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  ADLS — Pseudonymized Zone                                   │
│      │                                                       │
│      ▼                                                       │
│  ML Feature Store / Inference Endpoint / Analytics           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- Pseudonymization is applied to enriched, cross-table-joined entities — the gold layer represents a cleaner, more complete view of each entity before pseudonymization
- Cross-table referential integrity is naturally handled by the medallion joins; all entity references in the gold output are pseudonymized in a single pass
- Raw PII exists in the cloud (Raw Zone and Bronze/Silver layers) — these layers require their own access controls, break-glass procedures, and PIPA-compliant safeguards
- Additional data round-trip (cloud → on-prem → cloud) adds pipeline latency and egress cost
- The PIPA compliance burden is higher: raw-data cloud storage must be documented and justified in the risk assessment

## Option Comparison

| Concern | Option A (Pre-Ingestion) | Option B (Post-Medallion) |
|---|---|---|
| Raw PII in cloud | Never | Yes (Raw/Bronze/Silver zones) |
| PIPA compliance complexity | Lower | Higher (raw-zone controls required) |
| Data quality at pseudonymization | Source-level (pre-enrichment) | Gold-level (post-enrichment, joined) |
| Cross-table referential integrity | Requires coordinated manifest across source tables | Handled naturally by medallion joins |
| Pipeline round-trips | One (on-prem → cloud) | Three (on-prem → cloud → on-prem → cloud) |
| Pseudonymization service complexity | Simpler (reads source tables directly) | Slightly higher (reads gold-layer export) |
| Cloud architecture | Simpler (no raw zone controls needed) | More complex (raw zone isolation required) |

## Supporting Services

The following services are shared across both options.

```
On-Prem Key Management
┌─────────────────────────────────────────────────────────┐
│  HashiCorp Vault (on-prem) — preferred for on-prem keys │
│  or                                                     │
│  Azure Key Vault (accessed via ExpressRoute/private link)│
│                                                         │
│  • FF1 keys (general tier, sensitive tier)              │
│  • HMAC signing key                                     │
│  • AES-SIV key                                          │
│  • No human access to key values in prod                │
│  • All access audited                                   │
└─────────────────────────────────────────────────────────┘

Azure Governance (Cloud Side)
┌─────────────────────────────────────────────────────────┐
│  Microsoft Purview                                      │
│  • Sensitivity labels on all ADLS assets                │
│  • Data lineage tracking                                │
│                                                         │
│  Azure Monitor + Log Analytics                          │
│  • Pipeline execution logs                              │
│  • ADLS access logs (raw zone if Option B)              │
│  • Pseudonymization audit events (forwarded from on-prem)│
└─────────────────────────────────────────────────────────┘
```

## Pseudonymization Service — Internal Flow (Both Options)

Regardless of integration point, the pseudonymization service executes the same internal steps:

```
Input data (source DB extract or gold-layer export)
    │
    ▼
Column scan (Presidio Analyzer + classification manifest lookup)
    │ → Identify PII/SPII columns and their tiers
    ▼
Key retrieval (on-prem KMS or Azure Key Vault via private link)
    │ → One key fetch per tier per job run; held in memory only
    ▼
Technique dispatch per column
    ├── FF1 (general tier key)    → general PII fields
    ├── FF1 (sensitive tier key)  → SPII fields
    ├── HMAC-SHA-256              → referential join keys
    └── Presidio redact           → free-text fields
    │
    ▼
Polars columnar transform (in-memory, streaming for large files)
    │
    ▼
Pseudonymized output (Parquet) + Parquet schema metadata (technique, key version per column)
    │
    ▼
Audit log event emitted (job ID, timestamp, source, row count, columns, key versions, status)
```

## Scalability Notes

The on-prem pseudonymization service is stateless and horizontally scalable — multiple instances can run concurrently on different source tables without coordination, since each instance only needs its own key fetch and classification manifest. For the current MB-scale data volumes, a single instance is sufficient. For GB-scale files, Polars streaming mode (`.sink_parquet()`) processes without materializing the full file in memory.

If data volumes grow to TB-scale in the future, the pseudonymization logic (FF1/HMAC column transforms) is portable to a distributed engine (PySpark UDFs) without changing the technique, key management design, or classification manifest schema.
