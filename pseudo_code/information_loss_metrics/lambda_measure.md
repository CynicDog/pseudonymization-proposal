# λ Measure

> Per-column mean absolute deviation between original and protected values, normalised by the original's range, then averaged across columns — the handbook's primary scalar utility summary.

## Inputs

- `X`, `Z` — original and protected microdata, row-aligned (`n × p`, continuous).

## Outputs

- `λ_k` — per-column information-loss measure for each `k ∈ {1, …, p}`.
- `λ` — file-level mean across columns.

## Algorithm

1. For each column `k ∈ {1, …, p}`:
   - Compute the range `R_k ← max(x_{·k}) - min(x_{·k})`.
   - If `R_k = 0` (constant column), set `λ_k ← 0` and skip the next step.
   - Otherwise compute
     $$
     \lambda_k := \frac{1}{n} \sum_{i=1}^{n} \frac{|x_{ik} - z_{ik}|}{R_k}.
     $$
2. File-level scalar:
   $$
   \lambda := \frac{1}{p} \sum_{k=1}^{p} \lambda_k.
   $$
3. Return `({λ_k}_k, λ)`.

## Notes

- `λ_k ∈ [0, 1]`: a constant column or unchanged column reports `0`; a column where every protected value is at the opposite end of the range reports `1`.
- Range normalisation makes columns of different units (counts, currencies, ages) directly comparable in the file-level mean.
- The metric is a mean and so insensitive to the tail of the distortion distribution. A small number of strongly perturbed records can coexist with low `λ`. Pair with [IL1s](il1s.md), which is σ-normalised and tail-sensitive.
- For categorical variables, use [KL Divergence](kl_divergence.md) instead.
- For [Local Suppression](../sdc_methods/non_perturbative/local_suppression.md), each `*` cell counts as full range-fraction loss `|x_{ik} - z_{ik}| / R_k := 1`.
- Typical operational thresholds: `λ < 0.10` for moderately sensitive releases; `λ < 0.05` for analytical-priority releases.

## References

- Mateo-Sanz, Domingo-Ferrer and Sebé (2005). Probabilistic information loss measures in confidentiality protection of continuous microdata. *DMKD* 11(2):181–193.
- Hundepool et al. (2014), Chapter 4.
- Templ, Meindl, Kowarik and Chen (2014), §4 (IHSN).
- [docs/12 § λ Measure](../../docs/12-sdc-risk-and-utility-metrics.md#-measure).
