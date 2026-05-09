# 11 — SDC Implementation in Python + Polars

## Purpose and Scope

This document translates the methodology in two authoritative SDC references into implementation guidance for a Python package `sdc_polars` that uses Polars as the DataFrame engine:

- *Handbook on Statistical Disclosure Control*, 2nd ed. (Hundepool et al., 2026) — the comprehensive European reference covering all techniques, mathematical foundations, and risk models
- *Introduction to Statistical Disclosure Control*, IHSN Working Paper No. 007 (Templ, Meindl, Kowarik and Chen, 2014) — a practitioner-oriented guide whose workflow diagram and variable classification model are adopted here as the organizing framework

The document is a companion to doc 03 (techniques overview) and serves as the engineering specification for `sdc_polars`. Focus is microdata protection; tabular SDC is out of scope.

## SDC Workflow

The workflow below is adapted from Figure 2 in the IHSN guide. It is the operational sequence the `SDCPipeline` class implements.

```
 ┌──────────────────────────────────────────────────────────────────┐
 │  PRE-PROCESSING                                                   │
 │  Assess disclosure scenarios. Identify direct identifiers,        │
 │  key variables, and sensitive variables. Set acceptable risk      │
 │  threshold and acceptable information loss level.                 │
 └───────────────────────────┬──────────────────────────────────────┘
                             │
                             ▼
                  Delete direct identifiers
                             │
               ┌─────────────┴──────────────┐
               │                            │
               ▼                            ▼
    CATEGORICAL KEY VARIABLES    CONTINUOUS KEY VARIABLES
               │                            │
    Assess disclosure risk       (risk = 100% for original;
    (k-anonymity, global risk,    measured by record linkage
     individual risk)             after any perturbation)
               │                            │
    ┌──────────┼───────────┐    ┌───────────┼──────────────┐
    │          │           │    │           │              │
    ▼          ▼           ▼    ▼           ▼              ▼
 Recoding   Local       PRAM  Micro-    Adding         Shuffling
            Suppression       aggregation noise
               │                            │
               └─────────────┬──────────────┘
                             │
                             ▼
                  Assess risk AND data utility
                  (benchmarking indicators,
                   direct loss measures)
                             │
             ┌───────────────┴───────────────┐
             │                               │
             ▼                               ▼
    Risk acceptable &           Disclosure risk too high
    utility preserved            or data utility too low
             │                               │
             ▼                               │
       Release data             Change method / parameters
                                    and repeat ──────────┐
                                                         │
                                    ◄────────────────────┘
```

The two tracks — categorical and continuous — are executed independently and then reunited for the final risk + utility assessment. The loop continues until both the disclosure risk threshold and the information loss threshold are satisfied simultaneously.

## Variable Taxonomy

The IHSN guide gives the clearest operational classification. Every column falls into one of three functional roles, plus one data-type dimension that governs which methods apply.

### Functional roles

**Direct identifiers** — variables that unambiguously name a statistical unit: resident registration number, customer ID, name, email address, exact street address. These are removed as the absolute first step. They are never SDC-transformed because transformation gives a false sense of protection — a masked name is still a name-shaped field. Drop them completely.

**Key variables (quasi-identifiers)** — variables that individually seem innocuous but in combination can re-identify respondents by linking to external databases. Gender alone is not identifying; gender + age band + region + occupation often is. Key variables are the primary SDC target. The goal is to reduce their joint identifying power so that no combination singles out fewer than k individuals.

**Sensitive variables** — variables whose values must never be disclosed for a specific individual: salary, claim payout, medical diagnosis, credit score, criminal history. Sensitive variables require protection against attribute disclosure — an adversary who re-identifies someone via their key variables must not be able to read off the sensitive value.

**A variable can be both a key variable and sensitive.** Income is a classic example: it can be used with other variables to re-identify individuals (key function), and its specific value is itself confidential (sensitive function). The IHSN guide explicitly calls this out — such columns appear in both `quasi_identifiers` and `sensitive` lists in `SDCFrame`.

