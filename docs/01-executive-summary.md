# 01 — Executive Summary

## Problem Statement

The company's data platform ingests sensitive and personal data from on-premises databases into Azure cloud infrastructure (Azure Data Factory, Azure Data Lake Storage, Databricks). This data flows downstream into machine learning workloads for prediction and inference.

Raw, identifiable data must never reach the ML layer. Regulatory obligations under Korean PIPA (개인정보보호법) require that any secondary use of personal information — including for statistical analysis and machine learning — must operate on pseudonymized data. Beyond compliance, pseudonymization reduces breach impact: a compromised pseudonymized dataset is computationally infeasible to re-identify without the corresponding cryptographic key.

## Scope

This proposal covers the design of a **pseudonymization layer** sitting between the raw data ingestion zone and all downstream consumption (ML, analytics, reporting). It does not cover:

- Authorization / authentication at the application layer
- Re-identification workflows (break-glass procedures)
- End-user consent management

## Current State

The team previously used PySpark modules on Synapse/Databricks for data transformations. For the actual data volumes in production — primarily megabytes, occasionally gigabytes — Spark introduces unnecessary overhead: cluster spin-up latency, complex dependency management, and disproportionate infrastructure cost. This proposal establishes a right-sized approach.

## Recommended Approach

**Technique:** Format-Preserving Encryption (FF1, NIST SP 800-38G) as the primary pseudonymization method, with HMAC-SHA-256 for cross-dataset referential keys.

- FF1 is stateless, deterministic, reversible (with key), and format-preserving — a 13-digit government ID remains 13 digits after pseudonymization, preserving compatibility with downstream schema contracts.
- HMAC-SHA-256 provides one-way, deterministic pseudonymization for keys that must join across tables but must not be reversible.
- Both techniques produce consistent output for the same input and key, preserving referential integrity for ML feature engineering.

**Engine:** Polars 1.x + PyArrow — columnar, in-memory, Rust-based. Handles MB–GB workloads on a single node with millisecond-level latency. Replaces PySpark for the pseudonymization step; Databricks remains as the ML training and serving platform.

**Key Management:** Azure Key Vault with per-environment, per-classification-tier key namespacing. Managed Identity access for ADF and Databricks; no human access to production keys.

**Detection:** Microsoft Presidio as the PII column detection layer, extended with Korean-specific pattern recognizers.

## Architecture Summary

The pseudonymization service runs on-premises. Its exact placement in the pipeline — whether before data enters the cloud (pre-ingestion) or after the Databricks medallion enrichment (post-medallion egress) — is an open architectural decision pending governance policy alignment. Both options share the same module design and key management approach.

```
Option A — Pre-Ingestion

  On-Prem DB → [On-Prem Pseudo Service] → ADF SHIR → ADLS (pseudonymized) → Databricks → ML

Option B — Post-Medallion Egress

  On-Prem DB → ADF SHIR → ADLS (raw) → Databricks Medallion
                                              │ egress gold layer
                                              ▼
                                    [On-Prem Pseudo Service]
                                              │ re-ingest
                                              ▼
                                    ADLS (pseudonymized) → ML
```

Full architecture detail with tradeoff analysis: [05-architecture.md](05-architecture.md)

## Roadmap at a Glance

| Phase | Duration | Focus |
|---|---|---|
| Phase 1 — Foundation | Weeks 1–4 | Classification inventory, Key Vault setup, Presidio Korean recognizers, base Polars module |
| Phase 2 — Integration | Weeks 5–8 | ADF pipeline wiring, audit logging, referential integrity testing, ML pipeline validation |
| Phase 3 — Hardening | Weeks 9–12 | PIPA risk assessment docs, key rotation automation, re-identification penetration test, stakeholder sign-off |

Full roadmap: [09-implementation-roadmap.md](09-implementation-roadmap.md)

## Key Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Primary technique | FF1 (FPE) | Format-preserving; stateless; NIST-standardized; deterministic |
| Referential key technique | HMAC-SHA-256 | One-way; deterministic; no vault required |
| Transform engine | Polars 1.x | Right-sized for MB-GB; no Spark overhead; on-prem deployable without cloud runtime |
| Deployment target | On-premises | Pseudonymization logic and key access stay within on-prem security boundary |
| Pipeline placement | TBD (Option A or B) | Pre-ingestion or post-medallion; design is open to either — see [05-architecture.md](05-architecture.md) |
| Key store | HashiCorp Vault (on-prem) or Azure Key Vault via private link | Keys remain accessible to on-prem service; no key material in cloud compute |
| PII detection | Microsoft Presidio | Open-source; Python-native; extensible for Korean patterns |
| Do NOT use | FF3/FF3-1 | NIST-withdrawn Feb 2025 (Beyne 2021 cryptographic weakness) |
