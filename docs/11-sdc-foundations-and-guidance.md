# 11 — Statistical Disclosure Control: Foundations and Implementation Guidance

This document is a conceptual review and engineering guide derived from two primary references:

- *Handbook on Statistical Disclosure Control*, 2nd ed. (Hundepool et al., 2026) — the authoritative European methodological reference
- *Introduction to Statistical Disclosure Control*, IHSN Working Paper No. 007 (Templ, Meindl, Kowarik and Chen, 2014) — a practitioner-oriented guide with the clearest workflow model

The goal is to build a durable understanding of what SDC is, what the moving parts are, where the real difficulty lies, and what that means for a Python implementation using Polars.

## What SDC Is Actually About

Statistical Disclosure Control is not primarily a technical problem. It is an adversarial modelling problem: given that you intend to release a dataset, and given that a motivated adversary will try to learn something they are not supposed to learn from it, what transformations must you apply to the data so that the adversary fails while the legitimate analyst does not?

This framing is important because it immediately rules out the naive interpretation — that SDC is about "masking" fields or "removing sensitive columns." Masking a name still leaves a name-shaped field. Removing a column an adversary already knows from another source does nothing. SDC must be understood as a systematic process of reducing the information content of the data in targeted, measurable ways, with explicit models of what the adversary knows and what they can infer.

The two properties that must hold simultaneously after any SDC process are:

**Privacy**: the probability that an adversary can successfully re-identify an individual, or correctly infer a sensitive value for an identified individual, falls below an acceptable threshold.

**Utility**: the data remains analytically valid — summary statistics, regression coefficients, distributional properties, and policy-relevant indicators are sufficiently preserved that the intended uses of the dataset remain possible.

These two objectives are fundamentally in tension. Every transformation that reduces disclosure risk also reduces the precision of the data, which reduces its utility. SDC methodology is the science of navigating this tradeoff systematically rather than arbitrarily.

## Disclosure Types

The handbook distinguishes three types of disclosure. Understanding which type you are protecting against determines which methods are appropriate.

**Identity disclosure** occurs when an adversary successfully links a record in the released dataset to a specific real individual. The adversary does not need to learn anything new — re-identification itself is the breach. This is the primary concern when releasing microdata (individual-level records), because once a record is linked to a known person, all variables in that record become disclosed about that person.

**Attribute disclosure** occurs when an adversary learns the value of a sensitive variable for a specific individual, either by re-identifying them through identity disclosure or by inferring the value from the structure of the data without re-identifying anyone. A dataset where everyone in a given postcode has the same diagnosis value discloses that diagnosis for every person in that postcode, even if no record can be individually identified. Attribute disclosure is possible even in a k-anonymous dataset — this is why k-anonymity alone is insufficient.

**Inferential disclosure** occurs when the released data allows an adversary to infer a sensitive value with significantly higher confidence than was possible before the data was released. This is the subtlest and hardest to prevent: even well-aggregated data can reveal that membership in a group is correlated with a sensitive outcome in a way that the adversary can exploit. In practice, full prevention of inferential disclosure is intractable; the goal is to ensure that the marginal gain in inference probability from the released data is negligible.

For most practical microdata protection work — including insurance data, survey data, and administrative records — the primary targets are identity and attribute disclosure. Inferential disclosure is the concern of output checking in remote-access facilities and is largely out of scope for pre-release microdata protection.

## Variable Taxonomy

Every column in a microdata set plays a specific role in the disclosure risk model. The IHSN guide provides the cleanest classification, and it is the framework adopted here.

### Direct Identifiers

Direct identifiers are variables that uniquely name or directly reference a statistical unit without any linking: resident registration number, customer ID, full name, email address, phone number, passport number, account number. A single direct identifier is sufficient for an adversary to re-identify a record.

The correct action for direct identifiers is deletion, not masking. This is a critical design principle. Replacing a name with `XXXX` or hashing a social security number still leaves a functionally identifying field in the record — worse, it may give a false sense of protection. The handbook is emphatic: direct identifiers must be stripped from the dataset before any other processing begins. They are never SDC-transformed because transformation does not address the fundamental problem.

### Key Variables (Quasi-Identifiers)

Key variables are variables that are not directly identifying in isolation but become identifying in combination. Age, gender, region, occupation, and marital status are individually innocuous; their joint distribution is not. A study by Sweeney (2002) demonstrated that 87% of the US population could be uniquely identified from {date of birth, gender, five-digit zip code} alone — all three are quasi-identifiers present in publicly available records.

