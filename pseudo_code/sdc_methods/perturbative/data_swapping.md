# Data Swapping

> Exchange the values of sensitive variables between pairs (or groups) of records that are similar on non-sensitive background variables, so marginal distributions are preserved while the linkage to identifying attributes is broken.

$$
\text{For records } (r_a, r_b) \text{ with } \mathrm{dist}(r_a, r_b) < \delta:\quad s(r_a) \leftrightarrow s(r_b)
$$

## Inputs

- `X` — microdata with `n` records.
- `B ⊆ {1, …, p}` — index set of background (matching) variables.
- `S ⊆ {1, …, p}` — index set of sensitive variables to swap.
- `δ > 0` — proximity threshold on background variables.
- `α ∈ (0, 1]` — swap fraction (`1` = full swap; typical partial swap: `0.10` – `0.20`).
- `d_B(·, ·)` — distance on background variables.

## Outputs

- `Z` — protected microdata with swapped sensitive values.

## Algorithm

1. Initialise `Z ← X`. Mark all records as unmatched.
2. Choose the swap set: select `m := ⌊α · n⌋` records uniformly at random from `{1, …, n}`. Records outside this set are not modified.
3. For each record `r_a` in the swap set (in random order, while still unmatched):
   - Search for `r_b` among the *unmatched* records (excluding `r_a`) satisfying `d_B(B(r_a), B(r_b)) < δ`.
   - If multiple candidates exist, choose `r_b ← arg min_b d_B(B(r_a), B(r_b))` (closest neighbour). If no candidate exists, either (a) relax `δ`, or (b) leave `r_a` unmatched.
   - If matched: swap sensitive values, `Z[r_a, S] ← X[r_b, S]` and `Z[r_b, S] ← X[r_a, S]`. Mark `r_a, r_b` as matched.
4. Return `Z`.

## Notes

- The marginal distribution of every swapped column in `Z` is *exactly* identical to that of `X` (no values are added or removed, only permuted). This is the defining utility property of swapping.
- The joint distribution between `S` and `B` is preserved approximately because swapped pairs are close on `B`. Cross-column correlations within `S` are preserved when *all* sensitive columns are swapped together for the same pair.
- Protection is *probabilistic per record*: an intruder who learns which records were swapped (or unswapped) can re-identify them. Full swapping (`α = 1`) maximises this uncertainty.
- The matching geometry can use rank-based tolerances rather than absolute `δ` — see [Rank Swapping](rank_swapping.md) for the continuous variant.
- Native risk metric: top-1 [Record Linkage](../../risk_metrics/record_linkage_risk.md) on swapped attributes; when `α < 1` supplement with [Individual Risk](../../risk_metrics/individual_risk.md).
- Native utility metric: [KL Divergence](../../information_loss_metrics/kl_divergence.md) on the swapped variable's marginal (≈ 0 by construction); track joint distortion with [λ](../../information_loss_metrics/lambda_measure.md) or a cross-tabulation chi-squared.

## References

- Dalenius and Reiss (1978), `papers/perturbative/Dalenius-Reiss-1978-data-swapping-disclosure.pdf`.
- Reiss, Post and Dalenius (1982); Reiss (1984).
- Fienberg and McIntyre (2004), `papers/perturbative/Fienberg-McIntyre-2004-data-swapping-variations.pdf`.
- [docs/11 § Data Swapping](../../../docs/11-sdc-microdata-protection-methods.md#data-swapping).
