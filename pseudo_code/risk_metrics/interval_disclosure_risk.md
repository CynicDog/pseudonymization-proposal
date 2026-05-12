# Interval Disclosure Risk

> The fraction of records whose original value lies within a published ±ε tolerance of the masked value, modelling an adversary who treats the masked value as an approximate pointer to the original.

## Inputs

- `X`, `Z` — original and protected microdata, row-aligned (`n × p`, continuous).
- `ε > 0` — tolerance fraction declared by the data custodian (typical range: `0.01` to `0.10`).
- `scale_rule ∈ {sd, IQR, fixed_unit}` — per-column scale rule. For `fixed_unit`, the data custodian supplies a unit (e.g. 100,000 KRW).

## Outputs

- `IDR_k(ε)` — per-column interval disclosure rate, for each `k ∈ {1, …, p}`.
- `IDR(ε)` — file-level mean across columns.

## Algorithm

1. For each column `k`, determine the per-column scale `s_k` according to `scale_rule`.
2. For each record `i` and column `k`, flag `b_{ik} ← 1` iff `|x_{ik} - z_{ik}| ≤ ε · s_k`, else `b_{ik} ← 0`.
3. Per-column aggregate:
   $$
   \mathrm{IDR}_k(\varepsilon) := \frac{1}{n} \sum_{i=1}^{n} b_{ik}.
   $$
4. File-level aggregate:
   $$
   \mathrm{IDR}(\varepsilon) := \frac{1}{p} \sum_{k=1}^{p} \mathrm{IDR}_k(\varepsilon).
   $$
5. Return `({IDR_k(ε)}_k, IDR(ε))`. Optionally repeat for multiple `ε` values and report a profile.

## Notes

- For [Rounding](../sdc_methods/perturbative/rounding.md) with base `b`, the interval is analytically known: every original lies in `[ub, (u+1)b]`, so `IDR(b / (2 s_k))` is exact.
- For [Noise Addition](../sdc_methods/perturbative/noise_addition.md), `IDR(ε)` under the assumption that the adversary knows the noise distribution is computable analytically from the noise CDF — match `ε · s_k` to a confidence band of `ε`.
- The metric is meaningful only for continuous variables. For categorical variables use the [k-Anonymity](k_anonymity.md) / [Individual Risk](individual_risk.md) family.
- Publishing IDR at multiple `ε` levels (e.g. 0.01, 0.05, 0.10) gives a more informative profile than a single point estimate.

## References

- Domingo-Ferrer and Torra (2001), `papers/perturbative/Domingo-Ferrer-Torra-2001-disclosure-protection-information-loss.pdf`.
- Mateo-Sanz, Sebé and Domingo-Ferrer (2004). Outlier protection in continuous microdata masking. *PSD 2004*, LNCS 3050.
- Hundepool et al. (2014), Chapter 4.
- [docs/12 § Interval Disclosure Risk](../../docs/12-sdc-risk-and-utility-metrics.md#interval-disclosure-risk).
