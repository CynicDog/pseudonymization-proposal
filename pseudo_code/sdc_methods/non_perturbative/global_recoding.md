# Global Recoding

> Replace original fine-grained categories or measurement values with coarser groupings applied uniformly to every record, reducing the granularity of the entire variable.

$$
x' := g(x), \quad g: \mathcal{X} \to \mathcal{X}', \quad |\mathcal{X}'| < |\mathcal{X}|
$$

## Inputs

- `X` — microdata (categorical or continuous columns).
- For each column `k` to recode:
  - **continuous**: ordered cut-points `c_0 < c_1 < … < c_M` defining `M` intervals.
  - **categorical**: a many-to-one mapping `g_k : C_k → C'_k` with `|C'_k| < |C_k|`.

## Outputs

- `Z` — protected microdata with recoded columns.
- `{g_k}_k` — the publicly documented mapping (this may be released).

## Algorithm

1. For each column `k` to recode:
   - If continuous, define `g_k(x) := [c_{j-1}, c_j)` for the unique `j` with `c_{j-1} ≤ x < c_j`. Equivalently, assign a single label per interval (e.g. interval midpoint, or `"[c_{j-1}, c_j)"`).
   - If categorical, the practitioner supplies `g_k` directly (e.g. ISCO-08 5-digit → ISCO-08 1-digit; rare ethnic categories → `"other"`).
   - For each record `i`, set `z_{ik} ← g_k(x_{ik})`.
2. Return `(Z, {g_k})`. The mapping `g_k` may be published alongside the dataset.

## Notes

- **Transparent and reversible** in the sense that the mapping is public; an analyst can re-aggregate other variables to the same granularity.
- Applies **uniformly** to all records, including those with low disclosure risk — this is inefficient compared to record-level methods. Common practice: apply recoding first, then [Local Suppression](local_suppression.md) for residual cases.
- Cut-points should be chosen to balance utility against k-anonymity. Common heuristics: equal-frequency bins (`g_k` puts ≈ `n/M` records in each bin); domain-meaningful bands (5-year age groups, official income brackets).
- Native risk metric: [k-Anonymity](../../risk_metrics/k_anonymity.md) on the recoded quasi-identifier combination; residual [SUDA Score](../../risk_metrics/suda_score.md) as diagnostic.
- Native utility metrics: [λ](../../information_loss_metrics/lambda_measure.md) (loss is the interval-midpoint distance from the original); [KL Divergence](../../information_loss_metrics/kl_divergence.md) on the recoded marginal.
- For [Top and Bottom Coding](top_bottom_coding.md), the bottom interval `(-∞, c_1)` collapses to `T_bot` and the top interval `[c_{M-1}, ∞)` collapses to `T_top`; otherwise the construction is identical.

## References

- De Waal and Willenborg (1995, 1999).
- Hundepool et al. (2014), Chapter 5.
- [docs/11 § Global Recoding](../../../docs/11-sdc-microdata-protection-methods.md#global-recoding).
