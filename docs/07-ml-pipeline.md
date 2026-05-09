# 07 — ML Pipeline Integration

## Core Requirement

Machine learning models in this platform must operate exclusively on pseudonymized data. This requirement holds across the entire ML lifecycle:

- **Feature engineering:** derived features computed from pseudonymized entity IDs and pseudonymized attributes
- **Model training:** training datasets contain no raw PII; all entity references are pseudonymized
- **Model artifacts:** trained weights, embeddings, and model metadata contain no recoverable PII
- **Inference:** input features are pseudonymized; output predictions are attached to pseudonyms, not raw identifiers
- **Logging and monitoring:** model serving logs contain no PII; pseudonyms only

---

## Design Requirements for Pseudonymized ML Data

For pseudonymized data to be useful for ML, three properties must hold:

**1. Consistency (Determinism)**
The same source value must always produce the same pseudonym, across pipeline runs, across tables, and across environments (staging, production). Without consistency, cross-table JOINs break and feature engineering loses its logical relationships.

Achieved by: FF1 with a fixed key, HMAC-SHA-256 with a fixed key. Both are deterministic for the same input and key.

**2. Referential Integrity**
If `customer_id = 10042` appears in three tables (policy, claim, payment), it must pseudonymize to the same token `PX-9k3m2` in all three tables. This allows Databricks feature engineering to JOIN across these tables using the pseudonymized key just as it would with the raw key.

Achieved by: using the same key and the same technique for the same logical entity identifier across all tables. The implementation must enforce this via a column-classification configuration that is shared across all pipeline runs.

**3. Statistical Utility Preservation**
Pseudonymization must not distort the statistical relationships between features. FF1 (encryption) and HMAC (keyed hash) are both bijective transformations — they preserve the cardinality, uniqueness, and co-occurrence structure of the original data. This means:

- All feature correlations remain intact in pseudonymized data
- Count, frequency, and ratio features computed on pseudonymized data are equivalent to those computed on raw data
- Time-series ordering (event timestamps) is not affected — timestamps are not pseudonymized by default, or are shifted uniformly per entity (see date handling note below)

---

## Pseudonymized Feature Store Design

```
Pseudonymized Zone (ADLS)
    │
    ▼
Databricks Feature Engineering Job
    │
    ├── JOIN: pseudo_policy ON pseudo_customer_id = pseudo_id
    ├── JOIN: pseudo_claim ON pseudo_policy_id = pseudo_id
    └── JOIN: pseudo_payment ON pseudo_policy_id = pseudo_id
    │
    ▼
Feature Table (Databricks Feature Store or ADLS Parquet)
    │
    • Entity key: pseudo_customer_id (FF1-pseudonymized, deterministic)
    • Features: claim_count_6m, premium_amount, last_payment_date, ...
    • No raw PII in any feature column
    │
    ▼
ML Training (Databricks MLflow experiment)
    │
    ▼
Registered Model (MLflow Model Registry)
    • Weights trained on pseudonymized features
    • No PII recoverable from weights
```

---

## Inference Pipeline Design

```
Application Layer (Authorized)
    │
    ├── Receives request: { raw_customer_id: "10042" }
    │
    ├── Pseudonymize: pseudo_id = FF1.encrypt("10042", key)
    │   (Application layer performs this; key access requires RBAC)
    │
    ▼
ML Inference Endpoint
    │
    ├── Input: { pseudo_customer_id: "PX-9k3m2", feature_1: ..., feature_2: ... }
    │   (Pseudonymized features fetched from feature store by pseudo_id)
    │
    ├── Model prediction: { score: 0.87 }
    │
    ▼
Application Layer
    │
    └── Attach prediction to original request context (in-memory, not logged)
        Output to authorized requester
        Log: { pseudo_id: "PX-9k3m2", score: 0.87, timestamp: ... }
        (Audit log contains only pseudonym, never raw ID)
```

**Key constraint:** The application layer that translates raw IDs to pseudonyms before calling the inference endpoint must have its own RBAC scope and audit trail, separate from the data platform. The inference endpoint itself never receives raw IDs.

