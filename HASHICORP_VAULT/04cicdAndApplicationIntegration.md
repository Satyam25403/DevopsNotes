# HashiCorp Vault - CI/CD & Application Integration

Getting secrets from Vault into real running applications and pipelines, without hardcoding tokens or manually copy-pasting values.

## Table of Contents
- [The Integration Problem](#the-integration-problem)
- [Vault Agent](#vault-agent)
- [Vault Agent Injector (Kubernetes Sidecar)](#vault-agent-injector-kubernetes-sidecar)
- [Jenkins Integration](#jenkins-integration)
- [GitHub Actions Integration](#github-actions-integration)
- [GitLab CI Integration](#gitlab-ci-integration)
- [Terraform + Vault](#terraform--vault)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## The Integration Problem

**Visual:**
```
Naive Approach (still a secret-sprawl problem):
CI Pipeline: vault login → gets token → passes token as env var →
             App reads token from env → App calls Vault directly

Problem: The Vault TOKEN itself is now a secret that needs
protecting, injecting, and rotating — you've just moved the
problem, not solved it.

Better Approach: Auth methods bound to PLATFORM IDENTITY
(AppRole for CI, Kubernetes auth for pods) — the credential
used to get INTO Vault is itself short-lived, scoped, and
tied to something the platform already manages for you.
```

---

## Vault Agent

**A helper process that handles authentication and secret retrieval automatically, so the application itself never talks to Vault's API directly.**

**Visual:**
```
Without Vault Agent:
App code must:
  1. Know how to authenticate (AppRole logic embedded in app)
  2. Handle token renewal
  3. Handle secret retrieval and caching
  4. Handle re-auth if the token expires
  → Every app reimplements this logic, inconsistently

With Vault Agent:
┌─────────────┐         ┌──────────────┐         ┌──────────┐
│  Vault Agent   │  ────→  │  Renders secret │  ────→  │  App reads  │
│  (handles auth, │         │  to a local FILE │         │  a plain      │
│  renewal,        │         │  (e.g. .env or     │         │  file — no     │
│  caching)          │         │   config.json)       │         │  Vault-aware    │
└─────────────┘         └──────────────┘         │  code needed)  │
                                                    └──────────┘
```

### Vault Agent Config

```hcl
# agent-config.hcl
auto_auth {
  method "approle" {
    config = {
      role_id_file_path   = "/etc/vault/role-id"
      secret_id_file_path = "/etc/vault/secret-id"
    }
  }

  sink "file" {
    config = {
      path = "/etc/vault/token"
    }
  }
}

template {
  source      = "/etc/vault/db-creds.tpl"
  destination = "/etc/app/db-creds.json"
}

vault {
  address = "https://vault.internal:8200"
}
```

**Template file (`db-creds.tpl`):**
```
{{ with secret "database/creds/myapp-role" }}
{
  "username": "{{ .Data.username }}",
  "password": "{{ .Data.password }}"
}
{{ end }}
```

**Visual:**
```
Vault Agent Lifecycle:
Starts → Authenticates via AppRole → Fetches secret →
   Writes rendered file → Watches lease → Auto-renews/re-fetches
   BEFORE expiry → Re-writes the file with fresh credentials

App just reads /etc/app/db-creds.json — it has NO idea
Vault even exists. This decouples app code from secrets
management entirely.
```

---

## Vault Agent Injector (Kubernetes Sidecar)

**On Kubernetes, an admission webhook automatically injects a Vault Agent sidecar container into pods with the right annotations.**

**Visual:**
```
Pod Definition (with annotations):
┌────────────────────────────────────┐
│  apiVersion: v1                        │
│  kind: Pod                              │
│  metadata:                                │
│    annotations:                             │
│      vault.hashicorp.com/agent-inject: "true"│
│      vault.hashicorp.com/role: "payments-app"  │
│      vault.hashicorp.com/agent-inject-secret-  │
│        db-creds: "database/creds/payments-role" │
└────────────────────────────────────┘
                    ↓
         Admission Webhook intercepts pod creation
                    ↓
┌────────────────────────────────────┐
│  Pod (after injection)                  │
│  ┌──────────────┐  ┌───────────────┐  │
│  │  App Container   │  │  Vault Agent     │  │
│  │  (reads secret     │  │  Sidecar           │  │
│  │  from shared         │  │  (fetches, writes,   │  │
│  │  volume mount)          │  │   renews secret)       │  │
│  └──────────────┘  └───────────────┘  │
│         shared emptyDir volume            │
└────────────────────────────────────┘
```

**Why this is powerful:** Application teams add a few annotations to their pod spec — they don't write any Vault client code, don't manage tokens, and don't handle renewal logic. The platform team manages the injector once, centrally, for the whole cluster.

---

## Jenkins Integration

```groovy
pipeline {
    agent any
    environment {
        VAULT_ADDR = 'https://vault.internal:8200'
    }
    stages {
        stage('Fetch Secrets') {
            steps {
                withVault(
                    configuration: [vaultUrl: env.VAULT_ADDR, vaultCredentialId: 'vault-approle'],
                    vaultSecrets: [[
                        path: 'secret/data/myapp/deploy',
                        secretValues: [
                            [envVar: 'DEPLOY_API_KEY', vaultKey: 'api_key']
                        ]
                    ]]
                ) {
                    sh 'echo Deploying with fetched key...'
                    sh './deploy.sh'
                }
            }
        }
    }
}
```

**Visual:**
```
Jenkins HashiCorp Vault Plugin Flow:
Pipeline stage starts → Plugin authenticates via AppRole
(credential stored in Jenkins Credential Store, itself just
 a Role ID/Secret ID pair) → Fetches secret → Injects as env
var ONLY within the withVault block → Automatically scrubbed
from logs and unavailable outside that block
```

---

## GitHub Actions Integration

```yaml
name: Deploy
on: push

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Import Secrets from Vault
        uses: hashicorp/vault-action@v3
        id: secrets
        with:
          url: https://vault.internal:8200
          method: jwt
          role: github-actions-role
          secrets: |
            secret/data/myapp/deploy api_key | DEPLOY_API_KEY

      - name: Deploy
        run: ./deploy.sh
        env:
          DEPLOY_API_KEY: ${{ steps.secrets.outputs.DEPLOY_API_KEY }}
```

**Visual:**
```
Note the auth method: "jwt"
GitHub Actions can present its own OIDC token to Vault,
which validates it against GitHub's OIDC provider —
NO static secret needs to be stored in GitHub Secrets at all
for the Vault authentication step itself.

This is the modern, most secure pattern: identity federation
instead of a long-lived stored credential.
```

---

## GitLab CI Integration

```yaml
deploy:
  stage: deploy
  image: hashicorp/vault:1.16
  id_tokens:
    VAULT_ID_TOKEN:
      aud: https://vault.internal:8200
  script:
    - export VAULT_TOKEN="$(vault write -field=token auth/jwt/login role=gitlab-ci-role jwt=$VAULT_ID_TOKEN)"
    - export DEPLOY_API_KEY="$(vault kv get -field=api_key secret/myapp/deploy)"
    - ./deploy.sh
```

**Visual:**
```
GitLab CI/CD ID Tokens + Vault JWT auth:
Same OIDC federation pattern as GitHub Actions —
GitLab issues a short-lived, job-specific JWT token,
Vault validates it, issues a Vault token scoped to
that specific pipeline run only.
```

---

## Terraform + Vault

Terraform itself can both **read secrets from** and **write secrets to** Vault as part of infrastructure provisioning.

```hcl
data "vault_generic_secret" "db_creds" {
  path = "database/creds/myapp-role"
}

resource "aws_db_instance" "example" {
  # ...
  password = data.vault_generic_secret.db_creds.data["password"]
}
```

**Visual:**
```
Why this matters:
Terraform state files can contain sensitive values in plaintext
if secrets are passed as plain variables. Pulling directly from
Vault at apply-time means the secret's ORIGIN is Vault, and
access to it is still governed by Vault's policies/audit log —
though note Terraform state itself may still cache the value,
so state file encryption/access control remains important.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is migrating a company's CI/CD pipelines away from long-lived static tokens stored in GitHub Secrets/Jenkins Credentials, toward Vault-brokered dynamic access.

**What they build:**
1. For **GitHub Actions**, configures Vault's **JWT auth method** using GitHub's OIDC provider, so pipelines authenticate using a token GitHub generates fresh for each job run — eliminating a long-lived Vault token stored as a GitHub Secret that could otherwise leak and remain valid indefinitely.
2. For **Kubernetes workloads**, rolls out the **Vault Agent Injector** cluster-wide, so application teams only add pod annotations — the platform team handles Vault authentication and secret rendering centrally, without each team writing custom Vault client code.
3. For a **legacy on-prem Jenkins setup** that can't easily use OIDC federation, configures **AppRole** with a Secret ID that's regenerated by a scheduled Jenkins job every 24 hours, limiting the exposure window if that Secret ID were ever logged accidentally.
4. Sets up **Vault audit logging** to a centralized SIEM, so security can see exactly which pipeline/pod fetched which secret and when — closing a visibility gap that existed when secrets lived in scattered CI credential stores with inconsistent logging.
5. Runs a **deprecation timeline**: old static Jenkins credentials are set to expire in 90 days, forcing every team to migrate to the new Vault-based flow rather than letting the old and new systems coexist indefinitely.

**Why this matters:** The real security win isn't just "we use Vault now" — it's replacing every long-lived, manually-managed secret with a short-lived credential brokered through an identity the platform already manages (GitHub's OIDC token, Kubernetes' service account, etc.), which is a fundamentally different risk profile.

---

Next: **05advanced_realworld_usecases.md** — PKI/certificate issuance, high availability with Raft, auto-unseal, and mature real-world Vault operating practices.