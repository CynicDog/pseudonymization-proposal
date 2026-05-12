# SUDA Score (MSU)

> Identify records that are unique on some *minimal* subset of quasi-identifier attributes, then weight each minimal sample-unique (MSU) by the inverse of its size — small MSUs indicate a record an adversary can re-identify with very few attributes.

## Inputs

- `X` — microdata with `n` records.
- `A := {a_1, …, a_ATT}` — set of `ATT` declared quasi-identifier attributes.
- `M ≤ ATT` — maximum MSU size to search (typical: 3 or 4).

## Outputs

- `MSU(i)` for each record `i` — set of minimal sample-unique attribute subsets of size ≤ `M`.
- `SUDA_i` — per-record SUDA score.

## Algorithm

1. For each subset size `m ∈ {1, 2, …, M}`, enumerate all attribute subsets `S ⊆ A` of size `m` (in order of increasing `m`):
   - For each subset `S`, group records by their projection onto `S`. A record `i` is **sample-unique on `S`** iff its projected tuple appears in exactly one row of `X`.
2. Filter sample-uniques on `S` to retain only *minimal* sample-uniques: a sample-unique `(i, S)` is **minimal** iff no proper subset `S' ⊊ S` already makes `i` sample-unique. Use the candidate-pruning recursion of Manning, Haglin and Keane (2008): when checking a size-`m` subset `S`, skip any `i` that has already been recorded as sample-unique on some `S' ⊊ S`.
3. For each record `i`, collect `MSU(i) ← {S : (i, S) is a minimal sample unique with |S| ≤ M}`.
4. For each record `i`, compute the SUDA score:
   $$
   \mathrm{SUDA}_i := \sum_{S \in \mathrm{MSU}(i)} \mathrm{ATT}^{\,M - |S|}.
   $$
   A record unique on a single attribute (|S| = 1) contributes `ATT^{M-1}`; a record unique only on a four-attribute combination contributes `ATT^{M-4}`.
5. Return `({MSU(i)}_i, {SUDA_i}_i)`.

## Notes

- Computational cost: `O(Σ_{m=1..M} C(ATT, m))` group-by passes; tractable for typical `ATT ∈ [5, 10]` and `M ∈ [3, 4]`.
- The exponential weighting reflects adversary effort: a small MSU means the adversary needs few facts to re-identify, hence a higher score.
- SUDA is the natural residual-risk diagnostic for [Global Recoding](../sdc_methods/non_perturbative/global_recoding.md) and [MASSC](../sdc_methods/perturbative/massc.md), which target overall k-anonymity but may leave small-MSU records untouched.
- An `i` with `MSU(i) = ∅` (no MSU of size ≤ `M`) has `SUDA_i = 0`.

## References

- Elliot, Manning and Ford (2002). A computational algorithm for handling the special uniques problem. *IJUFKS* 10(5):493–509.
- Manning, Haglin and Keane (2008). A recursive search algorithm for statistical disclosure assessment. *DMKD* 16(2):165–196.
- [docs/12 § SUDA Score](../../docs/12-sdc-risk-and-utility-metrics.md#suda-score-msu).
