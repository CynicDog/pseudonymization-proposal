# 01 — Executive Summary

## Problem Statement

The data platform ingests sensitive and personal data from on-premises databases into cloud infrastructure. This data flows downstream into machine learning workloads for prediction and inference.

Raw, identifiable data must never reach the ML layer. Regulatory obligations under Korean PIPA (개인정보보호법) require that any secondary use of personal information — including for statistical analysis and machine learning — must operate on pseudonymized data. Beyond compliance, pseudonymization reduces breach impact: a compromised pseudonymized dataset is computationally infeasible to re-identify without the corresponding cryptographic key.

## Scope

This proposal covers the design of a **pseudonymization layer** sitting between the raw data ingestion zone and all downstream consumption (ML, analytics, reporting). It does not cover:

- Authorization / authentication at the application layer
- Re-identification workflows (break-glass procedures)
- End-user consent management

## Current State

For the actual data volumes in scope — primarily megabytes, occasionally gigabytes — distributed processing tooling introduces unnecessary overhead: cluster spin-up latency, complex dependency management, and disproportionate infrastructure cost. This proposal establishes a right-sized approach.

## Recommended Approach

The core pseudonymization challenge in an insurance data platform is **statistical**, not cryptographic. The fields that matter for ML — income, claim amounts, premiums, benefit levels — must remain numerically meaningful after pseudonymization. Encrypting them produces data that is both private and useless to a model. The approach is therefore primarily Statistical Disclosure Control (SDC), applied to sensitive numerical attributes, with identifier surrogation as a supporting step.

**For sensitive numerical attributes** (income, claim amounts, policy values, medical-adjacent fields):
Statistical Disclosure Control — rounding to appropriate precision, top-coding at the 99th percentile to suppress identifying extreme values, and micro-aggregation where a k-anonymity guarantee is needed. These techniques preserve the distributional shape used in ML modeling while suppressing the rare extreme values that enable individual re-identification. This is the established methodology for insurance and actuarial microdata, grounded in Hundepool et al. (2012) and Domingo-Ferrer (2016).

**For direct identifiers** (customer ID, resident registration number, policy number, phone number):
Consistent surrogation — a deterministic transformation that replaces each identifier with a stable, opaque surrogate, enabling cross-table joins without revealing the original identity. These fields are not ML features; the surrogation technique's primary requirement is consistency, not cryptographic strength.

**Engine:** Polars 1.x + PyArrow — columnar, in-memory, deployed on-prem. Handles MB–GB workloads on a single node without cloud compute overhead.

**Detection:** A structured PII detection library with regex-based pattern recognizers for structured tabular fields. No NER or language model inference required for 100% tabular data.

## Architecture Summary

The pseudonymization service runs on-premises. Its exact placement in the pipeline — whether before data enters the cloud (pre-ingestion) or after cloud-side enrichment (post-enrichment egress) — is an open architectural decision pending governance policy alignment. Both options share the same module design and key management approach.

```
Option A — Pre-Ingestion

  On-Prem DB → [On-Prem Pseudo Service] → encrypted transfer → cloud storage (pseudonymized) → ML

Option B — Post-Enrichment Egress

  On-Prem DB → encrypted transfer → cloud storage (raw) → cloud analytics platform (enrichment)
                                                                    │ egress enriched layer
                                                                    ▼
                                                         [On-Prem Pseudo Service]
                                                                    │ re-ingest
                                                                    ▼
                                                         cloud storage (pseudonymized) → ML
```

Full architecture detail with tradeoff analysis: [05-architecture.md](05-architecture.md)

## Key Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Primary approach for numerical fields | SDC (round → top-code → micro-aggregate) | Preserves distributional utility for ML; suppresses identifying extremes |
| Approach for direct identifiers | Deterministic surrogation (HMAC or FF1) | Consistent join-key replacement; these fields are not ML features |
| Transform engine | Polars 1.x + PyArrow | Right-sized for MB-GB; on-prem deployable; no Spark overhead |
| Deployment target | On-premises | Pseudonymization logic stays within on-prem security boundary |
| Pipeline placement | TBD (pre-ingestion or post-enrichment) | Open to either — see [05-architecture.md](05-architecture.md) |
| Column detection | Regex-based structured PII detection library | No NER model needed for 100% tabular data |
| Key store (for identifier surrogation) | On-prem KMS or cloud KMS via private network | Keys stay within on-prem boundary |
| Do NOT use | FF3/FF3-1 | NIST-withdrawn Feb 2025; cryptographic weakness in tweak schedule |
| Do NOT apply to analytical fields | Encryption (FF1, HMAC, AES) | Encrypted numerical fields are meaningless for ML |
