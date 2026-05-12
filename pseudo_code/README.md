# Pseudo Code Reference

Algorithmic companion to [docs/11](../docs/11-sdc-microdata-protection-methods.md) (SDC methods) and [docs/12](../docs/12-sdc-risk-and-utility-metrics.md) (risk and utility metrics). Each file is implementation-ready pseudocode for one method or metric — numbered steps, English with inline math, no Python syntax. Read these alongside the prose chapters and the primary papers in [`papers/`](../papers/).

## Conventions

- **Indexing.** 1-based across the math. `i ∈ {1, …, n}` ranges over records, `k ∈ {1, …, p}` over variables, `g ∈ {1, …, G}` over groups.
- **Assignment.** `:=` for definition; `←` for procedural assignment inside an algorithm step.
- **Matrices.** `X ∈ ℝ^{n×p}` is the original microdata; `Z ∈ ℝ^{n×p}` is the released (protected) microdata. `x_i` is row `i` as a column vector in `ℝ^p`; `x_{ik}` is the scalar entry.
- **Quasi-identifier projection.** `Q(x_i)` returns the sub-tuple of `x_i` on the declared quasi-identifier columns. Sensitive attribute is `s_i`.
- **Distances.** `‖·‖_σ` is Euclidean distance after per-column standardisation by sample standard deviation. `EMD(P, Q)` is Wasserstein-1 (Earth Mover's Distance).
- **Per-record risk.** `r_i` is the individual disclosure risk of record `i`; `ξ := mean_i r_i` is the file-level summary.

## Notation Glossary

| Symbol | Meaning |
|---|---|
| `n` | record count |
| `p` | variable count |
| `X` | original data matrix (`n × p`) |
| `Z` | protected data matrix (`n × p`) |
| `Q(·)` | quasi-identifier projection |
| `s_i` | sensitive attribute value of record `i` |
| `f_k` | sample frequency of quasi-identifier combination `k` |
| `F̂_k` | design-weighted population-frequency estimate of combination `k` |
| `r_i`, `ξ` | per-record / file-level disclosure risk |
| `λ`, `λ_k` | file-level / per-column λ information-loss measure |
| `MSU(i)` | set of minimal sample-unique attribute subsets for record `i` |
| `σ_k`, `s_k` | population / sample std. dev. of column `k` |
| `Σ_X` | covariance matrix of `X` |
| `μ̂`, `Σ̂_MCD` | MCD-robust location / scatter estimates |

## Methods × Metrics Map

| SDC Method | Native Risk Metric | Native Utility Metric |
|---|---|---|
| [Noise Addition](sdc_methods/perturbative/noise_addition.md) | [Record Linkage](risk_metrics/record_linkage_risk.md), [Interval Disclosure](risk_metrics/interval_disclosure_risk.md) | [λ](information_loss_metrics/lambda_measure.md), [Eigenvalue Spectrum](information_loss_metrics/eigenvalue_spectrum.md) |
| [Multiplicative Noise](sdc_methods/perturbative/multiplicative_noise.md) | [Record Linkage](risk_metrics/record_linkage_risk.md), [RMDID1](risk_metrics/rmdid1.md) | [λ](information_loss_metrics/lambda_measure.md), [Regression Comparison](information_loss_metrics/regression_coefficient.md) |
| [Microaggregation](sdc_methods/perturbative/microaggregation.md) | [k-Anonymity](risk_metrics/k_anonymity.md), [Record Linkage](risk_metrics/record_linkage_risk.md) | [SSE / SST](information_loss_metrics/sse_sst_ratio.md) |
| [Data Swapping](sdc_methods/perturbative/data_swapping.md) | [Record Linkage](risk_metrics/record_linkage_risk.md), [Individual Risk](risk_metrics/individual_risk.md) | [KL Divergence](information_loss_metrics/kl_divergence.md), [λ](information_loss_metrics/lambda_measure.md) |
| [Rank Swapping](sdc_methods/perturbative/rank_swapping.md) | [Record Linkage](risk_metrics/record_linkage_risk.md) | [λ](information_loss_metrics/lambda_measure.md), [Regression Comparison](information_loss_metrics/regression_coefficient.md) |
| [Rounding](sdc_methods/perturbative/rounding.md) | [Interval Disclosure](risk_metrics/interval_disclosure_risk.md) | [λ](information_loss_metrics/lambda_measure.md) |
| [Resampling](sdc_methods/perturbative/resampling.md) | [Record Linkage](risk_metrics/record_linkage_risk.md), [RMDID1](risk_metrics/rmdid1.md) | [λ](information_loss_metrics/lambda_measure.md), [Eigenvalue Spectrum](information_loss_metrics/eigenvalue_spectrum.md) |
| [PRAM](sdc_methods/perturbative/pram.md) | [Individual Risk](risk_metrics/individual_risk.md), [t-Closeness](risk_metrics/t_closeness.md) | [KL Divergence](information_loss_metrics/kl_divergence.md) |
| [MASSC](sdc_methods/perturbative/massc.md) | [k-Anonymity](risk_metrics/k_anonymity.md), [SUDA Score](risk_metrics/suda_score.md) | [KL Divergence](information_loss_metrics/kl_divergence.md), [λ](information_loss_metrics/lambda_measure.md) |
| [Sampling](sdc_methods/non_perturbative/sampling.md) | [Individual Risk](risk_metrics/individual_risk.md) | — (use [Regression Comparison](information_loss_metrics/regression_coefficient.md)) |
| [Global Recoding](sdc_methods/non_perturbative/global_recoding.md) | [k-Anonymity](risk_metrics/k_anonymity.md), [SUDA Score](risk_metrics/suda_score.md) | [λ](information_loss_metrics/lambda_measure.md), [KL Divergence](information_loss_metrics/kl_divergence.md) |
| [Top and Bottom Coding](sdc_methods/non_perturbative/top_bottom_coding.md) | [RMDID1](risk_metrics/rmdid1.md), [Individual Risk](risk_metrics/individual_risk.md) | [λ](information_loss_metrics/lambda_measure.md) |
| [Local Suppression](sdc_methods/non_perturbative/local_suppression.md) | [k-Anonymity](risk_metrics/k_anonymity.md), [Benchmark Risk Outlier](risk_metrics/benchmark_risk_outlier.md) | [λ](information_loss_metrics/lambda_measure.md), [KL Divergence](information_loss_metrics/kl_divergence.md) |

For sensitive variables, pair any method's native risk metric with [l-Diversity](risk_metrics/l_diversity.md); escalate per-record risk through [Hierarchical Risk](risk_metrics/hierarchical_risk.md) on grouped data.

## Index

**Risk metrics** — [k-Anonymity](risk_metrics/k_anonymity.md), [l-Diversity](risk_metrics/l_diversity.md), [t-Closeness](risk_metrics/t_closeness.md), [Individual Risk (Benedetti-Franconi)](risk_metrics/individual_risk.md), [Global Risk Rate ξ](risk_metrics/global_risk_rate.md), [Benchmark Risk Outlier](risk_metrics/benchmark_risk_outlier.md), [Hierarchical (Household) Risk](risk_metrics/hierarchical_risk.md), [SUDA Score (MSU)](risk_metrics/suda_score.md), [Record Linkage Risk](risk_metrics/record_linkage_risk.md), [Interval Disclosure Risk](risk_metrics/interval_disclosure_risk.md), [RMDID1](risk_metrics/rmdid1.md).

**Information loss metrics** — [λ Measure](information_loss_metrics/lambda_measure.md), [IL1s](information_loss_metrics/il1s.md), [SSE / SST](information_loss_metrics/sse_sst_ratio.md), [Eigenvalue Spectrum](information_loss_metrics/eigenvalue_spectrum.md), [Regression Coefficient Comparison](information_loss_metrics/regression_coefficient.md), [KL Divergence](information_loss_metrics/kl_divergence.md).

**SDC methods — perturbative** — [Noise Addition](sdc_methods/perturbative/noise_addition.md), [Multiplicative Noise](sdc_methods/perturbative/multiplicative_noise.md), [Microaggregation](sdc_methods/perturbative/microaggregation.md), [Data Swapping](sdc_methods/perturbative/data_swapping.md), [Rank Swapping](sdc_methods/perturbative/rank_swapping.md), [Rounding](sdc_methods/perturbative/rounding.md), [Resampling](sdc_methods/perturbative/resampling.md), [PRAM](sdc_methods/perturbative/pram.md), [MASSC](sdc_methods/perturbative/massc.md).

**SDC methods — non-perturbative** — [Sampling](sdc_methods/non_perturbative/sampling.md), [Global Recoding](sdc_methods/non_perturbative/global_recoding.md), [Top and Bottom Coding](sdc_methods/non_perturbative/top_bottom_coding.md), [Local Suppression](sdc_methods/non_perturbative/local_suppression.md).