### Data type dimension

| Data type | Typical methods |
|---|---|
| Categorical (finite value set; no arithmetic order implied) | Global recoding, local suppression, PRAM |
| Continuous (numerical; arithmetic operations valid) | Microaggregation, noise addition, shuffling, top/bottom coding |

The split between categorical and continuous governs which branch of the Figure 2 workflow is used for each variable.

### SDCFrame

The `SDCFrame` dataclass captures the taxonomy and is the entry point for all operations:

```python
@dataclass
class SDCFrame:
    lf: pl.LazyFrame
    identifiers: list[str]        # removed immediately; never transformed
    quasi_identifiers: list[str]  # key variables; primary SDC target
    sensitive: list[str]          # confidential outcomes; may overlap with quasi_identifiers
    weights: str | None = None    # sampling weight column for individual risk
    strata: str | None = None     # stratification variable
    hierarchy: str | None = None  # household/group ID for hierarchical risk
```

## Package Structure

```
sdc_polars/
├── __init__.py
├── types.py                        # SDCFrame, variable role constants
├── risk/
│   ├── frequency.py                # k-anonymity, l-diversity, t-closeness, ARGUS
│   ├── individual.py               # Benedetti-Franconi negative binomial risk + benchmark
│   ├── hierarchical.py             # Household-level risk (1 - Π(1 - r_i))
│   ├── outlier.py                  # Robust Mahalanobis Distance (RMD) for outlier risk
│   ├── suda.py                     # SUDA minimal sample unique detection (ATT^(M-k))
│   └── record_linkage.py           # Distance-based, probabilistic, interval disclosure
├── methods/
│   ├── non_perturbative/
│   │   ├── recoding.py             # Global recoding (categorical + continuous intervals)
│   │   ├── suppression.py          # Local suppression (min-suppression heuristic)
│   │   ├── top_bottom.py           # Top/bottom coding
│   │   └── sampling.py             # Record sampling
│   └── perturbative/
│       ├── noise.py                # Uncorrelated, correlated, multiplicative, Kim-linear
│       ├── microaggregation.py     # Univariate optimal, MDAV multivariate, k-modes
│       ├── rank_swap.py            # Rank swapping
│       ├── rounding.py             # Controlled and stochastic rounding
│       ├── pram.py                 # Post-Randomization Method
│       ├── shuffling.py            # Data shuffling (Muralidhar-Sarathy 2006)
│       └── synthetic.py            # Distribution fitting, bootstrap synthesis
├── loss/
│   ├── continuous.py               # SSE/SST, IL1s, eigenvalue diff, lm, lambda
│   └── categorical.py              # KL divergence, frequency distortion
└── pipeline.py                     # SDCPipeline fluent orchestrator
```

## Polars Design Principles

**Lazy first.** All public functions accept `pl.LazyFrame` and return `pl.LazyFrame`. `.collect()` is called only at the final step or where an algorithm fundamentally requires the full dataset in memory (covariance computation, iterative nearest-neighbour search).

**Expression API for everything column-local.** Operations that act on individual values — rounding, clipping, categorical replacement, noise injection from a pre-computed array — use `pl.Expr` and stay in the Polars query plan with zero Python overhead per row.

**Group operations via `group_by` + `agg` + `join`.** Risk assessment and microaggregation group-mean replacement are expressed as aggregations joined back to the original frame, not as Python loops over groups. Example pattern:

```python
counts = lf.group_by(qi_cols).agg(pl.len().alias("_n"))
lf.join(counts, on=qi_cols).with_columns((pl.col("_n") >= k).alias("is_safe"))
```

**Collect to NumPy for inherently iterative algorithms.** MDAV multivariate microaggregation (iterative nearest-neighbour), Hansen-Mukherjee optimal univariate (dynamic programming on a sorted array), correlated noise (requires a covariance matrix and Cholesky decomposition), and rank swapping (random partner selection with range constraints) all require temporary collection to NumPy. After computation the result is wrapped back as a `pl.Series` and joined or assigned.

