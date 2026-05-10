# 08 — Key Management

## Design Principles

- **Keys never co-locate with data.** Key material and pseudonymized data are stored in separate systems with separate access controls.
- **Keys stay within the on-prem boundary.** The pseudonymization service runs on-prem; keys are retrieved from an on-prem KMS or from a cloud KMS over a private network. No key material is exposed to cloud compute environments.
- **No human access to production key values.** Production keys are accessed only by the pseudonymization service process. Human admins manage key lifecycle (create, rotate, retire) but never retrieve key values directly.
- **Separate key namespaces by classification tier.** A key compromise affecting general PII must not affect sensitive PII. Keys are isolated per tier.
- **Key versioning, not invalidation.** Rotation creates a new key version; existing pseudonymized data retains a reference to the version used. Historical data remains readable without mass re-pseudonymization.
- **All key access is audited.** Every key retrieval is logged immutably (KMS-native audit log or SIEM, depending on KMS choice).


## Key Store Options

Two key store configurations are viable. The choice depends on existing infrastructure and operational preference. The key hierarchy, naming convention, and rotation strategy are identical in both cases.

**Option 1 — On-Prem KMS**

An on-prem KMS keeps all key material entirely within the on-prem network. The pseudonymization service authenticates via a service credential (e.g., AppRole or service token). The audit log streams to a local SIEM.

Configuration requirements:
- KV secrets engine with versioning support
- Service-identity-based auth; one role per service (pseudonymization service, admin)
- Audit device: file or syslog (feed to SIEM for immutable retention)
- Soft-delete / versioning policy to prevent accidental key loss
- HA: 3+ node cluster for production resilience

**Option 2 — Cloud KMS via Private Network**

A cloud KMS deployed with a private endpoint, accessible from on-prem over a dedicated or VPN private link. The pseudonymization service authenticates via a service principal credential. This option can unify key lifecycle management with cloud-side audit integration for any cloud-side key consumers.

Configuration requirements:
- Private endpoint only; no public network access
- Soft delete: enabled (90-day recovery window)
- Purge protection: enabled (meets PIPA audit retention)
- Audit logs forwarded to the cloud audit logging platform (immutable, 90-day minimum)
- One key store per environment (dev, staging, prod)

### Key Hierarchy

```
Key Store (prod) — pseudo-platform-prod
│
├── pseudo-ff1-general-v1         AES-256 key for FF1 encryption of PI-class fields
│   └── pseudo-ff1-general-v2     (after rotation)
│
├── pseudo-ff1-sensitive-v1       AES-256 key for FF1 encryption of SPII-class fields
│   └── pseudo-ff1-sensitive-v2   (after rotation)
│
├── pseudo-hmac-ref-v1            HMAC-SHA-256 signing key for referential (join) keys
│   └── pseudo-hmac-ref-v2        (after rotation)
│
└── pseudo-aes-siv-v1             AES-256 key for AES-SIV encryption of unstructured fields
    └── pseudo-aes-siv-v2         (after rotation)
```

**Naming convention:** `pseudo-{algorithm}-{tier}-v{version}`

**Why separate keys per tier:**
- A cryptographic breach, side-channel attack, or insider threat affecting the general PII key does not compromise sensitive PII (health, financial, government ID) data
- PIPA risk assessment can be scoped per tier: compromise of general PII key is a lower-severity incident than compromise of SPII key
- Access control can be differentiated: a broader set of authorized systems may access the general PII key; the SPII key is restricted to a smaller set of explicitly authorized compute identities


## Access Control Model

| Principal | Key Access | Scope |
|---|---|---|
| On-prem pseudonymization service | All keys (via service credential) | Read key value; latest version for new runs; specific version for re-pseudonymization |
| Data orchestration service identity | No direct key access | Orchestrates the pipeline but does not perform pseudonymization |
| ML compute | No key access | ML compute never retrieves pseudonymization keys; operates on already-pseudonymized data |
| Data engineers (humans) | No key value access in prod | Metadata/policy management only; never retrieve secret values |
| Security admin | Emergency break-glass | Separate procedure; logged; requires approval workflow |
| Dev / staging key store | Dev team service accounts | Full access to non-prod environments only |

