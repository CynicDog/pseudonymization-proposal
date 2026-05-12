# IL1s — Yancey-Winkler Scaled Distance

> Sum of absolute deviations between original and protected values, scaled per column by the standard deviation — more sensitive than λ to large per-record deviations.

## Inputs

- `X`, `Z` — original and protected microdata, row-aligned (`n × p`, continuous).

## Outputs

- `IL1s` — file-level scalar utility loss.
- Optionally `IL1s_k` per column.

## Algorithm

1. For each column `k ∈ {1, …, p}`, compute the sample standard deviation `s_k` of the original column. If `s_k = 0`, exclude column `k` from the computation (undefined).
2. For each record `i` and each retained column `k`, compute the scaled per-cell deviation `δ_{ik} ← |x_{ik} - z_{ik}| / s_k`.
3. Aggregate:
   $$
   \mathrm{IL1s} := \frac{1}{n \cdot p'} \sum_{i=1}^{n} \sum_{k=1}^{p'} \delta_{ik},
   $$
   where `p'` is the number of retained (non-constant) columns. Per-column `IL1s_k := mean_i δ_{ik}`.
4. Return `(IL1s, {IL1s_k}_k)`.

## Notes

- σ-normalisation differs sharply from λ's range-normalisation: in a heavy-tailed distribution the range can be many multiples of the standard deviation, so a deviation that contributes modestly to `λ` contributes proportionally more to `IL1s`.
- Report alongside [λ](lambda_measure.md): `λ` for the global summary, `IL1s` for tail-sensitivity. Where `IL1s ≫ λ`, examine which column is driving the gap.
- Affected disproportionately by [Top and Bottom Coding](../sdc_methods/non_perturbative/top_bottom_coding.md), since both the censored deviation and the original `s_k` move together.
- Undefined for constant columns (`s_k = 0`); they must be excluded.

## References

- Yancey, Winkler and Creecy (2002). Disclosure risk assessment in perturbative microdata protection. *Inference Control in Statistical Databases*, LNCS 2316.
- Mateo-Sanz, Domingo-Ferrer and Sebé (2005). *DMKD* 11(2):181–193.
- Hundepool et al. (2014), Chapter 4.
- [docs/12 § IL1s](../../docs/12-sdc-risk-and-utility-metrics.md#il1s--yancey-winkler-scaled-distance).