## Risk Assessment

### `risk/frequency.py` — k-Anonymity, l-Diversity, t-Closeness, ARGUS

**k-Anonymity** — a dataset satisfies k-anonymity if every combination of quasi-identifier values appears in at least k records. Implemented as: `group_by(qi_cols)` → compute count → join back to original frame → filter records where count < k. These are the violations to be resolved by recoding or suppression.

**l-Diversity** — within each k-anonymous group, the sensitive attribute must take at least l distinct values. Without this, an adversary who narrows a target to a k-anonymous group may still infer the sensitive value if all group members share it. Implemented as: `group_by(qi_cols).agg(n_unique(sensitive))` → join → filter groups where n_unique < l.

**t-Closeness** — the distribution of the sensitive attribute within each group must be within distance t of the population distribution. Uses Earth Mover's Distance (Wasserstein-1); for numerical attributes a practical approximation is `|group_mean − pop_mean| / pop_range`. More conservative than l-diversity: it prevents inference even when the group has diversity if the group distribution is skewed toward one value.

**ARGUS threshold rule** — a quasi-identifier combination is flagged as unsafe if its estimated population frequency (weighted count if weights are available, otherwise sample count) falls below a threshold (typically 3). This is the primary rule used by μ-ARGUS.

Key parameters: `qi_cols`, `k` / `l` / `t` / `threshold`, optional `weight_col`.

### `risk/individual.py` — Individual Risk (Benedetti-Franconi) and Benchmark

**Individual risk** (Benedetti-Franconi, Section 3.3.5) — the risk of record i is `r_i = f_k / F̂_k` where `f_k` is the sample count of key combination k and `F̂_k = Σ w_i` is the design-weight sum, estimating population frequency. This is the posterior mean of `1/F_k` under a Negative Binomial model. Computed by joining group-level aggregates back to individual records. The global risk rate ξ is the record-weighted average across all records.

**Benchmark approach** (IHSN Section 3.4) — flags records that are outliers in the risk distribution rather than just above a fixed threshold. A record is a benchmark outlier if both:
- `r_i > threshold` (absolute threshold, e.g. 0.05)
- `r_i > median(r) + c · MAD(r)` (relative outlier condition, typically c = 2–3)

The MAD computation requires collecting individual risks to a NumPy array. This is intentional — the purpose is to identify the tail of the risk distribution, not apply a fixed rule.

Key parameters: `qi_cols`, `weight_col`, `abs_threshold`, `mad_multiplier`.

### `risk/hierarchical.py` — Household Risk

For household or group-level data, re-identifying any member of a household reveals information about all members. The household-level risk is:

`r_hh = 1 − Π_i (1 − r_i)` across all members i in the household

This is computed after individual risks are assigned: `group_by(household_col).agg((1 - pl.col("individual_risk")).product())` → household_risk = 1 − product → join back to individual records.

Key parameters: `qi_cols`, `weight_col`, `household_col`.

### `risk/outlier.py` — Robust Mahalanobis Distance (RMD)

Outlier records are disproportionately identifiable because adversaries specifically look for extreme values. Templ and Meindl (2008) define the RMDID1 measure using Robust Mahalanobis Distance:

1. Estimate robust location and scatter via Minimum Covariance Determinant (MCD, `sklearn.covariance.MinCovDet`).
2. Compute RMD for each record in the original data.
3. For each record, construct an interval of width `scale_factor × RMD × std(col)` around the original value.
4. After masking, a record is unsafe if its masked value falls inside that interval.
5. RMDID1 = proportion of unsafe records, reported per column and overall.

This measure is specifically designed to identify which records — after any SDC method has been applied — still allow an adversary to recover the original value by exploiting the outlier's rarity.

Key parameters: `original`, `protected`, `continuous_cols`, `scale_factor`.

### `risk/suda.py` — SUDA Special Uniques

A "special unique" is a record that is unique on some minimal subset of the quasi-identifiers. A minimal sample unique (MSU) is a subset S on which the record is unique and no proper subset of S is already unique for that record. Records with small MSUs are especially at risk — an adversary only needs to know a few attributes.

