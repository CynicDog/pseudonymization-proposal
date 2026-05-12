# Rank Swapping

> Sort values of each continuous or ordinal variable, then randomly exchange values between record pairs whose ranks differ by no more than `p%` of the total, preserving the overall rank distribution.

$$
\text{Swap } x_{(i)} \leftrightarrow x_{(j)} \quad \text{if } |i - j| \leq \lfloor p \cdot n / 100 \rfloor
$$

## Inputs

- `X ∈ ℝ^{n × p}` — original microdata (continuous or ordinal columns).
- `p%` — rank-window width as percentage of `n` (typical `10` – `30`).

## Outputs

- `Z ∈ ℝ^{n × p}` — protected microdata.

## Algorithm

1. For each column `k ∈ {1, …, p}`:
   - Compute the rank window width `w ← ⌊p% · n / 100⌋`.
   - Sort records by value `x_{·k}` ascending, giving ordered indices `π_k(1), …, π_k(n)` such that `x_{π_k(1), k} ≤ x_{π_k(2), k} ≤ …`.
   - Initialise a list `swapped ← []` (records already involved in a swap on this column).
   - For each rank position `i ∈ {1, …, n}` (in random order):
     - If `π_k(i) ∈ swapped`, skip.
     - Choose a partner rank `j` uniformly at random from `{max(1, i - w), …, min(n, i + w)} \ {i}` with `π_k(j) ∉ swapped`.
     - If no valid partner exists, skip.
     - Exchange values: temporarily swap `x_{π_k(i), k}` and `x_{π_k(j), k}` in a working column copy.
     - Mark `π_k(i)` and `π_k(j)` as swapped.
2. After processing all columns, the working data is `Z`.
3. Return `Z`.

## Notes

- Treats each column **independently** — this is the property that distorts cross-column correlations and is the reason to pair the method with [Regression Coefficient Comparison](../../information_loss_metrics/regression_coefficient.md).
- The rank window `w` controls the protection-utility trade-off: larger `w` → values travel farther → stronger protection but more distributional distortion.
- For ordinal categorical variables, the same procedure applies after mapping categories to ranks.
- Distributional shape is broadly preserved because each value lands within `w` rank positions of its original; extreme tails are perturbed only slightly (Moore 1996).
- Native risk metric: [Record Linkage](../../risk_metrics/record_linkage_risk.md) (top-1 and top-2).
- Native utility metric: [λ](../../information_loss_metrics/lambda_measure.md); supplement with [Regression Coefficient Comparison](../../information_loss_metrics/regression_coefficient.md) to detect cross-column attenuation.

## References

- Greenberg (1987).
- Moore (1996).
- Domingo-Ferrer and Torra (2001), `papers/perturbative/Domingo-Ferrer-Torra-2001-disclosure-protection-information-loss.pdf`.
- [docs/11 § Rank Swapping](../../../docs/11-sdc-microdata-protection-methods.md#rank-swapping).
