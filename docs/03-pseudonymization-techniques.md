# 03 — Pseudonymization Techniques

## Two Distinct Problems

The data in this platform contains two fundamentally different categories of sensitive information, and they require fundamentally different treatments.

**Direct identifiers** are fields whose sole purpose is identification — customer ID, resident registration number, policy number, phone number. In an ML pipeline, these fields are almost never input features; they are join keys or record handles. Pseudonymizing them means replacing them with a consistent, opaque surrogate that preserves their role as a join key without revealing the original identity. The surrogate value itself carries no information useful to a model.

**Sensitive numerical attributes** are fields like income, annual premium, claim payout amount, medical diagnosis codes, number of dependents. These are the features ML models actually learn from. The distribution of these values — their ranges, their concentrations, their correlations with other fields — is where re-identification risk lives, and also where model utility lives. An encrypted income field is useless to a model. A statistically transformed income field that preserves the distributional shape while suppressing extreme values that could single out an individual is both privacy-protective and ML-useful.

The primary challenge in this platform is the second category. Direct identifier pseudonymization is a well-solved supporting concern.

## Statistical Disclosure Control — Primary Approach

Statistical Disclosure Control (SDC) is the methodology for transforming sensitive numerical attributes so that the statistical properties useful for analysis and modeling are preserved, while the information that enables re-identification of specific individuals is suppressed. This is the standard methodology in actuarial science, official statistics, and healthcare data sharing, and it is the correct framing for insurance data at this platform.

The foundational reference is Hundepool et al., *Statistical Disclosure Control* (Wiley, 2012), which remains the authoritative text covering all techniques below. Domingo-Ferrer's work on microaggregation and utility measurement provides the theoretical grounding for the ML-utility tradeoffs.

### Rounding

Rounding reduces numerical precision to a coarser unit, limiting the information conveyed without eliminating the value. A salary of ₩52,374,000 rounded to the nearest ₩10,000,000 becomes ₩50,000,000 — still useful for an income-based risk model, but no longer a precise fingerprint.

Rounding is typically the first step applied, before any other transformation. It reduces the effective cardinality of the field and makes subsequent techniques (top-coding, micro-aggregation) more effective because the reduced-precision values group more naturally.

The appropriate rounding unit depends on the field's range and the analytical precision required downstream:

| Field | Indicative rounding unit |
|---|---|
| Annual income / salary | ₩10,000,000 |
| Monthly premium | ₩10,000 |
| Claim payout | ₩1,000,000 |
| Policy face value | ₩10,000,000 |
| Age | 5-year band |

Final rounding units are confirmed in the data classification inventory (Phase 1) in consultation with the actuarial and ML teams, who define the minimum precision required for model accuracy.

### Top-Coding and Bottom-Coding

Extreme values are disproportionately identifying. In an insurance portfolio of tens of thousands of policyholders, a single individual with an annual income of ₩15,000,000,000 or a claim payout of ₩3,000,000,000 is nearly uniquely identifiable from that value alone, without needing any other field. Top-coding replaces values above a threshold with the threshold value or a category label; bottom-coding does the same for values below a lower threshold.

**Threshold setting**: The standard methodology sets the threshold at a percentile of the empirical distribution — the 95th, 97th, or 99th percentile depending on the concentration of outliers and the sensitivity classification of the field. The choice must be documented and justified in the PIPA risk assessment. For heavy-tailed distributions common in insurance (claim severity, income of high-net-worth policyholders), the 99th percentile threshold is typical.

The key property of top-coding is that it leaves the bulk of the distribution completely intact. All records below the threshold retain their exact (rounded) values. Only the small number of extreme records are affected, and precisely those records are the ones carrying the highest re-identification risk.

Top-coding and rounding are complementary and should both be applied: round first to reduce precision across the distribution, then top-code to suppress the identifying tail.

### Micro-Aggregation

Micro-aggregation is a grouping technique that replaces individual values with group summaries while maintaining population-level statistical properties. Records are sorted by the sensitive variable, partitioned into groups of at least k records, and each record's value is replaced by the group mean. The result guarantees that no individual value is distinguishable from at least k−1 other values in the same group — a k-anonymity guarantee on that attribute.

The choice of k controls the privacy-utility tradeoff:
- Larger k: stronger privacy (more individuals share the same pseudonymized value), more information loss
- Smaller k: less information loss, weaker protection
- k = 5 is the statistical minimum; k = 10 is recommended for sensitive insurance fields