SUDA score (IHSN Table 3) — for each MSU of size k in a record: contribution = `ATT^(M−k)`, where ATT = total number of key attributes and M = user-specified maximum MSU size. Smaller MSUs score exponentially higher. Record's total score = sum over all its MSUs.

Example: a record with MSU {age 60s} (size 1) and MSU {Male, University} (size 2), with M=3 and ATT=4, gets score `4^(3−1) + 4^(3−2) = 16 + 4 = 20`.

Implementation: iterate over all subsets of qi_cols up to size max_msu_size using `itertools.combinations`. For each subset, `group_by` the subset columns and identify size-1 groups. Track which subsets are MSUs (unique but no unique proper subset already found for that record). This requires `O(C(ATT, M))` group-by operations and a collect to DataFrame, making it the most computationally intensive risk measure.

Key parameters: `qi_cols`, `max_msu_size` (default 4).

### `risk/record_linkage.py` — Distance-Based Linkage and Interval Disclosure

**Distance-based record linkage** is the primary risk measure for continuous key variables (the right branch of the Figure 2 workflow). Before any SDC, linkage rate = 100% by definition. After perturbation, it should fall substantially. For each protected record, find the k nearest original records using standardized Euclidean distance (`scipy.spatial.cdist`). Two metrics are reported:

- `linked_1st`: fraction where the nearest original record is the correct match
- `linked_within_top2`: fraction where the correct match is 1st or 2nd nearest — the IHSN upper-bound metric

IHSN Listing 3 benchmark: after MDAV k=3, rate falls to [0%, 61.42%] (exact vs. upper bound).

**Interval disclosure risk** — construct an interval of `± half_width_pct × range(col)` around each masked value; risk = proportion of records where the original value falls within this interval. This is an adversary-model where the masking method is known and the attacker uses the published masked value as a pointer to the approximate original. Both measures should be reported together as lower and upper bounds on true linkage risk.

Key parameters for distance linkage: `original`, `protected`, `key_cols`, `top_n`.
Key parameters for interval disclosure: `original`, `protected`, `continuous_cols`, `interval_half_width_pct` (default 0.05).

Implementation note: both functions must collect to NumPy for pairwise distance computation. For large datasets (n > 100k), consider `sklearn.neighbors.BallTree` for approximate nearest-neighbour.

## Non-Perturbative Methods

### `methods/non_perturbative/recoding.py` — Global Recoding (Section 3.4.3.2)

Global recoding reduces the precision of quasi-identifiers by merging categories or discretizing continuous ranges. It applies uniformly to the entire dataset — not only to unsafe records — to maintain consistent categorization across all records.

Two variants:

**Categorical recoding** — merge existing categories into broader groups. Accepts a `mapping` dict `{old_category: new_category}`. Implemented with `pl.col(col).replace(mapping)` — O(n), stays in the Polars expression plan.

**Continuous interval recoding** — replace numeric values with interval labels. Accepts `breaks` (cut points) and `labels`. Implemented with `pl.col(col).cut(breaks, labels=labels)`. The output column changes from numeric to categorical.

Key design decision: recoding is irreversible in the released dataset. The analyst must know the new category definitions. For the categorical case, document any categories not present in the mapping — they pass through unchanged unless an explicit catch-all is added.

### `methods/non_perturbative/suppression.py` — Local Suppression (Section 3.4.3.4)

Local suppression sets individual field values to null for records whose quasi-identifier combination is unsafe (count < k). Unlike global recoding, it targets only unsafe records. The goal is to minimize the number of suppressions while bringing all records to safety.

True minimum suppression is NP-hard. The practical heuristic suppresses the quasi-identifier column with the highest cardinality first (highest uniqueness impact), applied iteratively until the record is safe. In the Polars implementation: identify unsafe records via the k-anonymity violation join, then for each suppression column use `pl.when(unsafe).then(None).otherwise(pl.col(c))`.

