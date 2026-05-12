# Global Risk Rate ξ

> The mean individual risk across the file — the single scalar that summarises the disclosure risk of the full release.

## Inputs

- `{r_i}_{i=1..n}` — per-record individual risks from [Individual Risk (Benedetti-Franconi)](individual_risk.md).
- `ξ*` — pre-declared acceptable threshold (typical operational targets: `0.01 ≤ ξ* ≤ 0.05`).

## Outputs

- `ξ` — file-level mean risk.
- `acceptable := (ξ ≤ ξ*)` — boolean release decision indicator.

## Algorithm

1. Compute `ξ := (1/n) Σ_{i=1}^{n} r_i`.
2. Compare against the pre-declared threshold: `acceptable ← (ξ ≤ ξ*)`.
3. Return `(ξ, acceptable)`.

## Notes

- `ξ` is the headline risk scalar reported alongside the headline utility scalar [λ](../information_loss_metrics/lambda_measure.md) at the end of each SDC iteration.
- The mean hides tail records. A release that passes on `ξ` may still contain a small number of records with very high `r_i` — exactly the records an adversary targets. Always pair with [Benchmark Risk Outlier](benchmark_risk_outlier.md).
- `ξ` is insensitive to attribute disclosure (sensitive value inference within a class). Read alongside [l-Diversity](l_diversity.md) and [t-Closeness](t_closeness.md) when sensitive variables are present.
- For grouped data, compute `ξ` using the household-escalated risks from [Hierarchical (Household) Risk](hierarchical_risk.md), not the raw per-record risks.

## References

- Benedetti and Franconi (1998).
- Hundepool et al. (2014), Chapter 4.
- Templ, Meindl, Kowarik and Chen (2014), §3 (IHSN).
- [docs/12 § Global Risk Rate ξ](../../docs/12-sdc-risk-and-utility-metrics.md#global-risk-rate-).
