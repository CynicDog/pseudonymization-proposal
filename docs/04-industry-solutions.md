# 04 — Approaches and Research Grounding

## How the Industry Frames This Problem

Pseudonymization of insurance and financial data is treated in the literature and in regulatory guidance primarily as a **statistical disclosure control** (SDC) problem, not a cryptographic one. The distinction matters: cryptography protects data in transit and at rest from unauthorized access; SDC protects the analytical content of data from enabling re-identification when the data is shared, processed, or used for modeling.

The authoritative body of work on SDC for microdata — individual-level records of the kind that appear in insurance portfolios — is organized around three questions:

**1. What makes a record identifying?** A record is identifying when a combination of its attribute values is rare enough that it can be linked to a specific individual using external information (public registries, social media, other datasets). High income, unusual claim patterns, rare diagnosis combinations, and extreme ages are common quasi-identifiers in insurance data.

**2. What transformations reduce identifiability while preserving analytical utility?** The answer depends on the field type. For continuous numerical attributes (income, claim severity), the SDC toolkit — rounding, top-coding, micro-aggregation, generalization — is designed to suppress the rare extremes that enable singling out while leaving the bulk of the distribution intact for modeling.

**3. How is the residual risk measured?** Re-identification risk is quantified through measures like k-anonymity (how many records share a given quasi-identifier combination), information loss metrics, and adversarial attack simulations (singling-out, linkability, inference).

## Research Foundation

The technique choices in this proposal are grounded in the following authoritative sources. These should be accessible to the implementation team as reference material.

**Hundepool, A., Domingo-Ferrer, J., Franconi, L., Giessing, S., Schulte Nordholt, E., Spicer, K., & de Wolf, P.P. (2012). *Statistical Disclosure Control*. Wiley.**

The reference textbook for SDC methodology. Covers the full range of techniques for microdata (rounding, top-coding, micro-aggregation, generalization, data swapping, synthetic data, noise addition) with a rigorous treatment of information loss measurement and disclosure risk. This is the primary methodological reference for the numerical field treatment in this platform.

**Domingo-Ferrer, J. (2016). *Database Anonymization: Privacy Models, Data Utility, and Microaggregation-based Inter-model Connections*. Morgan & Claypool Synthesis Lectures.**

Connects k-anonymity, t-closeness, and differential privacy through the micro-aggregation framework, and shows that applying formal privacy models to micro-aggregated data achieves better utility than applying them to raw data. Directly relevant to the ML pipeline design where both analytical utility and formal privacy guarantees are desired.

**Oganian, A. & Domingo-Ferrer, J. (2001). On the Complexity of Optimal Microaggregation for Statistical Disclosure Control. *Statistical Journal of the UN Economic Commission for Europe*, 18(4), 345–353.**

Establishes that optimal micro-aggregation is NP-hard, motivating the greedy contiguous-sort approximation that is the practical standard. Understanding this result clarifies why all production implementations use approximations and what information loss to expect.

**El Emam, K., Jonker, E., Arbuckle, L., & Malin, B. (2011). A Systematic Review of Re-Identification Attacks on Health Data. *PLoS ONE*, 6(12), e28071.**

A comprehensive review of known re-identification attacks on de-identified health data — the domain most analogous to insurance microdata. The attack methodologies described (quasi-identifier matching, probabilistic linkage) inform the risk assessment approach required by the PIPA preliminary plan.

**Dwork, C. & Roth, A. (2014). *The Algorithmic Foundations of Differential Privacy*. Foundations and Trends in Theoretical Computer Science, 9(3–4).**

The formal reference for differential privacy, covering Laplace and Gaussian mechanisms, composition theorems, and privacy budget allocation. Referenced when noise perturbation is applied or when a formal DP bound is required for external data sharing.

**NIST SP 800-38G (2016, Draft Rev. 1 February 2025). *Recommendation for Block Cipher Modes of Operation: Methods for Format-Preserving Encryption*.**

The standard specifying FF1, the format-preserving encryption algorithm used for direct identifier pseudonymization where the original field format must be preserved. The 2025 revision removes FF3/FF3-1 from the standard due to a cryptographic weakness; only FF1 is compliant.

**NIST SP 800-188 (2023). *De-Identifying Government Datasets*. Garfinkel, S.L.**

Practical federal guidance on de-identification, covering the full data lifecycle, quasi-identifier identification, k-anonymity, and formal privacy models. More accessible than the primary research literature; useful as a reference for compliance documentation.

**PIPC (2020). *Comprehensive Guidelines on Processing Pseudonymized Data*. Personal Information Protection Commission, Korea.**

The authoritative regulatory guidance for PIPA-compliant pseudonymization in Korea. Establishes the four-stage process (preliminary plan, risk assessment, execution, evaluation) and the organizational accountability framework. The pseudonymization approach in this proposal is designed to satisfy these requirements.

## How the SDC Approach Applies Here

The insurance portfolio data in this platform is a textbook SDC scenario. Policy records contain a mix of direct identifiers (which are handled by consistent surrogation and dropped from ML features) and sensitive numerical attributes (income, premiums, claim amounts, benefit levels) that are the actual modeling targets and features.

For the numerical attributes, the established approach — round to reduce precision, top-code to suppress identifying extremes, micro-aggregate if a k-anonymity guarantee is needed — has been validated in actuarial and statistical contexts for decades. It is the approach used by national statistical offices when publishing insurance and financial microdata for research use.

The key property that makes this approach appropriate for ML is that all three techniques (rounding, top-coding, micro-aggregation) preserve the distributional shape of the data in the central mass of the distribution — precisely where most records and most model training signal live. Only the rare extreme values, which are simultaneously the most identifying and the least representative of the overall population, are meaningfully altered.

## The PII Detection Layer

Identifying which columns in the data require treatment — and which SDC or surrogation technique to apply — is the classification problem. For this platform's tabular data, the classification approach relies on:

- **Schema-level knowledge**: The data classification inventory produced in Phase 1 is the primary source. Data owners and the actuarial team know which columns contain income, which contain claim amounts, and which contain direct identifiers.
- **Pattern-based detection**: For string-type columns that may contain structured identifiers (RRN, policy number, phone number), regex-based pattern matching against the column values confirms and validates the classification.
- **Column-name heuristics**: Systematic naming conventions (`*_income`, `*_amount`, `*_id`, `*_rnn`) supplement the inventory for columns added after the initial classification.

Microsoft Presidio's `StructuredEngine` is well-suited to the pattern-based detection step for structured fields. For a 100% tabular dataset with no free text, it operates purely on regex `PatternRecognizer` instances — no language model or NER inference is involved.

## What the Platform Does Not Need

**Vaulted tokenization**: Stores original values in a secure vault and replaces production records with random tokens. The vault lookup overhead makes it incompatible with ML feature pipelines. For this platform, consistent surrogation via deterministic hash achieves the same join-key function without the vault infrastructure.

**Heavy encryption for analytical fields**: Format-preserving encryption (FF1) and symmetric encryption are appropriate for direct identifiers that need an opaque, format-consistent surrogate. Applying them to income, claim amounts, or any field that a model must learn from produces a result that is both privacy-protecting and analytically useless. The correct tool for analytical fields is SDC.

**Differential privacy as a data layer**: Laplace/Gaussian noise applied field-by-field at the data storage layer is rarely the right answer for insurance microdata with high-range fields. The noise required to achieve a meaningful privacy budget at the field level destroys the signal. DP belongs at the model training boundary (DP-SGD), not at the data layer, except for fields with naturally bounded small ranges.