Key tradeoff: suppression reduces information more than recoding for individual records, but affects fewer records than global recoding. For highly sparse quasi-identifier combinations, suppression can produce many nulls — recoding the offending variable may be preferable.

Key parameters: `qi_cols`, `suppress_cols`, `k`, `strategy` (heuristic ordering).

### `methods/non_perturbative/top_bottom.py` — Top and Bottom Coding (Section 3.4.3.3)

Extreme values are disproportionately identifying. Top-coding replaces values above a threshold with the threshold value; bottom-coding does the same for the lower tail. The threshold is typically derived from a quantile of the empirical distribution.

Polars implementation: `pl.col(col).clip(upper_bound=threshold)` and `pl.col(col).clip(lower_bound=threshold)`. If no explicit threshold is given, compute it from the quantile first (one `.collect()` call for the quantile), then inject as a literal into the expression.

Key parameters: `col`, `threshold` (explicit value) or `quantile` (default 0.99 for top, 0.01 for bottom).

## Perturbative Methods

### `methods/perturbative/noise.py` — Noise Addition (Sections 3.4.4.1–3.4.4.3)

Four noise variants from the handbook. All preserve means; they differ in what additional properties are preserved and how much protection they provide.

**Uncorrelated noise** — `Z_j = X_j + ε_j` where `ε_j ~ N(0, α · Var(X_j))`. Each column receives independent noise scaled to a fraction α of its variance. Preserves means and cross-variable covariances. Inflates individual variances by factor `(1 + α)`. Simplest to implement: compute per-column std from a collected frame, generate noise with NumPy, add back as Polars expressions.

**Correlated noise** — draws noise from `N(0, α · Σ)` where Σ is the sample covariance matrix. Preserves means and correlation coefficients. Inflates variances and covariances uniformly by `(1 + α)`. Requires a Cholesky decomposition of the scaled covariance matrix (NumPy), from which correlated noise vectors are drawn and added back column-wise.

**Multiplicative noise (Höhne 2004)** — each record receives a multiplicative factor of `1 ± f ± s`, where the ± alternation is balanced so the column sum is exactly preserved. Better suited for skewed business data (claim severity, turnover) because perturbation scales with the value magnitude. Requires collect to NumPy for the sum-preservation adjustment.

**Noise + linear transform (Kim 1986)** — applies uncorrelated noise then rescales so `Var(G_j) = Var(X_j)` and `E(G_j) = E(X_j)`. The scale parameter `c = sqrt(Var(X_j) / Var(Z_j))` must be published or the analyst cannot correct for it — disclosure of c is a design decision.

Key parameters: `cols`, `alpha` (noise fraction, default 0.1), `seed`, `mode` (for the pipeline).

### `methods/perturbative/microaggregation.py` — Microaggregation (Sections 3.4.2.3, 3.4.5)

Microaggregation partitions records into groups of size ≥ k and replaces each record's values with group means (or modes for categorical). This achieves k-anonymity on continuous variables while minimizing information loss.

**Univariate optimal (Hansen-Mukherjee, O(k²n))** — the globally optimal univariate partition is computable in polynomial time via shortest-path dynamic programming on the sorted values. Sort values ascending, build a cost matrix where `cost(i,j) = SSE(values[i..j])` (computed in O(1) using prefix sums), then solve `dp[j] = min(dp[i] + cost(i,j))` with group sizes constrained to `[k, 2k)`. Backtrack to get group boundaries, compute means, join back via row index.

This runs entirely in NumPy after a single collect. It is O(k²n) in time and O(n) in space — feasible for typical microdata sizes (n < 10M). For very large n, approximate partitions using the MDAV heuristic on a single variable.

**Multivariate MDAV (Domingo-Ferrer and Torra, 2005)** — the heuristic from μ-ARGUS. Iteratively: find the record most distant from the current centroid (x_r), find the record most distant from x_r (x_s), assign k nearest records to each as groups, remove them, repeat. Any residual < 2k records form a final group. Replace all column values within each group with group column means.

