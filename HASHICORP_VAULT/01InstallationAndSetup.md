# HashiCorp Vault - Installation & Setup

Getting Vault running in dev mode for learning, and the real production initialization/unseal flow.

## Table of Contents
- [Installing Vault](#installing-vault)
- [Dev Mode Server](#dev-mode-server)
- [Storing Your First Secret (Dev Mode)](#storing-your-first-secret-dev-mode)
- [Production Server Configuration](#production-server-configuration)
- [Initializing Vault](#initializing-vault)
- [Unsealing Vault](#unsealing-vault)
- [Logging In](#logging-in)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Installing Vault

### Linux (apt)

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault
```

### Mac (Homebrew)

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

### Docker

```bash
docker run --cap-add=IPC_LOCK -d --name=vault-dev -p 8200:8200 hashicorp/vault
```

**Verify:**
```bash
vault version
# Vault v1.16.0
```

---

## Dev Mode Server

**Dev mode runs an in-memory, auto-unsealed Vault server — great for learning, never for production.**

```bash
vault server -dev
```

**Visual:**
```
Terminal Output:
┌───────────────────────────────────────────────┐
│  Root Token: hvs.AbCdEf123456                    │  ← use this to log in
│  Unseal Key: xY9z...                              │  ← auto-unsealed already
│                                                    │
│  WARNING! dev mode is enabled!                     │
│  In this mode, Vault is initialized and unsealed    │
│  automatically. Do not run this in production.        │
└───────────────────────────────────────────────┘

Dev Mode Characteristics:
┌───────────────────────────────┐
│  ✗ In-memory storage             │  ← data lost on restart
│  ✗ Auto-unsealed                  │  ← no real seal/unseal process
│  ✗ Single unseal key (not split)   │  ← no Shamir's Secret Sharing
│  ✗ HTTP (no TLS)                    │  ← insecure by default
│  ✓ Fast to start, zero config        │  ← perfect for local testing
└───────────────────────────────┘
```

**In a new terminal, set the address and token:**
```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='hvs.AbCdEf123456'
```

---

## Storing Your First Secret (Dev Mode)

```bash
# Enable the KV secrets engine (v2) at path "secret/" (often enabled by default in dev mode)
vault secrets enable -path=secret kv-v2

# Write a secret
vault kv put secret/myapp/config username="admin" password="s3cr3t"
```

**Visual:**
```
Terminal Output:
======= Secret Path =======
secret/data/myapp/config

======= Metadata =======
Key                Value
---                -----
created_time       2026-07-12T10:00:00Z
version            1
```

**Read it back:**
```bash
vault kv get secret/myapp/config
```

**Visual:**
```
====== Data ======
Key         Value
---         -----
password    s3cr3t
username    admin
```

**Visual — the request flow:**
```
CLI command → HTTP API call → Vault Server
                                    ↓
                        Barrier encrypts the data
                                    ↓
                        Stored in backend (in-memory, dev mode)
```

---

## Production Server Configuration

Real deployments use a config file, not `-dev`.

```hcl
# config.hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-1"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_cert_file = "/opt/vault/tls/cert.pem"
  tls_key_file  = "/opt/vault/tls/key.pem"
}

api_addr = "https://vault-node-1.internal:8200"
cluster_addr = "https://vault-node-1.internal:8201"
ui = true
```

```bash
vault server -config=config.hcl
```

**Visual:**
```
Production vs Dev Differences:
┌───────────────────┬─────────────────┬─────────────────────┐
│ Aspect               │ Dev Mode           │ Production              │
├───────────────────┼─────────────────┼─────────────────────┤
│ Storage              │ In-memory           │ Raft/Consul (persistent)│
│ TLS                  │ None (HTTP)          │ Required (HTTPS)          │
│ Seal state            │ Auto-unsealed         │ Sealed, needs manual/     │
│                       │                       │ auto-unseal process         │
│ Unseal keys            │ Single, not split      │ Shamir's Secret Sharing     │
│                       │                       │ (multiple key holders)       │
└───────────────────┴─────────────────┴─────────────────────┘
```

---

## Initializing Vault

**Initialization happens exactly ONCE per Vault cluster** — it generates the unseal keys and the initial root token.

```bash
vault operator init
```

**Visual:**
```
Terminal Output:
┌───────────────────────────────────────────────┐
│ Unseal Key 1: aBc123...                          │
│ Unseal Key 2: dEf456...                          │
│ Unseal Key 3: gHi789...                          │
│ Unseal Key 4: jKl012...                          │
│ Unseal Key 5: mNo345...                          │
│                                                    │
│ Initial Root Token: hvs.RootTokenHere              │
│                                                    │
│ Vault initialized with 5 key shares and a          │
│ key threshold of 3. Please securely distribute      │
│ the key shares.                                      │
└───────────────────────────────────────────────┘
```

⚠️ **Critical practice:** these 5 unseal keys should go to **5 different trusted individuals** (e.g., different team leads), never all saved in one place. The root token should be used once for initial setup, then revoked in favor of scoped tokens/auth methods.

---

## Unsealing Vault

Immediately after init (or after any restart), Vault is **sealed** and must be unsealed with the threshold number of keys.

```bash
vault operator unseal aBc123...
# Unseal Progress: 1/3

vault operator unseal dEf456...
# Unseal Progress: 2/3

vault operator unseal gHi789...
# Unseal Progress: 3/3 → Sealed: false
```

**Visual:**
```
Unseal Progress Bar:
[1/3] ██████░░░░░░░░░░░  Sealed: true
[2/3] ████████████░░░░  Sealed: true
[3/3] ██████████████████  Sealed: false ✓ Vault is now UNSEALED

After a server restart, this process must repeat —
Vault does NOT remember it was unsealed before.
(unless auto-unseal via cloud KMS is configured — file 05)
```

---

## Logging In

```bash
vault login hvs.RootTokenHere
```

**Visual:**
```
Terminal Output:
Success! You are now authenticated. The token information
displayed below is already stored in the token helper. You
do NOT need to run "vault login" again.

Key                  Value
---                  -----
token                hvs.RootTokenHere
token_policies       ["root"]
```

⚠️ **Best practice:** the root token is meant for initial bootstrapping only. Real usage should immediately set up proper **Auth Methods and Policies** (file 02) and revoke/avoid using the root token day-to-day.

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is standing up a new production Vault cluster for the first time.

**What they do:**
1. Deploys Vault with the **Raft storage backend** across 3 nodes for high availability (not dev mode, not a single node).
2. Runs `vault operator init` exactly once, and immediately **distributes the 5 unseal key shares to 5 different senior engineers/leads** across different teams — ensuring no single person can unseal Vault alone.
3. Configures **auto-unseal via AWS KMS** so that routine restarts (deployments, patching) don't require paging 3 different people at 2 AM just to manually unseal the cluster — reserving manual Shamir unsealing for true disaster-recovery scenarios where the KMS key itself is unavailable.
4. Uses the **root token exactly once** to configure the Kubernetes auth method and initial policies, then **revokes the root token** and stores a sealed, offline copy in a physical safe for emergency-only use.
5. Documents the unseal-key holders and rotation procedure in the team's runbook, since a departing key-holder means that key share must be rotated (via `vault operator rekey`).

**Why this matters:** Mishandling the initialization step — e.g., storing all 5 unseal keys in the same password manager, or leaving the root token active for daily operations — defeats the entire security model Vault was chosen for in the first place.

---

Next: **02auth_methods_and_policies.md** — how identities authenticate to Vault, and how policies control what they can access.