The threat model for quasi-identifiers is **record linkage**: an adversary holds an external database (public records, social media, another leaked dataset) that also contains some or all of the quasi-identifier columns. By matching on the combination, they can link records in the released dataset to known individuals, exposing all other variables.

Quasi-identifiers are the primary target of SDC. The goal is to reduce their joint identifying power — typically to ensure that no combination of quasi-identifier values singles out fewer than k individuals.

### Sensitive Variables

Sensitive variables are variables whose values are confidential for specific individuals: income, medical diagnosis, credit score, claim payout, criminal history, religion, political affiliation. The disclosure risk here is attribute disclosure — an adversary who has successfully linked a record to a known individual reads off the sensitive value.

The key insight from the IHSN guide is that **a variable can be both a key variable and sensitive**. Income is the canonical example: it is simultaneously useful for record linkage (people have roughly known income ranges based on their occupation and region) and confidential in its precise value. For such variables, the SDC treatment must address both roles.

### Data Type Dimension

Cutting across the three functional roles above is a second classification by data type. This determines which SDC methods are applicable:

**Categorical variables** take values from a finite, unordered (or weakly ordered) set: gender, region code, occupation class, product type, marital status. There is no meaningful arithmetic on categories — averaging "Male" and "Female" is undefined. Methods for categorical variables must operate on the category memberships: merging categories, nulling them out, or stochastically reclassifying them.

**Continuous variables** take numerical values for which arithmetic operations are valid: age, income, premium, claim amount, years of employment. Their risk profile is different — they are identifying through their precise values and their correlational structure with other continuous variables. Methods for continuous variables can exploit the metric structure: adding noise, replacing groups of values with their mean, or swapping values within a bounded rank window.

## The Risk-Utility Loop

The IHSN guide's Figure 2 establishes the operational workflow. It is not a one-shot pipeline — it is an iterative loop.

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
                                    and repeat
