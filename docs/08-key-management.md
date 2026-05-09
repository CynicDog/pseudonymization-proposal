# 08 — Key Management

## Design Principles

- **Keys never co-locate with data.** Key material and pseudonymized data are stored in separate Azure services with separate access controls.
- **Managed Identity only.** No human ever possesses production encryption keys. All compute (ADF, Databricks, Azure Functions) accesses keys via Azure Managed Identity.
- **Separate key namespaces by classification tier.** A key compromise affecting general PII must not affect sensitive PII. Keys are isolated per tier.
- **Key versioning, not invalidation.** Rotation creates a new key version; existing pseudonymized data retains a reference to the version used. Historical data remains readable without mass re-pseudonymization.
- **All key access is audited.** Every key retrieval is logged immutably in Azure Monitor / Key Vault audit logs.

---

## Azure Key Vault Structure

One Key Vault per environment (dev, staging, prod). Key Vaults are deployed with:
- **Purview mode:** `Enable for Azure Purview` (allows Purview to scan and classify secrets metadata)
- **Soft delete:** enabled (90-day recovery window)
- **Purge protection:** enabled (prevents permanent deletion for 90 days; meets PIPA audit retention)
- **Network:** private endpoint only; no public access
- **Diagnostic settings:** Key Vault audit logs → Log Analytics Workspace (immutable, 90-day retention minimum)

### Key Hierarchy

```
Key Vault (prod) — kv-data-platform-prod
│
├── kv-ff1-general-v1         AES-256 key for FF1 encryption of PI-class fields
│   └── kv-ff1-general-v2     (after Q1 rotation)
│
├── kv-ff1-sensitive-v1       AES-256 key for FF1 encryption of SPII-class fields
│   └── kv-ff1-sensitive-v2   (after Q1 rotation)
│
├── kv-hmac-ref-v1            HMAC-SHA-256 signing key for referential (join) keys
│   └── kv-hmac-ref-v2        (after Q1 rotation)
│
└── kv-aes-siv-v1             AES-256 key for AES-SIV encryption of unstructured fields
    └── kv-aes-siv-v2         (after Q1 rotation)
```

**Naming convention:** `kv-{algorithm}-{tier}-v{version}`

**Why separate keys per tier:**
- A cryptographic breach, side-channel attack, or insider threat affecting the general PII key does not compromise sensitive PII (health, financial, government ID) data
- PIPA risk assessment can be scoped per tier: compromise of general PII key is a lower-severity incident than compromise of SPII key
- Access control can be differentiated: a broader set of authorized systems may access the general PII key; the SPII key is restricted to a smaller set of explicitly authorized compute identities

---

## Access Control Model

| Principal | Key Access | Scope |
|---|---|---|
| ADF Managed Identity (prod) | `kv-ff1-general-*`, `kv-hmac-ref-*` | Get secret, latest version only |
| Pseudonymization Fn / Databricks MI (prod) | All keys | Get secret, specific version (for re-pseudonymization) |
| Databricks ML cluster MI (prod) | No key access | ML compute never retrieves pseudonymization keys |
| Data engineers (humans) | No key access in prod | Key Vault RBAC: reader on metadata only, not secret value |
| Security admin | Emergency break-glass | Separate break-glass procedure; logged; requires approval |
| Dev / staging Key Vault | Dev team | Full access to dev/staging Key Vault only |

**Key principle:** ML training and serving compute (Databricks ML clusters, inference endpoints) **never** have access to pseudonymization keys. They consume pseudonymized data and produce predictions; they are not capable of reversing pseudonymization.

---

## Key Rotation Strategy

### Rotation Schedule

| Key | Rotation Frequency | Trigger |
|---|---|---|
| FF1 general (PI-class) | Quarterly | Automated (Azure Key Vault rotation policy) |
| FF1 sensitive (SPII-class) | Quarterly | Automated + manual confirmation |
| HMAC referential | Quarterly | Automated |
| AES-SIV blob | Quarterly | Automated |

Additional ad-hoc rotation triggers:
- Suspected key compromise (incident response)
- Personnel change in Key Vault admin role
- PIPA audit finding requiring immediate remediation

### Rotation Without Invalidation

Key rotation does not immediately invalidate existing pseudonymized data. The approach:

1. **New key version created:** Azure Key Vault generates `v2` of a key; `v1` remains active (not deleted)
2. **New ingestion uses new key:** ADF pipeline configuration updated to reference latest version (via `GetSecret` without version pin → automatically retrieves latest)
3. **Existing pseudonymized data retains version tag:** Parquet metadata (see [06-technology-stack.md](06-technology-stack.md)) records which key version was used per column
4. **Historical re-pseudonymization (optional):** A background job can re-encrypt historical data with the new key, reading `v1` to decrypt and `v2` to re-encrypt; run asynchronously to avoid production impact
5. **Old version retirement:** After re-pseudonymization of all historical data is confirmed, `v1` is disabled (not deleted — soft delete retained for audit purposes)

This approach ensures continuous availability of pseudonymized data during key rotation with no pipeline downtime.

---

## Tweak Management for FF1

FF1 supports an optional "tweak" parameter — a domain-specific constant that diversifies the encryption space. Even with the same key, different tweaks produce different ciphertexts for the same plaintext.

**Recommended tweak strategy:**
- Use a per-column tweak derived from the column's semantic type (e.g., `"customer_id"`, `"policy_number"`, `"phone"`)
- This ensures that even if two columns share the same key (e.g., both use the general-tier FF1 key), the same raw value (e.g., the number `10042`) produces different pseudonyms in `customer_id` vs. `claim_id` columns — preventing cross-column correlation attacks
- Tweaks are not secret; they are configuration constants, not key material

Tweak values should be stored in pipeline configuration (not Key Vault) alongside the column classification inventory.

---

## Key Metadata and Audit Records

For every pseudonymization job run, the following must be recorded in the audit log (Azure Monitor):

```json
{
  "event_type": "pseudonymization_job",
  "timestamp": "<ISO8601>",
  "pipeline_run_id": "<ADF run ID>",
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

---

## Key Compromise Response

If a pseudonymization key is suspected to be compromised:

1. **Immediate:** Disable the compromised key version in Key Vault (not delete — preserves audit trail)
2. **Within 24 hours:** Generate new key version; update pipeline configuration; re-run pseudonymization on all affected datasets with new key
3. **Incident assessment:** Determine which pseudonymized datasets were produced with the compromised key; assess re-identification risk (is the attacker likely to have the key + dataset?)
4. **Notification:** If re-identification risk is non-trivial, escalate to PIPA breach notification evaluation (PIPC notification within 72 hours of confirmed breach determination)
5. **Post-incident:** Root cause analysis; update key management procedures; document in compliance records

The break-glass procedure for key compromise access is maintained separately by the security team and is outside the scope of this proposal.
