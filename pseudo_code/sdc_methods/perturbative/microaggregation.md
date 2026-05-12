# Microaggregation (MDAV)

> Partition records into groups of at least `k`, then replace each value within a group with the group mean, so no record can be distinguished from the others in its group.

$$
\text{Minimise} \quad \mathrm{SSE} = \sum_{g=1}^{G} \sum_{j \in G_g} (\mathbf{x}_j - \bar{\mathbf{x}}_g)^\top (\mathbf{x}_j - \bar{\mathbf{x}}_g)
$$

## Inputs

- `X ∈ ℝ^{n × p}` — original microdata (continuous, or categorical with an admissible distance).
- `k ≥ 2` — minimum group size (typical: 3 or 5).
- `d(·, ·)` — distance function on the value space (Euclidean for continuous; user-supplied for categorical, e.g. Torra 2004).

## Outputs

- Partition `{G_1, …, G_G}` of records into groups of size ≥ `k`.
- `Z ∈ ℝ^{n × p}` — protected microdata, where `z_i := centroid(G_{g(i)})` for the group `g(i)` containing `i`.

## Algorithm (MDAV — Maximum Distance to Average Vector)

1. Initialise `U ← {1, …, n}` (the set of unassigned record indices). Initialise the partition `P ← ∅`.
2. While `|U| ≥ 3 · k`:
   - Compute the centroid `c̄ ← mean({x_i : i ∈ U})` of the unassigned records.
   - Find `r ← arg max_{i ∈ U} d(x_i, c̄)` (record farthest from the centroid).
   - Form group `G_r ← {r}` ∪ {(k - 1) records in `U` closest to `x_r` by `d(·, ·)`}.
   - Find `s ← arg max_{i ∈ U \ G_r} d(x_i, x_r)` (record farthest from `r` among remaining).
   - Form group `G_s ← {s}` ∪ {(k - 1) records in `U \ G_r` closest to `x_s` by `d(·, ·)`}.
   - Append `G_r, G_s` to `P`. Update `U ← U \ (G_r ∪ G_s)`.
3. While `|U| ≥ 2 · k` (only one extreme group can be formed):
   - Compute `c̄ ← mean({x_i : i ∈ U})`.
   - Find `r ← arg max_{i ∈ U} d(x_i, c̄)`.
   - Form `G_r ← {r}` ∪ {(k - 1) records closest to `x_r`}.
   - Append `G_r` to `P`; update `U ← U \ G_r`.
4. If `|U| > 0`, form a final group `G_final ← U` and append to `P`. By construction `|G_final| ≥ k`.
5. For each group `G_g ∈ P`:
   - Compute the centroid `x̄_g ← (1/|G_g|) Σ_{i ∈ G_g} x_i`.
   - For each `i ∈ G_g`, set `z_i ← x̄_g`.
6. Return `(P, Z)`.

## Notes

- MDAV guarantees `k`-anonymity on the microaggregated variables by construction: every record's masked value equals the centroid shared by ≥ `k - 1` other records.
- For the univariate case, Hansen and Mukherjee (2003) showed that the optimal partition is sorted-value equal-sized groups of exactly `k`; the dynamic-programming algorithm achieves the exact optimum in `O(n · k)`.
- For multivariate continuous data, optimal microaggregation is NP-hard (Oganian and Domingo-Ferrer 2001); MDAV runs in `O(n²)` per pass and consistently achieves near-optimal SSE.
- For **categorical** variables, replace `mean` with `mode` and use a categorical distance (Torra 2004); the algorithm structure is unchanged.
- Information loss grows with `k` and with within-group heterogeneity. Native utility metric: [SSE / SST](../../information_loss_metrics/sse_sst_ratio.md) (which MDAV minimises directly).
- Native risk guarantee: [k-Anonymity](../../risk_metrics/k_anonymity.md) by construction; residual disclosure should still be checked via top-1 [Record Linkage](../../risk_metrics/record_linkage_risk.md).

## References

- Defays and Nanopoulos (1993).
- Mateo-Sanz and Domingo-Ferrer (1999).
- Domingo-Ferrer and Mateo-Sanz (2002), `papers/perturbative/Domingo-Ferrer-Mateo-Sanz-2002-practical-data-oriented-microaggregation.pdf`.
- Hansen and Mukherjee (2003).
- Oganian and Domingo-Ferrer (2001), `papers/perturbative/Oganian-Domingo-Ferrer-2001-complexity-optimal-microaggregation.pdf`.
- Torra (2004); Domingo-Ferrer and Torra (2005), `papers/perturbative/Domingo-Ferrer-Torra-2005-ordinal-k-anonymity-microaggregation.pdf`.
- [docs/11 § Microaggregation](../../../docs/11-sdc-microdata-protection-methods.md#microaggregation).
