# k-Anonymity

> Every combination of quasi-identifier values must appear in at least `k` records, so no record can be exact-matched on those quasi-identifiers alone.

## Inputs

- `X` — microdata table with `n` records and `p` columns.
- `K ⊆ {1, …, p}` — index set of quasi-identifier columns.
- `k ≥ 2` — anonymity threshold (typical: 3 or 5).

## Outputs

- `violations` — count of records whose equivalence class is smaller than `k`.
- `violation_rate := violations / n`.
- `sample_uniques` — count of records with equivalence-class size exactly 1.
- `holds := (violations = 0)` — boolean indicator that the file is `k`-anonymous on `K`.

## Algorithm

1. Project each record `x_i` onto its quasi-identifier tuple: `q_i := Q(x_i)`, where `Q(·)` selects columns indexed by `K`.
2. Group records by tuple equality on `q_i`. For each distinct tuple `q`, let `n_q := |{i : q_i = q}|` be the equivalence-class size.
3. For each record `i`, define its class size `c_i := n_{q_i}`.
4. Compute summaries:
   - `violations ← |{i : c_i < k}|`.
   - `sample_uniques ← |{i : c_i = 1}|`.
   - `violation_rate ← violations / n`.
   - `holds ← (violations = 0)`.
5. Return `(violations, violation_rate, sample_uniques, holds)`.

## Notes

- The metric depends on the choice of `K`. Document which columns are declared quasi-identifiers before computing.
- A sample-unique record (`c_i = 1`) is not necessarily population-unique; for sample-vs-population assessment use [Individual Risk](individual_risk.md).
- `k`-anonymity does not protect against homogeneity or background-knowledge attacks — pair with [l-Diversity](l_diversity.md) for sensitive attributes.
- Implementation cost is one group-by pass over the `K` columns: `O(n · |K|)`.

## References

- Samarati and Sweeney (1998), `papers/non-perturbative/Samarati-Sweeney-1998-k-anonymity-SRI-tech-report.pdf`.
- Sweeney (2002), `papers/non-perturbative/Sweeney-2002-k-anonymity.pdf`.
- Samarati (2001), `papers/non-perturbative/Samarati-2001-protecting-respondents-identities.pdf`.
- [docs/12 § k-Anonymity](../../docs/12-sdc-risk-and-utility-metrics.md#k-anonymity).
