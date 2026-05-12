# Resampling

> Draw `t` independent samples from the original data, sort each by the variable of interest, then replace each record's value with the average of the corresponding rank values across samples.

$$
Z^{(j)} := \frac{1}{t} \sum_{s=1}^{t} x^{(s)}_{(j)}, \quad j = 1, \ldots, n
$$

## Inputs

- `X ∈ ℝ^{n × p}` — original microdata (continuous columns).
- `t ≥ 2` — number of bootstrap samples (typical `t ∈ [10, 100]`).

## Outputs

- `Z ∈ ℝ^{n × p}` — protected microdata.

## Algorithm

1. For each column `k ∈ {1, …, p}`:
   - For each `s ∈ {1, …, t}`:
     - Draw bootstrap sample `B^{(s)}_k` of size `n` from `x_{·k}` (simple random sample with replacement).
     - Sort `B^{(s)}_k` ascending; let `x^{(s)}_{(j), k}` denote its `j`-th order statistic.
   - Compute the averaged order statistics:
     $$
     y^{(j)}_k := \frac{1}{t} \sum_{s=1}^{t} x^{(s)}_{(j), k}, \quad j = 1, \ldots, n.
     $$
   - Compute the rank of each original value `x_{ik}` in `x_{·k}`: `rank_k(i) ∈ {1, …, n}` (break ties consistently).
   - Assign the protected value by rank: `z_{ik} ← y^{(rank_k(i))}_k`.
2. Return `Z`.

## Notes

- The rank order of each column is preserved exactly: `rank(z_{·k}) = rank(x_{·k})`.
- Column means are exactly preserved: `mean(z_{·k}) = mean(y^{(j)}_k) = mean(x_{·k})` (averaging preserves means).
- As `t → ∞`, the averaged order statistics converge to the expected order statistics of the underlying distribution — smooth and continuous even when the empirical distribution has ties or spikes.
- Cross-column correlations are approximately preserved because each variable is rank-aligned with its original; correlation is mediated by the rank correspondence across variables.
- Especially effective for heavy-tailed right-skewed columns (income, expenditure) where a few exact extreme values dominate disclosure risk.
- Native risk metric: [Record Linkage](../../risk_metrics/record_linkage_risk.md) plus [RMDID1](../../risk_metrics/rmdid1.md) for tail records.
- Native utility metric: [λ](../../information_loss_metrics/lambda_measure.md); secondary check with [Eigenvalue Spectrum](../../information_loss_metrics/eigenvalue_spectrum.md) because resampling tends to flatten the covariance spectrum.

## References

- Heer (1993).
- Domingo-Ferrer and Mateo-Sanz (1999).
- [docs/11 § Resampling](../../../docs/11-sdc-microdata-protection-methods.md#resampling).
