# Eigenvalue Spectrum Comparison

> Compare the sorted eigenvalues of the original and protected covariance matrices; large deviations indicate that perturbation has distorted the principal-component structure.

## Inputs

- `X`, `Z` — original and protected microdata (`n × p`, continuous).

## Outputs

- `Δ_max` — maximum absolute deviation of corresponding sorted eigenvalues (standardised).
- `Δ_sum` — aggregate absolute deviation (standardised).
- Per-pair table `{(λ_j(X), λ_j(Z))}_{j=1..p}`.

## Algorithm

1. Compute the covariance matrices `Σ_X ← Cov(X)` and `Σ_Z ← Cov(Z)`.
2. Compute the eigenvalue spectra and sort each in *descending* order:
   - `(λ_1(X), λ_2(X), …, λ_p(X))` from `Σ_X`.
   - `(λ_1(Z), λ_2(Z), …, λ_p(Z))` from `Σ_Z`.
3. Compute deviations per rank `j ∈ {1, …, p}`:
   $$
   \Delta_j := |\lambda_j(\Sigma_X) - \lambda_j(\Sigma_Z)|.
   $$
4. Standardise by `λ_1(Σ_X)` (largest original eigenvalue) and aggregate:
   $$
   \Delta_{\max} := \max_j \frac{\Delta_j}{\lambda_1(\Sigma_X)}, \qquad \Delta_{\mathrm{sum}} := \frac{1}{\lambda_1(\Sigma_X)} \sum_{j=1}^{p} \Delta_j.
   $$
5. Return `(Δ_max, Δ_sum, {(λ_j(X), λ_j(Z))}_j)`.

## Notes

- The metric is invariant to which `n × n` orthogonal rotation underlies `Cov(Z)`, so it does **not** detect rotations of the principal axes. For PCA-driven downstream use, supplement with the **canonical correlation** between principal-component scores of `X` and `Z`, or with **subspace angles** between the top-r eigenvector subspaces.
- Uncorrelated [Noise Addition](../sdc_methods/perturbative/noise_addition.md) inflates all eigenvalues uniformly by the noise variance; correlated noise preserves ratios (Kim 1986); [Resampling](../sdc_methods/perturbative/resampling.md) tends to flatten the spectrum.
- Eigenvalue computation is `O(p³)` — trivial for typical microdata `p ≤ 100`.

## References

- Domingo-Ferrer, Mateo-Sanz and Torra (2001). Comparing SDC methods for microdata. *ETK-NTTS*.
- Mateo-Sanz, Domingo-Ferrer and Sebé (2005). *DMKD* 11(2):181–193.
- Hundepool et al. (2014), Chapter 4.
- [docs/12 § Eigenvalue Spectrum Comparison](../../docs/12-sdc-risk-and-utility-metrics.md#eigenvalue-spectrum-comparison).
