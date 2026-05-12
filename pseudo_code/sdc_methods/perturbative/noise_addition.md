# Noise Addition

> Replace each continuous value with the original value plus a random draw from a noise distribution, preserving distributional shape while obscuring exact values.

$$
Z = X + \varepsilon, \quad \varepsilon \sim \mathcal{N}(0, \Sigma_\varepsilon), \quad \Sigma_\varepsilon = \alpha \cdot \mathrm{diag}(\sigma_1^2, \ldots, \sigma_p^2)
$$

## Inputs

- `X ∈ ℝ^{n × p}` — original microdata (continuous columns only).
- `α > 0` — noise magnitude parameter (typical `α ∈ [0.05, 0.20]`).
- `mode ∈ {uncorrelated, correlated, linear_transform}` — noise design variant.

## Outputs

- `Z ∈ ℝ^{n × p}` — protected microdata.

## Algorithm

1. For each column `k ∈ {1, …, p}`, compute the column variance `σ_k² ← Var(x_{·k})`.
2. Construct the noise covariance `Σ_ε` according to `mode`:
   - **uncorrelated**: `Σ_ε ← α · diag(σ_1², …, σ_p²)` (preserves column means exactly; inflates covariance).
   - **correlated** (Kim 1986): compute `Σ_X ← Cov(X)`, then `Σ_ε ← α · Σ_X` (preserves the correlation matrix in expectation).
   - **linear_transform** (Brand 2002): sample a random near-orthogonal `p × p` matrix `T`, set `Σ_ε ← α · Σ_X` *and* compose the masked output through `T` (apply step 4 with the orthogonal pre-multiplication after the draw).
3. Draw the noise matrix: for each record `i ∈ {1, …, n}`, sample independently `ε_i ~ N(0, Σ_ε)` (`p`-dimensional multivariate normal).
4. Form the protected matrix:
   $$
   z_i := x_i + \varepsilon_i \quad \text{for } i = 1, \ldots, n.
   $$
   For `linear_transform` mode, `z_i := T · (x_i + ε_i)`.
5. Return `Z`.

## Notes

- Under uncorrelated noise, `E[Z] = E[X]` (column means exact), but `Cov(Z) = Cov(X) + Σ_ε` — analysts unaware of the noise inflate variances and attenuate correlations.
- Under correlated noise, both means and the correlation matrix are preserved in expectation.
- The method does **not** provide formal disclosure bounds: an intruder who knows `α` can partially invert the perturbation; outlier records remain identifiable.
- Native risk metric: [Record Linkage](../../risk_metrics/record_linkage_risk.md) (top-1 lower, top-2 upper); supplement with [Interval Disclosure](../../risk_metrics/interval_disclosure_risk.md).
- Native utility metric: [λ](../../information_loss_metrics/lambda_measure.md); check covariance distortion with [Eigenvalue Spectrum](../../information_loss_metrics/eigenvalue_spectrum.md).
- Sampling `N(0, Σ_ε)` requires a Cholesky factor `L` such that `L Lᵀ = Σ_ε`; draw `u ~ N(0, I_p)` and set `ε_i := L u_i`.

## References

- Kim (1986). Kim (1990). Sullivan (1989). Tendick (1991). Brand (2002).
- `papers/perturbative/Brand-Giessing-2002-Sullivan-algorithm-tests-CASC.pdf`.
- `papers/perturbative/Domingo-Ferrer-Sebe-Castella-2004-security-noise-addition.pdf`.
- `papers/perturbative/Domingo-Ferrer-Torra-2001-disclosure-protection-information-loss.pdf`.
- [docs/11 § Noise Addition](../../../docs/11-sdc-microdata-protection-methods.md#noise-addition).
