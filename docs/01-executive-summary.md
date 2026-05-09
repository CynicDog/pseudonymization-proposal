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

The core pseudonymization challenge in an insurance data platform is **statistical**, not cryptographic. The fields that matter for ML — income, claim amounts, premiums, benefit levels — must remain numerically meaningful after pseudonymization. Encrypting them produces data that is both private and useless to a model. The approach is therefore primarily Statistical Disclosure Control (SDC), applied to sensitive numerical attributes, with identifier surrogation as a supporting step.

**For sensitive numerical attributes** (income, claim amounts, policy values, medical-adjacent fields):
Statistical Disclosure Control — rounding to appropriate precision, top-coding at the 99th percentile to suppress identifying extreme values, and micro-aggregation where a k-anonymity guarantee is needed. These techniques preserve the distributional shape used in ML modeling while suppressing the rare extreme values that enable individual re-identification. This is the established methodology for insurance and actuarial microdata, grounded in Hundepool et al. (2012) and Domingo-Ferrer (2016).

**For direct identifiers** (customer ID, resident registration number, policy number, phone number):
Consistent surrogation — a deterministic transformation that replaces each identifier with a stable, opaque surrogate, enabling cross-table joins without revealing the original identity. These fields are not ML features; the surrogation technique's primary requirement is consistency, not cryptographic strength.

**Engine:** Polars 1.x + PyArrow — columnar, in-memory, deployed on-prem. Handles MB–GB workloads on a single node. Replaces PySpark for the pseudonymization step; Databricks remains as the ML training platform.

**Detection:** Microsoft Presidio `StructuredEngine` with regex-based pattern recognizers for structured tabular fields. No NER or language model inference required for 100% tabular data.

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
| Primary approach for numerical fields | SDC (round → top-code → micro-aggregate) | Preserves distributional utility for ML; suppresses identifying extremes |
| Approach for direct identifiers | Deterministic surrogation (HMAC or FF1) | Consistent join-key replacement; these fields are not ML features |
| Transform engine | Polars 1.x + PyArrow | Right-sized for MB-GB; on-prem deployable; no Spark overhead |
| Deployment target | On-premises | Pseudonymization logic stays within on-prem security boundary |
| Pipeline placement | TBD (pre-ingestion or post-medallion) | Open to either — see [05-architecture.md](05-architecture.md) |
| Column detection | Microsoft Presidio StructuredEngine (regex-based) | No NER model needed for 100% tabular data |
| Key store (for identifier surrogation) | HashiCorp Vault (on-prem) or Azure Key Vault via private link | Keys stay within on-prem boundary |
| Do NOT use | FF3/FF3-1 | NIST-withdrawn Feb 2025; cryptographic weakness in tweak schedule |
| Do NOT apply to analytical fields | Encryption (FF1, HMAC, AES) | Encrypted numerical fields are meaningless for ML |
