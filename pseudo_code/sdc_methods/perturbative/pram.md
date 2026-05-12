# PRAM (Post-Randomisation Method)

> Apply a Markov transition matrix to each categorical variable, so that the released category for any record is drawn probabilistically from a specified distribution over all categories, enabling unbiased estimation after correction.

$$
p_{kl} := P(X' = c_l \mid X = c_k), \qquad \hat{T}_X := (P^{-1})^\top T_\xi
$$

## Inputs

- `X` — microdata with one or more categorical columns to protect.
- For each protected column with category set `C := {c_1, …, c_K}`:
  - `P` — `K × K` Markov transition matrix with rows summing to 1.
  - `invariant ∈ {true, false}` — whether to enforce `π^T P = π^T` (marginal-preserving).
- Random seed (recorded for reproducibility but not disclosed).

## Outputs

- `Z` — protected microdata with each categorical column replaced per the Markov matrix.
- (Optionally) `T̂_X` — corrected frequency-table estimator for analysts.

## Algorithm — Apply

1. (Optional design step) If `invariant = true`, design `P` so that `π^T P = π^T`, where `π` is the marginal frequency of the column in `X`. A standard construction: pick a low diagonal target `p_kk ≥ p_min` (e.g. `0.8`) and solve the constrained system for off-diagonals (Kooiman, Willenborg and Gouweleeuw 1998).
2. For each record `i` and each protected column `k`:
   - Let `c_a := x_{ik}` be the original category.
   - Draw `c_b ∈ C` according to row `P[a, ·]`: i.e. `P(z_{ik} = c_l) = p_{a l}`.
   - Assign `z_{ik} ← c_b`.
3. Return `Z`. (The Markov matrix `P` may be retained internally but not disclosed.)

## Algorithm — Corrected Marginal Estimator (for analysts)

1. Compute the perturbed frequency vector `T_ξ ∈ ℝ^K` from `Z`.
2. Verify that `P` is invertible (Kooiman et al. 1998 gave conditions; diagonal-dominant `P` is sufficient).
3. Recover an unbiased estimator of the original marginal:
   $$
   \hat{T}_X := (P^{-1})^\top T_\xi.
   $$
4. Return `T̂_X`.

## Notes

- **Invariant PRAM** preserves column marginals in expectation; the corrected estimator is unbiased and KL divergence between `X` and `Z` marginals approaches zero by construction (see [KL Divergence](../../information_loss_metrics/kl_divergence.md)).
- **Diagonal-dominant PRAM** sets `p_kk` close to 1 so most records retain their category; protection comes from intruder uncertainty about which records changed.
- The estimator `T̂_X` has variance that depends on `P`'s eigenvalues; near-singular `P` produces unstable estimators. Choose `P` with bounded condition number.
- Per-record categorical replacement, the categorical counterpart of [Noise Addition](noise_addition.md).
- Native risk metric: [Individual Risk](../../risk_metrics/individual_risk.md) under the perturbed marginal; for sensitive variables, [t-Closeness](../../risk_metrics/t_closeness.md).
- Native utility metric: [KL Divergence](../../information_loss_metrics/kl_divergence.md).

## References

- Gouweleeuw, Kooiman, Willenborg and de Wolf (1998a), `papers/perturbative/Gouweleeuw-1998a-PRAM-theory-implementation-JOS.pdf`.
- Gouweleeuw, Kooiman, Willenborg and de Wolf (1998b), `papers/perturbative/Gouweleeuw-1998b-PRAM-protecting-microdata-Questiio.pdf`.
- Kooiman et al. (1998); De Wolf et al. (1998–1999), `papers/perturbative/De-Wolf-1998-Reflections-on-PRAM.pdf`.
- De Wolf and Van Gelder (2004), `papers/perturbative/De-Wolf-Van-Gelder-2004-empirical-evaluation-PRAM.pdf`.
- Gross, Guiblin and Merrett (2004), `papers/perturbative/Gross-Guiblin-Merrett-2004-PRAM-2001-Census-SAR.pdf`.
- [docs/11 § PRAM](../../../docs/11-sdc-microdata-protection-methods.md#pram-post-randomisation-method).