---

## Date and Timestamp Handling

Timestamps and dates are often not considered PII in isolation but can become quasi-identifiers when combined with other fields (e.g., date of birth + region = highly identifying). The recommended approach:

- **Event timestamps** (created_at, updated_at): do not pseudonymize. These are operational metadata, not personal information.
- **Date of birth and similar personal dates**: apply **consistent date shifting** — add a fixed random offset (e.g., ±N days, where N is derived deterministically from the entity's pseudonymized key). This preserves age ranges and time intervals while preventing direct re-identification via exact date of birth.
- **Policy/claim effective dates**: do not pseudonymize. These are contractual dates, not personal identifiers.

Date shifting ensures that time-series features (e.g., "days since last claim", age-based risk features) remain mathematically valid after pseudonymization.

---

## ML Scenario Variants

### Scenario 1: Internal Supervised Learning (Default)

Training on pseudonymized operational data for internal prediction tasks (churn, risk scoring, claim prediction).

- **Approach:** FF1 + HMAC pseudonymization on all PII/SPII fields; train directly on pseudonymized dataset
- **Utility impact:** Zero — bijective transforms preserve all statistical relationships
- **Regulatory posture:** PIPA-compliant for internal research/analytics secondary use
- **Recommended for:** Most ML use cases in this platform

### Scenario 2: Fully Synthetic Training Data

Generate a synthetic dataset that statistically mirrors the real data distribution but contains no real records.

- **Tools:** Synthetic Data Vault (SDV), CTGAN, or TVAE for tabular data; `sdv.multi_table.HMASynthesizer` for relational schemas
- **Approach:** Fit synthesizer on pseudonymized data; generate N synthetic records; train ML model on synthetic records
- **Utility impact:** Moderate — edge-case distributions may be underrepresented; rare events poorly synthesized
- **Regulatory posture:** Strongest possible — synthetic data is entirely outside PIPA scope
- **Recommended for:** Externally shared research datasets; cases requiring zero re-identification risk even with key compromise

### Scenario 3: Differential Privacy at Training Time

Apply DP-SGD during model training to provide a formal, measurable privacy bound.

- **Tools:** Opacus (PyTorch), TensorFlow Privacy, Objax (JAX)
- **Approach:** Train model with DP-SGD; set privacy budget ε ≤ 1.0 for strong privacy; tune noise multiplier and gradient clipping
- **Utility impact:** Small at ε = 1.0 with sufficient data (ΔAUC < 0.01 on typical classification tasks); larger impact with small datasets
- **Regulatory posture:** Formally bounded re-identification risk; strongest defense against model inversion and membership inference attacks
- **Recommended for:** Externally published models; federated learning across organizations; regulatory requirements for formal privacy proof

### Scenario 4: Re-identification at Prediction Serving (Authorized)

Some downstream use cases require attaching model predictions to identifiable customer records (e.g., a customer-facing recommendation or a case worker view). This must not happen within the ML platform itself.

- **Design:** Prediction output carries pseudonym only; re-identification occurs in a separate authorized application layer with its own RBAC, audit logging, and break-glass procedure
- **No raw PII ever enters the ML platform or model serving logs**

---

## Anti-Patterns to Avoid

| Anti-pattern | Risk | Correct approach |
|---|---|---|
| Logging raw customer IDs in model serving | PII in logs; breach exposure | Log pseudonyms only |
| Using non-deterministic masking (Faker) for training data | Breaks cross-table JOINs; unusable feature store | Use FF1/HMAC with fixed key |
| Storing keys in model configuration files | Key exposure if repo/config leaked | Key Vault at runtime |
| Pseudonymizing only some tables in a JOIN | Inconsistent JOIN keys; feature engineering failure | Pseudonymize all tables with shared classification config |
| Training on raw data in a "trusted" Databricks notebook | PIPA violation; breach risk | All Databricks compute accesses pseudonymized zone only |
| Embedding raw IDs in model feature names or metadata | PII leakage in MLflow artifacts | Use pseudonymized IDs as feature names |
