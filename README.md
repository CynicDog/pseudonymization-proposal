# Pseudonymization Proposal

Research-backed design proposal for a pseudonymization layer at the enterprise data platform. Covers technique selection, architecture, technology stack, ML pipeline integration, key management, and PIPA compliance.

This repository is a **proposal document set** — not implementation code. It is intended to be read by the implementation team (or a separate implementation repository) as the authoritative design specification.


## Intended Audience

| Role | Primary Documents |
|---|---|
| Security Architect | 02, 03, 05, 08, 10 |
| Data Engineer | 03, 05, 06, 07, 09 |
| ML Engineer | 03, 06, 07 |
| Legal / Compliance | 02, 10 |
| Engineering Lead | 01, 05, 09 |


## Document Index

| # | Document | Description |
|---|---|---|
| 01 | [Executive Summary](docs/01-executive-summary.md) | Problem statement, scope, recommended approach, roadmap at a glance |
| 02 | [Regulatory Context](docs/02-regulatory-context.md) | Korean PIPA (개인정보보호법), PIPC guidelines, compliance posture |
| 03 | [Pseudonymization Techniques](docs/03-pseudonymization-techniques.md) | Taxonomy of techniques, decision criteria, field-type mapping |
| 04 | [Industry Solutions](docs/04-industry-solutions.md) | Survey of open-source and commercial tools; recommendations |
| 05 | [Architecture](docs/05-architecture.md) | Three-zone data platform architecture with ASCII diagrams |
| 06 | [Technology Stack](docs/06-technology-stack.md) | Polars/PyArrow-based stack; why not PySpark; Korean NLP |
| 07 | [ML Pipeline Integration](docs/07-ml-pipeline.md) | Pseudonymized feature design, inference boundary, DP options |
| 08 | [Key Management](docs/08-key-management.md) | Azure Key Vault design, key hierarchy, rotation strategy |
| 09 | [Implementation Roadmap](docs/09-implementation-roadmap.md) | Three-phase sprint checklist for the implementation team |
| 10 | [Compliance & Audit Framework](docs/10-compliance-audit.md) | PIPA compliance checklist, audit architecture, risk metrics |


## How to Use This Repo

1. Start with **01-executive-summary** for the full picture.
2. Implementation team: use **09-implementation-roadmap** as a sprint backlog.
3. Security / legal review: focus on **02**, **08**, and **10**.
4. Each document is self-contained and cross-references others where relevant.