Complexity O(n²/k) — for n > 100k, use `sklearn.neighbors.BallTree` for approximate nearest-neighbour at each centroid step. Standardize columns before distance computation to avoid scale bias.

**Categorical k-modes (Torra 2004)** — cluster categorical records using Hamming distance and mode aggregation. Use the `kmodes` library. Target n_clusters ≈ n/k. Within each cluster, replace each categorical variable with the cluster mode. Information loss measured by frequency distortion (KL divergence between original and protected distributions).

Key parameters: `col` / `cols`, `k` (minimum group size), `standardize` (MDAV).

### `methods/perturbative/rank_swap.py` — Rank Swapping (Section 3.4.2.4)

Rank swapping (Moore 1996) sorts values by rank and swaps each record's value with a randomly chosen partner within a window of ±p·n ranks. Applied independently per column. Values in the final dataset are all original values — only their record-level associations change.

Implementation: sort to get rank order (use `pl.Series.arg_sort()`), collect to NumPy for the random swap pass (to allow range-constrained partner selection), write back via Polars expression. Key invariant: every value appears exactly once in the output.

Tradeoff: empirically competitive with microaggregation on the risk/utility frontier. Preserves marginal distributions exactly (only within-column; cross-column correlations are weakened). A larger window p reduces protection but reduces information loss.

Key parameters: `cols`, `p` (rank window as fraction of n, default 0.1), `seed`.

### `methods/perturbative/rounding.py` — Rounding (Section 3.4.2.5)

**Controlled rounding** — replace each value with the nearest multiple of a base b. Pure Polars expression: `(pl.col(col) / base).round() * base`. Stays in the query plan, zero collection.

**Stochastic rounding** — round up or down to the two adjacent multiples of b with probability proportional to distance from each. Unbiased: `E[result] = original`. Requires a uniform random draw per record, so collection to NumPy is needed.

Rounding is typically applied as the first continuous perturbation step (before noise or microaggregation) to reduce the precision of quasi-identifiers like age or income without full recoding.

Key parameters: `col`, `base`.

### `methods/perturbative/pram.py` — PRAM (Section 3.4.6)

Post-Randomization Method is the primary perturbative method for categorical variables. Each record's value on a categorical variable is changed (or retained) according to a Markov transition matrix P, where `P_kl = P(output=l | original=k)`. The mechanism is stochastic and applied independently per record.

Key properties:
- Because the matrix P is known, analysts can correct frequency table estimates: `T̂_original = (P⁻¹)ᵀ · T_perturbed`
- An invariant PRAM matrix can be constructed such that marginal distributions are preserved in expectation (`P·π = π`). Off-diagonal entries are set uniformly at rate `off_diag / (K-1)`; diagonal entries absorb the rest.
- The correction matrix `(P⁻¹)ᵀ` should be published with the dataset so analysts can unbias frequency estimates.

Implementation: for small category counts (K < 50), the simplest approach is to iterate over categories, generate `numpy.random.choice` draws using the row's probability vector, and write back via Polars `replace`. For large K, build a lookup DataFrame with cumulative probabilities and use an inequality join to map uniform random draws to new categories.

Key parameters: `col`, `transition_matrix` (dict of dicts, row sums must be 1.0), `seed`.

### `methods/perturbative/shuffling.py` — Shuffling (Muralidhar and Sarathy 2006)

Shuffling protects continuous sensitive variables by preserving all original values but reassigning which record each value belongs to. Unlike noise-based methods, the released data contains no modified values — only re-associations.

Algorithm:
1. Fit a regression model: `sensitive_cols ~ predictor_cols` (e.g., ordinary least squares via `numpy.linalg.lstsq`).
2. Generate predicted values for each record (the "synthetic" sensitive values).
3. Rank the predicted values. Assign to each rank position the original value that occupies that rank position in the original data — effectively rank-map original values through the prediction order.
4. The result: original values survive intact, but their associations with predictor variables are broken.

