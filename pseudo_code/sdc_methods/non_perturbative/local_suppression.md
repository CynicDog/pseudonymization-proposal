# Local Suppression

> Set specific attribute values to missing for selected records, chosen to eliminate unique or rare combinations of key variables and thereby achieve k-anonymity, while minimising the total number of suppressions.

$$
x'_{ij} := \begin{cases} * & \text{if record } i \text{ is unique on key set } K \\ x_{ij} & \text{otherwise} \end{cases}
$$

$$
\text{Minimise } \sum_{i, j} \mathbf{1}[x'_{ij} = *] \text{ subject to: every key combination appears} \geq k \text{ times.}
$$

## Inputs

- `X` — microdata with `n` records.
- `K ⊆ {1, …, p}` — index set of quasi-identifier columns.
- `k ≥ 2` — anonymity threshold.
- `solver ∈ {greedy, ilp}` — optimisation strategy.
- Optional per-cell suppression cost `c_{ij} > 0` (if analytical priority differs by column).

## Outputs

- `Z` — protected microdata where some cells in `K`-columns are replaced by the suppression symbol `*`.
- `holds := (Z is k-anonymous on K)`.

## Algorithm — Greedy Heuristic

1. Project records onto quasi-identifiers `K` and group; compute the equivalence-class size `c_i` for each record (see [k-Anonymity](../../risk_metrics/k_anonymity.md)).
2. Identify the set of *violating* records `V := {i : c_i < k}`.
3. While `V ≠ ∅`:
   - For each violating record `i`, for each candidate column `j ∈ K`:
     - Hypothetically set `x_{ij} ← *`. Recompute equivalence-class sizes treating `*` as a wildcard matching any value on that column.
     - Compute the gain `Δ_{ij} := (number of records moved out of V by this suppression)` and the cost `c_{ij}`.
   - Choose the `(i, j)` pair with maximum `Δ_{ij} / c_{ij}`. Apply the suppression: `Z[i, j] ← *`.
   - Recompute `V`.
4. Return `(Z, holds)`.

## Algorithm — ILP Formulation (Optimal)

1. For each cell `(i, j)` with `i ∈ {1, …, n}, j ∈ K`, introduce a binary variable `y_{ij} ∈ {0, 1}` (1 = suppress).
2. For each k-anonymity constraint `q` (the projection of the constraint that the recoded equivalence class of record `i` must contain ≥ `k` records), encode `y_{ij}` patterns that satisfy the constraint as linear inequalities.
3. Solve:
   $$
   \min \sum_{i, j} c_{ij} \cdot y_{ij} \quad \text{subject to k-anonymity constraints}.
   $$
4. Decode `y_{ij}` into suppressions: `Z[i, j] ← *` iff `y_{ij} = 1`. Use a standard ILP solver (e.g. CBC, Gurobi).
5. Return `(Z, holds)`.

## Notes

- The optimisation problem (minimum cells to suppress for k-anonymity) is **NP-hard** in general (Meyerson and Williams 2004), but tractable in practice for typical microdata via greedy heuristics or ILP solvers (μ-Argus uses this approach).
- Typically applied **after** [Global Recoding](global_recoding.md): recoding bulk-reduces uniqueness; suppression handles residual cases efficiently.
- Suppression introduces **non-random missingness** concentrated in the most unusual records. Analysts must be told to use missing-data methods (multiple imputation, MLE with missing-data mechanisms) and that the missing pattern is informative.
- For the [λ Measure](../../information_loss_metrics/lambda_measure.md), each `*` cell counts as full range-fraction loss (`|x_{ik} - z_{ik}| / R_k := 1` by handbook convention).
- Native risk guarantee: [k-Anonymity](../../risk_metrics/k_anonymity.md) by construction; pair with [Benchmark Risk Outlier](../../risk_metrics/benchmark_risk_outlier.md) to identify which records demanded suppression.
- Native utility metrics: [λ](../../information_loss_metrics/lambda_measure.md) and [KL Divergence](../../information_loss_metrics/kl_divergence.md) on the affected variables' marginals.

## References

- De Waal and Willenborg (1995).
- Hundepool et al. (2014), Chapter 5.
- Hundepool et al. (2014), `papers/non-perturbative/Hundepool-2014-mu-ARGUS-v5.1-manual.pdf`.
- Meyerson and Williams (2004). On the complexity of optimal k-anonymity.
- [docs/11 § Local Suppression](../../../docs/11-sdc-microdata-protection-methods.md#local-suppression).
