# Individual Risk (Benedetti-Franconi)

> Estimate, for each record, the probability that an adversary holding that record's quasi-identifier values would correctly re-identify it, using a negative-binomial posterior on the population frequency given the sample frequency.

## Inputs

- `X` — sample microdata with `n` records and quasi-identifier index set `K`.
- `w_i` — design weight of record `i` (set `w_i := N / n` for simple random sampling, where `N` is the population size).
- Optional dispersion / informative-prior parameters for the negative-binomial posterior (Franconi and Polettini 2004).

## Outputs

- `r_i` for each record `i ∈ {1, …, n}` — per-record re-identification risk.
- `ξ := (1/n) Σ_i r_i` — file-level mean risk (see [Global Risk Rate ξ](global_risk_rate.md)).

## Algorithm

1. Project each record onto its quasi-identifier tuple: `q_i := Q(x_i)`.
2. Group by `q_i`. For each distinct tuple `q`, compute:
   - sample frequency `f_q := |{i : q_i = q}|`.
   - design-weighted population estimate `F̂_q := Σ_{i : q_i = q} w_i`.
3. For each record `i` with tuple `q := q_i`:
   - **closed form (sample-uniques only)**: if `f_q = 1`, the negative-binomial posterior mean of the population count yields `r_i ≈ (log F̂_q) / (F̂_q - 1)` (Benedetti-Franconi 1998). For larger `f_q`, use the same posterior model on `(f_q, F̂_q)` with the project's dispersion parameter.
   - **handbook approximation**: `r_i ≈ f_q / F̂_q` is the leading-order approximation used when the posterior need not be computed exactly.
4. Optionally cap `r_i ≤ 1` and floor `r_i ≥ 0` to absorb numerical edge cases (e.g. `f_q > F̂_q` from weight noise).
5. Aggregate `ξ ← (1/n) Σ_i r_i`. Return `({r_i}_i, ξ)`.

## Notes

- The estimator is appropriate when the released file is a *sample* rather than a census; for a census, sample-unique ⇒ population-unique, so `r_i = 1` directly.
- A record with `f_q > 1` (not sample-unique) still has nonzero risk if `F̂_q` is small.
- Per-record `r_i` feeds [Hierarchical (Household) Risk](hierarchical_risk.md) for grouped data and [Benchmark Risk Outlier](benchmark_risk_outlier.md) for tail detection.
- Implementation: one group-by pass on quasi-identifier columns to get `(f_q, F̂_q)`, then a join back to record level.

## References

- Benedetti and Franconi (1998). Statistical and technological solutions for controlled data dissemination. *NTTS Pre-proceedings*.
- Franconi and Polettini (2004). Individual risk estimation in μ-Argus: A review. *PSD 2004*, LNCS 3050.
- Hundepool et al. (2014), Chapter 4.
- Templ, Meindl, Kowarik and Chen (2014), §3 (IHSN).
- [docs/12 § Individual Risk](../../docs/12-sdc-risk-and-utility-metrics.md#individual-risk-benedetti-franconi).
