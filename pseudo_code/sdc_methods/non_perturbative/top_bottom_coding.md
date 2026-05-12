# Top and Bottom Coding

> Replace values above a top threshold with the threshold value, and values below a bottom threshold with the threshold value, eliminating the most extreme observations that are most likely to be unique and thus re-identifiable.

$$
x' := \min(x, T_{\mathrm{top}}), \qquad x' := \max(x, T_{\mathrm{bot}})
$$

## Inputs

- `X` — microdata (continuous or ordered-categorical columns).
- For each column `k` to code:
  - `T_top,k` — upper threshold (typical: 98th or 99th percentile of `x_{·k}`).
  - `T_bot,k` — lower threshold (typical: 1st or 2nd percentile).
- `label_mode ∈ {clip, plus_marker, interval}` — how to render coded values for analysts.

## Outputs

- `Z` — protected microdata with extremes clipped.

## Algorithm

1. For each column `k` to code:
   - For each record `i`, compute the coded value `z_{ik}` according to `label_mode`:
     - **clip**: if `x_{ik} > T_top,k`, set `z_{ik} ← T_top,k`; if `x_{ik} < T_bot,k`, set `z_{ik} ← T_bot,k`; otherwise `z_{ik} ← x_{ik}`.
     - **plus_marker**: same as clip but encode the censored cells with a marker (e.g. `"T_top,k+"`, `"T_bot,k-"`) so the analyst can apply censored-regression methods.
     - **interval**: emit an interval label `[T_top,k, ∞)` or `(-∞, T_bot,k]` for censored cells.
2. Document `(T_top,k, T_bot,k)` per coded column in the release metadata; analysts using these columns must account for the censoring (e.g. Tobit, interval regression).
3. Return `Z`.

## Notes

- Preserves the interior of the distribution exactly; only the tails are modified. Most records receive no perturbation, which is what makes the method analytically neutral for the bulk.
- The protection rationale is that extreme values are disproportionately identifying (a billionaire's income, a centenarian's age, a very small firm's employee count). Collapsing tails to a single coded value removes that uniqueness.
- For categorical variables, top/bottom coding generalises to collapsing the highest and lowest ordered categories into aggregated codes.
- Often **combined with [Rounding](../perturbative/rounding.md)** so that the interior remains protected at coarser granularity while the tails are censored.
- Native risk metric: [RMDID1](../../risk_metrics/rmdid1.md) — top/bottom coding directly targets robust-Mahalanobis outliers; complement with [Individual Risk](../../risk_metrics/individual_risk.md) to check whether the thresholds are aggressive enough.
- Native utility metric: [λ](../../information_loss_metrics/lambda_measure.md), where the loss is concentrated on the censored tail records but contributes proportionally to the column mean.

## References

- Hundepool et al. (2014), Chapter 5.
- Willenborg and De Waal (2001). *Elements of Statistical Disclosure Control*. Springer.
- [docs/11 § Top and Bottom Coding](../../../docs/11-sdc-microdata-protection-methods.md#top-and-bottom-coding).