```

The pre-processing step is where the majority of analytical work happens: deciding which columns are which, what the plausible adversary population looks like, what external databases they might have access to, and what risk level is acceptable for this release context. These are judgement calls that cannot be automated. The SDC practitioner must reason about the threat model explicitly before touching the data.

The categorical and continuous tracks are executed independently. This is methodologically correct — the methods do not compose across data types, and the risk model is different for each. They are reunited for the final risk and utility assessment.

The loop continues until both thresholds are met simultaneously. In practice this requires several iterations, often involving coarsening the quasi-identifiers more aggressively or increasing the noise level, at each stage checking whether the utility loss is acceptable.

## Risk Assessment

### k-Anonymity and Its Limits

k-anonymity, introduced by Samarati and Sweeney (1998), is the baseline property for microdata protection: every combination of quasi-identifier values must appear in at least k records. A record that is the only one with a particular combination of quasi-identifier values is a **sample unique** — it can be directly re-identified by an adversary who holds those quasi-identifier values for the target individual.

k=3 is a common minimum threshold in European national statistics practice. It means an adversary who correctly matches on the quasi-identifiers still faces a 1-in-3 uncertainty about which record is their target's.

However, k-anonymity has two well-known structural weaknesses:

**Homogeneity attack**: if all k records in a group share the same sensitive variable value, attribute disclosure is trivially achieved even without identity disclosure. Knowing that someone is in a group does not protect their diagnosis if everyone in the group has the same diagnosis.

**Background knowledge attack**: an adversary with prior knowledge about the target (e.g., knowing they do not smoke) can eliminate records within the k-anonymous group that contradict that knowledge, potentially reducing the effective group size below k.

### l-Diversity

l-diversity (Machanavajjhala et al., 2006) strengthens k-anonymity by requiring that within each k-anonymous group, the sensitive attribute takes at least l meaningfully different values. This prevents the homogeneity attack: an adversary who narrows a target to a group cannot infer the sensitive value if the group contains genuine diversity.

"Meaningfully different" has several formulations: distinct (l distinct values), entropy (entropy of the distribution ≥ log l), or recursive (the most frequent value appears less often than the sum of the rest). The recursive form is the strictest.

### t-Closeness

t-closeness (Li et al., 2007) extends l-diversity by requiring that the distribution of the sensitive attribute within each group is statistically close to the distribution in the full population. The distance is measured by Earth Mover's Distance (Wasserstein-1). This prevents a more subtle form of attribute disclosure: an attacker who knows the group contains diverse values may still infer that the target has an above-average income if the group's income distribution is skewed high relative to the population.

t-closeness is substantially harder to achieve than k-anonymity or l-diversity and imposes larger information loss. It is the right standard when the sensitive variable's distributional relationship to the quasi-identifiers is itself the sensitive information.

### Individual Risk — Benedetti-Franconi Model

Population uniqueness (the true re-identification risk) and sample uniqueness (what we can measure in the released data) diverge in a non-trivial way when the data is a sample from a larger population. A record may be unique in the sample simply because the sampling rate was low, not because the individual is truly rare.

The Benedetti-Franconi model (1998) addresses this using a Negative Binomial posterior. Given that a key combination appears `f_k` times in the sample with design-weight sum `F̂_k` (the estimate of how many people in the population share this key combination), the individual risk of a record in that combination is:

`r_i ≈ f_k / F̂_k`

For a population unique (F̂_k ≈ 1), this approaches 1. For a record in a large population group, this is small. The global risk rate ξ = (1/n) Σ r_i is the primary scalar summary of overall dataset risk. The handbook reports this as the number to track across SDC iterations.

The benchmark approach (IHSN guide) complements the global rate with a relative outlier measure: a record is a **benchmark risk outlier** if its individual risk exceeds both an absolute threshold and the value `median(r) + c·MAD(r)`. This identifies the tail of the risk distribution — records that are not just above threshold but anomalously risky relative to the rest.

### Hierarchical Risk

For household or grouped survey data, the risk of one member being re-identified propagates to all members of the same household. An adversary who identifies a head of household immediately knows who the other members are. The household risk is:

`r_hh = 1 − Π_i (1 − r_i)` for all individuals i in the household.

This grows rapidly when a household contains even one high-risk individual. Household-level datasets must assess risk at the household level, not only the individual level.

### SUDA — Special Uniques

SUDA (Special Uniqueness Detection Algorithm) identifies records that are unique on some **minimal** subset of quasi-identifier attributes. A minimal sample unique (MSU) is a set of attributes S on which the record is unique and for which no proper subset of S is already unique. MSUs are significant because they indicate the minimum information an adversary needs to re-identify the record.

The SUDA score weights MSUs by their size: each MSU of size k contributes `ATT^(M−k)` to the record's score, where ATT is the total number of key attributes and M is the maximum MSU size to search. Smaller MSUs score exponentially higher — a record unique on a single attribute is far more dangerous than one unique on four attributes combined.

This is the most computationally intensive risk measure: it requires checking all subsets of size 1 through M, making it O(C(ATT, M)) group-by operations. For typical microdata configurations (ATT ≈ 5–10, M ≈ 3–4) this is manageable.

### Record Linkage Risk for Continuous Variables

For continuous key variables, the standard risk measure is distance-based record linkage. Before any SDC, every record in the dataset links to itself with probability 1 (the linkage rate is 100% by definition). After perturbation, the linkage rate is the proportion of protected records whose nearest neighbour in the original dataset — by standardized Euclidean distance — is their own pre-perturbation record. This is the adversary's success rate if they have access to the original microdata structure and are trying to recover values after release.

The IHSN guide also defines an upper-bound metric: a record is "linked" if the correct original is among the top-2 nearest neighbours. The lower bound (exact nearest) and upper bound (within top-2) bracket the true linkage risk. After MDAV microaggregation with k=3, the IHSN guide reports this interval as [0%, 61.42%] — the lower bound is zero (no record's exact original is the nearest neighbour) but the upper bound remains high.

**Interval disclosure risk** is a complementary measure: given the masked value and knowledge of the masking method, what fraction of records have their original value within ±ε of the masked value? This models an adversary who uses the published masked value as an approximate pointer to the original.

### Outlier Risk — Robust Mahalanobis Distance

Outlier records are disproportionately identifiable in continuous data: an adversary looking for "the enterprise with the highest turnover" or "the individual with the largest claim" exploits extreme values directly, not linkage. Templ and Meindl (2008) define the RMDID1 measure: compute Robust Mahalanobis Distance for each record (using Minimum Covariance Determinant for robustness), use the RMD to scale a disclosure interval around the original value, and count what fraction of masked values fall within this interval. Records with high RMD (outliers) have larger disclosure intervals — they are intrinsically harder to protect.

## SDC Methods

### Non-Perturbative Methods

Non-perturbative methods reduce the resolution of the data without introducing distortion. They do not add noise — they simply throw away information.

**Global recoding** merges categories or discretizes continuous ranges uniformly across the entire dataset. Occupation categories might be coarsened from 400 ISCO codes to 20 broad occupational groups; exact age might be replaced with five-year age bands; region might be aggregated from district to province level. Global recoding is the preferred first step for categorical quasi-identifiers because it reduces the number of distinct combinations in the dataset systematically, without creating inconsistencies between records. The key cost: the analyst loses the ability to distinguish between merged categories entirely.

**Local suppression** sets individual field values to null for records whose quasi-identifier combination is unsafe, while leaving the rest of the record intact. Unlike global recoding, it targets only unsafe records. The minimum-suppression problem (find the smallest set of values to null that brings all records to k-anonymity) is NP-hard; the practical heuristic suppresses the highest-cardinality quasi-identifier column first, iterating until the record is safe. Local suppression is the tool of last resort for records that remain unsafe after recoding — it avoids throwing away information for the majority of records.

**Top and bottom coding** replaces extreme values with threshold values: all incomes above the 99th percentile become the 99th-percentile value; all claims below the 1st percentile become the 1st-percentile value. Extreme values are disproportionately identifying and disproportionately rare — a single extremely high income may correspond to one identifiable person. Top coding is both a disclosure protection measure and a standard statistical practice for handling outliers. The threshold values should be published.

### Perturbative Methods — Categorical

**PRAM (Post-Randomization Method)** is the primary perturbative method for categorical variables. Each record's value on a categorical variable is stochastically reclassified according to a Markov transition matrix P, where `P_kl` is the probability that a record with original category k is reclassified to category l. The mechanism is known and fixed, which enables analysts to correct frequency table estimates: if the analyst knows P, they can compute `T̂_original ≈ (P⁻¹)ᵀ · T_perturbed`.

The invariant PRAM variant constructs P such that the marginal distribution of the variable is preserved in expectation — `P·π = π`. This ensures that frequency-based analyses are unbiased even without explicit correction. The tradeoff is that an invariant P has lower off-diagonal probabilities, meaning less perturbation and therefore less protection.

PRAM introduces genuine uncertainty at the record level: even for a record whose value did not change, the analyst cannot know whether the value is original or the result of a reclassification back to the same category. This is the source of the privacy guarantee.

### Perturbative Methods — Continuous

**Noise addition** injects random error drawn from a distribution centered at zero. The four variants from the handbook differ in what properties they preserve:

*Uncorrelated noise* adds independent Gaussian noise to each column scaled to a fraction α of its variance. Means are preserved; individual variances inflate by factor (1+α); correlations between variables are not affected in expectation. This is the simplest variant and the right default.

*Correlated noise* draws noise from a multivariate normal with covariance αΣ, where Σ is the sample covariance matrix. Means and correlation coefficients are preserved; variances inflate uniformly. This is appropriate when preserving the correlation structure of the protected data is a priority.

*Multiplicative noise* (Höhne 2004) perturbs each value by a multiplicative factor of approximately `1 ± f`, with alternating signs balanced so the column sum is exactly preserved. Perturbation scales with the value magnitude, making this the correct choice for heavily skewed business data (claim severity, revenue) where additive noise would over-perturb large values and under-perturb small ones.

*Noise with linear transform* (Kim 1986) applies uncorrelated noise then rescales the result so both the mean and variance of each column are exactly preserved. It does not preserve correlations exactly. Because the scale parameter c must be published for analysts to use it correctly, its disclosure is a design decision.

**Microaggregation** achieves k-anonymity on continuous variables by partitioning records into groups of size ≥ k and replacing each record's values with the group mean. Every group has exactly the same values for the protected columns — k-anonymity is by construction. The design challenge is the partitioning: the optimal partition minimises within-group variance (SSE), which minimises information loss.

For a single variable, Hansen and Mukherjee (2003) showed that the globally optimal partition is computable in O(k²n) time using dynamic programming on the sorted values — a rare case where an NP-looking optimization has a polynomial solution.

For multiple variables simultaneously, optimal partitioning is NP-hard. The MDAV (Maximum Distance to Average Vector) heuristic from μ-ARGUS is the standard practical solution: iteratively identify the two most extreme records (by distance from the current centroid and from each other), assign the k nearest records to each as groups, remove them, and repeat. MDAV is O(n²/k), acceptable for typical microdata sizes; for very large n, approximate nearest-neighbour (BallTree) reduces this substantially.

Categorical microaggregation (Torra 2004) uses the k-modes algorithm (Hamming distance, mode aggregation) to cluster records, then replaces each record's categorical values with the cluster mode.

**Rank swapping** (Moore 1996) sorts records by value and swaps each record's value with a randomly chosen partner within a window of ±p·n rank positions, applied independently per column. All original values survive in the output — only their record-level assignments change. Rank swapping preserves the marginal distribution of each variable exactly (by construction) and is empirically competitive with microaggregation on the risk-utility frontier. Because swaps are independent per column, correlations between columns are weakened.

**Shuffling** (Muralidhar and Sarathy 2006) is the correct method when the goal is to protect continuous sensitive variables against regression-based inference while preserving all original values. The algorithm: fit a regression model predicting the sensitive variables from non-sensitive predictor variables; generate predicted (synthetic) values for each record; rank-map the original values through the prediction order so that each record receives an original value that corresponds to its predicted rank. The result: original values survive intact, but the relationship between sensitive values and the predictor variables is broken. An adversary who knows the predictors for a target cannot use regression to recover the target's sensitive value.

**Rounding** replaces each value with the nearest multiple of a base b: ages rounded to 5-year multiples, incomes to the nearest 100,000 won. Controlled rounding is deterministic and easy to understand. Stochastic rounding rounds up or down with probability proportional to the fractional distance from each multiple, preserving expected value (unbiased). Rounding is often applied as a first pass before noise or microaggregation, reducing the precision of quasi-identifiers without full recoding.

## Information Loss Measurement

Every SDC transformation must be followed by measurement of information loss. Without this, there is no way to know whether the protected data remains analytically useful.

**λ (lambda) measure** — the primary scalar summary in the handbook (Table 3.12). For each column: `λ_k = mean|original − protected| / range(original)`. Overall λ = mean across columns. The handbook reports approximately 14% overall λ for a realistic mix of PRAM, noise, and suppression on a survey dataset. This is the number to track across iterations.

**IL1s (Yancey-Winkler scaled distance)** — `Σ_ij |x_ij − z_ij| / s_j` where s_j is the standard deviation of column j. More sensitive to large deviations than λ because it normalizes by standard deviation rather than range — useful when the variable has a long tail.

**SSE/SST ratio** — within-group sum of squares divided by total sum of squares. The natural measure for evaluating microaggregation quality: lower SSE/SST means tighter groups and less information loss. Polars can compute this efficiently via `group_by → agg(mean) → join → compute squared deviations`.

**Eigenvalue comparison** — comparing the eigenvalue spectrum of the covariance matrix of original versus protected data detects whether perturbation has distorted the principal components. This matters for any analysis that depends on the correlation or variance structure — factor analysis, PCA, regression. Large eigenvalue deviations indicate that the perturbation has changed what the data "explains."

**Regression coefficient comparison (lm)** — fit the same linear model on original and protected data; compare coefficients. This directly tests whether the analytical relationships that the dataset was designed to support (e.g., how income predicts claim frequency) survive the SDC process. This is the most interpretation-grounded utility measure.

**KL divergence** (categorical) — `D_KL(P_orig || P_prot) = Σ P(x) log(P(x)/Q(x))` per column. Quantifies how much the category frequency distribution has shifted. A value near zero means PRAM or recoding has preserved the distribution. Small KL divergence is achievable with invariant PRAM.

## What to Focus on for Pseudonymizing Data

Given the context of this project — insurance policyholder data, survey-linked administrative records — the practical priorities are:

**The quasi-identifier combination is almost always the problem.** In insurance data, the combination of {age band, gender, region, occupation class, product type} is often enough to narrow a policyholder to a very small group. Before applying any method, compute the k-anonymity violation rate. If 30% of records are sample uniques (k=1), the data needs substantial recoding before perturbative methods are even meaningful.

**Global recoding first, perturbation second.** Non-perturbative methods should be applied first: coarsen age to bands, merge low-frequency region codes, collapse rare occupation categories. This brings the majority of k-violations to safety with minimal analytical cost. The remaining unsafe records are handled by local suppression or — if suppression is too aggressive — additional recoding of the most cardinality-dense quasi-identifier.

**For continuous sensitive variables, microaggregation or shuffling — not noise alone.** Noise addition is well-understood and easy to implement, but it has a fundamental weakness: the adversary knows you added noise, and if they have approximate prior knowledge of the original value, they can construct a neighbourhood around the masked value and recover it with non-trivial probability. Microaggregation provides a structural guarantee (every record has at least k − 1 identical neighbours on the protected columns) that noise cannot. Shuffling is the right choice specifically when the sensitive variable has a strong analytical relationship with the predictors that must be preserved — it breaks the individual-level association while preserving the distributional properties.

**Treat variables that are both quasi-identifiers and sensitive with extra care.** Income or premium in insurance data may be both identifying (used for linkage) and sensitive (the value itself is confidential). These require protection against both disclosure types. The practical approach: recode income into bands (reducing its linkage power) and apply noise or microaggregation to the banded or original values (reducing attribute disclosure risk). Whether to perturb before or after banding depends on which analytical use case is primary.

**The individual risk score, not the k-violation count, is the right scalar to minimize.** k-anonymity is a necessary condition, not a sufficient one. After achieving k-anonymity through recoding, compute individual risk rates ξ for the remaining dataset and use the benchmark outlier measure to identify records that remain disproportionately risky. These are your residual problem records — they require additional treatment (further suppression, microaggregation into their group, or top-coding).

**Information loss must be measured before declaring success.** Each iteration of the risk-utility loop requires both a risk measurement and a utility measurement. The λ measure gives a fast, interpretable summary. For the specific analytical use cases of this dataset — if the primary use is regression modelling, the lm coefficient comparison is essential; if it is frequency analysis of categorical variables, KL divergence is primary.

## Implementation Guidance

The core delivery is a **Python library packaged as a wheel** (`sdc_polars`). This establishes a clean, testable, importable unit with no runtime dependencies on a web framework. The library owns the domain model, the SDC execution engine, and the risk and loss measurement layer.

The primary design goal for the library is that every SDC operation is driven by a **declarative rule** — a named method with explicit, serializable parameters — rather than imperative function calls with inlined logic. This means the library can later be surfaced as a **Flask API** with minimal additional code: routes receive JSON rule lists, deserialize them into the library's domain objects, execute via the same engine, and return structured JSON reports. A UI sits naturally on top of that API, rendering each rule as a form and assembling the pipeline interactively.

### What this means for the library design

**The domain model is the shared contract.** Column roles (identifiers, quasi-identifiers, sensitive, weights, hierarchy), SDC rules (method + parameters), risk thresholds, and utility requirements are all first-class domain objects. They must be serializable to and from JSON without loss of information. This keeps the library's core logic independent of both the web layer and any particular data format.

**SDC rules carry all parameters explicitly.** Each rule specifies its method, the target columns, and all numeric parameters (k, α, quantile, base, seed, etc.). There are no global defaults embedded in the engine. This is essential for auditability: a stored rule list is a complete, reproducible description of the SDC process applied to a dataset.

**The library exposes three logical operations** that map directly to future API endpoints: assess risk on the original data; apply a rule list and return the protected data together with a risk + utility report; and simulate (run the pipeline, return only the report, not the data) for interactive parameter tuning.

**Polars is the execution backend.** Internally, the engine translates each rule into Polars LazyFrame operations, collects to NumPy only where required by specific algorithms (MDAV, DP microaggregation, correlated noise, SUDA), and defers `.collect()` until the final output step. This is an implementation detail that the library's public interface does not expose.

### Implementation sequence

1. Domain model — rule types, parameter schemas, column role annotation. No data processing; just the configuration and validation layer.
2. Risk assessment — k-anonymity, individual risk, record linkage. Establishes the measurement baseline.
3. Non-perturbative methods — recoding, suppression, top/bottom coding.
4. Loss measures — λ, KL divergence, SSE/SST. Required to evaluate all subsequent methods.
5. Perturbative methods in order of complexity — noise variants, rounding, rank swap, PRAM, microaggregation (univariate then MDAV), shuffling.
6. Pipeline orchestrator — chains rules into a single execution, collects the risk + utility report at the end.
7. Advanced risk measures — SUDA, hierarchical risk, RMD outlier risk — as a second phase once the core loop is validated.
8. Flask wrapper — thin API layer over the library's three logical operations. This phase defines the JSON request/response contracts and is the integration point for a future UI.

## References

Hundepool, A., Domingo-Ferrer, J., Franconi, L., Giessing, S., Lenz, R., Naylor, J., Schulte Nordholt, E., Seri, G., De Wolf, P-P., Tent, R., Młodak, A., Gussenbauer, J., Wilak, K. (2026). *Handbook on Statistical Disclosure Control*, 2nd Edition. Centre of Excellence SDC / STACE project.

Templ, M., Meindl, B., Kowarik, A., Chen, S. (2014). *Introduction to Statistical Disclosure Control (SDC)*. IHSN Working Paper No. 007. International Household Survey Network.
