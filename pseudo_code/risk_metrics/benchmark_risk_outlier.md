# Benchmark Risk Outlier

> Flag any record whose individual risk exceeds both an absolute threshold and a robust multiple of the median, identifying the tail of the risk distribution that the global mean cannot localise.

## Inputs

- `{r_i}_{i=1..n}` — per-record individual risks from [Individual Risk](individual_risk.md).
- `r*` — absolute risk floor (operationally chosen, e.g. `0.1` = 10% adversary success probability).
- `c > 0` — robust outlier multiplier (typical `c = 3`, matching the MAD outlier rule).

## Outputs

- `flagged` — index set of records flagged as benchmark outliers.
- `count := |flagged|`.
- `close_out := (count = 0)` — boolean: IHSN's recommended SDC-iteration close-out indicator.

## Algorithm

1. Compute `med ← median({r_i})`.
2. Compute `MAD ← median({|r_i - med|})`. (Optionally scale to σ-equivalence by `1.4826 · MAD`.)
3. Form the dynamic threshold `τ ← max(r*, med + c · MAD)`.
4. For each record `i`, flag `i ∈ flagged ⟺ r_i > τ`.
5. Set `count ← |flagged|`, `close_out ← (count = 0)`. Return `(flagged, count, close_out)`.

## Notes

- The `max(absolute floor, robust adaptive)` construction guards both ends: the absolute floor prevents declaring a release safe when many records sit just above adversary-acceptable risk; the MAD adaptive term catches records anomalous *relative to the bulk* even after SDC has pulled the bulk down.
- Records flagged here are the natural candidates for [Local Suppression](../sdc_methods/non_perturbative/local_suppression.md) or further [Global Recoding](../sdc_methods/non_perturbative/global_recoding.md).
- For grouped data, run on the household-escalated risks (see [Hierarchical Risk](hierarchical_risk.md)) rather than raw `r_i`.
- The MAD construction is robust to up to 50% contamination, which is what makes it suitable after several SDC iterations have already shifted the distribution.

## References

- Templ, Meindl, Kowarik and Chen (2014), §3.4 (IHSN).
- Hundepool et al. (2014), Chapter 4.
- [docs/12 § Benchmark Risk Outlier](../../docs/12-sdc-risk-and-utility-metrics.md#benchmark-risk-outlier).
