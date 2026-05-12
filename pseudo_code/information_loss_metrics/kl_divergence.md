# KL Divergence (Categorical)

> Per-column Kullback-Leibler divergence between the original and protected category-frequency distributions — the natural loss measure for categorical perturbation.

## Inputs

- `X`, `Z` — original and protected microdata, with at least one categorical column.
- `K_cat ⊆ {1, …, p}` — set of categorical columns to compare.
- Smoothing parameter `ε > 0` for Laplace smoothing (typical `ε = 10⁻⁶`).

## Outputs

- `D_KL,k` — per-column KL divergence for each `k ∈ K_cat`.
- `D_KL` — file-level summary (mean over `K_cat`).

## Algorithm

1. For each categorical column `k ∈ K_cat`:
   - Let `V_k := set of values appearing in either x_{·k} or z_{·k}`.
   - For each value `v ∈ V_k`:
     - `P_k(v) ← |{i : x_{ik} = v}| / n` (original marginal).
     - `Q_k(v) ← |{i : z_{ik} = v}| / n` (protected marginal).
   - Apply Laplace smoothing to `Q_k`: replace `Q_k(v) ← (Q_k(v) · n + ε) / (n + ε · |V_k|)` to avoid division by zero where `Q_k(v) = 0` but `P_k(v) > 0`.
   - Compute
     $$
     D_{\mathrm{KL},k} := \sum_{v \in V_k} P_k(v) \cdot \log \frac{P_k(v)}{Q_k(v)},
     $$
     omitting terms with `P_k(v) = 0` (by convention `0 · log 0 := 0`).
2. File-level summary `D_KL ← (1 / |K_cat|) Σ_{k ∈ K_cat} D_{KL,k}`.
3. Return `({D_KL,k}_k, D_KL)`.

## Notes

- KL divergence is **asymmetric**: by convention the original `P` is the first argument, so the divergence measures information lost when treating `Q` as an approximation to `P`.
- For [PRAM](../sdc_methods/perturbative/pram.md) under the invariant-marginal design constraint `π^T P = π^T`, the divergence approaches zero by construction; non-trivial deviation reflects sample-level variance around an unbiased target.
- Captures only the **marginal** distribution. It does not detect changes in the *joint* distribution of two categorical variables. For joint utility, run KL on a cross-tabulation or use the chi-squared statistic.
- Log base is conventional: `log_2` gives bits, `log_e` gives nats. Publish the choice.
- For [Local Suppression](../sdc_methods/non_perturbative/local_suppression.md), include the suppression symbol `*` as a value in `V_k`.

## References

- Kullback and Leibler (1951). On information and sufficiency. *Annals of Mathematical Statistics* 22:79–86.
- Domingo-Ferrer, Mateo-Sanz and Torra (2001). Comparing SDC methods for microdata. *ETK-NTTS*.
- Hundepool et al. (2014), Chapter 4.
- [docs/12 § KL Divergence](../../docs/12-sdc-risk-and-utility-metrics.md#kl-divergence-categorical).