**Key principle:** Only the on-prem pseudonymization service ever retrieves key values. Cloud compute (ML, orchestration) has no key access. This is enforced by network boundary (keys accessible from on-prem only) and access control policy.


## Key Rotation Strategy

### Rotation Schedule

| Key | Rotation Frequency | Trigger |
|---|---|---|
| FF1 general (PI-class) | Quarterly | Automated (KMS rotation policy) |
| FF1 sensitive (SPII-class) | Quarterly | Automated + manual confirmation |
| HMAC referential | Quarterly | Automated |
| AES-SIV blob | Quarterly | Automated |

Additional ad-hoc rotation triggers:
- Suspected key compromise (incident response)
- Personnel change in Key Vault admin role
- PIPA audit finding requiring immediate remediation

### Rotation Without Invalidation

Key rotation does not immediately invalidate existing pseudonymized data. The approach:

1. **New key version created:** The KMS generates `v2` of a key; `v1` remains active (not deleted)
2. **New ingestion uses new key:** Pipeline configuration updated to reference latest version (via version-less secret fetch → automatically retrieves latest)
3. **Existing pseudonymized data retains version tag:** Parquet metadata (see [06-technology-stack.md](06-technology-stack.md)) records which key version was used per column
4. **Historical re-pseudonymization (optional):** A background job can re-encrypt historical data with the new key, reading `v1` to decrypt and `v2` to re-encrypt; run asynchronously to avoid production impact
5. **Old version retirement:** After re-pseudonymization of all historical data is confirmed, `v1` is disabled (not deleted — soft delete retained for audit purposes)

This approach ensures continuous availability of pseudonymized data during key rotation with no pipeline downtime.


## Tweak Management for FF1

FF1 supports an optional "tweak" parameter — a domain-specific constant that diversifies the encryption space. Even with the same key, different tweaks produce different ciphertexts for the same plaintext.

**Recommended tweak strategy:**
- Use a per-column tweak derived from the column's semantic type (e.g., `"customer_id"`, `"policy_number"`, `"phone"`)
- This ensures that even if two columns share the same key (e.g., both use the general-tier FF1 key), the same raw value (e.g., the number `10042`) produces different pseudonyms in `customer_id` vs. `claim_id` columns — preventing cross-column correlation attacks
- Tweaks are not secret; they are configuration constants, not key material

Tweak values should be stored in pipeline configuration (not Key Vault) alongside the column classification inventory.


## Key Metadata and Audit Records

For every pseudonymization job run, the following must be recorded in the audit log:

```json
{
  "event_type": "pseudonymization_job",
  "timestamp": "<ISO8601>",
  "pipeline_run_id": "<run ID>",
  "source_dataset": "<table/file name>",
  "row_count": <N>,
  "columns_pseudonymized": [
    {
      "column": "customer_id",
      "technique": "FF1",
      "key_name": "kv-ff1-general",
      "key_version": "v2",
      "tweak": "customer_id"
    },
    ...
  ],
  "status": "success | failure",
  "initiated_by": "<Managed Identity principal ID>"
}
```

This record constitutes the audit trail required by the PIPC guidelines (see [02-regulatory-context.md](02-regulatory-context.md)) and enables PIPA compliance documentation.


## Key Compromise Response

If a pseudonymization key is suspected to be compromised:

1. **Immediate:** Disable the compromised key version in the KMS (not delete — preserves audit trail)
2. **Within 24 hours:** Generate new key version; update pipeline configuration; re-run pseudonymization on all affected datasets with new key
3. **Incident assessment:** Determine which pseudonymized datasets were produced with the compromised key; assess re-identification risk (is the attacker likely to have the key + dataset?)
4. **Notification:** If re-identification risk is non-trivial, escalate to PIPA breach notification evaluation (PIPC notification within 72 hours of confirmed breach determination)
5. **Post-incident:** Root cause analysis; update key management procedures; document in compliance records

The break-glass procedure for key compromise access is maintained separately by the security team and is outside the scope of this proposal.