The key privacy argument: an adversary who knows a respondent's predictor values cannot use a regression model to predict their sensitive value, because the original regression relationship has been replaced with a shuffled one.

Implementation requires collect to NumPy for the regression fit and rank mapping. Return a LazyFrame with the shuffled columns replacing the originals.

Limitations: only protects the sensitive columns against regression-based inference from the predictor columns. Does not address k-anonymity or re-identification via key variable combinations.

Key parameters: `sensitive_cols`, `predictor_cols`, `seed`.

### `methods/perturbative/synthetic.py` — Synthetic Data (Section 3.4.7)

Two approaches from the handbook:

**Distribution fitting (Liew et al. 1985)** — for each column, fit the best-matching parametric distribution from a candidate set (normal, lognormal, gamma, Pareto, etc.) using minimum KS statistic. Draw n samples from the fitted distribution, then rank-map the drawn samples to the original rank order to preserve ordering structure without exactly copying original values.

**Bootstrap resampling (Fienberg)** — draw t independent bootstrap samples of the full dataset, sort all by the same rank criterion, and report the average of j-th ranked values across t samples. Smooths out individual values while preserving the distributional shape.

Both require collect to NumPy. The distribution-fitting approach is the more principled option for unimodal columns; bootstrap is simpler and distribution-free.

Key parameters: `cols`, `seed`, `t` (number of bootstrap draws, for resampling only).

## Information Loss Measures (Section 3.5)

### `loss/continuous.py`

**λ (lambda) measure** — the primary overall measure in the handbook. For each column: `λ_k = mean|original − protected| / range(original)`. The overall λ is the mean across columns. The handbook reports ~14% overall λ for a mix of PRAM + noise + suppression on a real survey dataset (Table 3.12).

**IL1s (Yancey-Winkler scaled distance)** — `IL1s = Σ_ij |x_ij − z_ij| / s_j` where s_j is the standard deviation of column j. More sensitive to large absolute deviations than λ because it normalizes by std rather than range.

**SSE/SST ratio** — within-group sum of squares divided by total sum of squares. The primary quality measure for microaggregation: lower SSE/SST means tighter groups (less information loss). Polars: `group_by(group_col).agg(mean)` → join → compute `(value − group_mean)²` for SSE; `(value − global_mean)²` for SST.

**Eigenvalue differences** — compare the eigenvalue spectrum of the covariance matrix of original vs. protected data. If perturbation inflates or deflates principal components, this will show in the eigenvalue ratios. Computed via `numpy.linalg.eigh` after collecting both frames.

**lm (regression coefficient comparison)** — fit the same linear regression model on original and protected data; compare regression coefficients. Large deviations indicate that the perturbation has distorted the analytical relationships that matter to users. Requires collect to NumPy for `numpy.linalg.lstsq`.

### `loss/categorical.py`

**KL divergence** — `D_KL(P_orig || P_prot) = Σ P(x) log(P(x)/Q(x))` per column. Measures how much the category frequency distribution has changed. A value near zero means the distribution is preserved. Add a small ε (1e-12) to avoid division by zero for categories that appear in the original but not the protected data.

**Frequency distortion** — mean absolute difference in category frequencies: `Σ|f_orig − f_prot| / K`. Simpler and more interpretable than KL divergence; expressed in percentage points. Both measures should be reported; KL divergence is more sensitive to small deviations in rare categories.

## Pipeline Orchestrator

`SDCPipeline` wraps an `SDCFrame`, chains operations in a fluent style, and tracks the original frame separately for information loss computation at the end.

