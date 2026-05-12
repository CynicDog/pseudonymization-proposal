# Multiplicative Noise

> Multiply each original value by an independent positive random factor, then rescale to restore the first two moments.

$$
X^a_{ij} := Z_{ij} \cdot X_{ij}, \qquad X^{aR}_i := \frac{\sigma_X}{\sigma_{X^a}}\left(X^a_i - \mu_{X^a}\right) + \mu_X
$$

## Inputs

- `X ∈ ℝ^{n × p}` — original microdata (continuous, strictly positive columns).
- Noise distribution `f_Z` — typically log-normal centred at 1 with parameter `σ_log > 0` (e.g. `Z ~ LogNormal(0, σ_log²)` so `E[Z] = exp(σ_log²/2)`).

## Outputs

- `X^{aR} ∈ ℝ^{n × p}` — protected microdata with the first two moments restored to those of `X`.

## Algorithm

1. For each record `i` and each column `k`:
   - Draw `Z_{ik}` independently from `f_Z` (positive multiplicative factor).
   - Form the raw perturbed value `X^a_{ik} ← Z_{ik} · X_{ik}`.
2. For each column `k`:
   - Compute the original moments `μ_X,k ← mean(X_{·k})`, `σ_X,k ← sd(X_{·k})`.
   - Compute the raw perturbed moments `μ_{X^a},k ← mean(X^a_{·k})`, `σ_{X^a},k ← sd(X^a_{·k})`.
3. Apply the moment-preserving rescaling (Höhne 2004): for each record `i` and column `k`,
   $$
   X^{aR}_{ik} := \frac{\sigma_{X,k}}{\sigma_{X^a,k}}\,(X^a_{ik} - \mu_{X^a,k}) + \mu_{X,k}.
   $$
4. Return `X^{aR}`.

## Notes

- Strictly applicable to positive variables; the log-normal multiplier keeps `Z_{ik} · X_{ik}` positive almost surely.
- After step 3 the first two moments match exactly; higher moments and cross-column correlations are approximately preserved, with distortion controlled by `σ_log`.
- Variance of `Z` must be chosen so that `Z · X` remains in a plausible range. A common choice is `σ_log ∈ [0.05, 0.25]`.
- Particularly suitable for income, wealth, turnover — variables where additive noise of fixed variance is implausibly small for large values and dominates small ones.
- Native risk metric: [Record Linkage](../../risk_metrics/record_linkage_risk.md), supplemented by [RMDID1](../../risk_metrics/rmdid1.md) because perturbation scales with magnitude.
- Native utility metric: [λ](../../information_loss_metrics/lambda_measure.md); verify no bias in conditional means via [Regression Coefficient Comparison](../../information_loss_metrics/regression_coefficient.md).

## References

- Höhne (2004). Varianten zur multiplikativen Anonymisierung. *Statistisches Bundesamt*.
- [docs/11 § Multiplicative Noise](../../../docs/11-sdc-microdata-protection-methods.md#multiplicative-noise).
