# 10 — Compliance & Audit Framework

## Overview

This document provides the compliance checklist, audit architecture, and operational metrics for maintaining PIPA (개인정보보호법) compliance in the pseudonymization layer. It is intended for the DPO function, legal team, and information security team.

For regulatory background, see [02-regulatory-context.md](02-regulatory-context.md). For key management audit controls, see [08-key-management.md](08-key-management.md).


## PIPA Compliance Checklist

Based on the PIPC "Comprehensive Guidelines on Processing Pseudonymized Data" (Sept 2020) and PIPA Article 28-2 through 28-7.

### Pre-Processing Requirements

- [ ] **Preliminary plan documented** — Purpose of pseudonymization, scope (which data, which systems), selected technique, responsible organizational unit, and planned use of pseudonymized data are documented in writing before processing begins
- [ ] **Purpose is lawful** — Processing purpose falls within PIPA Article 28-2 permitted uses: statistics, scientific/industrial research, or public interest record-keeping. ML model development qualifies as industrial research.
- [ ] **Classification inventory completed** — All in-scope columns classified by PII/SPII tier; classification manifest reviewed by data owner and legal/compliance stakeholder
- [ ] **Re-identification risk assessment completed** — Entropy analysis, quasi-identifier analysis, and singling-out risk documented; risk level assessed as acceptable for stated purpose
- [ ] **Minimum-use principle applied** — Only fields required for the stated downstream purpose are retained in the pseudonymized zone; all other PII/SPII fields are dropped at the pseudonymization boundary

### Technical Control Requirements

- [ ] **Pseudonymization technique documented** — FF1 (NIST SP 800-38G) + HMAC-SHA-256; algorithm selection rationale recorded
- [ ] **Keys managed separately from data** — an isolated key management service; no key material co-located with pseudonymized data; service-identity access only
- [ ] **Encryption strength sufficient** — AES-256 key material; computational infeasibility of brute-force attack documented
- [ ] **Raw data access restricted** — Raw storage zone accessible only via the authorized ingestion service identity and documented break-glass procedure; no data engineer or ML engineer direct access
- [ ] **Pseudonymized zone access controlled** — RBAC implemented; access limited to data engineering and ML roles; no raw-zone cross-access
- [ ] **Key rotation policy active** — Quarterly automated rotation; key versioning ensures no data loss; rotation events logged

### Operational Control Requirements

- [ ] **Audit logging active and immutable** — Audit logging platform; KMS access logs; storage access logs; pipeline execution logs; 90-day active retention + 1-year archive
- [ ] **Re-identification prohibition enforced** — Technical controls (no keys in ML compute) and policy controls (contractual prohibition for any team accessing pseudonymized data)
- [ ] **Incident response plan in place** — Key compromise, unauthorized access, and suspected re-identification scenarios documented with response timelines
- [ ] **Third-party provision controls** — If pseudonymized data is shared with any external party, additional contractual safeguards and PIPC notification procedures are in place

### Ongoing Compliance Requirements

- [ ] **Annual risk assessment refresh** — Re-identification risk re-assessed when new data sources are added, when query patterns change, or annually at minimum
- [ ] **Quarterly key rotation executed** — Rotation log reviewed by security team; old key versions retired per schedule
- [ ] **Quarterly access review** — RBAC assignments reviewed; any unnecessary permissions revoked
- [ ] **PIPA compliance documentation maintained** — All required records (preliminary plan, risk assessment, audit logs) retained for minimum 3 years (PIPA retention guidance)


