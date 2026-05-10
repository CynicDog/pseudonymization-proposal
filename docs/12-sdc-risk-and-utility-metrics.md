# SDC Risk and Utility Metrics

SDC metrics split into two families that mirror the two objectives of the risk-utility loop. **Disclosure risk metrics** quantify how successfully an adversary could re-identify a record or infer a sensitive value from the released data — they answer the privacy question. **Information loss metrics** quantify how much the analytical content of the data has been degraded by protection — they answer the utility question. Both must be computed and reported after every iteration of the SDC pipeline described in [doc 10](10-sdc-foundations-and-guidance.md#the-risk-utility-loop): a release is acceptable only when both fall within their pre-declared thresholds. Neither family is sufficient alone — a dataset can be perfectly safe by being empty, and a dataset can be perfectly useful by being unprotected. The choice of which specific metric drives an iteration depends on the methods applied (see the methods×metrics map below) and on the variable taxonomy from [doc 10](10-sdc-foundations-and-guidance.md#variable-taxonomy).

## Methods × Metrics Reference

*Each masking method has a "native" risk metric (the disclosure dimension it most directly addresses or fails to address) and a "native" utility metric (the loss measure most natural to the method's mechanics). This table maps every method in [doc 11](11-sdc-microdata-protection-methods.md) to the metrics that should appear in its post-iteration risk-utility report. Methods are listed in doc 11's order.*

| Method | Native Risk Metric | Native Utility Metric |
|---|---|---|
| [Noise Addition](11-sdc-microdata-protection-methods.md#noise-addition) | [Record linkage (top-1 / top-2)](#record-linkage-risk-distance-based), [Interval disclosure](#interval-disclosure-risk) | [λ measure](#-measure), [Eigenvalue spectrum](#eigenvalue-spectrum-comparison) |
| [Multiplicative Noise](11-sdc-microdata-protection-methods.md#multiplicative-noise) | [Record linkage](#record-linkage-risk-distance-based), [RMDID1](#rmdid1--robust-mahalanobis-disclosure-risk) | [λ measure](#-measure), [Regression coefficient (lm)](#regression-coefficient-comparison-lm) |
| [Microaggregation](11-sdc-microdata-protection-methods.md#microaggregation) | [k-Anonymity](#k-anonymity) (by construction), [Record linkage (top-1)](#record-linkage-risk-distance-based) | [SSE / SST](#sse--sst-ratio) |
| [Data Swapping](11-sdc-microdata-protection-methods.md#data-swapping) | [Record linkage (top-1)](#record-linkage-risk-distance-based), [Individual risk](#individual-risk-benedetti-franconi) | [KL divergence](#kl-divergence-categorical), [λ measure](#-measure) |
| [Rank Swapping](11-sdc-microdata-protection-methods.md#rank-swapping) | [Record linkage (top-1 / top-2)](#record-linkage-risk-distance-based) | [λ measure](#-measure), [Regression coefficient (lm)](#regression-coefficient-comparison-lm) |
| [Rounding](11-sdc-microdata-protection-methods.md#rounding) | [Interval disclosure](#interval-disclosure-risk) | [λ measure](#-measure) |
| [Resampling](11-sdc-microdata-protection-methods.md#resampling) | [Record linkage](#record-linkage-risk-distance-based), [RMDID1](#rmdid1--robust-mahalanobis-disclosure-risk) | [λ measure](#-measure), [Eigenvalue spectrum](#eigenvalue-spectrum-comparison) |
| [PRAM](11-sdc-microdata-protection-methods.md#pram-post-randomisation-method) | [Individual risk](#individual-risk-benedetti-franconi), [t-Closeness](#t-closeness) | [KL divergence](#kl-divergence-categorical) |
| [MASSC](11-sdc-microdata-protection-methods.md#massc) | [k-Anonymity](#k-anonymity) (by construction), [SUDA score](#suda-score-msu) | [KL divergence](#kl-divergence-categorical), [λ measure](#-measure) |
| [Sampling](11-sdc-microdata-protection-methods.md#sampling) | [Individual risk](#individual-risk-benedetti-franconi) (via inclusion uncertainty) | None native — values are intact; check [Regression coefficient (lm)](#regression-coefficient-comparison-lm) for design-effect bias |
| [Global Recoding](11-sdc-microdata-protection-methods.md#global-recoding) | [k-Anonymity](#k-anonymity), [SUDA score](#suda-score-msu) | [λ measure](#-measure), [KL divergence](#kl-divergence-categorical) |
| [Top and Bottom Coding](11-sdc-microdata-protection-methods.md#top-and-bottom-coding) | [RMDID1](#rmdid1--robust-mahalanobis-disclosure-risk), [Individual risk](#individual-risk-benedetti-franconi) | [λ measure](#-measure) (concentrated on the censored tail) |
| [Local Suppression](11-sdc-microdata-protection-methods.md#local-suppression) | [k-Anonymity](#k-anonymity) (by construction), [Benchmark risk outlier](#benchmark-risk-outlier) | [λ measure](#-measure) (each `*` cell as full range loss), [KL divergence](#kl-divergence-categorical) |

For sensitive variables, every method should additionally be evaluated against [l-Diversity](#l-diversity) and, where the sensitive distribution is itself disclosive, [t-Closeness](#t-closeness). For grouped or household data, escalate any individual-risk score through [Hierarchical (Household) Risk](#hierarchical-household-risk).

## Disclosure Risk Metrics

### k-Anonymity

> Every combination of quasi-identifier values must appear in at least $k$ records, so no record can be exact-matched on those quasi-identifiers alone.

$$
\forall i \in \{1, \ldots, n\}: \; \big|\{j : Q(x_j) = Q(x_i)\}\big| \geq k
$$

where $Q(\cdot)$ projects a record onto its quasi-identifier columns. The two operational scalars derived from $k$-anonymity are the **violation rate** — the share of records whose equivalence class is smaller than $k$ — and the **sample-unique count**, the number of records with $|\{j : Q(x_j) = Q(x_i)\}| = 1$. A sample unique can be matched directly by an adversary holding those quasi-identifier values.

$k$-anonymity is the baseline target for any release of microdata with quasi-identifiers; $k = 3$ is the conventional minimum in European national statistics, $k = 5$ a stricter operational threshold. It is straightforward to compute via a `group_by(quasi_identifiers).agg(len())` and is the natural target of [global recoding](11-sdc-microdata-protection-methods.md#global-recoding), [local suppression](11-sdc-microdata-protection-methods.md#local-suppression), [microaggregation](11-sdc-microdata-protection-methods.md#microaggregation) and [MASSC](11-sdc-microdata-protection-methods.md#massc).

$k$-anonymity does not protect against the homogeneity attack (all $k$ records in a group share the same sensitive value, so the sensitive value is disclosed without re-identification) nor against background-knowledge attacks (an adversary with prior knowledge eliminates records within the group). Pair $k$-anonymity with [l-Diversity](#l-diversity) for sensitive attributes and with [Individual Risk](#individual-risk-benedetti-franconi) for sample-vs-population uncertainty.

**References:** Samarati and Sweeney (1998); Sweeney (2002); Hundepool et al. (2014).

### l-Diversity

> Within each $k$-anonymous equivalence class the sensitive attribute must take at least $l$ "well-represented" values, preventing trivial inference of the sensitive value from group membership.

$$
\text{distinct}(s_i \mid Q(x_i) = q) \geq l \quad \forall q
$$

The simplest formulation requires $l$ distinct values per group. **Entropy $l$-diversity** strengthens this to $H(S \mid Q = q) \geq \log l$, where $H$ is the Shannon entropy of the sensitive attribute distribution within the group; this rules out classes that are technically $l$-distinct but dominated by a single value. **Recursive $(c, l)$-diversity** (Machanavajjhala et al. 2007) is the strictest form: ordering the in-group sensitive frequencies $r_1 \geq r_2 \geq \ldots$, it requires $r_1 < c \cdot (r_l + r_{l+1} + \ldots)$, which forces the most common value to be bounded by a multiple of the long tail.

$l$-diversity defends against the homogeneity attack that $k$-anonymity misses, and is the right metric for sensitive variables in any post-release evaluation. It is computed as a per-group aggregate after $k$-anonymity has been verified.

The metric does not protect against skewness attacks: an attribute distribution within a group can be $l$-diverse yet still concentrated heavily relative to the population (e.g., 99% of the class earns above the population median), letting an adversary infer "above-median income" with high confidence. That gap is closed by [t-Closeness](#t-closeness).

**References:** Machanavajjhala, Kifer, Gehrke and Venkitasubramaniam (2007); Hundepool et al. (2014).

### t-Closeness

> The distribution of the sensitive attribute within every equivalence class must be within Earth Mover's Distance $t$ of the population distribution, eliminating skewness-based attribute disclosure.

$$
\mathrm{EMD}\big(P(S \mid Q = q),\; P(S)\big) \leq t \quad \forall q
$$

Earth Mover's Distance is the Wasserstein-1 metric between two distributions: the minimum work required to transport probability mass from one distribution into the other. For ordinal or continuous sensitive variables it is computed from the sorted CDFs as $\mathrm{EMD}(P, Q) = \int |F_P(x) - F_Q(x)| \, dx$; for nominal variables a ground distance over the category space must be specified.

$t$-closeness is the strictest of the three group-based metrics. It is the right standard when the *relationship* between the sensitive attribute and the quasi-identifiers is itself disclosive — for example, when knowing that someone is in a particular age × region group already shifts the prior over income substantially. [PRAM](11-sdc-microdata-protection-methods.md#pram-post-randomisation-method) is the natural perturbative pathway to a $t$-closeness target, since it can be designed to push the in-group distribution toward the population marginal.

The metric is more expensive to satisfy than $k$-anonymity or $l$-diversity, requires a population reference distribution, and imposes larger information loss. Reserve it for releases where the analytical use case explicitly depends on protecting the sensitive distribution's relationship to the quasi-identifiers.

**References:** Li, Li and Venkatasubramanian (2007); Hundepool et al. (2014).

### Individual Risk (Benedetti-Franconi)

> Estimate, for each record, the probability that an adversary holding the quasi-identifier values of that record's individual would correctly re-identify it, using a negative-binomial posterior on the population frequency.

$$
r_i \approx \frac{f_k}{\hat{F}_k}
$$

$$
\xi = \frac{1}{n} \sum_{i=1}^{n} r_i
$$

where $f_k$ is the sample frequency of the quasi-identifier combination $k$ to which record $i$ belongs and $\hat{F}_k$ is the design-weighted estimate of the corresponding population frequency. The Benedetti-Franconi model derives this ratio as the posterior mean of a negative-binomial distribution placed on the population count given the sample count, accounting for the sampling design.

The estimator is the right risk score whenever the released file is a **sample** rather than a census: a record that is sample-unique ($f_k = 1$) is not necessarily population-unique, and the risk should reflect the chance that other matching individuals exist in the population. For a true population-unique record, $\hat{F}_k \approx 1$ and $r_i$ approaches 1; for a record in a large population stratum, $r_i$ is small even if the sample frequency is low.

Both $r_i$ and the file-level summary $\xi$ should be tracked across SDC iterations: $\xi$ is the headline scalar that the IHSN guide and the handbook recommend as the primary risk indicator, and $r_i$ is the per-record diagnostic for identifying which records demand further treatment. For grouped data, escalate $r_i$ through [Hierarchical (Household) Risk](#hierarchical-household-risk); for residual outliers, use [Benchmark Risk Outlier](#benchmark-risk-outlier).

**References:** Benedetti and Franconi (1998); Franconi and Polettini (2004); Hundepool et al. (2014); Templ, Meindl, Kowarik and Chen (2014).

### Global Risk Rate ξ

> The mean individual risk across the file — the single scalar that summarises the disclosure risk of the full release.

$$
\xi = \frac{1}{n} \sum_{i=1}^{n} r_i
$$

$\xi$ converts the per-record risk distribution into a single comparable number, making it the natural target for the SDC iteration loop: at each pass the practitioner observes whether $\xi$ has fallen below the pre-declared release threshold (typical operational targets in national statistics are $\xi < 0.01$ to $\xi < 0.05$, depending on release sensitivity). It is reported alongside the chosen utility headline ([λ Measure](#-measure)) so that one risk number and one utility number summarise the release.

The metric averages over a heavy-tailed distribution. A $\xi$ below threshold can coexist with a small number of records whose individual risk is unacceptably high — exactly the records an adversary will target. Pair $\xi$ with the per-record [Benchmark Risk Outlier](#benchmark-risk-outlier) measure to detect this situation.

$\xi$ is also insensitive to the quasi-identifier-to-sensitive linkage that defines attribute disclosure: a low-$\xi$ release can still permit attribute disclosure if the sensitive distribution within groups is concentrated. Read $\xi$ alongside [l-Diversity](#l-diversity) and [t-Closeness](#t-closeness) for sensitive variables.

**References:** Benedetti and Franconi (1998); Hundepool et al. (2014); Templ, Meindl, Kowarik and Chen (2014).

### Benchmark Risk Outlier

> Flag any record whose individual risk exceeds both an absolute threshold and a robust multiple of the median, identifying the tail of the risk distribution that the global mean cannot localise.

$$
r_i > \max\!\big(r^{*},\; \mathrm{median}(r) + c \cdot \mathrm{MAD}(r)\big)
$$

where $r^{*}$ is an absolute floor (operationally chosen — for example $0.1$, meaning a 10% adversary success probability) and $c$ is a robust outlier multiplier (typically $c = 3$, mirroring the standard MAD-based outlier rule). MAD is the median absolute deviation from the median, scaled to $\sigma$-equivalence by a factor of $1.4826$ when desired.

The benchmark measure complements [Global Risk Rate ξ](#global-risk-rate-): $\xi$ tells you whether the release is acceptable on average, the benchmark count tells you how many residual problem records need additional treatment. It is the IHSN guide's recommended diagnostic for closing out the SDC iteration loop — once the count of benchmark outliers reaches zero, the practitioner knows that no record is anomalously risky relative to the rest of the dataset.

The metric's robustness comes from the median + MAD construction: it identifies records that stand out from the bulk of the risk distribution even after the bulk has been pulled down by SDC, which a fixed threshold alone would miss. Records flagged by this measure are the natural candidates for [local suppression](11-sdc-microdata-protection-methods.md#local-suppression) or further [global recoding](11-sdc-microdata-protection-methods.md#global-recoding).

**References:** Templ, Meindl, Kowarik and Chen (2014); Hundepool et al. (2014).

### Hierarchical (Household) Risk

> When records share a group structure (household, family, firm, schoolclass), the re-identification risk of the group is the union of its members' individual risks: identifying any one member identifies all.

$$
r_{\mathrm{hh}} = 1 - \prod_{i \in H} (1 - r_i)
$$

For a household $H$ with members $i$ each carrying individual risk $r_i$, the household risk is the probability that *at least one* member is re-identified, computed under independence across members. The household risk is then propagated back to every member: each member of $H$ inherits $r_{\mathrm{hh}}$ as their effective risk, since identifying any household member exposes the household structure and therefore all other members.

The metric is mandatory for any household survey or grouped administrative dataset. A household with seven members each carrying $r_i = 0.1$ has $r_{\mathrm{hh}} \approx 0.52$ — more than half the time some member of that household is re-identifiable. Computing risk only at the individual level vastly understates the true exposure for grouped data.

The hierarchical-risk computation is straightforward as a `group_by(household_id).agg(1 - (1 - r_i).prod())` followed by a join back to the record level. The same construction generalises to any nested grouping (firm × employees, schoolclass × pupils, ward × admissions); the practitioner must declare the relevant grouping column in the pre-processing variable taxonomy.

**References:** Hundepool et al. (2014); Templ, Meindl, Kowarik and Chen (2014).

### SUDA Score (MSU)

> Identify records that are unique on some *minimal* subset of quasi-identifier attributes, then weight each minimal sample unique by the inverse of its size — small MSUs indicate a record that an adversary can re-identify with very few attributes.

$$
\mathrm{SUDA}_i = \sum_{S \in \mathrm{MSU}(i)} \mathrm{ATT}^{\,M - |S|}
$$

where $\mathrm{MSU}(i)$ is the set of minimal sample uniques of record $i$ — subsets $S$ of quasi-identifier attributes on which record $i$ is unique and for which no proper subset of $S$ is itself a sample unique — $\mathrm{ATT}$ is the total number of quasi-identifier attributes considered, and $M$ is the maximum MSU size searched (typically $3$ or $4$).

A record unique on a single attribute (an MSU of size $1$) contributes $\mathrm{ATT}^{M-1}$ to its score; a record unique only on a four-attribute combination contributes $\mathrm{ATT}^{M-4}$. The exponential weighting reflects the adversary's task: re-identifying a single-attribute MSU requires knowing only one fact, while re-identifying a four-attribute MSU requires knowing four facts simultaneously. SUDA is the right risk score when the adversary's information is fragmentary — for instance, when public records expose some quasi-identifiers but not all.

The score is the most computationally intensive of the file-level risk measures: it requires checking all subsets of size $1$ through $M$, giving $O\!\big(\binom{\mathrm{ATT}}{M}\big)$ group-by passes over the data. For typical microdata configurations ($\mathrm{ATT} \approx 5$–$10$, $M \approx 3$–$4$) this is tractable. SUDA is the natural diagnostic for the residual risk of [global recoding](11-sdc-microdata-protection-methods.md#global-recoding) and [MASSC](11-sdc-microdata-protection-methods.md#massc), which target overall $k$-anonymity but may leave records with small MSUs untouched.

**References:** Elliot, Manning and Ford (2002); Manning, Haglin and Keane (2008); Hundepool et al. (2014).

### Record Linkage Risk (Distance-Based)

> Treat the protected file as the adversary's input and the original file as the target; measure the share of protected records whose nearest original neighbour by standardised distance is the correct pre-perturbation record.

$$
\mathrm{RL}_{1} = \frac{1}{n} \sum_{i=1}^{n} \mathbf{1}\!\left[\arg\min_{j} \|z_i - x_j\|_{\sigma} = i\right]
$$

$$
\mathrm{RL}_{2} = \frac{1}{n} \sum_{i=1}^{n} \mathbf{1}\!\left[i \in \mathrm{top}\text{-}2_{j}\,\|z_i - x_j\|_{\sigma}\right]
$$

where $z_i$ is the masked record, $x_j$ is the $j$-th original record, and $\|\cdot\|_{\sigma}$ is the Euclidean distance after each column is standardised by its sample standard deviation. $\mathrm{RL}_{1}$ is the **lower bound** on linkage risk: the share of records the adversary recovers exactly. $\mathrm{RL}_{2}$ is the **upper bound**: the share whose original is in the adversary's two-best candidates. The interval $[\mathrm{RL}_{1}, \mathrm{RL}_{2}]$ brackets the true success rate of a nearest-neighbour adversary.

This is the native risk metric for continuous perturbative methods — [noise addition](11-sdc-microdata-protection-methods.md#noise-addition), [microaggregation](11-sdc-microdata-protection-methods.md#microaggregation), [rank swapping](11-sdc-microdata-protection-methods.md#rank-swapping), [resampling](11-sdc-microdata-protection-methods.md#resampling), [multiplicative noise](11-sdc-microdata-protection-methods.md#multiplicative-noise) — because $k$-anonymity does not naturally apply to continuous values. The IHSN guide reports for MDAV microaggregation with $k = 3$ a typical interval of approximately $[0\%, 61\%]$: zero records are recovered exactly (the centroid replaces every record's exact value) but the centroid lies among the two nearest originals for the majority.

The metric is sensitive to the standardisation choice: standardising by population $\sigma$ versus sample $\sigma$ versus interquartile range gives meaningfully different rates. Publish the standardisation rule alongside the metric. The complementary measure for adversaries with approximate prior knowledge (rather than exact original values) is [Interval Disclosure Risk](#interval-disclosure-risk).

**References:** Domingo-Ferrer and Mateo-Sanz (2002); Pagliuca and Seri (1999); Hundepool et al. (2014).

### Interval Disclosure Risk

> The fraction of records whose original value lies within a published $\pm \varepsilon$ tolerance of the masked value, modelling an adversary who treats the masked value as an approximate pointer to the original.

$$
\mathrm{IDR}(\varepsilon) = \frac{1}{n} \sum_{i=1}^{n} \mathbf{1}\!\left[\,|x_i - z_i| \leq \varepsilon \cdot s\,\right]
$$

where $s$ is a per-column scale (typically the standard deviation, the IQR, or a fixed business-meaningful unit such as 100,000 won), and $\varepsilon$ is a tolerance fraction declared by the data custodian. The metric reports the share of records the adversary correctly localises within a $\pm \varepsilon$ band around the released value.

Interval disclosure is the right metric whenever the released value is itself an obvious pointer to the original — most clearly for [rounding](11-sdc-microdata-protection-methods.md#rounding), where every original lies in the analytically known interval $[ub, (u+1)b]$ and the metric is exact, but also for [noise addition](11-sdc-microdata-protection-methods.md#noise-addition) under the assumption that the adversary knows the noise distribution and can compute the expected band. It is also informative for an adversary with approximate prior knowledge: even if record linkage gives them only a long candidate list, the published value tells them which band to focus on.

The choice of $\varepsilon$ is a release decision and should be reported. Typical values range from $\varepsilon = 0.01$ (very tight band — measures near-exact recovery) to $\varepsilon = 0.10$ (loose band — measures recovery to within one decile). Publishing IDR at multiple $\varepsilon$ levels gives a more informative profile than a single point estimate.

**References:** Domingo-Ferrer and Torra (2001); Mateo-Sanz, Sebé and Domingo-Ferrer (2004); Hundepool et al. (2014).

### RMDID1 — Robust Mahalanobis Disclosure Risk

> Use Robust Mahalanobis Distance to scale a per-record disclosure interval, so that outliers (which are intrinsically harder to mask) get larger tolerances; report the share of records where the masked value falls inside the scaled interval.

$$
\mathrm{RMD}(x_i) = \sqrt{(x_i - \hat{\mu})^{\top}\,\hat{\Sigma}_{\mathrm{MCD}}^{-1}\,(x_i - \hat{\mu})}
$$

$$
\mathrm{RMDID1} = \frac{1}{n} \sum_{i=1}^{n} \mathbf{1}\!\left[\,|x_i - z_i| \leq \varepsilon \cdot \mathrm{RMD}(x_i) \cdot s\,\right]
$$

where $\hat{\mu}$ and $\hat{\Sigma}_{\mathrm{MCD}}$ are the location and scatter estimates from Minimum Covariance Determinant (Rousseeuw 1985), giving outlier-robust estimates of the data centre and shape; $s$ is a column-level scale; and $\varepsilon$ is the same tolerance fraction used in [Interval Disclosure Risk](#interval-disclosure-risk). The MCD-scaled interval expands proportionally to how far each record sits from the robust centre.

RMDID1 is the right risk metric for outlier-driven continuous data — claim severity, turnover, top wages, large transfers — where a fixed-band interval-disclosure measure understates the real risk for the very records that matter most. An extreme value is identifiable to an adversary even after substantial perturbation, and its disclosure interval should reflect that. Templ and Meindl (2008) demonstrated on European Structural Business Survey data that RMDID1 reveals residual risk that simple linkage misses.

The metric requires a robust covariance estimator (MCD is the standard choice, but other M-estimators are valid) and is therefore more involved than the simple distance-based linkage measures. It is the natural risk pairing for [top and bottom coding](11-sdc-microdata-protection-methods.md#top-and-bottom-coding), [multiplicative noise](11-sdc-microdata-protection-methods.md#multiplicative-noise), and [resampling](11-sdc-microdata-protection-methods.md#resampling) — methods whose effectiveness on heavy-tailed data is precisely what RMDID1 measures.

**References:** Templ and Meindl (2008); Rousseeuw (1985); Templ, Meindl, Kowarik and Chen (2014).

## Information Loss Metrics

### λ Measure

> Per-column mean absolute deviation between original and protected values, normalised by the original's range, then averaged across columns — the handbook's primary scalar utility summary.

$$
\lambda_k = \frac{1}{n}\sum_{i=1}^{n} \frac{|x_{ik} - z_{ik}|}{\max(x_{\cdot k}) - \min(x_{\cdot k})}, \qquad \lambda = \frac{1}{p}\sum_{k=1}^{p} \lambda_k
$$

where $x_{ik}$ is the original value of record $i$ in column $k$, $z_{ik}$ is the protected value, and the denominator is the range of the original column. Per-column $\lambda_k$ values are bounded in $[0, 1]$ (a constant column reports zero loss; a column where every protected value is at the opposite end of the range reports $\lambda_k = 1$); the file-level $\lambda$ averages across columns.

$\lambda$ is the primary utility scalar reported alongside $\xi$ in the handbook's Table 3.12 example: a realistic mix of PRAM, noise and suppression yielded approximately $\lambda \approx 0.14$ on a survey dataset. Pre-declared utility thresholds are typically of the form $\lambda < 0.10$ for moderately sensitive releases, $\lambda < 0.05$ for analytical priority releases. The metric's range-normalisation makes columns with different units directly comparable — important for any heterogeneous dataset combining counts, currencies and ages.

The metric is a mean and therefore insensitive to the tail of the per-record distortion distribution. A small number of records perturbed strongly can coexist with low $\lambda$. For tail-aware utility measurement, pair $\lambda$ with [IL1s](#il1s--yancey-winkler-scaled-distance), which normalises by standard deviation rather than range and penalises tail distortion more heavily. For categorical variables, use [KL Divergence](#kl-divergence-categorical) instead.

**References:** Mateo-Sanz, Domingo-Ferrer and Sebé (2005); Hundepool et al. (2014); Templ, Meindl, Kowarik and Chen (2014).

### IL1s — Yancey-Winkler Scaled Distance

> Sum of absolute deviations between original and protected values, scaled per column by the standard deviation — more sensitive than λ to large per-record deviations.

$$
\mathrm{IL1s} = \frac{1}{n \cdot p} \sum_{i=1}^{n} \sum_{k=1}^{p} \frac{|x_{ik} - z_{ik}|}{s_k}
$$

where $s_k$ is the sample standard deviation of column $k$. The σ-normalisation differs sharply from $\lambda$'s range-normalisation: in a heavy-tailed distribution the range can be many multiples of the standard deviation, so a deviation that contributes modestly to $\lambda$ contributes proportionally more to IL1s. This makes IL1s the right utility measure when the analytical use case is sensitive to the tails — for example, when a regression model relies on the top decile of incomes for identification.

IL1s and $\lambda$ are typically reported together: $\lambda$ for the global summary, IL1s for sensitivity to tail distortion. Where the two diverge — IL1s much higher than $\lambda$ — the data custodian should examine which column is driving the difference and whether the SDC method choice is appropriate for that column's distribution.

The metric is undefined for constant columns ($s_k = 0$), which should be excluded before computation. It is also affected more strongly than $\lambda$ by [top and bottom coding](11-sdc-microdata-protection-methods.md#top-and-bottom-coding), since the censored extremes contribute disproportionately to both numerator and the original $s_k$.

**References:** Yancey, Winkler and Creecy (2002); Mateo-Sanz, Domingo-Ferrer and Sebé (2005); Hundepool et al. (2014).

### SSE / SST Ratio

> Within-group sum of squares divided by total sum of squares, after partitioning records into the groups produced by [microaggregation](11-sdc-microdata-protection-methods.md#microaggregation) — directly the quantity MDAV minimises.

$$
\frac{\mathrm{SSE}}{\mathrm{SST}} = \frac{\sum_{g=1}^{G}\sum_{i \in G_g} \|x_i - \bar{x}_g\|^2}{\sum_{i=1}^{n} \|x_i - \bar{x}\|^2}
$$

where $G_1, \ldots, G_G$ is the partition of records into groups of size $\geq k$, $\bar{x}_g$ is the centroid of group $g$, and $\bar{x}$ is the overall centroid. The numerator measures the variance lost by collapsing each group to its centroid; the denominator measures the original total variance. The ratio is bounded in $[0, 1]$: zero corresponds to all groups being singletons (no information loss, but no protection), one corresponds to all records in a single group (maximum protection, maximum loss).

SSE/SST is the native loss measure for any group-replacement method — [microaggregation](11-sdc-microdata-protection-methods.md#microaggregation) with MDAV minimises SSE directly, [MASSC](11-sdc-microdata-protection-methods.md#massc) Step 1 produces groups for which SSE/SST is the natural utility report, and any clustering-based pseudonymisation strategy can be evaluated against it. Per-column SSE/SST values reveal which columns have suffered the most aggregation.

The metric does not measure record-level distortion the way $\lambda$ does; it measures variance preservation. Two methods can have similar $\lambda$ but very different SSE/SST if one preserves group structure and the other breaks it. Reporting SSE/SST alongside $\lambda$ and $\xi$ gives the most complete utility-risk profile for microaggregation-based releases.

**References:** Domingo-Ferrer and Mateo-Sanz (2002); Mateo-Sanz, Domingo-Ferrer and Sebé (2005); Hundepool et al. (2014).

### Eigenvalue Spectrum Comparison

> Compare the sorted eigenvalues of the original and protected covariance matrices; large deviations indicate that perturbation has distorted the principal-component structure.

$$
\Delta_{\lambda} = \max_{j} \big|\lambda_j(\mathrm{Cov}(X)) - \lambda_j(\mathrm{Cov}(Z))\big|
$$

where the eigenvalues $\lambda_j$ are sorted in descending order. The maximum absolute deviation gives a worst-case summary; the sum $\sum_j |\lambda_j(\mathrm{Cov}(X)) - \lambda_j(\mathrm{Cov}(Z))|$ gives an aggregate measure. Both should be reported in standardised form — divide by the largest original eigenvalue $\lambda_1(\mathrm{Cov}(X))$ — to make values comparable across datasets.

This metric is the right utility check for any analysis that depends on the variance or correlation structure: principal-component analysis, factor analysis, multivariate regression, dimensionality reduction. [Uncorrelated noise addition](11-sdc-microdata-protection-methods.md#noise-addition) inflates all eigenvalues uniformly by the noise variance; [correlated noise addition](11-sdc-microdata-protection-methods.md#noise-addition) preserves the eigenvalue ratios (Kim 1986); [resampling](11-sdc-microdata-protection-methods.md#resampling) tends to flatten the spectrum because variable-by-variable rank-averaging weakens cross-column structure.

The eigenvalue test does not capture asymmetric distortions of the principal axes (rotations of the eigenvectors). For a release whose downstream use is PCA-based, supplement with the **canonical correlation** between the principal-component scores of $X$ and $Z$, or with explicit comparison of the top-$k$ eigenvectors via subspace angles.

**References:** Domingo-Ferrer, Mateo-Sanz and Torra (2001); Hundepool et al. (2014); Mateo-Sanz, Domingo-Ferrer and Sebé (2005).

### Regression Coefficient Comparison (lm)

> Fit the same linear model on the original and protected data; compare coefficients in magnitude, sign, and significance — the most interpretation-grounded utility measure.

$$
\Delta_{\beta} = \big\|\hat{\beta}_{\mathrm{orig}} - \hat{\beta}_{\mathrm{prot}}\big\|_2, \qquad \Delta_{\beta,k} = \frac{|\hat{\beta}_{\mathrm{orig},k} - \hat{\beta}_{\mathrm{prot},k}|}{\mathrm{SE}(\hat{\beta}_{\mathrm{orig},k})}
$$

The first scalar is the Euclidean distance between coefficient vectors; the second is the per-coefficient deviation in standard-error units, which gives a meaningful comparison to inferential precision (a deviation $\Delta_{\beta,k} < 1$ is within a single original standard error). A complete report includes the per-coefficient deviations, sign-flip count, and any change in statistical significance at the conventional $\alpha = 0.05$ threshold.

This metric directly tests whether the analytical relationships the dataset was designed to support survive the SDC process. For an insurance release whose primary use is modelling claim frequency from policyholder characteristics, the lm comparison on that exact model is the only utility measure that directly answers the question that matters. Any method that breaks individual-level associations between sensitive variables and predictors — notably [shuffling](11-sdc-microdata-protection-methods.md#rank-swapping) and aggressive [data swapping](11-sdc-microdata-protection-methods.md#data-swapping) — will register here even if $\lambda$ and SSE/SST look acceptable.

The metric is conditional on the model specification: a release that preserves coefficients for a linear model can fail catastrophically for a model with interactions or non-linear terms. Specify and publish the model(s) used for the lm utility check as part of the release metadata. For a release intended to support a portfolio of analytical use cases, run the comparison on each.

**References:** Mateo-Sanz, Domingo-Ferrer and Sebé (2005); Hundepool et al. (2014).

### KL Divergence (Categorical)

> Per-column Kullback-Leibler divergence between the original and protected category-frequency distributions — the natural loss measure for categorical perturbation.

$$
D_{\mathrm{KL}}(P \,\|\, Q) = \sum_{x \in \mathcal{X}} P(x) \log \frac{P(x)}{Q(x)}
$$

where $P$ is the original category-frequency distribution of the column and $Q$ is the protected distribution. The divergence is non-negative, zero exactly when the two distributions coincide, and unbounded above (when $Q(x) = 0$ but $P(x) > 0$, the divergence diverges — apply Laplace smoothing to $Q$ in this case).

KL divergence is the categorical counterpart of $\lambda$: it summarises how much the column's marginal has been distorted by [PRAM](11-sdc-microdata-protection-methods.md#pram-post-randomisation-method), [global recoding](11-sdc-microdata-protection-methods.md#global-recoding), [data swapping](11-sdc-microdata-protection-methods.md#data-swapping), [MASSC](11-sdc-microdata-protection-methods.md#massc), or [local suppression](11-sdc-microdata-protection-methods.md#local-suppression). For invariant PRAM the divergence approaches zero by construction (the design constraint $\pi^{\top} P = \pi^{\top}$ forces marginal preservation in expectation), and any non-trivial deviation reflects sample-level variance around an unbiased target. For non-invariant PRAM and for global recoding, the divergence directly measures the distributional cost of the protection.

KL divergence is asymmetric: $D_{\mathrm{KL}}(P \,\|\, Q) \neq D_{\mathrm{KL}}(Q \,\|\, P)$ in general. By convention the original $P$ is the first argument, so the divergence measures information lost when treating $Q$ as an approximation to $P$. The metric also captures only the marginal distribution; it does not detect changes in the joint distribution of two categorical variables. For joint-distribution utility, supplement with KL divergence on cross-tabulations or with the chi-squared statistic on the joint table.

**References:** Kullback and Leibler (1951); Domingo-Ferrer, Mateo-Sanz and Torra (2001); Hundepool et al. (2014).

## References

Benedetti, R., Franconi, L. (1998). *Statistical and technological solutions for controlled data dissemination*. Pre-proceedings of New Techniques and Technologies for Statistics, Vol. 1, 225–232.

Domingo-Ferrer, J., Mateo-Sanz, J. M., Torra, V. (2001). Comparing SDC methods for microdata on the basis of information loss and disclosure risk. *Pre-proceedings of ETK-NTTS*, 807–826.

Domingo-Ferrer, J., Mateo-Sanz, J. M. (2002). Practical data-oriented microaggregation for statistical disclosure control. *IEEE Transactions on Knowledge and Data Engineering*, 14(1), 189–201.

Domingo-Ferrer, J., Torra, V. (2001). A quantitative comparison of disclosure control methods for microdata. In *Confidentiality, Disclosure and Data Access* (pp. 111–134). North-Holland.

Elliot, M. J., Manning, A. M., Ford, R. W. (2002). A computational algorithm for handling the special uniques problem. *International Journal of Uncertainty, Fuzziness and Knowledge-Based Systems*, 10(5), 493–509.

Franconi, L., Polettini, S. (2004). Individual risk estimation in μ-Argus: A review. *Privacy in Statistical Databases*, LNCS 3050, 262–272.

Hundepool, A., Domingo-Ferrer, J., Franconi, L., Giessing, S., Schulte Nordholt, E., Spicer, K., De Wolf, P-P. (2014). *Statistical Disclosure Control*. Wiley Series in Survey Methodology.

Hundepool, A., Domingo-Ferrer, J., Franconi, L., Giessing, S., Lenz, R., Naylor, J., Schulte Nordholt, E., Seri, G., De Wolf, P-P., Tent, R., Młodak, A., Gussenbauer, J., Wilak, K. (2026). *Handbook on Statistical Disclosure Control*, 2nd Edition. Centre of Excellence SDC / STACE project.

Kullback, S., Leibler, R. A. (1951). On information and sufficiency. *Annals of Mathematical Statistics*, 22(1), 79–86.

Li, N., Li, T., Venkatasubramanian, S. (2007). t-Closeness: Privacy beyond k-anonymity and l-diversity. *Proceedings of ICDE 2007*, 106–115.

Machanavajjhala, A., Kifer, D., Gehrke, J., Venkitasubramaniam, M. (2007). l-Diversity: Privacy beyond k-anonymity. *ACM Transactions on Knowledge Discovery from Data*, 1(1), Article 3.

Manning, A. M., Haglin, D. J., Keane, J. A. (2008). A recursive search algorithm for statistical disclosure assessment. *Data Mining and Knowledge Discovery*, 16(2), 165–196.

Mateo-Sanz, J. M., Domingo-Ferrer, J., Sebé, F. (2005). Probabilistic information loss measures in confidentiality protection of continuous microdata. *Data Mining and Knowledge Discovery*, 11(2), 181–193.

Mateo-Sanz, J. M., Sebé, F., Domingo-Ferrer, J. (2004). Outlier protection in continuous microdata masking. *Privacy in Statistical Databases*, LNCS 3050, 201–215.

Pagliuca, D., Seri, G. (1999). *Some results of individual ranking method on the system of enterprise accounts annual survey*. ESPRIT SDC Project Deliverable MI-3/D2.

Rousseeuw, P. J. (1985). Multivariate estimation with high breakdown point. *Mathematical Statistics and Applications*, B, 283–297.

Samarati, P., Sweeney, L. (1998). *Protecting privacy when disclosing information: k-anonymity and its enforcement through generalization and suppression*. SRI International Technical Report.

Sweeney, L. (2002). k-Anonymity: A model for protecting privacy. *International Journal of Uncertainty, Fuzziness and Knowledge-Based Systems*, 10(5), 557–570.

Templ, M., Meindl, B. (2008). Robust statistics meets SDC: New disclosure risk measures for continuous microdata masking. *Privacy in Statistical Databases*, LNCS 5262, 113–126.

Templ, M., Meindl, B., Kowarik, A., Chen, S. (2014). *Introduction to Statistical Disclosure Control (SDC)*. IHSN Working Paper No. 007. International Household Survey Network.

Yancey, W. E., Winkler, W. E., Creecy, R. H. (2002). Disclosure risk assessment in perturbative microdata protection. *Inference Control in Statistical Databases*, LNCS 2316, 135–152.
