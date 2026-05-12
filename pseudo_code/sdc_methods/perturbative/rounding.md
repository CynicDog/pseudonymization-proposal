# Rounding

> Replace each continuous value with a nearby multiple of a chosen base, reducing precision uniformly across the variable.

$$
x' \in \{u b,\, (u+1) b\}, \quad u := \lfloor x / b \rfloor
$$

## Inputs

- `X ∈ ℝ^{n × p}` — original microdata (continuous columns).
- `b_k > 0` — rounding base for column `k`.
- `mode ∈ {zero_restricted_down, zero_restricted_up, deterministic_nearest, random_unbiased}`.

## Outputs

- `Z ∈ ℝ^{n × p}` — protected microdata, every value is a multiple of `b_k`.

## Algorithm

1. For each column `k` and each record `i`:
   - Compute `u_{ik} ← ⌊x_{ik} / b_k⌋` and the fractional part `f_{ik} ← (x_{ik} - u_{ik} · b_k) / b_k ∈ [0, 1)`.
   - Decide the rounding direction according to `mode`:
     - **zero_restricted_down**: `z_{ik} ← u_{ik} · b_k` (always round to the lower adjacent multiple).
     - **zero_restricted_up**: `z_{ik} ← (u_{ik} + 1) · b_k`.
     - **deterministic_nearest**: `z_{ik} ← u_{ik} · b_k` if `f_{ik} < 0.5`, else `(u_{ik} + 1) · b_k`.
     - **random_unbiased** (probabilistic / "random rounding"): with probability `1 - f_{ik}` set `z_{ik} ← u_{ik} · b_k`, else `z_{ik} ← (u_{ik} + 1) · b_k`. Under this rule `E[z_{ik}] = x_{ik}`, making frequency-estimator output unbiased.
2. Return `Z`.

## Notes

- The protected value is contained in the analytically known interval `[u_{ik} · b_k, (u_{ik} + 1) · b_k]`. This makes [Interval Disclosure Risk](../../risk_metrics/interval_disclosure_risk.md) **exact**, not estimated.
- Information loss is bounded by `b_k / 2` per record under deterministic-nearest; native utility metric is [λ](../../information_loss_metrics/lambda_measure.md), which directly averages this bounded loss.
- Choose `b_k` relative to the column's dispersion: a base spanning several standard deviations gives strong protection but heavy utility loss. A common rule is `b_k ≈ σ_k / 4` to `σ_k / 2`.
- Random rounding is the variant used in official statistics for cell counts because it preserves expected totals.
- For categorical variables, the analogous operation is [Global Recoding](../non_perturbative/global_recoding.md).
- Frequently combined with [Top and Bottom Coding](../non_perturbative/top_bottom_coding.md) to handle outliers that lie far from any rounded multiple.

## References

- Willenborg and De Waal (2001). *Elements of Statistical Disclosure Control*. Springer.
- [docs/11 § Rounding](../../../docs/11-sdc-microdata-protection-methods.md#rounding).
