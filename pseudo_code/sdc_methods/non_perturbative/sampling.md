# Sampling

> Release only a random subset of records rather than the full population file, so an intruder cannot be certain whether any specific individual appears in the data.

$$
S \subset U,\quad |S| = n,\quad \pi_i := P(i \in S)
$$

## Inputs

- `U` — population of size `N`.
- `n` — desired sample size (`n < N`).
- `design ∈ {SRS, stratified, cluster, PPS}` — sampling design.
- For stratified: strata `H_1, …, H_L` and per-stratum sample sizes `n_h`.
- For cluster / PPS: cluster definitions and size measures.

## Outputs

- `S ⊂ U` — released sample of size `n`.
- `{π_i}_{i ∈ S}` — inclusion probabilities (needed by analysts for Horvitz-Thompson estimation).
- `{w_i := 1 / π_i}` — design weights.

## Algorithm

1. According to `design`:
   - **SRS (simple random sampling without replacement)**:
     - Draw `n` distinct units from `U` uniformly at random.
     - Set `π_i ← n / N` for all `i ∈ U`.
   - **Stratified**:
     - For each stratum `h`, draw `n_h` units from `H_h` by SRS.
     - Set `π_i ← n_h / |H_h|` for `i ∈ H_h`.
   - **Cluster**:
     - Sample `m` clusters by SRS or PPS; include every unit in the sampled clusters.
     - `π_i ← P(cluster of i is sampled) · 1` (units within sampled clusters are taken with certainty).
   - **PPS (probability proportional to size)**:
     - Set `π_i ∝ size(i)` and use a PPS sampler (e.g. Sampford or systematic-PPS) to draw `n` units.
2. Compute design weights: `w_i ← 1 / π_i` for `i ∈ S`.
3. Return `(S, {π_i}, {w_i})`.

## Notes

- Sampling provides disclosure protection through **inclusion uncertainty**: an intruder matching to a known individual cannot confirm a hit because the individual may not have been sampled.
- Protection strength is proportional to `(1 - n / N)`. A near-census provides negligible protection on its own and must be combined with other methods.
- Sampling does **not** distort values of variables that are included — this is its key advantage over perturbative methods for analytical microdata.
- Disclosure asymmetry: absence from the sample can itself reveal information for rare subpopulations (small strata, ethnic minorities). Document this in the release metadata.
- Native risk metric: [Individual Risk (Benedetti-Franconi)](../../risk_metrics/individual_risk.md) — this is exactly the negative-binomial uncertainty that sampling generates.
- No native value-distortion utility metric; check design-effect bias with [Regression Coefficient Comparison](../../information_loss_metrics/regression_coefficient.md) against full-population or larger-sample baselines.

## References

- Hundepool et al. (2014), Chapter 5.
- Templ, Meindl, Kowarik and Chen (2014) (IHSN).
- Cochran (1977). *Sampling Techniques*. Wiley.
- [docs/11 § Sampling](../../../docs/11-sdc-microdata-protection-methods.md#sampling).
