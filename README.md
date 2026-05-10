# Pseudonymization Proposal

Research-backed design proposal for a pseudonymization layer at the enterprise data platform. Covers technique selection, architecture, technology stack, ML pipeline integration, key management, and PIPA compliance.

This repository is a **proposal document set** — not implementation code. It is intended to be read by the implementation team (or a separate implementation repository) as the authoritative design specification.


## Intended Audience

| Role | Primary Documents |
|---|---|
| Security Architect | 02, 03, 05, 08, 09 |
| Data Engineer | 03, 05, 06, 07 |
| ML Engineer | 03, 06, 07 |
| Legal / Compliance | 02, 09 |
| Engineering Lead | 01, 05, 10, 11 |


## Document Index

| # | Document | Description |
|---|---|---|
| 01 | [Executive Summary](docs/01-executive-summary.md) | Problem statement, scope, recommended approach, roadmap at a glance |
| 02 | [Regulatory Context](docs/02-regulatory-context.md) | Korean PIPA (개인정보보호법), PIPC guidelines, compliance posture |
| 03 | [Pseudonymization Techniques](docs/03-pseudonymization-techniques.md) | Taxonomy of techniques, decision criteria, field-type mapping |
| 04 | [Approaches and Research Grounding](docs/04-industry-solutions.md) | SDC methodology, research references, detection layer design |
| 05 | [Architecture](docs/05-architecture.md) | Three-zone data platform architecture with ASCII diagrams |
| 06 | [Technology Stack](docs/06-technology-stack.md) | Polars/PyArrow-based stack; why not PySpark; Korean NLP |
| 07 | [ML Pipeline Integration](docs/07-ml-pipeline.md) | Pseudonymized feature design, inference boundary, DP options |
| 08 | [Key Management](docs/08-key-management.md) | Azure Key Vault design, key hierarchy, rotation strategy |
| 09 | [Compliance & Audit Framework](docs/09-compliance-audit.md) | PIPA compliance checklist, audit architecture, risk metrics |
| 10 | [SDC Foundations and Guidance](docs/10-sdc-foundations-and-guidance.md) | Statistical disclosure control concepts, risk-utility framework |
| 11 | [SDC Microdata Protection Methods](docs/11-sdc-microdata-protection-methods.md) | All 13 masking methods with formulas, explanations, and citations |


## How to Use This Repo

1. Start with **01-executive-summary** for the full picture.
2. Security / legal review: focus on **02**, **08**, and **09**.
3. SDC method selection: start with **10**, then use **11** as a reference.
4. Each document is self-contained and cross-references others where relevant.
