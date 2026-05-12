# Hierarchical (Household) Risk

> When records share a group structure (household, family, firm, schoolclass), the re-identification risk of the group is the union of its members' individual risks: identifying any one member identifies all.

## Inputs

- `{r_i}_{i=1..n}` — per-record individual risks from [Individual Risk](individual_risk.md).
- `h_i ∈ H` — household (or grouping) identifier for record `i`.

## Outputs

- `r_hh(h)` — household-level risk for each group `h`.
- `r̃_i` — escalated per-record risk: every member of household `h` inherits `r_hh(h)`.

## Algorithm

1. Group records by `h_i`. For each group `h ∈ H` with members `I_h := {i : h_i = h}`:
   - Compute the union risk under independence: `r_hh(h) ← 1 - ∏_{i ∈ I_h} (1 - r_i)`.
2. For each record `i`, set the escalated risk: `r̃_i ← r_hh(h_i)`.
3. Return `({r_hh(h)}_{h ∈ H}, {r̃_i}_i)`. Downstream summaries (ξ, benchmark outlier) should use `r̃_i` instead of `r_i`.

## Notes

- Independence is an upper-bound assumption; correlated risks within a household typically yield smaller true `r_hh` but the formula is the operational standard.
- Without escalation, household members each report low individual risks while the group is effectively exposed once any member is matched. A 7-member household with `r_i = 0.1` each has `r_hh ≈ 0.52`.
- Generalises to any nested grouping: firm × employees, schoolclass × pupils, ward × admissions. Declare the grouping column in the pre-processing variable taxonomy.
- Implementation: `group_by(h_i).agg(1 - prod(1 - r_i))` then a join back to record level.

## References

- Hundepool et al. (2014), Chapter 4.
- Templ, Meindl, Kowarik and Chen (2014), §3.5 (IHSN).
- [docs/12 § Hierarchical (Household) Risk](../../docs/12-sdc-risk-and-utility-metrics.md#hierarchical-household-risk).
