# Regression Coefficient Comparison (lm)

> Fit the same linear model on the original and protected data; compare coefficients in magnitude, sign, and significance — the most interpretation-grounded utility measure.

## Inputs

- `X`, `Z` — original and protected microdata, row-aligned.
- `f` — a model formula `y ~ X_1 + X_2 + …` declared by the data custodian as part of the release metadata.
- `α` — significance threshold (typical `α = 0.05`).

## Outputs

- `Δ_β` — Euclidean distance between coefficient vectors.
- `{Δ_{β,k}}_k` — per-coefficient deviation in standard-error units.
- `sign_flips` — count of coefficients whose sign changed.
- `sig_changes` — count of coefficients whose significance at `α` changed.

## Algorithm

1. Fit the linear model on the original data: obtain `β̂_orig ∈ ℝ^q` and per-coefficient standard errors `SE(β̂_orig,k)` for the `q` model coefficients.
2. Fit the **same** model specification `f` on the protected data: obtain `β̂_prot ∈ ℝ^q` and `SE(β̂_prot,k)`.
3. Compute the global distance:
   $$
   \Delta_\beta := \|\hat{\beta}_{\mathrm{orig}} - \hat{\beta}_{\mathrm{prot}}\|_2.
   $$
4. Per-coefficient deviation in original standard-error units:
   $$
   \Delta_{\beta,k} := \frac{|\hat{\beta}_{\mathrm{orig},k} - \hat{\beta}_{\mathrm{prot},k}|}{\mathrm{SE}(\hat{\beta}_{\mathrm{orig},k})}.
   $$
5. For each coefficient `k`:
   - Sign flip: `sign(β̂_orig,k) ≠ sign(β̂_prot,k)`.
   - Significance change: `(p-value_orig,k < α) ≠ (p-value_prot,k < α)`.
6. Aggregate:
   - `sign_flips ← |{k : sign flip}|`.
   - `sig_changes ← |{k : significance change}|`.
7. Return `(Δ_β, {Δ_{β,k}}_k, sign_flips, sig_changes)`.

## Notes

- The metric is **conditional on the model specification**: a release that preserves linear-model coefficients may still fail for models with interactions, splines, or nonlinear terms. Specify and publish the model(s) as part of release metadata.
- For a release supporting a portfolio of analytical use cases, run the comparison on each. For an insurance release intended for claim-frequency modelling, the lm comparison on that exact model is the only direct utility measure.
- `Δ_{β,k} < 1` means the deviation is within one original standard error — typically acceptable. `Δ_{β,k} > 3` means more than three SEs — typically a release blocker.
- Methods that break individual-level associations — notably [Rank Swapping](../sdc_methods/perturbative/rank_swapping.md) when applied independently per column and aggressive [Data Swapping](../sdc_methods/perturbative/data_swapping.md) — register here even if [λ](lambda_measure.md) and [SSE/SST](sse_sst_ratio.md) look acceptable.

## References

- Mateo-Sanz, Domingo-Ferrer and Sebé (2005). *DMKD* 11(2):181–193.
- Hundepool et al. (2014), Chapter 4.
- [docs/12 § Regression Coefficient Comparison](../../docs/12-sdc-risk-and-utility-metrics.md#regression-coefficient-comparison-lm).
