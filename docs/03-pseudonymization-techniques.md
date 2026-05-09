# 03 — Pseudonymization Techniques

## Technique Taxonomy

| Technique | Reversible | Format-Preserving | Deterministic | ML Utility | Complexity |
|---|---|---|---|---|---|
| FF1 (FPE) | Yes (keyed) | Yes | Yes | High | Medium |
| HMAC-SHA-256 | No | No | Yes | High | Low |
| AES-SIV | Yes (keyed) | No | Yes | High | Medium |
| Vaulted Tokenization | Yes (vault lookup) | Optional | Yes | High | High |
| Faker / Masking | No | No | No | Low | Low |
| k-anonymity / ℓ-diversity | No | No | No | Medium | High |
| Differential Privacy | N/A (noise) | N/A | N/A | Medium | High |


## Technique Detail

### FF1 — Format-Preserving Encryption (Primary Recommendation)

**Standard:** NIST SP 800-38G (March 2016; Draft Rev. 1, February 2025)

FF1 is a Feistel-based cipher that encrypts data within its original domain: a 16-digit number produces a 16-digit ciphertext; a 10-character alphanumeric string produces a 10-character alphanumeric ciphertext. The transform is deterministic (same input + same key → same output), reversible with the key, and stateless (no external database lookup required).

**Why FF1:**
- Downstream systems (databases, APIs) often enforce format constraints (field length, character set, check digits). FF1 preserves these constraints.
- Determinism enables referential integrity: the same customer ID pseudonymizes to the same token in every table, every pipeline run, every environment — enabling JOINs on pseudonymized keys.
- Stateless computation scales horizontally without vault infrastructure.
- NIST-standardized, widely implemented, auditable.

**Critical Warning — FF3/FF3-1:** NIST removed FF3 and FF3-1 from SP 800-38G in the February 2025 Draft Revision 1, citing a cryptographic weakness in the tweak schedule identified by Beyne (2021). **Do not implement FF3 or FF3-1.** Use FF1 exclusively.

**Python library:** `ff3` (PyPI) — implements FF1 and FF3-1; use the FF1 mode only.

**Domain size constraint:** FF1 requires a minimum domain size (radix^minlen ≥ 100). For most PII field types (numeric IDs, phone numbers, alphanumeric codes), this is trivially satisfied. Binary or very short fields should use AES-SIV instead.


### HMAC-SHA-256 — Keyed One-Way Hash (Referential Key Recommendation)

HMAC-SHA-256 produces a fixed-length, deterministic, one-way pseudonym for any input, given a secret key. Unlike FF1, it is not reversible — there is no decrypt operation. The same input always produces the same output (deterministic), making it suitable for linking records across tables via pseudonymized join keys.

**When to use:**
- Entity keys (customer ID, policy number) that must join across multiple tables but must not be reversible
- Credentials and authentication artifacts: passwords, tokens, session identifiers (these should never be reversible)
- Any field where re-identification is prohibited even with key access

**Salting:** A static HMAC key is sufficient for deterministic cross-dataset joins. Adding a per-record random salt would break determinism and is not suitable for production pseudonymization (only for one-time masking in dev/test environments).

**Python:** `hmac` (standard library) + `hashlib`. No external dependency required.


### AES-SIV — Authenticated Deterministic Encryption

**Standard:** RFC 5297

AES-SIV derives its initialization vector deterministically from the plaintext and optional authenticated data, making it nonce-misuse resistant. Unlike standard AES modes (CBC, GCM with random IV), AES-SIV produces the same ciphertext for the same plaintext and key — deterministic and authenticated.

**When to use:**
- Fields that are not format-constrained but require reversibility and tamper detection
- Unstructured or variable-length values (free-text contact notes, internal reference codes) where FF1's domain constraints are limiting
- Cases where ciphertext authenticity must be verifiable (the MAC is embedded in the output)

**Python:** `cryptography` library (PyPI) — `AESSIV` class under `cryptography.hazmat.primitives.ciphers.aead`.


### Vaulted Tokenization (Legacy Reference)

In vaulted tokenization, the original value is stored in a separate secure vault, and the production database contains only a random token. Re-identification requires a round-trip to the vault.

**Advantages:** Strongest regulatory separation; original data physically isolated.

**Disadvantages for this use case:**
- Every pseudonymized field access requires a vault lookup → high latency for ML feature pipelines
- Vault becomes a single point of failure and a scaling bottleneck
- Operational overhead: vault HA, failover, audit, rotation
- Not compatible with high-throughput columnar batch processing (Polars/PyArrow)

