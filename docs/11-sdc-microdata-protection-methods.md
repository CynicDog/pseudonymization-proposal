# SDC Microdata Protection Methods

Microdata masking methods split into two families. **Perturbative** methods alter the values in the released dataset while keeping all records — an intruder sees every record but cannot fully trust the values. **Non-perturbative** methods reduce the information content by suppressing records, collapsing categories, or releasing only a sample — values that do appear are authentic, but the dataset is coarser or incomplete. Both families are grounded in a trade-off between statistical utility and disclosure risk, and the choice between them depends on the variable types and the analytical goals of the data users.

## Perturbative Methods vs. Data Types

*Table 3.2 from the Handbook on Statistical Disclosure Control. "X" = applicable; "(X)" = applicable with adaptation.*

| Method | Continuous | Categorical |
|---|---|---|
| Noise addition | X | |
| Microaggregation | X | (X) |
| Data swapping | (X) | X |
| Rank swapping | X | X |
| Rounding | X | |
| Resampling | X | |
| PRAM | | X |
| MASSC | | X |

## Non-perturbative Methods vs. Data Types

*Table 3.6 from the Handbook on Statistical Disclosure Control.*

| Method | Continuous | Categorical |
|---|---|---|
| Sampling | | X |
| Global recoding | X | X |
| Top and bottom coding | X | X |
| Local suppression | | X |

## Perturbative Methods

### Noise Addition

> Replace each continuous value with the original value plus a random draw from a noise distribution, preserving distributional shape while obscuring exact values.

$$
Z = X + \varepsilon, \quad \varepsilon \sim \mathcal{N}(0,\, \Sigma_\varepsilon), \quad \Sigma_\varepsilon = \alpha \cdot \mathrm{diag}(\sigma_1^2, \ldots, \sigma_p^2)
$$

The original data matrix $X$ ($n \times p$) is transformed by adding a noise matrix $\varepsilon$ whose rows are drawn independently from a zero-mean multivariate normal distribution. The scalar $\alpha$ controls noise magnitude relative to each variable's marginal variance $\sigma_j^2$, so protection is proportional across variables of different scales.

**Uncorrelated noise** (diagonal $\Sigma_\varepsilon$) preserves column means exactly, since $\mathbb{E}[Z] = \mathbb{E}[X]$, but inflates the covariance matrix: $\mathrm{Cov}(Z) = \mathrm{Cov}(X) + \Sigma_\varepsilon$. Analysts who are unaware of the noise will overestimate variances and attenuate correlations.

**Correlated noise** sets $\Sigma_\varepsilon = \alpha \cdot \Sigma_X$, drawing noise that matches the correlation structure of $X$. Kim (1986) showed that under this design the correlation matrix is exactly preserved in expectation, making the masked data more analytically faithful for multivariate tasks.

**Linear-transformation noise** (Brand 2002) applies a random orthogonal or near-orthogonal linear map to $X$ before releasing, achieving stronger theoretical guarantees on both utility and protection by confounding the rotation as part of the noise.

Noise addition is the simplest perturbative method to implement and is theoretically well understood. Its main limitation is that it does not provide formal disclosure bounds: a motivated intruder who knows the noise parameters can partially undo the perturbation, and records with very unusual values may still be identifiable despite the added noise.

