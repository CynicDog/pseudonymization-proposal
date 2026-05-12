# MASSC

> Protect categorical key variables through four sequential steps — Micro Agglomeration, Substitution, Subsampling, and Calibration — achieving k-anonymity while preserving marginal distributions via post-stratification weights.

$$
k\text{-anonymity on key variables} \;\wedge\; \pi_{\mathrm{post}} = \pi_{\mathrm{original}}
$$

## Inputs

- `X` — microdata with `n` records and categorical key columns `K`.
- `k ≥ 2` — anonymity threshold.
- `f_sub ∈ (0, 1]` — subsample fraction (typical `0.5` – `0.8`).
- `C` — calibration variables (subset of `K` or auxiliary columns) and corresponding population totals `T_C` to match.
- Distance `d_K(·, ·)` on the categorical key space (Hamming or domain-specific).

## Outputs

- `Z` — protected microdata, smaller than `X`, k-anonymous on `K`, with calibrated weights.
- `{w_i}_{i ∈ Z}` — post-stratification calibration weights for analysts.

## Algorithm

1. **Step 1 — Micro Agglomeration.**
   - Cluster the records of `X` into groups of size ≥ `k` using `d_K` on the key columns. Any k-anonymous clustering method works; the canonical choice is MDAV adapted to categorical distance ([Microaggregation](microaggregation.md)).
   - Let `{C_1, …, C_G}` be the resulting partition. For each `C_g`, identify a representative value tuple `q_g` on `K` (e.g. the mode tuple) and assign every member of `C_g` the value tuple `q_g`. The dataset is now k-anonymous on `K`.

2. **Step 2 — Substitution.**
   - For each cluster `C_g` and each record `i ∈ C_g`:
     - Draw a record `j ∈ C_g` uniformly at random.
     - Replace `x_{i, K}` with `x_{j, K}` (substitute key-variable values with those of a randomly chosen cluster-mate; this breaks the deterministic original-to-masked link while keeping the distribution within `C_g` unchanged).

3. **Step 3 — Subsampling.**
   - Draw a random subsample of size `n_sub := ⌊f_sub · n⌋` without replacement from the substituted dataset. Call this `X^{sub}`.

4. **Step 4 — Calibration.**
   - For each record `i ∈ X^{sub}`, compute a post-stratification weight `w_i` so that the weighted marginal distributions of the calibration variables `C` match the original / population totals `T_C`. Standard methods: raking (iterative proportional fitting) or generalised regression (GREG) calibration.
   - Verify `Σ_{i ∈ X^{sub}} w_i · 1[x_{iC} = c] = T_C(c)` for each calibration cell `c` (tolerance `≤ ε_cal`).

5. Return `(Z := X^{sub}, {w_i})`.

## Notes

- The result is simultaneously k-anonymous (Step 1), distributionally faithful in expectation (Step 4), and smaller than `X` (Step 3 — sampling adds an extra inclusion-uncertainty layer).
- Choosing `(k, f_sub, C)` is a joint optimisation: aggressive `k` may overshoot the calibration constraints; aggressive `f_sub` increases design effects.
- Native risk guarantee: [k-Anonymity](../../risk_metrics/k_anonymity.md) by Step 1; residual diagnostic [SUDA Score](../../risk_metrics/suda_score.md) catches small MSUs that survive subsampling.
- Native utility metrics: [KL Divergence](../../information_loss_metrics/kl_divergence.md) on calibrated marginals (Step 4 forces near zero); [λ](../../information_loss_metrics/lambda_measure.md) on substituted key-variable values.
- Variance estimation under MASSC requires custom replicate-weight schemes; analysts must be told to use the weights `w_i`.

## References

- Singh, Yu and Dunteman (2003). MASSC: A new data mask for limiting statistical information loss and disclosure.
- [docs/11 § MASSC](../../../docs/11-sdc-microdata-protection-methods.md#massc).