**Recommendation:** Do not use vaulted tokenization for the data platform pseudonymization layer. Use stateless FF1/HMAC instead, with Azure Key Vault used only for key storage (not data storage). See [08-key-management.md](08-key-management.md).


### Faker / Data Masking (Dev/Test Only)

Libraries such as `Faker` (PyPI) and `Mimesis` generate synthetic but realistic-looking values (names, addresses, phone numbers, dates). Output is non-deterministic and irreversible.

**Appropriate use:** Generating synthetic development and test datasets. These libraries have no place in the production pseudonymization pipeline — they break referential integrity (same input → different output each run) and produce values that may not satisfy format constraints.


### k-anonymity, ℓ-diversity, t-closeness (Aggregate Release Only)

These are generalization/suppression techniques that ensure each record is indistinguishable from at least k-1 other records across quasi-identifier combinations. They are designed for **publishing aggregate datasets** (e.g., open government data, research registries).

**Not recommended for internal ML pipelines:** Generalization and suppression destroy the fine-grained signal needed for predictive models. A k=5 anonymized dataset where age is bucketed to decade ranges and region is generalized to province level is unusable for a churn prediction model. These techniques introduce significant data utility loss without providing stronger re-identification protection than FF1+HMAC for internal, access-controlled systems.


### Differential Privacy (Model Training Boundary)

Differential privacy (DP) adds calibrated statistical noise to ensure that the presence or absence of any individual record cannot be inferred from the output of a computation. In ML, this is applied as DP-SGD (differentially private stochastic gradient descent) during model training.

**When to use:**
- Publishing trained model weights externally (e.g., open research, federated learning)
- Generating aggregate statistics for external release
- Formal privacy guarantee requirement (ε-bounded re-identification risk)

**Not a replacement for pseudonymization:** DP is a training-time technique, not a data-at-rest protection mechanism. The training data must still be pseudonymized before entering the training pipeline. DP and pseudonymization are complementary, not alternatives.

**Implementation libraries:** Opacus (PyTorch DP-SGD), TensorFlow Privacy, Objax (JAX). At ε = 1.0, empirical studies show re-identification risk < 0.1% with negligible model utility loss (ΔAUC ≈ 0) on classification tasks with sufficient training data.


## Field-Type to Technique Mapping

This mapping applies to the company's defined PII/SPII classification tiers. Field types are referenced generically; the implementation team should map concrete column names to these categories using the data classification inventory produced in Phase 1 (see [09-implementation-roadmap.md](09-implementation-roadmap.md)).

| Field Type | Technique | Reversible | Notes |
|---|---|---|---|
| Numeric identifiers (IDs, policy numbers) | FF1 | Yes | Format preserved; deterministic join key |
| Government-issued IDs (national ID, passport, license) | FF1 | Yes | Format preservation critical; separate SPII key namespace |
| Phone numbers | FF1 | Yes | Preserve format (e.g., 010-XXXX-XXXX structure) |
| Email addresses | HMAC-SHA-256 | No | Format not contractually required downstream; one-way sufficient |
| Names (structured fields) | FF1 or HMAC | Optional | If name is a join key: FF1. If name is only displayed: HMAC or masking |
| IP addresses, device IDs, MAC addresses | HMAC-SHA-256 | No | Technical identifiers; one-way hash sufficient |
| Authentication credentials (passwords, tokens) | HMAC-SHA-256 | No | Must never be reversible |
| Health / financial numeric data | FF1 | Yes | SPII key namespace; strictest access control |
| Free-text fields (notes, comments) | Presidio redact + masking | No | PII entities detected by Presidio; replaced with `[REDACTED]` or synthesized placeholder |
| Dates of birth / event dates | FF1 on components or date shifting | Optional | Consider date-shift (± N days with consistent offset per entity) to preserve time-series utility |
| Biometric / genetic data | AES-SIV or drop | Yes | Separate key; access control; evaluate if needed downstream |
| Social graph / relationship data | HMAC-SHA-256 on all entity IDs | No | Preserve graph structure; pseudonymize all node identifiers |


## Technique Selection Decision Tree

```
Is the field used as a JOIN key across datasets?
├── Yes → Must be deterministic
│   ├── Must re-identify for authorized use cases? → FF1
│   └── Never re-identify (one-way only)? → HMAC-SHA-256
└── No → Is format preservation required by downstream schema?
    ├── Yes → FF1
    └── No → Is field free-text / unstructured?
        ├── Yes → Presidio detection + masking/redaction
        └── No → HMAC-SHA-256 or AES-SIV
```
