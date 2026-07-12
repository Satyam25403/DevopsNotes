# HashiCorp Vault - Auth Methods & Policies

The core access-control theory: how clients prove who they are, and how Vault decides what they're allowed to do.

## Table of Contents
- [Authentication vs Authorization](#authentication-vs-authorization)
- [Token Auth Method](#token-auth-method)
- [AppRole Auth Method](#approle-auth-method)
- [Kubernetes Auth Method](#kubernetes-auth-method)
- [Other Auth Methods](#other-auth-methods)
- [Policies Deep Dive](#policies-deep-dive)
- [Writing Policy Files](#writing-policy-files)
- [Token Lifecycle: TTL, Renewal, Revocation](#token-lifecycle-ttl-renewal-revocation)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Authentication vs Authorization

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│  Authentication            Authorization                  │
│  "Who are you?"             "What are you allowed to do?"   │
├─────────────────────────────────────────────────────┤
│  Handled by: Auth Methods    Handled by: Policies             │
│  (Token, AppRole, K8s, LDAP)  (attached to tokens/identities)   │
└─────────────────────────────────────────────────────┘

Flow:
Client ──authenticates via──→ Auth Method ──→ Vault issues a Token
                                                       ↓
                                            Token has Policies attached
                                                       ↓
                                    Policies determine which paths/actions
                                    are permitted for subsequent requests
```

---

## Token Auth Method

**The most basic method — every successful authentication of any kind ultimately results in a token.**

```bash
# Create a token with a specific policy attached
vault token create -policy="myapp-policy" -ttl=1h
```

**Visual:**
```
Terminal Output:
Key                  Value
---                  -----
token                hvs.CAESIJ...
token_accessor       8g3H2j...
token_duration       1h
token_policies       ["default" "myapp-policy"]

Every token has:
- An ID (the secret itself, used in requests)
- An Accessor (a non-secret reference, used to manage/revoke
  the token WITHOUT knowing the actual token value — useful for
  admins revoking a leaked token without needing to know it)
- Attached Policies
- A TTL (time-to-live)
```

Direct token creation is rarely how real applications authenticate though — that's what **AppRole** and platform-specific methods are for.

---

## AppRole Auth Method

**Designed for machine-to-machine authentication — the standard way CI/CD pipelines and applications authenticate.**

**Visual:**
```
AppRole Concept:
┌─────────────┐         ┌─────────────┐
│  Role ID       │    +    │  Secret ID     │   ───→   Vault Token
│ (like a         │         │ (like a          │
│  username,       │         │  password,        │
│  semi-public,       │         │  should be          │
│  can be in configs)  │         │  protected)           │
└─────────────┘         └─────────────┘
```

**Setup:**
```bash
vault auth enable approle

vault write auth/approle/role/jenkins-ci \
  token_policies="jenkins-policy" \
  token_ttl=1h \
  token_max_ttl=4h

# Get the Role ID (semi-public, safe to store in CI config)
vault read auth/approle/role/jenkins-ci/role-id
# role_id: 12345678-abcd-...

# Generate a Secret ID (sensitive! generated separately)
vault write -f auth/approle/role/jenkins-ci/secret-id
# secret_id: 87654321-dcba-...
```

**Login (what the CI pipeline does at runtime):**
```bash
vault write auth/approle/login \
  role_id="12345678-abcd-..." \
  secret_id="87654321-dcba-..."
```

**Visual:**
```
Why split Role ID and Secret ID?
Role ID    → can live in a CI config file (like a username, low risk alone)
Secret ID  → injected as a protected CI secret/env var at runtime,
             short-lived, can be automatically rotated per pipeline run

Even if Role ID leaks alone, it's useless without the Secret ID.
```

---

## Kubernetes Auth Method

**Lets pods authenticate using their own Kubernetes Service Account token — no separate secret to manage at all.**

**Visual:**
```
┌──────────────┐   Service Account Token   ┌─────────────┐
│  Pod            │ ─────────────────────────→│  Vault          │
│  (has a K8s      │   (automatically mounted    │  (validates      │
│  SA token         │    into every pod by K8s)     │  token against    │
│  mounted by         │                              │  the K8s API)      │
│  default)             │                              └─────────────┘
└──────────────┘
```

**Setup (Vault side):**
```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443"

vault write auth/kubernetes/role/payments-app \
  bound_service_account_names="payments-sa" \
  bound_service_account_namespaces="payments" \
  policies="payments-policy" \
  ttl=1h
```

**Login (from inside the pod, usually done by Vault Agent automatically):**
```bash
vault write auth/kubernetes/login \
  role="payments-app" \
  jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
```

**Visual:**
```
Why this is elegant for Kubernetes-based apps:
No secret to provision, rotate, or leak —
the pod's IDENTITY (its Service Account) IS the credential,
already provided by the platform itself.
```

---

## Other Auth Methods

**Visual:**
```
┌───────────────────────────────────────────────────┐
│ Auth Method        Use Case                            │
├───────────────────────────────────────────────────┤
│ LDAP/Active Directory  Human users logging in with        │
│                       existing corporate credentials        │
│ GitHub                 Developers authenticate with their     │
│                       GitHub org membership/teams               │
│ AWS IAM                 EC2 instances/Lambda authenticate        │
│                       using their AWS IAM identity                │
│ Azure/GCP                Equivalent cloud-native identity          │
│                       methods for Azure VMs / GCP instances          │
│ TLS Certificates          mTLS-based machine authentication            │
│ OIDC/JWT                  Federated identity (Okta, Auth0, etc.)         │
└───────────────────────────────────────────────────┘
```

---

## Policies Deep Dive

**A Policy is a set of rules defining allowed operations on specific paths.**

**Visual:**
```
Policy Structure:
┌─────────────────────────────────────────┐
│  path "secret/data/myapp/*" {              │
│    capabilities = ["read", "list"]           │
│  }                                          │
│                                             │
│  path "database/creds/myapp-role" {          │
│    capabilities = ["read"]                    │
│  }                                            │
└─────────────────────────────────────────┘

Capabilities:
┌────────┬──────────────────────────────────┐
│ create   │ Write a NEW secret at this path      │
│ read     │ Read the secret's value               │
│ update    │ Modify an existing secret               │
│ delete    │ Remove the secret                        │
│ list      │ List secret names at a path (not values)  │
│ sudo      │ Perform privileged/root-protected operations│
│ deny      │ Explicitly block access, overrides other    │
│           │ grants (deny always wins)                     │
└────────┴──────────────────────────────────┘
```

---

## Writing Policy Files

```hcl
# payments-policy.hcl
path "secret/data/payments/*" {
  capabilities = ["read", "list"]
}

path "database/creds/payments-role" {
  capabilities = ["read"]
}

path "secret/data/shared-configs/*" {
  capabilities = ["read"]
}

# Explicitly deny access to another team's secrets,
# even if a broader wildcard elsewhere might otherwise match
path "secret/data/hr/*" {
  capabilities = ["deny"]
}
```

```bash
vault policy write payments-policy payments-policy.hcl
```

**Visual:**
```
Principle of Least Privilege in Action:
┌──────────────┐        ┌─────────────────────┐
│  payments-app   │  ──→   │  Can access:            │
│  (via AppRole    │        │  - secret/payments/*      │
│   or K8s auth)     │        │  - database/creds/         │
│                  │        │    payments-role             │
│                  │        │  Cannot access:                │
│                  │        │  - secret/hr/*                   │
│                  │        │  - any other team's secrets        │
└──────────────┘        └─────────────────────┘
```

---

## Token Lifecycle: TTL, Renewal, Revocation

**Visual:**
```
Token Lifecycle:
Created (TTL=1h) ──→ Used for requests ──→ Approaching expiry
                                                    │
                                     ┌─────────────┴─────────────┐
                                     ↓                             ↓
                          Renewed (if renewable)         Expires naturally
                          extends TTL                    (no longer valid)
                                     │
                                     ↓
                        Can also be manually REVOKED early
                        (e.g. security incident, employee offboarding)
```

**Commands:**
```bash
# Renew a token before it expires
vault token renew hvs.CAESIJ...

# Revoke a token immediately (e.g. compromised)
vault token revoke hvs.CAESIJ...

# Revoke using the accessor (doesn't require knowing the token itself)
vault token revoke -accessor 8g3H2j...
```

**Visual — why the accessor matters:**
```
Scenario: A CI log accidentally printed a token value.
Admin needs to revoke it, but doesn't want to further expose
the leaked token by typing/logging it again.

Using the ACCESSOR (a safe, non-secret reference) to revoke:
vault token revoke -accessor <accessor>
→ Achieves the same result without ever re-handling the secret token.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is setting up secret access for three different consumers: a Jenkins CI pipeline, a Kubernetes-deployed microservice, and a human on-call engineer.

**What they configure:**
1. **Jenkins CI** → **AppRole auth method**. The Role ID is stored in Jenkins' job configuration (low sensitivity), and the Secret ID is injected as a Jenkins Credential, regenerated periodically via a scheduled job — so even if old CI logs leak the Role ID, it's useless without a fresh Secret ID.
2. **Kubernetes microservice** → **Kubernetes auth method**. No secret to manage at all — the pod's Service Account identity, already provisioned by the cluster, is the credential. This eliminates an entire class of "how do we securely inject the Vault credential into the pod" problems.
3. **On-call human engineer** → **LDAP auth method** tied to the company's existing Active Directory, so engineers use their normal corporate login rather than a separate Vault-specific password to remember and rotate.
4. Writes **three separate, narrowly-scoped policies** — `jenkins-policy` (can only read/rotate CI-specific secrets), `payments-app-policy` (can only access the payments team's database credentials), and `oncall-readonly-policy` (read-only access to a break-glass emergency secrets path) — instead of one broad policy shared by everyone.
5. Sets **short TTLs** (1 hour) with auto-renewal for machine identities (Jenkins, K8s pods), and configures alerts if the Root Token is ever used outside of the initial bootstrap window — since ongoing root token usage is a red flag indicating policies aren't properly scoped.

**Why this matters:** Using one shared, long-lived token for "convenience" across different systems is the single most common Vault anti-pattern — it defeats the audit trail, blast-radius containment, and rotation benefits that are the entire point of adopting Vault.

---

Next: **03secrets_engines_practical.md** — hands-on with the KV, Database, and Transit secrets engines: storing static secrets, generating dynamic database credentials, and encryption-as-a-service.