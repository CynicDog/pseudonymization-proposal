# 04 — Industry Solutions Survey

## Overview

This document surveys the current landscape of pseudonymization and data privacy tooling — open-source libraries, cloud-managed services, and commercial enterprise platforms. Each is evaluated in the context of this company's infrastructure (Azure-centric, ADF + Databricks, MB-GB data scale, Korean PII patterns).

---

## Open-Source Tools (Recommended Stack)

### Microsoft Presidio

**Repository:** github.com/microsoft/presidio  
**License:** MIT

Presidio is Microsoft's open-source framework for PII detection and anonymization in text, structured data, and images. It operates in two stages:

1. **Presidio Analyzer:** Detects PII entities using a combination of named entity recognition (NER via spaCy), regex patterns, rule-based context enhancement, and a confidence scoring model.
2. **Presidio Anonymizer:** Applies configurable operators to detected entities — redact, replace, mask, hash, encrypt, or custom operator.

**Why Presidio for this stack:**
- Python-native; integrates naturally with Polars/Pandas pipelines via the `anonymize` operator
- Documented integration with Azure Data Factory and Databricks (microsoft.github.io/presidio/samples/deployments/data-factory/)
- Fully customizable: add custom recognizers for Korean PII patterns (주민등록번호 regex, Korean phone format, 사업자등록번호)
- Active maintenance by Microsoft; Azure alignment reduces integration friction
- The analyzer can scan structured DataFrames column-by-column and return entity-type annotations, enabling automated column classification without manual inventory

**Role in this proposal:** Detection and classification layer. Presidio identifies which columns contain PII and of what type; the pseudonymization engine (Polars + FF1/HMAC) then applies the appropriate technique.

**Installation:**
```
pip install presidio-analyzer presidio-anonymizer
python -m spacy download en_core_web_lg
```

For Korean text, a Korean spaCy model or a `kiwipiepy`-based custom NER component should be added.

---

### ff3 (PyPI)

**Package:** `ff3`  
**License:** Apache 2.0

Implements Format-Preserving Encryption (FPE) using the FF1 algorithm (NIST SP 800-38G). Lightweight, no heavy dependencies. Supports configurable radix (numeric, alphanumeric, arbitrary character set) and tweak (domain separator for key derivation per field type).

**Usage pattern:**
```python
from ff3 import FF3Cipher
cipher = FF3Cipher(key, tweak, radix=10)
pseudonym = cipher.encrypt(plaintext)
original  = cipher.decrypt(pseudonym)
```

**Limitations:**
- Pure Python; for high-throughput columnar transforms (millions of rows), wrap in a Polars expression or batch-vectorize
- Minimum domain constraint: radix^minlen ≥ 100 (satisfied by all standard PII field types)
- Use FF1 mode only (the library supports both FF1 and FF3-1; FF3-1 is cryptographically broken — see [03-pseudonymization-techniques.md](03-pseudonymization-techniques.md))

---

### py4phi

**Repository:** github.com/phi-friday/py4phi  
**License:** MIT

A DataFrame encryption library supporting Polars, Pandas, and PySpark as backends. Provides column-level AES encryption/decryption with built-in key and salt management. Includes a CLI interface for batch operations.

**Role in this proposal:** AES-based encryption for fields where FPE format preservation is not required (e.g., AES-SIV equivalent for variable-length blobs). Also useful as a wrapper for key file management when prototyping the pseudonymization module.

---

### scrubadub

**Package:** `scrubadub`  
**License:** MIT

Regex and rule-based PII scrubber for unstructured text. Detects and redacts names, email addresses, phone numbers, credit card numbers, and other patterns in free-text strings.

**Role:** Secondary detection layer for free-text fields (claim notes, customer service transcripts, policy remarks) where Presidio NER may be supplemented with lightweight regex fallback.

---

### Faker / Mimesis

**Packages:** `faker`, `mimesis`

Synthetic data generation libraries producing realistic-looking but entirely fictitious PII (names, addresses, phone numbers, dates, IDs). Faker supports Korean locale (`ko_KR`) via its locale system.

**Role:** Development and test dataset generation only. These libraries are non-deterministic and irreversible — they are not suitable for production pseudonymization pipelines where referential integrity across runs is required.

---

### Apache Ranger / Apache Atlas

