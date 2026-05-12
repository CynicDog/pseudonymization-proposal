# RMDID1 — Robust Mahalanobis Disclosure Risk

> Use Robust Mahalanobis Distance to scale a per-record disclosure interval, so that outliers (which are intrinsically harder to mask) get larger tolerances; report the share of records where the masked value falls inside the scaled interval.

## Inputs

- `X`, `Z` — original and protected microdata, row-aligned (`n × p`, continuous).
- `ε > 0` — tolerance fraction (same role as in [Interval Disclosure Risk](interval_disclosure_risk.md)).
- `s_k` — per-column scale for column `k` (sample sd or IQR).
- MCD breakdown fraction `α ∈ [0.5, 0.875]` (typical `α = 0.75`).

## Outputs

- `RMDID1` — share of records where the masked value lies inside the MCD-scaled interval around the original.
- `RMD_i` — per-record Robust Mahalanobis Distance (useful as a diagnostic and as an outlier indicator).

## Algorithm

1. Compute the MCD (Minimum Covariance Determinant) location and scatter estimates from `X` at breakdown fraction `α`:
   - Search over `h := ⌈α · n⌉`-subsets of rows of `X` to find the subset with minimum-determinant sample covariance. Use the FAST-MCD algorithm (Rousseeuw and Van Driessen 1999) for tractability.
   - Set `μ̂ ← mean of the optimal subset`, `Σ̂_MCD ← cov of the optimal subset` (consistency-adjusted).
2. For each record `i ∈ {1, …, n}`, compute
   $$
   \mathrm{RMD}_i := \sqrt{(x_i - \hat{\mu})^{\top}\,\hat{\Sigma}_{\mathrm{MCD}}^{-1}\,(x_i - \hat{\mu})}.
   $$
3. For each record `i` and each column `k`, flag inside-interval:
   $$
   b_{ik} \leftarrow \mathbf{1}\!\left[\,|x_{ik} - z_{ik}| \leq \varepsilon \cdot \mathrm{RMD}_i \cdot s_k\,\right].
   $$
4. Aggregate to a single scalar:
   $$
   \mathrm{RMDID1} := \frac{1}{n \cdot p} \sum_{i=1}^{n} \sum_{k=1}^{p} b_{ik}.
   $$
   (Or report `RMDID1_k` per column and average.)
5. Return `(RMDID1, {RMD_i}_i)`.

## Notes

- The intuition: an outlier (large `RMD_i`) gets a wider disclosure band because it is intrinsically harder to mask. A method that achieves low RMDID1 on outliers genuinely protects the tail.
- Requires `Σ̂_MCD` to be invertible — guaranteed if `h > p` and the columns are not collinear.
- RMDID1 is the right risk pairing for [Top and Bottom Coding](../sdc_methods/non_perturbative/top_bottom_coding.md), [Multiplicative Noise](../sdc_methods/perturbative/multiplicative_noise.md), and [Resampling](../sdc_methods/perturbative/resampling.md) — methods whose effectiveness on heavy-tailed data is what RMDID1 measures.
- FAST-MCD is `O(n · p² + p³)` per random subset, with a small number of subsets sampled; tractable for `p ≤ 50` and `n ≤ 10⁵`.

## References

- Templ and Meindl (2008). Robust statistics meets SDC: New disclosure risk measures for continuous microdata masking. *PSD 2008*, LNCS 5262.
- Rousseeuw (1985). Multivariate estimation with high breakdown point.
- Rousseeuw and Van Driessen (1999). A fast algorithm for the minimum covariance determinant estimator. *Technometrics* 41:212–223.
- [docs/12 § RMDID1](../../docs/12-sdc-risk-and-utility-metrics.md#rmdid1--robust-mahalanobis-disclosure-risk).
