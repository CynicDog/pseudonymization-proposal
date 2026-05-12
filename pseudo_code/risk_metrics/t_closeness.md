# t-Closeness

> The distribution of the sensitive attribute within every equivalence class must be within Earth Mover's Distance `t` of the population distribution, eliminating skewness-based attribute disclosure.

## Inputs

- `X` — microdata with quasi-identifiers `K` and a sensitive column `s`.
- `t ∈ (0, 1]` — closeness threshold.
- `d(·, ·)` — ground distance on the sensitive value space (Euclidean for ordinal/continuous; user-supplied for nominal).

## Outputs

- `violations` — count of equivalence classes whose EMD to the population marginal exceeds `t`.
- `max_emd := max_q EMD(P(S | Q = q), P(S))`.
- `holds := (violations = 0)`.

## Algorithm

1. Compute the population marginal `π := empirical distribution of s over all n records`.
2. Group by `Q(x_i)`. For each equivalence class `q`, compute the in-class marginal `π_q := empirical distribution of s_i for i ∈ I_q`.
3. For each class `q`, compute `EMD(π_q, π)`:
   - **Ordinal / continuous sensitive**: sort the value space; let `F_q, F` be the cumulative distribution functions of `π_q, π`. Then `EMD(π_q, π) := Σ_v |F_q(v) - F(v)| · Δ_v`, where `Δ_v` is the spacing to the next value.
   - **Nominal sensitive**: solve the Wasserstein-1 transportation LP with cost matrix `d(v, v')` between every pair of categories `(v, v')`. Use any standard EMD solver.
4. Flag class `q` as violating if `EMD(π_q, π) > t`. Aggregate:
   - `violations ← |{q : EMD(π_q, π) > t}|`.
   - `max_emd ← max_q EMD(π_q, π)`.
   - `holds ← (violations = 0)`.
5. Return `(violations, max_emd, holds)`.

## Notes

- t-closeness is strictly stronger than l-diversity: it bounds the *shape* of the in-class distribution against the population, not just its diversity.
- Requires a reference population marginal. If the released file is a sample, use the (design-weighted) sample marginal as the proxy.
- [PRAM](../sdc_methods/perturbative/pram.md) is the natural perturbative pathway to a t-closeness target — its Markov matrix can be designed to push in-class distributions toward `π`.
- LP cost dominates for nominal sensitive variables; for ordinal/continuous the CDF formula is `O(|V|)` per class.

## References

- Li, Li and Venkatasubramanian (2007). t-Closeness: Privacy beyond k-anonymity and l-diversity. *ICDE 2007*.
- [docs/12 § t-Closeness](../../docs/12-sdc-risk-and-utility-metrics.md#t-closeness).
