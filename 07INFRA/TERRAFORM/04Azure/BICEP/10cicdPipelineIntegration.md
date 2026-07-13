# CI/CD Pipeline Integration for Bicep

This file covers wiring Bicep into automated pipelines — GitHub Actions and Azure DevOps — including authentication patterns, the PR-review-then-merge-deploy workflow (`what-if` on PR, `create` on merge, directly analogous to `terraform plan`/`apply` pipelines), environment promotion, and approval gates.

```
Concepts covered:
  OIDC / Workload Identity Federation     →  passwordless pipeline authentication
  GitHub Actions workflow                   →  full example, PR validation + merge deployment
  Azure DevOps pipeline                       →  full example, using the AzureCLI/AzureResourceManagerTemplateDeployment tasks
  Environment promotion (dev → staging → prod)  →  parameter-file-per-environment pattern in a pipeline
  Approval gates                                  →  manual sign-off before production deployment
```

---

## Authentication in Pipelines: OIDC / Workload Identity Federation

Both file 00 and the Terraform set's storage-account lesson stress avoiding long-lived credentials. In CI/CD specifically, the modern best practice — for **both** Bicep and Terraform pipelines — is **OIDC (OpenID Connect) federation**: the pipeline proves its identity to Azure using a short-lived, automatically-rotated token issued by the CI platform itself (GitHub/Azure DevOps), with **no client secret stored anywhere, ever**.

```
Traditional approach:  Service Principal + CLIENT SECRET stored as a pipeline secret
                        (long-lived, must be rotated manually, can leak)

OIDC approach:          Service Principal + FEDERATED CREDENTIAL trust relationship
                        (no secret exists at all — GitHub/Azure DevOps mints a
                         short-lived token per pipeline run, Azure trusts it
                         because of a pre-configured trust relationship)
```

### One-Time Setup: Federated Credential on the Service Principal

```bash
# Create the Service Principal (or use an existing one)
az ad sp create-for-rbac --name bicep-cicd-sp --skip-assignment

# Assign it the minimum role it needs (principle of least privilege —
# same principle stated explicitly in file 00)
az role assignment create \
  --assignee <sp-app-id> \
  --role Contributor \
  --scope /subscriptions/<subscription-id>/resourceGroups/example-resource

# Configure the federated credential — this trusts a SPECIFIC GitHub repo +
# branch (or Azure DevOps service connection) to authenticate AS this SP,
# without ever needing a client secret
az ad app federated-credential create \
  --id <sp-app-id> \
  --parameters '{
    "name": "github-main-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

> This setup step is **identical in spirit** for a Terraform pipeline — OIDC federation isn't a Bicep-specific feature, it's an Azure AD / Entra ID capability that any tool authenticating via Azure CLI or the ARM API can take advantage of. The only Bicep-specific part of this whole file is everything *after* authentication succeeds.

---

## GitHub Actions — Full Example

### `.github/workflows/bicep-deploy.yml`

```yaml
name: Bicep Deploy

on:
  pull_request:
    branches: [main]
    paths: ['infra/**']
  push:
    branches: [main]
    paths: ['infra/**']

permissions:
  id-token: write   # REQUIRED for OIDC — lets GitHub mint the federated token
  contents: read

jobs:
  validate-and-whatif:
    # Runs on EVERY pull request — the equivalent of "terraform plan" on PR
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC — no secret used)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          # Note: no client-secret input at all — this is the OIDC flow

      - name: Bicep Build (compile-time validation)
        run: az bicep build --file infra/main.bicep

      - name: What-If
        run: |
          az deployment group what-if \
            --resource-group example-resource \
            --template-file infra/main.bicep \
            --parameters infra/params/dev.bicepparam

  deploy:
    # Runs only on merge to main — the equivalent of "terraform apply" on merge
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production   # ties into GitHub's native Environments feature —
                                # this is where approval gates are configured (see below)
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy
        run: |
          az deployment group create \
            --resource-group example-resource \
            --template-file infra/main.bicep \
            --parameters infra/params/prod.bicepparam \
            --name "deploy-${{ github.sha }}"
            # Naming the deployment after the commit SHA makes it trivially
            # traceable back to the exact code change in ARM's deployment
            # history (file 06) — extremely useful during incident review
```

### Side-by-Side: Terraform's Equivalent Pipeline Shape

```yaml
# Terraform's PR job would look almost identical, just swapping CLI commands:
- name: Terraform Init
  run: terraform init
- name: Terraform Plan
  run: terraform plan -var-file=dev.tfvars

# And the merge job swaps in:
- name: Terraform Apply
  run: terraform apply -auto-approve -var-file=prod.tfvars
```

> The overall **pipeline shape** — validate/plan on PR, deploy on merge, gated by environment — is identical between Bicep and Terraform pipelines. The only real differences are: (1) Bicep needs no `init` step at all (no providers to download, per file 00), and (2) Bicep has no separate persisted plan artifact to pass between jobs the way Terraform's `terraform plan -out=tfplan` artifact is sometimes uploaded and later consumed by `terraform apply tfplan` for guaranteed plan/apply consistency — `what-if` is always a fresh, throwaway check (file 06), never an artifact you feed into the next step.

---

## Azure DevOps Pipeline — Full Example

### `azure-pipelines.yml`

```yaml
trigger:
  branches:
    include: [main]
  paths:
    include: ['infra/*']

pr:
  branches:
    include: [main]
  paths:
    include: ['infra/*']

