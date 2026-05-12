# SSE / SST Ratio

> Within-group sum of squares divided by total sum of squares, after partitioning records into the groups produced by [Microaggregation](../sdc_methods/perturbative/microaggregation.md) — directly the quantity MDAV minimises.

## Inputs

- `X` — original microdata (`n × p`, continuous).
- `{G_1, …, G_G}` — partition of record indices into groups of size ≥ `k`, produced by a microaggregation routine.

## Outputs

- `SSE / SST` — scalar utility loss in `[0, 1]`.
- Optionally per-column ratios `(SSE_k / SST_k)` for `k ∈ {1, …, p}`.

## Algorithm

1. Compute the overall centroid `x̄ ← (1/n) Σ_{i=1}^{n} x_i`.
2. Compute the total sum of squares
   $$
   \mathrm{SST} := \sum_{i=1}^{n} \|x_i - \bar{x}\|^2.
   $$
3. For each group `g ∈ {1, …, G}`:
   - Compute the group centroid `x̄_g ← (1/|G_g|) Σ_{i ∈ G_g} x_i`.
4. Compute the within-group sum of squares
   $$
   \mathrm{SSE} := \sum_{g=1}^{G} \sum_{i \in G_g} \|x_i - \bar{x}_g\|^2.
   $$
5. Return the ratio `SSE / SST`. Optionally compute per-column variants by replacing the squared norm with the per-column squared deviation.

## Notes

- `SSE / SST ∈ [0, 1]`: zero if every group is a singleton (no information loss, no protection); one if all records are in a single group (max loss, max protection).
- The MDAV objective minimises `SSE` directly, so this is the native loss measure for microaggregation-based methods and the natural utility report for [MASSC](../sdc_methods/perturbative/massc.md) Step 1.
- The metric measures *variance preservation*, not record-level distortion. Two methods can have similar `λ` but very different SSE/SST.
- Per-column reporting reveals which columns have suffered the most aggregation.
- Complexity: linear in `n · p` after the partition is given.

## References

- Domingo-Ferrer and Mateo-Sanz (2002), `papers/perturbative/Domingo-Ferrer-Mateo-Sanz-2002-practical-data-oriented-microaggregation.pdf`.
- Mateo-Sanz, Domingo-Ferrer and Sebé (2005). *DMKD* 11(2):181–193.
- Hundepool et al. (2014), Chapter 4.
- [docs/12 § SSE / SST Ratio](../../docs/12-sdc-risk-and-utility-metrics.md#sse--sst-ratio).