## Audit Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Event Sources                                               │
│                                                              │
│  ┌────────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │  KMS           │  │  Storage     │  │  Pipeline       │  │
│  │  Audit Log     │  │  Access Log  │  │  Execution Log  │  │
│  │  (key access)  │  │  (raw zone + │  │  (pseudo jobs)  │  │
│  │                │  │  pseudo zone)│  │                 │  │
│  └───────┬────────┘  └──────┬───────┘  └────────┬────────┘  │
│          │                  │                    │           │
│          └──────────────────┴────────────────────┘           │
│                             │                                │
│                    Audit Log Aggregator                      │
│                    (cloud-side or on-prem SIEM)              │
│                             │                                │
│                             ▼                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Immutable Log Store                                   │  │
│  │  • 90-day active retention                             │  │
│  │  • Archive: 1 year                                     │  │
│  │  • Ad-hoc query for audit and compliance reports       │  │
│  └────────────────────────────────────────────────────────┘  │
│                             │                                │
│              ┌──────────────┴──────────────┐                 │
│              ▼                             ▼                 │
│  ┌─────────────────────┐      ┌────────────────────────┐    │
│  │  Alert Rules        │      │  Data Lineage Tracker   │    │
│  │  • Unexpected raw   │      │  on-prem → raw →        │    │
│  │    zone access      │      │  pseudonymized → ML     │    │
│  │  • KMS errors       │      │  (PIPC audit evidence)  │    │
│  │  • Job failure      │      │                        │    │
│  │  • Key expiry warns │      └────────────────────────┘    │
│  └─────────────────────┘                                     │
└──────────────────────────────────────────────────────────────┘
```

### Log Event Types

| Event | Source | Retention | PIPA Evidence |
|---|---|---|---|
| Key retrieval | KMS audit log | 1 year | Who accessed keys, when, for which job |
| Raw zone read | Storage access log | 1 year | Every access to raw PII data |
| Raw zone write (ingestion) | Storage access log | 1 year | Data ingestion audit trail |
| Pseudonymization job run | Audit platform (custom event) | 1 year | Which columns, which key version, row count |
| Pseudonymized zone access | Storage access log | 90 days | Normal operational access |
| RBAC change | Identity provider audit log | 90 days | Access control change tracking |


## Re-identification Risk Metrics

The following metrics should be computed and reviewed quarterly as part of the risk assessment refresh:

### Entropy Metric
For each pseudonymized column, the entropy of the pseudonym space should exceed the entropy of the raw value space (by definition for FF1, since it is a permutation cipher). The metric to track:

```
effective_entropy(column) = log2(distinct_pseudonym_count)
```

For a column with 50,000 distinct customer IDs pseudonymized with FF1 over a 10-digit numeric domain (domain size = 10^10), the pseudonym is one of 10^10 possible values — entropy ≈ 33 bits. At AES-256, the key alone provides 256 bits of security; brute-force enumeration is computationally infeasible.

### Quasi-Identifier Risk Score
Identify columns in the pseudonymized dataset that, in combination, could narrow re-identification. Common quasi-identifiers in insurance/financial data:

- Age range (derived from birth year if not pseudonymized)
- Gender (if retained)
- Geographic region (city/district)
- Policy type / product category
- Event date (claim submission date, policy start date)

Use the PIPC-recommended analysis approach: count the size of equivalence classes for the top-N quasi-identifier combinations. If any combination produces equivalence classes of size 1 (i.e., unique individuals), those fields should either be pseudonymized or suppressed/generalized.

**Target:** Minimum equivalence class size ≥ 5 for all quasi-identifier combinations after pseudonymization of direct identifiers. Document the actual measured value quarterly.

### Key Age Compliance
Track the age of each active key version against the 90-day rotation target:

```
key_age_days = (current_date - key_creation_date)
compliance = key_age_days < 90
```

Alert threshold: 75 days (15-day warning before rotation due).

### Raw Zone Access Rate
The raw zone should be accessed only by the authorized ingestion service identity and documented break-glass procedures. Track:

```
unexpected_access_count = raw_zone_access_events WHERE principal NOT IN [ingestion_service_identity, pseudo_job_identity]
```

Target: 0 unexpected accesses. Any non-zero value triggers an immediate security review.


## PIPA Audit Response Pack

Maintain the following documents in a designated, access-controlled location (DPO and legal access) for rapid response to a PIPC inquiry:

| Document | Owner | Review Frequency |
|---|---|---|
| Preliminary pseudonymization plan | DPO | Per new data source |
| Data classification manifest (current version) | Data Platform Lead | Quarterly |
| Re-identification risk assessment report | Security Team | Annually (minimum) |
| Key management architecture document | Security Architect | Per major change |
| RBAC assignment report | IT Admin | Quarterly |
| Pseudonymization job audit log export | Data Platform Lead | On-demand (PIPC request) |
| Incident log (if applicable) | DPO | Per incident |

In the event of a PIPC inspection or complaint:
1. Assemble the above pack within 48 hours
2. Prepare audit log queries to export specific time-range audit logs from the log store
3. Engage legal counsel with PIPA expertise before responding to the PIPC


## Data Lineage as Compliance Evidence

A data catalog or lineage tool integrated with the pipeline provides queryable records of data movement. For PIPA compliance, lineage tracking should demonstrate:

- The end-to-end data flow: on-prem source → raw zone → pseudonymized zone → ML/analytics
- That raw data did not flow directly to consumption layers (no lineage edge bypassing the pseudonymization boundary)
- Column-level sensitivity labels showing which columns are classified as PII/SPII and which are pseudonymized

Export lineage records and sensitivity label reports quarterly for inclusion in the compliance documentation pack.
