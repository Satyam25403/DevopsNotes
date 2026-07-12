# HashiCorp Vault - Advanced Features & Real-World Use Cases

Going beyond day-to-day secret storage: PKI/certificate management, high availability, auto-unseal, disaster recovery, and how mature organizations actually operate Vault.

## Table of Contents
- [PKI Secrets Engine (Certificate Authority)](#pki-secrets-engine-certificate-authority)
- [High Availability with Raft](#high-availability-with-raft)
- [Auto-Unseal with Cloud KMS](#auto-unseal-with-cloud-kms)
- [Disaster Recovery & Replication](#disaster-recovery--replication)
- [Dynamic Secrets for Cloud Providers (AWS Example)](#dynamic-secrets-for-cloud-providers-aws-example)
- [Audit Logging](#audit-logging)
- [Common Pitfalls & War Stories](#common-pitfalls--war-stories)
- [Real-Life DevOps Use Case (End-to-End)](#real-life-devops-use-case-end-to-end)

---

## PKI Secrets Engine (Certificate Authority)

**Vault can act as an internal Certificate Authority, issuing short-lived TLS certificates on demand.**

**Visual:**
```
Traditional Certificate Management:
Buy/generate a cert → valid for 1-2 years → manually track
expiry → renew manually → forget once → outage 🔥

Vault PKI Engine:
App requests a cert → Vault (acting as CA) issues one
INSTANTLY → valid for hours/days → automatically
regenerated well before expiry by Vault Agent
```

### Setup

```bash
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki

# Generate root CA
vault write -field=certificate pki/root/generate/internal \
  common_name="internal.company.com" \
  ttl=87600h > CA_cert.crt

# Configure issuing URLs
vault write pki/config/urls \
  issuing_certificates="https://vault.internal:8200/v1/pki/ca" \
  crl_distribution_points="https://vault.internal:8200/v1/pki/crl"

# Create a role defining what certs can be issued
vault write pki/roles/internal-services \
  allowed_domains="internal.company.com" \
  allow_subdomains=true \
  max_ttl="72h"
```

### Issuing a Certificate

```bash
vault write pki/issue/internal-services common_name="payments.internal.company.com"
```

**Visual:**
```
Terminal Output:
Key             Value
---             -----
certificate     -----BEGIN CERTIFICATE-----...
private_key     -----BEGIN RSA PRIVATE KEY-----...
ca_chain        -----BEGIN CERTIFICATE-----...
serial_number    3d:2a:...
expiration        2026-07-15T10:00:00Z  (72h from now)

Short-lived certs mean:
- A leaked private key is only useful for a few hours/days
- No manual renewal tracking — Vault Agent auto-renews
- Revocation (via CRL) is less critical since certs expire fast anyway
```

**Visual — mTLS between microservices using PKI:**
```
Service A ──cert issued by Vault PKI──→ mutual TLS handshake ←──cert issued by Vault PKI── Service B

Both services trust the same internal Vault CA,
enabling automatic, short-lived mTLS between all
internal services without manual cert distribution.
```

---

## High Availability with Raft

**Visual:**
```
┌─────────────────────────────────────────────────┐
│               Vault Raft Cluster (3 nodes)          │
│                                                       │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐  │
│   │  Node 1     │   │  Node 2     │   │  Node 3     │  │
│   │  (LEADER)    │←→│ (Follower)   │←→│ (Follower)   │  │
│   └──────────┘   └──────────┘   └──────────┘  │
│                                                       │
│   All writes go through the LEADER, replicated         │
│   to followers via the Raft consensus protocol           │
└─────────────────────────────────────────────────┘

If the leader fails:
Node 1 (LEADER) ✗ crashes
         ↓
Remaining nodes hold a Raft election
         ↓
Node 2 becomes new LEADER automatically
         ↓
Client requests transparently redirect to new leader
(minimal downtime, no manual intervention)
```

**Configuration:**
```hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-1"

  retry_join {
    leader_api_addr = "https://vault-node-2.internal:8200"
  }
  retry_join {
    leader_api_addr = "https://vault-node-3.internal:8200"
  }
}
```

---

## Auto-Unseal with Cloud KMS

**Removes the need for humans to manually enter unseal keys after every restart.**

**Visual:**
```
Manual Unseal (default):
Vault restarts → SEALED → needs 3 of 5 humans to enter
key shares → UNSEALED
(painful for routine restarts/patching/scaling events)

Auto-Unseal via AWS KMS:
Vault restarts → SEALED → Vault automatically calls AWS KMS →
KMS decrypts the master key → UNSEALED
(no human intervention for ROUTINE restarts)
```

**Configuration:**
```hcl
seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "arn:aws:kms:us-east-1:123456789:key/abcd-1234"
}
```

**Visual — important nuance:**
```
Auto-unseal does NOT eliminate the unseal key concept —
it shifts trust to the cloud KMS instead of human key-holders.

Recovery Keys (similar to unseal keys) still exist for
TRUE disaster recovery scenarios (e.g. the KMS key itself
becomes unavailable, or for operations like generating a
new root token) — these are still distributed to trusted
individuals via Shamir's Secret Sharing.
```

---

## Disaster Recovery & Replication

**Visual:**
```
┌─────────────────────┐         ┌─────────────────────┐
│   Primary Cluster        │  ────→  │   DR Replica Cluster    │
│   (us-east-1)              │  (async  │   (us-west-2)             │
│   Handles all reads/         │  replic-│   Standby, read-only,       │
│   writes normally               │  ation) │   ready for failover           │
└─────────────────────┘         └─────────────────────┘

If us-east-1 has a regional outage:
Promote DR replica to primary → applications redirect →
business continues with minimal secrets-management downtime
```

Enterprise-tier feature; smaller setups rely on **Raft's built-in HA within a single region** plus regular **snapshots** for backup:

```bash
vault operator raft snapshot save backup.snap
```

---

## Dynamic Secrets for Cloud Providers (AWS Example)

Beyond databases, Vault can generate temporary **cloud provider credentials** too.

```bash
vault secrets enable aws

vault write aws/config/root \
  access_key=AKIA... \
  secret_key=abc123... \
  region=us-east-1

vault write aws/roles/s3-readonly \
  credential_type=iam_user \
  policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": "*"
  }]
}
EOF
```

```bash
vault read aws/creds/s3-readonly
```

**Visual:**
```
Output:
access_key      AKIAI...
secret_key      wJalrXUt...
security_token  (if using STS)
lease_duration   3600  (1 hour)

Just like database credentials — a brand new,
temporary IAM user/session is created for each request,
and automatically cleaned up after the lease expires.
```

---

## Audit Logging

**Every single request to Vault can be logged — critical for compliance and incident investigation.**

```bash
vault audit enable file file_path=/var/log/vault_audit.log
```

**Visual:**
```
Audit Log Entry (simplified, sensitive values are HMAC-hashed):
{
  "time": "2026-07-12T10:15:00Z",
  "type": "request",
  "auth": { "display_name": "approle-payments-app" },
  "request": {
    "operation": "read",
    "path": "database/creds/payments-role"
  }
}

Even the audit log doesn't store plaintext secrets —
sensitive fields are HMAC-hashed, but the WHO/WHAT/WHEN
of every access is fully traceable.
```

---

## Common Pitfalls & War Stories

**Visual:**
```
Pitfall 1: "Root token used for daily app authentication"
Cause: Convenient during initial setup, never migrated away from
Fix: Root token used ONCE for bootstrap, then revoked;
     real auth via AppRole/Kubernetes/OIDC methods

Pitfall 2: "All 5 unseal keys stored in the same password manager"
Cause: Misunderstanding Shamir's Secret Sharing's purpose
Fix: Distribute keys to genuinely different individuals/systems;
     otherwise a single compromised vault (the password manager!)
     defeats the whole point

Pitfall 3: "Dynamic DB credentials cause connection pool churn"
Cause: TTL set too short (e.g. 5 minutes) for an app that holds
       long-lived DB connections, causing constant reconnects
Fix: Set TTL appropriate to the app's actual connection lifecycle
     (e.g. 1-24 hours), not arbitrarily short

Pitfall 4: "Vault Agent sidecar fails silently, app can't find secrets"
Cause: Missing Kubernetes RBAC permissions for the Vault
       Kubernetes auth method to validate service account tokens
Fix: Verify the auth/kubernetes/config token reviewer permissions
     during initial cluster setup, test failure paths explicitly

Pitfall 5: "Nobody can unseal Vault because a key-holder left the company"
Cause: No key rotation process when personnel changes
Fix: `vault operator rekey` whenever a key-holder leaves,
     documented as a mandatory offboarding step
```

---

## Real-Life DevOps Use Case (End-to-End)

**Scenario:** A financial services company needs Vault to be a core part of their compliance posture (SOC 2, PCI-DSS) — not just a convenient secrets store.

**Full workflow the DevOps/security team builds:**

1. **High availability:** Deploys a 5-node Raft cluster across 3 availability zones, so no single AZ failure takes down secrets access for the entire platform.
2. **Auto-unseal:** Configured via AWS KMS, so routine node restarts during patching don't require paging on-call key-holders — reserving manual Shamir unsealing for genuine disaster recovery.
3. **Zero standing static secrets:** Database credentials, cloud IAM credentials, and internal TLS certificates are ALL dynamically generated with short TTLs (1-24 hours) — there are effectively no long-lived static passwords left in the organization for these systems.
4. **PKI-based internal mTLS:** All internal microservice-to-microservice traffic uses short-lived certs issued by Vault's PKI engine, with Vault Agent handling automatic renewal — eliminating manual certificate expiry incidents entirely.
5. **Full audit trail:** Every secret access streams to the company's SIEM, satisfying compliance auditors who need to answer "who accessed what, when" for any given secret.
6. **Disaster recovery:** A DR replica cluster in a separate region can be promoted within minutes if the primary region has an outage, tested via quarterly failover drills.
7. **Governance discipline:** Root token usage is alerted on immediately (should never happen outside initial bootstrap); unseal/recovery keys are rekeyed automatically as part of the offboarding checklist whenever a key-holder leaves the company.
8. **Continuous policy review:** Quarterly access reviews check that each team's Vault policies still follow least-privilege — a payments-team AppRole that somehow gained read access to another team's path gets flagged and corrected.

**Why this is "real DevOps," not just running a tool:** Vault here isn't just "where we put passwords" — it's the backbone of dynamic, short-lived access across databases, cloud providers, and internal service identity (mTLS), fully auditable, highly available, and disaster-recovery tested. This is the difference between "we installed Vault" and "Vault is how our organization guarantees secrets are never long-lived, never shared, and always traceable."

---

This completes the HashiCorp Vault note series: **Introduction → Setup/Unseal → Auth Methods & Policies → Secrets Engines → CI/CD & App Integration → Advanced/Real-World Usage.**