**Measurement:** Native risk metric is [record linkage (top-1 lower / top-2 upper)](12-sdc-risk-and-utility-metrics.md#record-linkage-risk-distance-based) supplemented by [interval disclosure risk](12-sdc-risk-and-utility-metrics.md#interval-disclosure-risk); native utility metric is the [λ measure](12-sdc-risk-and-utility-metrics.md#-measure), with [eigenvalue spectrum comparison](12-sdc-risk-and-utility-metrics.md#eigenvalue-spectrum-comparison) as a secondary check on covariance distortion. Noise addition alters values continuously, so disclosure is best assessed through nearest-neighbour recovery and utility through range-normalised mean absolute deviation.

**References:** Kim (1986); Kim (1990); Sullivan (1989); Tendick (1991); Brand (2002).

### Multiplicative Noise

> Multiply each original value by an independent positive random factor, then rescale to restore the first two moments.

$$
X^a_{ij} = Z_{ij} \cdot X_{ij}
$$

$$
X^{aR}_i = \frac{\sigma_X}{\sigma_{X^a}}\!\left(X^a_i - \mu_{X^a}\right) + \mu_X
$$

Each element of the data matrix is perturbed by multiplying with a random variate $Z_{ij}$, typically drawn from a log-normal distribution centred at 1, so the noise is proportional to the magnitude of the original value. Large values receive proportionally larger perturbations than small values, which is often more realistic for income or expenditure data than additive noise with a fixed variance.

The raw multiplicative perturbation $X^a$ distorts the sample mean and variance. The moment-preserving rescaling (Höhne 2004) corrects this: the masked column is linearly mapped so that its sample mean equals $\mu_X$ and its sample standard deviation equals $\sigma_X$. After rescaling the first two moments are exactly restored; higher-order moments and correlations are only approximately preserved, and the degree of distortion depends on the variance of the noise distribution.

Multiplicative noise is especially appropriate when variables are strictly positive (wages, assets, turnover) and when analysts rely on means and variances but not higher moments. Its implementation is more demanding than additive noise because the rescaling must be performed correctly to avoid introducing bias, and because the distribution of $Z_{ij}$ must be chosen so that $Z_{ij} \cdot X_{ij}$ remains within a plausible range.

**Measurement:** Native risk metric is [record linkage (top-1 / top-2)](12-sdc-risk-and-utility-metrics.md#record-linkage-risk-distance-based) supplemented by [RMDID1](12-sdc-risk-and-utility-metrics.md#rmdid1--robust-mahalanobis-disclosure-risk); native utility metric is the [λ measure](12-sdc-risk-and-utility-metrics.md#-measure), with [regression coefficient comparison (lm)](12-sdc-risk-and-utility-metrics.md#regression-coefficient-comparison-lm) as a check that the moment-preserving rescaling has not introduced bias in conditional means. Because perturbation scales with magnitude, outlier-aware risk (RMDID1) matters more than for additive noise.

**References:** Höhne (2004).

### Microaggregation

> Partition records into groups of at least $k$, then replace each value within a group with the group mean, so no record can be distinguished from the others in its group.

$$
\text{Minimise} \quad \mathrm{SSE} = \sum_{i=1}^{g} \sum_{j \in G_i} (\mathbf{x}_j - \bar{\mathbf{x}}_i)^\top (\mathbf{x}_j - \bar{\mathbf{x}}_i)
$$

where $G_1, \ldots, G_g$ is a partition of $n$ records into groups each of size $\geq k$, $\bar{\mathbf{x}}_i$ is the centroid of group $G_i$, and $\mathrm{SSE}$ is the total within-group sum of squares.

The objective is to find the partition that minimises information loss (measured by SSE) subject to every group having at least $k$ members. This guarantees $k$-anonymity on the microaggregated variables: every released record is identical on all masked attributes to at least $k - 1$ other records in the dataset.

For the univariate case Hansen and Mukherjee (2003) proved that the optimal partition places equal-sized groups of exactly $k$ records on the sorted variable and derived a dynamic-programming algorithm. For the multivariate case the problem is NP-hard, so heuristics are used in practice. The **MDAV** (Maximum Distance to Average Vector) algorithm (Mateo-Sanz and Domingo-Ferrer 1999) works as follows: find the record farthest from the dataset centroid, form a group of $k$ nearest neighbours around it, remove the group, and repeat until all records are assigned. MDAV consistently achieves near-optimal SSE and runs in $O(n^2)$ per variable.

Information loss grows with $k$ and with the heterogeneity within groups. For categorical variables microaggregation can be applied with adaptation by using the mode instead of the mean and a suitable distance measure over the categorical domain (Torra 2004).

**Measurement:** Native utility measure is [SSE / SST](12-sdc-risk-and-utility-metrics.md#sse--sst-ratio) — the MDAV objective directly minimises within-group SSE — and the structural risk guarantee is [k-anonymity](12-sdc-risk-and-utility-metrics.md#k-anonymity) by construction. Residual disclosure should be checked via [top-1 record linkage](12-sdc-risk-and-utility-metrics.md#record-linkage-risk-distance-based), since k-anonymity does not bound similarity-based recovery from the centroids.

**References:** Defays and Nanopoulos (1993); Mateo-Sanz and Domingo-Ferrer (1999); Domingo-Ferrer and Mateo-Sanz (2002); Hansen and Mukherjee (2003); Oganian and Domingo-Ferrer (2001); Torra (2004).

### Data Swapping

> Exchange the values of sensitive variables between pairs (or groups) of records that are similar on non-sensitive background variables, so marginal distributions are preserved while the linkage to identifying attributes is broken.

$$
\text{For records } (r_a, r_b) \text{ with } \mathrm{dist}(r_a, r_b) < \delta: \quad s(r_a) \leftrightarrow s(r_b)
$$

where $s(\cdot)$ denotes the sensitive attribute values and $\delta$ is a proximity threshold on non-sensitive variables.

In data swapping (Dalenius and Reiss 1978), the dataset is partitioned into matched pairs or small groups based on background characteristics (e.g., age, region, industry) that are considered non-disclosive. Within each matched group the values of one or more sensitive variables — typically income, exact address, or health outcomes — are swapped between members. Because the swapped records are similar on background variables, the swap is hard to detect and marginal frequency distributions are exactly preserved.

**Partial swapping** exchanges only a fraction of records (say 10–20%), leaving most records intact. **Full swapping** swaps all records. The protection level depends critically on the swap fraction and on how tightly records are matched: loose matching (large $\delta$) makes swaps harder to detect but also reduces utility.

Data swapping is most natural for categorical variables because matching is easier to define, but it can be adapted to continuous variables by treating matched groups as quantile bands. Unlike noise addition, it produces no analytical bias on marginals, making it suitable for frequency-table production systems. Its main weakness is that the protection for any individual record is probabilistic — an intruder who learns that a record was not swapped can still identify the individual.

**Measurement:** Native risk metric is [record linkage (top-1)](12-sdc-risk-and-utility-metrics.md#record-linkage-risk-distance-based) on the swapped sensitive attribute, complemented by [individual risk](12-sdc-risk-and-utility-metrics.md#individual-risk-benedetti-franconi) when the swap fraction is below 100%; native utility metrics are [KL divergence](12-sdc-risk-and-utility-metrics.md#kl-divergence-categorical) on the marginal of the swapped variable (which by construction should be near zero) and [λ](12-sdc-risk-and-utility-metrics.md#-measure) on the joint distribution. Because swaps preserve marginals exactly, joint-distribution distortion is the loss to track.

**References:** Dalenius and Reiss (1978); Reiss, Post and Dalenius (1982); Reiss (1984).

### Rank Swapping

> Sort values of each continuous or ordinal variable, then randomly exchange values between record pairs whose ranks differ by no more than $p\%$ of the total, preserving the overall rank distribution.

$$
\text{Swap } x_{(i)} \leftrightarrow x_{(j)} \text{ if } |i - j| \leq \lfloor p \cdot n / 100 \rfloor
$$

where $x_{(i)}$ denotes the $i$-th order statistic and $p$ is the neighbourhood size parameter (typically 10–30).

Rank swapping (Greenberg 1987) treats each variable independently. After sorting, a random permutation of swaps is applied within a sliding window of width $p\%$. Because the window restricts how far a value can travel in the rank order, the distribution shape is broadly preserved and extreme outliers (which are most re-identifiable) are shifted only slightly. This makes rank swapping a good fit for skewed distributions such as income or wealth.

The parameter $p$ governs the protection-utility trade-off: larger $p$ allows values to travel farther, increasing protection but distorting distributional statistics more strongly. Moore (1996) showed that rank swapping outperforms simple random rounding on several utility measures while providing comparable protection.

Applied to categorical ordinal variables, rank swapping is equivalent to permuting codes within an ordered neighbourhood. Domingo-Ferrer and Torra (2001) provided a comparative evaluation against other perturbative methods on real microdata sets, finding rank swapping among the most utility-preserving options for continuous key variables.

**Measurement:** Native risk metric is [record linkage (top-1 / top-2)](12-sdc-risk-and-utility-metrics.md#record-linkage-risk-distance-based); native utility metric is the [λ measure](12-sdc-risk-and-utility-metrics.md#-measure), with [regression coefficient comparison (lm)](12-sdc-risk-and-utility-metrics.md#regression-coefficient-comparison-lm) used to detect the cross-column correlation attenuation that rank swapping causes when applied independently per variable.

**References:** Greenberg (1987); Moore (1996); Domingo-Ferrer and Torra (2001).

### Rounding

> Replace each continuous value with a nearby multiple of a chosen base, reducing precision uniformly across the variable.

$$
x' \in \{ub,\; (u+1)b\}, \quad u = \left\lfloor \frac{x}{b} \right\rfloor
$$

where $b > 0$ is the rounding base and $u b$ is the largest multiple of $b$ that does not exceed $x$.

In **zero-restricted rounding**, the masked value $x'$ is assigned to the lower adjacent multiple $ub$ or the upper adjacent multiple $(u+1)b$. To make frequency estimators unbiased, the assignment can be randomised with probability $1 - (x - ub)/b$ for the lower and $(x - ub)/b$ for the upper multiple; this is called **random rounding** and is widely used in official statistics for cell counts.

Rounding reduces the precision of continuous variables to a granularity that matches the base $b$. For example, rounding exact ages to five-year bands (base 5) prevents age-exact linkage to external databases. The protection level depends on $b$ relative to the dispersion of $x$: a base that spans several standard deviations provides strong protection but severe utility loss.

Rounding is strictly applicable to continuous (or fine-grained integer) variables; for categorical variables the analogous operation is global recoding. It is often combined with top and bottom coding to further protect outliers that fall far from any rounded multiple.

**Measurement:** Native risk metric is [interval disclosure risk](12-sdc-risk-and-utility-metrics.md#interval-disclosure-risk) — every masked value carries an exact deterministic interval $[ub, (u+1)b]$, which makes the disclosure interval analytically known; native utility metric is the [λ measure](12-sdc-risk-and-utility-metrics.md#-measure), since rounding's loss is bounded by $b/2$ per record and λ averages this directly.

**References:** Willenborg and De Waal (2001).

### Resampling

> Draw $t$ independent samples from the original data, sort each by the variable of interest, then replace each record's value with the average of the corresponding rank values across samples.

$$
Z^{(j)} = \frac{1}{t} \sum_{s=1}^{t} x^{(s)}_{(j)}, \quad j = 1, \ldots, n
$$

where $x^{(s)}_{(j)}$ is the $j$-th order statistic of the $s$-th bootstrap sample of size $n$ drawn from the original data.

Heer (1993) proposed resampling as a masking procedure that smooths the empirical distribution without distorting the global rank order. The procedure works variable by variable: draw $t$ simple random samples with replacement (bootstrap samples) of the same size $n$ as the original data, sort each sample, and average the $j$-th ranked values across all $t$ samples. The output $Z$ preserves the rank ordering of records on each variable and exactly matches the original sample mean (because averaging preserves means), but blurs exact values by pulling them toward their expected order statistics.

The key insight is that for large $t$ the averaged order statistics converge to the expected order statistics of the underlying distribution, which are smooth and continuous even if the original data has ties or spikes. This makes resampling especially effective for income and expenditure variables with heavy right tails, where a few exact large values dominate disclosure risk.

Correlation structure between variables is approximately preserved because each variable is sorted independently before averaging, maintaining the relative rank correspondence across variables. Domingo-Ferrer and Mateo-Sanz (1999) evaluated resampling against other perturbative methods, finding it provides a favourable utility-protection balance for continuous survey variables.

**Measurement:** Native risk metric is [record linkage (top-1 / top-2)](12-sdc-risk-and-utility-metrics.md#record-linkage-risk-distance-based) plus [RMDID1](12-sdc-risk-and-utility-metrics.md#rmdid1--robust-mahalanobis-disclosure-risk) for tail records; native utility metric is the [λ measure](12-sdc-risk-and-utility-metrics.md#-measure) with [eigenvalue spectrum comparison](12-sdc-risk-and-utility-metrics.md#eigenvalue-spectrum-comparison) as a secondary check, since resampling smooths the distribution and may flatten covariance eigenvalues.

**References:** Heer (1993); Domingo-Ferrer and Mateo-Sanz (1999).

### PRAM (Post-Randomisation Method)

> Apply a Markov transition matrix to each categorical variable, so that the released category for any record is drawn probabilistically from a specified distribution over all categories, enabling unbiased estimation after correction.

$$
p_{kl} = P\!\left(X' = c_l \mid X = c_k\right), \quad k, l \in \{1, \ldots, K\}
$$

$$
\hat{T}_X = \left(P^{-1}\right)^\top T_\xi
$$

where $\mathbf{P}$ is the $K \times K$ Markov matrix with entries $p_{kl}$, $T_\xi$ is the vector of observed (perturbed) category frequencies, and $\hat{T}_X$ recovers the true frequencies.

In PRAM (Gouweleeuw et al. 1997), each record's categorical value $X = c_k$ is independently replaced by a new value $X' = c_l$ with probability $p_{kl}$, where $\mathbf{P}$ is designed by the data custodian before release. The matrix $\mathbf{P}$ need not be disclosed publicly: releasing only the corrected marginal totals $\hat{T}_X$ allows analysts to reconstruct population-level frequency distributions while remaining unable to reverse-engineer the perturbation for any individual record.

**Invariant PRAM** imposes the constraint $\pi^\top \mathbf{P} = \pi^\top$ where $\pi$ is the marginal distribution of $X$, so that the release marginals are unchanged in expectation. This makes PRAM transparent for distributional analyses while still providing individual-level protection.

**Diagonal-dominant PRAM** sets $p_{kk}$ close to 1 so most records retain their original category; only a small fraction are changed. The protection against exact-match linkage stems from the uncertainty about which records were changed and which were not. Kooiman et al. (1998) and De Wolf et al. (1999) studied conditions under which $\mathbf{P}$ is invertible and under which the estimator $\hat{T}_X$ has acceptable variance.

PRAM is the categorical counterpart of noise addition. It is widely applicable to nominal and ordinal variables, particularly sensitive attributes such as health status, religion, or occupation in official microdata releases.

**Measurement:** Native risk metric is [individual risk](12-sdc-risk-and-utility-metrics.md#individual-risk-benedetti-franconi) under the perturbed marginal, with [t-closeness](12-sdc-risk-and-utility-metrics.md#t-closeness) when the affected variable is sensitive; native utility metric is [KL divergence](12-sdc-risk-and-utility-metrics.md#kl-divergence-categorical) on the column marginal, which under invariant PRAM approaches zero by construction. The strength of PRAM's protection is precisely what makes KL the right loss measure: any deviation reflects sample-level variance around an unbiased target.

**References:** Gouweleeuw et al. (1997); Kooiman et al. (1998); De Wolf et al. (1999).

### MASSC

> Protect categorical key variables through four sequential steps — Micro Agglomeration, Substitution, Subsampling, and Calibration — achieving $k$-anonymity while preserving marginal distributions via post-stratification weights.

$$
k\text{-anonymity on key variables} \wedge \pi_{\text{post}} = \pi_{\text{original}}
$$

MASSC (Singh, Yu and Dunteman 2003) was designed specifically for official microdata files with categorical key variables (e.g., age group, region, occupation). It combines four operations applied in sequence:

**Step 1 — Micro Agglomeration.** Records are clustered into groups of at least $k$ using a distance metric on the categorical key variables. All members of a group receive the same masked key-variable values, achieving $k$-anonymity.

**Step 2 — Substitution.** Within each cluster, each record's key-variable values are replaced with those of a randomly drawn member of the cluster. This breaks the deterministic link between original and masked values while preserving the distribution of values within the cluster.

**Step 3 — Subsampling.** A random subsample of the substituted records is retained. Subsampling reduces the absolute number of records in the release, further limiting the precision with which an intruder can estimate cell probabilities.

**Step 4 — Calibration.** Post-stratification weights are applied to the subsample so that the weighted marginal distributions of key variables match those of the original population. This step restores the analytical validity of the released dataset for frequency estimation and logistic regression.

MASSC produces a microdata file that is simultaneously $k$-anonymous (structural protection), distributionally faithful (calibrated), and smaller than the original (sampling protection). Its main limitation is the complexity of the four-step pipeline and the need to choose cluster size $k$, subsample fraction, and calibration variables jointly to avoid large design effects.

**Measurement:** Native risk guarantee is [k-anonymity](12-sdc-risk-and-utility-metrics.md#k-anonymity) by construction (Step 1 — Micro Agglomeration), with residual [SUDA score](12-sdc-risk-and-utility-metrics.md#suda-score-msu) as the check on minimal sample uniques that survive subsampling; native utility metrics are [KL divergence](12-sdc-risk-and-utility-metrics.md#kl-divergence-categorical) on calibrated marginals (which Step 4 forces near zero) and [λ](12-sdc-risk-and-utility-metrics.md#-measure) on the substituted key-variable values.

**References:** Singh, Yu and Dunteman (2003).

## Non-perturbative Methods

### Sampling

> Release only a random subset of records rather than the full population file, so an intruder cannot be certain whether any specific individual appears in the data.

$$
S \subset U, \quad |S| = n, \quad \pi_i = P(i \in S)
$$

In simple random sampling without replacement, $\pi_i = n/N$ for all units $i$ in the population $U$ of size $N$. For complex survey designs (stratified, cluster, probability-proportional-to-size), $\pi_i$ varies across units and must be accounted for in analysis using Horvitz-Thompson or calibration estimators.

Sampling provides disclosure limitation through **uncertainty of inclusion**: an intruder who finds a record matching a known individual cannot confirm the match because the individual may simply not have been sampled. The protection is probabilistic and strongest when the sampling fraction $n/N$ is small. If the sampling fraction approaches 1 (a near-census), sampling alone provides negligible protection.

Released microdata files from household surveys are almost always samples rather than censuses, making sampling a default first layer of protection. Sampling does not distort the values of variables that are included — this distinguishes it sharply from perturbative methods and makes it attractive for research microdata where analytical accuracy on retained records is paramount. However, rare subpopulations are likely to be underrepresented or absent in the sample, which can itself be a disclosure risk: absence confirms non-membership in the subgroup.

**Measurement:** Native risk metric is [individual risk](12-sdc-risk-and-utility-metrics.md#individual-risk-benedetti-franconi), since sampling's protection is exactly the inclusion uncertainty that the Benedetti-Franconi $f_k / \hat{F}_k$ posterior models; there is no native value-distortion utility metric because sampled values are intact. Utility loss enters only through design effects, which are captured indirectly by [regression coefficient comparison (lm)](12-sdc-risk-and-utility-metrics.md#regression-coefficient-comparison-lm) against the full population (when available) or against larger samples.

**References:** Hundepool et al. (2014); standard survey sampling literature.

### Global Recoding

> Replace original fine-grained categories or measurement values with coarser groupings applied uniformly to every record, reducing the granularity of the entire variable.

$$
x' = g(x), \quad g: \mathcal{X} \to \mathcal{X}', \quad |\mathcal{X}'| < |\mathcal{X}|
$$

where $g$ is a many-to-one mapping (surjective but not injective) from the original value space to a coarser space.

For **continuous variables**, global recoding groups values into intervals: exact age becomes an age band, exact income becomes a bracket. The intervals $\mathcal{X}'$ are defined globally and applied identically to all records. For **categorical variables**, it merges categories: specific occupation codes collapse into broad occupational groups, or rare ethnic categories are combined into an "other" category.

The protection mechanism is straightforward: the released variable carries less information than the original, so fewer records are unique on the recoded variable and exact-match attacks using external databases become harder. The information loss is also straightforward to measure: each record in $\mathcal{X}'$ corresponds to an interval or a merged set, and the width of the interval relative to the analyst's needs determines utility loss.

Global recoding is frequently the first non-perturbative intervention because it is transparent, reversible in the sense that the mapping $g$ can be published, and does not require any record-level decisions. Its main drawback is that it applies the same coarsening to all records equally, including those records that pose little disclosure risk. This motivates hybrid approaches combining global recoding with local suppression to handle residual uniqueness after recoding.

**Measurement:** Native risk metric is [k-anonymity](12-sdc-risk-and-utility-metrics.md#k-anonymity) — recoding's purpose is precisely to reduce sample uniques on the quasi-identifier combination — with [SUDA score](12-sdc-risk-and-utility-metrics.md#suda-score-msu) as a residual diagnostic; native utility metrics are the [λ measure](12-sdc-risk-and-utility-metrics.md#-measure) (interval midpoint distance from the original) and [KL divergence](12-sdc-risk-and-utility-metrics.md#kl-divergence-categorical) on the recoded marginal distribution.

**References:** De Waal and Willenborg (1995, 1999); Hundepool et al. (2014).

### Top and Bottom Coding

> Replace values above a top threshold with the threshold value, and values below a bottom threshold with the threshold value, eliminating the most extreme observations that are most likely to be unique and thus re-identifiable.

$$
x' = \min(x,\; T_{top}), \qquad x' = \max(x,\; T_{bot})
$$

Top coding bounds the upper tail of a distribution: any record with $x > T_{top}$ has its value replaced by exactly $T_{top}$ (or labelled as "$T_{top}+$"). Bottom coding does the analogous operation for the lower tail. The thresholds are typically set at extreme quantiles — often the 98th or 99th percentile for top coding, or the 1st or 2nd percentile for bottom coding — to affect only the outliers that represent the greatest re-identification risk.

The rationale is that extreme values are disproportionately identifying. A billionaire's income, a centenarian's age, or a very small firm's employee count may be unique in the dataset and directly linkable to public records. By collapsing all such extremes to a single coded value the released data no longer reveals exactly how extreme any individual record is.

Top and bottom coding preserves the entire interior of the distribution without alteration, so all records not near the extremes receive no perturbation. This makes the method analytically neutral for the bulk of the data while providing targeted protection for the outliers. Analysts working with the tails — for example, studying high-income earners — must account for the censoring explicitly (e.g., using Tobit models or interval regression). For categorical variables, top and bottom coding generalises to collapsing the highest and lowest ordered categories into aggregated codes.

**Measurement:** Native risk metric is [RMDID1](12-sdc-risk-and-utility-metrics.md#rmdid1--robust-mahalanobis-disclosure-risk), since top/bottom coding directly targets robust-Mahalanobis outliers, with [individual risk](12-sdc-risk-and-utility-metrics.md#individual-risk-benedetti-franconi) as the check on whether the threshold is aggressive enough; native utility metric is the [λ measure](12-sdc-risk-and-utility-metrics.md#-measure), where the loss is concentrated on the censored tail records but contributes proportionally to the column average.

**References:** Hundepool et al. (2014); Willenborg and De Waal (2001).

### Local Suppression

> Set specific attribute values to missing for selected records, chosen to eliminate unique or rare combinations of key variables and thereby achieve $k$-anonymity, while minimising the total number of suppressions.

$$
x'_{ij} = \begin{cases} {*} & \text{if record } i \text{ is unique on key set } K \\ x_{ij} & \text{otherwise} \end{cases}
$$

$$
\text{Minimise} \sum_{i,j} \mathbf{1}[x'_{ij} = {*}] \quad \text{subject to: every key combination appears} \geq k \text{ times}
$$

A record is **unique** if no other record in the dataset shares the same combination of values on a set of key variables (quasi-identifiers such as age, sex, region, occupation). Unique records have re-identification risk 1 under exact-match attacks. Local suppression removes uniqueness by replacing one or more key values with a missing-value code $*$, which reduces the apparent combination's specificity until at least $k - 1$ other records match.

Unlike global recoding, local suppression makes record-level decisions: only the records that cause uniqueness are modified, and only the minimum necessary values are suppressed. This is efficient when most records are already safe and only a small fraction require treatment. The minimisation problem (find the fewest suppressions that achieve $k$-anonymity) is NP-hard in general but is tractable in practice via greedy heuristics or integer programming solvers, as implemented in tools such as $\mu$-Argus.

Local suppression is typically applied after global recoding: recoding reduces the number of unique records substantially, and suppression handles the residual cases. Released data with suppressed values must be analysed with methods that handle missing data (multiple imputation, maximum likelihood with missing-data mechanisms), and the missingness pattern is non-random — it is concentrated in the most unusual records — which must be disclosed to analysts.

**Measurement:** Native risk guarantee is [k-anonymity](12-sdc-risk-and-utility-metrics.md#k-anonymity) by construction (the suppression objective enforces it), with [benchmark risk outlier](12-sdc-risk-and-utility-metrics.md#benchmark-risk-outlier) as the diagnostic for records that demanded suppression; native utility metrics are the [λ measure](12-sdc-risk-and-utility-metrics.md#-measure) — by handbook convention each `*` cell counts as full range-fraction loss — and [KL divergence](12-sdc-risk-and-utility-metrics.md#kl-divergence-categorical) on the affected variable's marginal.

**References:** De Waal and Willenborg (1995); Hundepool et al. (2014).
