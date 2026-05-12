# l-Diversity

> Within each `k`-anonymous equivalence class the sensitive attribute must take at least `l` "well-represented" values, preventing trivial inference of the sensitive value from group membership.

## Inputs

- `X` — microdata with quasi-identifiers `K` and a designated sensitive column `s ∈ {1, …, p} \ K`.
- `l ≥ 2` — diversity threshold.
- `mode ∈ {distinct, entropy, recursive}` — l-diversity variant.
- `c > 0` — multiplier for the recursive `(c, l)` variant (Machanavajjhala et al. 2007).

## Outputs

- `violations` — count of equivalence classes failing the l-diversity test.
- `violation_rate := violations / G`, where `G` is the number of equivalence classes.
- `holds := (violations = 0)`.

## Algorithm

1. Verify [k-anonymity](k_anonymity.md) holds with `k = l` (necessary precondition: a class cannot be l-diverse if its size is below `l`).
2. Group records by `Q(x_i)`. For each equivalence class `q` with member set `I_q := {i : Q(x_i) = q}`, gather the in-class sensitive multiset `S_q := {s_i : i ∈ I_q}`.
3. For each class `q`, evaluate the chosen variant:
   - **distinct**: pass iff `|set(S_q)| ≥ l`.
   - **entropy**: let `π_q(v) := |{i ∈ I_q : s_i = v}| / |I_q|`. Compute `H_q := - Σ_v π_q(v) · log π_q(v)`. Pass iff `H_q ≥ log l`.
   - **recursive `(c, l)`**: sort the in-class frequencies in decreasing order, `r_1 ≥ r_2 ≥ … ≥ r_m` where `m := |set(S_q)|`. Pass iff `r_1 < c · (r_l + r_{l+1} + … + r_m)`.
4. `violations ← |{q : class q fails}|`. `holds ← (violations = 0)`. Return `(violations, violation_rate, holds)`.

## Notes

- For a sensitive attribute that takes only a few values, even entropy l-diversity may be unsatisfiable without aggressive recoding.
- l-diversity does not bound skewness against the population marginal — use [t-Closeness](t_closeness.md) when the sensitive distribution itself is disclosive.
- If multiple sensitive columns exist, evaluate the test on each column independently and report per-column results.
- Per-class computations are independent and can be parallelised by group.

## References

- Machanavajjhala, Kifer, Gehrke and Venkitasubramaniam (2007). l-Diversity: Privacy beyond k-anonymity. *ACM TKDD* 1(1).
- [docs/12 § l-Diversity](../../docs/12-sdc-risk-and-utility-metrics.md#l-diversity).