Micro-aggregation is particularly effective for fields that must retain their aggregate statistical structure for ML (mean claim severity by cohort, income distribution by product line) while removing the individual-level precision that enables re-identification. It is more utility-preserving than generalization (binning into broad ranges) because the replacement value is the actual local mean rather than a fixed bin label.

The NP-hardness of optimal micro-aggregation (Oganian & Domingo-Ferrer, 2001) means production implementations use greedy contiguous-sort approximations, which achieve good practical results at O(n log n) cost.

### Generalization

Generalization replaces a precise value with a less precise category. Age 47 becomes the band "40–49"; income ₩52,000,000 becomes the range "₩50,000,000–₩59,999,999". Unlike micro-aggregation, the replacement value is a fixed label rather than a local mean, which means all records in the same band receive identical values regardless of their original distribution within the band.

Generalization is appropriate when the downstream analysis or model requires only ordinal or categorical resolution — a churn model that segments customers by income tier rather than exact income, or an actuarial table that uses age bands rather than exact ages. When the model requires numerical precision within a range, micro-aggregation is preferable.

Generalization combined with suppression of records that cannot be safely generalized is the basis of k-anonymity. For this platform's internal ML use case, k-anonymity as a formal model is less relevant than practical information-loss minimization, but the underlying generalization technique remains useful.

### Noise Perturbation

Noise perturbation adds random values to sensitive fields, shifting individual records while preserving aggregate population statistics. The approach has a formal grounding in differential privacy (Laplace and Gaussian mechanisms) and can provide provable bounds on re-identification risk.

The practical challenge for high-range insurance fields is calibration. The noise scale needed to achieve a meaningful privacy budget against the full range of income values (₩0 to ₩500,000,000+) is often large enough to destroy the individual-level signal the ML model needs. Noise perturbation works best as a complement to top-coding and rounding (which reduce the effective range and therefore the required noise scale), not as a standalone technique.

For this platform, noise perturbation is a secondary technique considered for:
- Fields with naturally small ranges (number of dependents, years of coverage, age after banding)
- Situations where a formal differential privacy bound is required (e.g., for external data sharing or model publishing)
- As an additional layer after rounding + top-coding have already reduced the field's sensitivity

## Direct Identifier Pseudonymization — Supporting Concern

Direct identifiers need a consistent surrogate value — the same entity must map to the same pseudonym wherever it appears, so that joins across tables work. The specific technique matters less than consistency, since these fields are typically not features in ML models.

**Deterministic hash (HMAC)**: A keyed one-way function that produces the same output for the same input and key. The surrogate is opaque and non-reversible. Sufficient for entity identifiers that do not need to be recovered.

**Format-preserving encryption (FF1)**: Produces a surrogate that maintains the original format (a 13-digit ID remains 13 digits). Relevant when downstream systems enforce format constraints, or when the original format must be preserved for operational system compatibility. Reversible with the key.

The choice between these is driven by whether the identifier ever needs to be reversed (de-pseudonymized for authorized use cases), not by ML requirements. From the model's perspective, both are equally opaque surrogate keys.

## Technique Assignment by Field Category

| Field category | Primary technique | Secondary |
|---|---|---|
| Continuous high-range numerics (income, claim amount, face value) | Round → top-code | Micro-aggregation if k-anonymity needed |
| Continuous low-range numerics (age, dependents, years) | Round or band | Noise if DP bound required |
| Ordinal codes (product type, risk tier) | Generalization or retain | — |
| Date of birth / event dates | Age band or date shift per entity | — |
| Direct identifiers used as join keys | HMAC (one-way) or FF1 (reversible) | — |
| Government-issued IDs | FF1 (format-preserving, reversible with key) | — |
| Free-text fields | Entity detection + redaction | — |

## What Pseudonymization Does Not Cover

Pseudonymization is not re-encryption and it is not a security boundary in the traditional sense. An ML model trained on properly SDC-treated data does not "know" original income values — but it learns the distributional shape. If that distributional shape is sufficiently identifying (e.g., a very small and homogeneous cohort), pseudonymization alone is insufficient and synthetic data or differential privacy at the model training step become relevant. These are addressed in [07-ml-pipeline.md](07-ml-pipeline.md).