```python
frame = SDCFrame(
    lf=pl.scan_parquet("data/policyholders.parquet"),
    identifiers=["customer_id", "rrn", "phone"],
    quasi_identifiers=["gender", "age_band", "region", "occupation"],
    sensitive=["income", "premium", "claim_amount"],
    weights="survey_weight",
)

pipeline = (
    SDCPipeline(frame)
    .drop_identifiers()
    .bottom_code(["income"], quantile=0.01)
    .top_code(["income", "claim_amount"], quantile=0.99)
    .round("income", base=1_000_000)
    .round("premium", base=10_000)
    .global_recode("region", mapping={"Seoul": "Capital", "Incheon": "Capital",
                                      "Busan": "Southeast", "Ulsan": "Southeast"})
    .global_recode("age_band", breaks=[0, 30, 40, 50, 60, 200],
                   labels=["<30", "30-39", "40-49", "50-59", "60+"])
    .add_noise(["income", "premium", "claim_amount"], alpha=0.1, mode="multiplicative")
    .microaggregate_mdav(["income", "premium"], k=3)
    .pram("occupation", matrix=occupation_transition_matrix)
    .shuffle(sensitive_cols=["claim_amount"], predictor_cols=["age_band", "region"])
    .enforce_k_anonymity(["gender", "age_band", "region"], k=3)
)

result = pipeline.collect()
print(pipeline.loss_report(continuous_cols=["income", "premium", "claim_amount"]))
print(pipeline.risk_report())
```

`risk_report()` returns: k-anonymity violation count and rate, global individual risk rate ξ, and distance-based linkage rate for continuous variables.

`loss_report()` returns: λ per column and overall, SSE/SST per continuous column (if microaggregation was applied), and KL divergence per categorical column.

## Suggested Implementation Order

1. **`types.py`** — `SDCFrame`. Zero dependencies; defines the shared data contract.
2. **`methods/non_perturbative/`** — recoding, top/bottom coding, suppression. Pure Polars expressions; testable immediately against known outputs.
3. **`loss/`** — loss metrics. Needed to validate all subsequent methods.
4. **`methods/perturbative/noise.py`** — uncorrelated noise first (simplest), then correlated, then multiplicative. Validate: mean preserved within 1% at n=10,000.
5. **`methods/perturbative/rounding.py`** and **`rank_swap.py`**.
6. **`methods/perturbative/microaggregation.py`** — univariate DP first, then MDAV. Validate: all groups ≥ k; SSE/SST decreases as k decreases.
7. **`methods/perturbative/pram.py`** — validate correction matrix: `(P⁻¹)ᵀ · T_perturbed ≈ T_original`.
8. **`methods/perturbative/shuffling.py`** — validate that regression of sensitive on predictors gives a weaker R² on protected data.
9. **`risk/frequency.py`** — k-anonymity, l-diversity, t-closeness, ARGUS.
10. **`risk/individual.py`** — Benedetti-Franconi risk + benchmark outliers.
11. **`risk/hierarchical.py`** and **`risk/outlier.py`**.
12. **`risk/suda.py`** — SUDA scores.
13. **`risk/record_linkage.py`** — distance-based linkage rate as final risk measure.
14. **`pipeline.py`** — fluent orchestrator wrapping all of the above.
15. **`methods/perturbative/synthetic.py`** — distribution fitting and bootstrap synthesis.

## Dependencies

| Purpose | Library |
|---|---|
| DataFrames | `polars >= 0.20` |
| Numerical computing | `numpy`, `scipy` |
| Nearest-neighbour (large n) | `scikit-learn` (`BallTree`, `MinCovDet`) |
| Categorical clustering | `kmodes` |
| Distribution fitting | `scipy.stats` |
| Optional type hints | `typing_extensions` |

For MDAV and all noise operations that require collect-to-numpy: collect once, compute in NumPy, return a `pl.LazyFrame`. The main query plan never materializes large intermediate frames unnecessarily.

## References

Hundepool, A., Domingo-Ferrer, J., Franconi, L., Giessing, S., Lenz, R., Naylor, J., Schulte Nordholt, E., Seri, G., De Wolf, P-P., Tent, R., Młodak, A., Gussenbauer, J., Wilak, K. (2026). *Handbook on Statistical Disclosure Control*, 2nd Edition. Centre of Excellence SDC / STACE project.

Templ, M., Meindl, B., Kowarik, A., Chen, S. (2014). *Introduction to Statistical Disclosure Control (SDC)*. IHSN Working Paper No. 007. International Household Survey Network.