pool:
  vmImage: ubuntu-latest

stages:
  - stage: ValidateAndWhatIf
    condition: eq(variables['Build.Reason'], 'PullRequest')
    jobs:
      - job: WhatIf
        steps:
          - task: AzureCLI@2
            inputs:
              azureSubscription: 'bicep-cicd-service-connection'   # configured via
                                                                      # Workload Identity Federation
                                                                      # in the Azure DevOps UI —
                                                                      # same OIDC principle as GitHub
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                az bicep build --file infra/main.bicep
                az deployment group what-if \
                  --resource-group example-resource \
                  --template-file infra/main.bicep \
                  --parameters infra/params/dev.bicepparam

  - stage: DeployProduction
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Deploy
        environment: 'production'   # Azure DevOps Environments — approval gates
                                       # configured here, same role as GitHub's
                                       # "environment:" key above
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'bicep-cicd-service-connection'
                    scriptType: bash
                    scriptLocation: inlineScript
                    inlineScript: |
                      az deployment group create \
                        --resource-group example-resource \
                        --template-file infra/main.bicep \
                        --parameters infra/params/prod.bicepparam \
                        --name "deploy-$(Build.SourceVersion)"
```

> Azure DevOps also offers a dedicated **`AzureResourceManagerTemplateDeployment@3`** task purpose-built for ARM/Bicep deployment (as an alternative to running raw `az` commands inside `AzureCLI@2`). Either approach works; the raw CLI approach shown above is generally preferred for consistency with local developer commands and easier debugging, since the exact same `az deployment group create` command can be copy-pasted and run locally to reproduce a pipeline failure.

---

## Environment Promotion: One Template, Many Parameter Files

Just like the Terraform set's pattern of one `.tf` configuration reused against `dev.tfvars`/`staging.tfvars`/`prod.tfvars`, the idiomatic Bicep CI/CD pattern is **one `main.bicep`, promoted unchanged through environments**, varying only the `.bicepparam` file consumed at each stage:

```
infra/
├── main.bicep
└── params/
    ├── dev.bicepparam
    ├── staging.bicepparam
    └── prod.bicepparam
```

```bicep
// params/prod.bicepparam
using '../main.bicep'

param environment = 'prod'
param storageAccountName = 'stprod${uniqueString('prod-seed')}'
param vmConfig = {
  size: 'Standard_DS3_v2'
  publisher: 'Canonical'
  offer: '0001-com-ubuntu-server-jammy'
  sku: '22_04-lts'
  version: 'latest'
}
```

A multi-stage pipeline can chain dev → staging → prod deployments sequentially, each stage depending on the previous one's success — promoting the *exact same compiled template* through environments is what gives you confidence that what was validated in staging is truly what reaches production, with only intentional, explicit parameter differences between them.

```yaml
# Azure DevOps — sequential stage dependency, enforcing promotion order
stages:
  - stage: DeployDev
    jobs: [ ... uses params/dev.bicepparam ... ]

  - stage: DeployStaging
    dependsOn: DeployDev
    condition: succeeded()
    jobs: [ ... uses params/staging.bicepparam ... ]

  - stage: DeployProd
    dependsOn: DeployStaging
    condition: succeeded()
    jobs: [ ... uses params/prod.bicepparam, gated by environment approval ... ]
```

---

## Approval Gates — Manual Sign-Off Before Production

Both platforms support pausing a pipeline before a sensitive stage until a designated human approves it — the automated equivalent of someone manually reviewing `terraform plan` output before typing `yes` at the `terraform apply` confirmation prompt.

- **GitHub Actions:** configure under repo **Settings → Environments → production → Required reviewers**. The `environment: production` key in the workflow YAML (shown above) is what ties a job to that protection rule.
- **Azure DevOps:** configure under **Pipelines → Environments → production → Approvals and checks**. The `environment: 'production'` key in the YAML (shown above) is the equivalent binding.

> Pair approval gates with a `what-if` step that **runs again immediately before the gate**, and have the approver actually read that output before clicking approve — otherwise the gate becomes a rubber stamp rather than a genuine safety check. This mirrors the same discipline good Terraform teams apply: never auto-approve production applies without a human having actually looked at the plan diff.

---

## CI/CD — Quick Reference Comparison

| Concern | Terraform Pipeline | Bicep Pipeline |
|---|---|---|
| Authentication | Service Principal (OIDC preferred) via `ARM_*` env vars | Service Principal (OIDC preferred) via `azure/login` action or service connection |
| Init step | `terraform init` (mandatory — downloads providers, configures backend) | None needed (file 00) |
| PR check | `terraform plan` | `az deployment ... what-if` |
| Merge deploy | `terraform apply` | `az deployment ... create` |
| Plan artifact passed between jobs | Common pattern: `-out=tfplan`, uploaded/downloaded as a build artifact | Not applicable — `what-if` is always freshly re-run, never persisted/consumed downstream |
| Environment promotion | Same `.tf`, different `.tfvars` per stage | Same `.bicep`, different `.bicepparam` per stage |
| Approval gates | Native to CI platform (GitHub Environments / ADO Environments) — not a Terraform-specific feature | Identical — native to CI platform, not a Bicep-specific feature |

The next (and final, for this batch) file is the **capstone**: a complete multi-tier application stack — Resource Group → VNet → Subnets → NSG → Storage Account → Key Vault → Virtual Machine — woven together using every concept from files 00–10: scopes, modules, loops, conditions, expressions, nested resources, and deployment-history-aware outputs.