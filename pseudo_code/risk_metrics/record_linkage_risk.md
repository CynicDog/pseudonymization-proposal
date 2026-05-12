# Record Linkage Risk (Distance-Based)

> Treat the protected file as the adversary's input and the original file as the target; measure the share of protected records whose nearest original neighbour by standardised distance is the correct pre-perturbation record.

## Inputs

- `X` — original microdata (`n × p`, continuous).
- `Z` — protected microdata (`n × p`, aligned row-wise to `X`).
- `scale ∈ {sample_sd, population_sd, IQR}` — standardisation rule for the per-column scale `σ_k` (publish alongside the result).

## Outputs

- `RL_1` — lower bound on linkage risk (share of records whose top-1 nearest original is the correct record).
- `RL_2` — upper bound (share whose correct original is among the top-2 nearest).
- Interval `[RL_1, RL_2]`.

## Algorithm

1. For each column `k ∈ {1, …, p}`, compute the chosen scale `σ_k` from `X` according to `scale`.
2. Standardise: `x̃_{ik} ← x_{ik} / σ_k`, `z̃_{ik} ← z_{ik} / σ_k` for all `i, k`.
3. For each masked record `i ∈ {1, …, n}`:
   - Compute distances `d_{ij} ← ‖z̃_i - x̃_j‖_2` for `j ∈ {1, …, n}`.
   - Let `j*_1(i) ← arg min_j d_{ij}` (top-1 nearest original).
   - Let `(j*_1(i), j*_2(i))` be the two smallest by `d_{ij}` (top-2).
4. Aggregate:
   $$
   \mathrm{RL}_1 := \frac{1}{n} \sum_{i=1}^{n} \mathbf{1}[j^*_1(i) = i],
   $$
   $$
   \mathrm{RL}_2 := \frac{1}{n} \sum_{i=1}^{n} \mathbf{1}[i \in \{j^*_1(i), j^*_2(i)\}].
   $$
5. Return `(RL_1, RL_2)`.

## Notes

- The metric requires row-alignment between `X` and `Z`. If the protection method has shuffled rows (e.g. [Data Swapping](../sdc_methods/perturbative/data_swapping.md), [Microaggregation](../sdc_methods/perturbative/microaggregation.md) followed by reordering), preserve the original index during masking.
- The result is sensitive to the choice of `scale`; population σ, sample σ, and IQR give meaningfully different rates. Always publish the rule.
- For categorical attributes use Hamming distance or a domain-specific metric in place of Euclidean; mixed-type data requires a Gower-style distance.
- Complexity: `O(n² · p)` naïvely. Use a KD-tree / ball tree for `p` small or LSH for `p` large.
- Reported in IHSN-style intervals: MDAV microaggregation with `k = 3` typically gives `[0%, 61%]`.

## References

- Domingo-Ferrer and Mateo-Sanz (2002), `papers/perturbative/Domingo-Ferrer-Mateo-Sanz-2002-practical-data-oriented-microaggregation.pdf`.
- Pagliuca and Seri (1999). Some results of individual ranking method on the system of enterprise accounts annual survey. *ESPRIT SDC*.
- Hundepool et al. (2014), Chapter 4.
- [docs/12 § Record Linkage Risk](../../docs/12-sdc-risk-and-utility-metrics.md#record-linkage-risk-distance-based).