**Role:** Access control (Ranger) and data governance / lineage (Atlas) for Hadoop/Spark ecosystems.

**Applicability:** If the team adopts Databricks Unity Catalog, Ranger/Atlas are superseded. If not, Ranger can enforce column-level masking policies in Databricks SQL; Atlas can track data lineage from raw to pseudonymized zone. These are optional infrastructure additions, not core pseudonymization components.

---

## Cloud-Managed Services

### Microsoft Purview (Azure Purview)

Microsoft Purview (formerly Azure Purview) is a unified data governance service with native integration into Azure Data Factory, ADLS, and Databricks.

**Relevant capabilities:**
- **Sensitivity labeling:** Automatically scan ADLS and annotate datasets with sensitivity classifications
- **Data lineage:** Track data flow from on-prem source through ADF pipelines to ADLS zones and Databricks
- **Data catalog:** Central inventory of all data assets with classification metadata

**Role in this proposal:** Metadata and governance layer. Purview labels datasets as containing PII/SPII classes; these labels inform the pseudonymization engine which technique to apply per column. Purview's lineage graph provides the audit trail evidence for PIPA compliance documentation.

**Recommendation:** Adopt Purview as the classification metadata layer. It is available as part of Azure enterprise agreements and integrates directly with the existing Azure infrastructure.

---

### Google Cloud Sensitive Data Protection (Cloud DLP)

Google's managed PII detection and pseudonymization service. Supports FPE-FFX (compatible with FF1), reversible tokenization, redaction, masking, and bucketing. 120+ built-in detectors covering global and regional PII types.

**Not recommended for this stack.** The company's infrastructure is Azure-centric (ADF, ADLS, Databricks on Azure). Introducing GCP DLP creates a cross-cloud dependency, adds latency for data that lives in Azure, complicates IAM management, and adds cost for data egress. The open-source stack (Presidio + ff3) achieves equivalent functionality without this coupling.

---

### AWS Macie / AWS Glue DataBrew

AWS-native PII discovery (Macie) and data preparation / masking (DataBrew). Not applicable to an Azure-based stack.

---

## Enterprise Commercial Platforms (Reference Only)

These platforms are included for completeness and competitive awareness. None are recommended for this use case given the available open-source alternatives and the MB-GB data scale.

### Protegrity

Enterprise data security platform offering tokenization, FPE (AES-256 based), and masking across structured and unstructured data in hybrid/multi-cloud environments. Strong enterprise support and compliance track record.

**Assessment:** Mature and proven at scale, but significantly overengineered and over-priced for MB-GB data volumes. FPE capability is equivalent to the `ff3` open-source library for this use case.

### Privitar Publisher

Policy-based de-identification and anonymization framework. Strong privacy risk quantification and policy management features.

**Assessment:** Better suited for large enterprises with dedicated privacy engineering teams managing complex multi-dataset anonymization policies. The open-source stack with documented policies is sufficient here.

### IBM InfoSphere Optim Data Privacy

Legacy enterprise platform for data masking and subsetting. Supports FPE (AES-256) with user-defined keys across on-premises and cloud data sources.

**Assessment:** Complex deployment, declining ecosystem support, not a fit for a modern Azure + Polars stack.

---

## Recommendation Summary

| Tool | Role | Adopt |
|---|---|---|
| Microsoft Presidio | PII detection and classification | Yes |
| ff3 (PyPI) | FF1 format-preserving encryption | Yes |
| py4phi | AES column encryption, key management helper | Yes |
| scrubadub | Free-text PII redaction fallback | Optional |
| Faker / Mimesis | Dev/test synthetic data | Yes (non-prod only) |
| Microsoft Purview | Data catalog, sensitivity labeling, lineage | Yes |
| Apache Ranger / Atlas | Access control / lineage (if no Unity Catalog) | Optional |
| Google Cloud DLP | FPE + tokenization (managed) | No |
| Protegrity | Enterprise FPE platform | No |
| Privitar / IBM Optim | Enterprise privacy platforms | No |

The recommended stack is entirely open-source for the core pseudonymization engine, with Azure-native services (Key Vault, Purview, Monitor) for governance and operations. This minimizes vendor lock-in, maximizes auditability, and is right-sized for MB-GB workloads.
