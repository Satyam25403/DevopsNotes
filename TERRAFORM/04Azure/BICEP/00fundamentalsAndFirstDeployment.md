# Bicep Fundamentals & First Deployment

## What Is Bicep?

Azure has a **native** Infrastructure-as-Code (IaC) language called **ARM (Azure Resource Manager) JSON templates**. ARM JSON is what Azure's control plane actually understands — every `az` CLI command, every Portal click, every Terraform `azurerm` resource eventually gets translated into a JSON payload sent to the ARM REST API.

ARM JSON, however, is verbose, hard to read, and painful to write by hand (no comments, heavy bracket nesting, awkward expressions like `[concat(resourceGroup().location, '-rg')]`). **Bicep** is a **Domain Specific Language (DSL)** built by Microsoft that **transpiles (compiles) down to ARM JSON**. You write clean, declarative Bicep — Azure CLI/PowerShell compiles it to ARM JSON behind the scenes — and that JSON is what actually gets sent to Azure.

```
You write:  main.bicep  (clean, typed, declarative)
         │
         ▼   (az deployment ... automatically runs "bicep build" first)
Compiles to: main.json   (ARM template — what Azure actually understands)
         │
         ▼
   ARM REST API  →  Azure Resource Manager  →  Actual resources created
```

> **Key mental model:** Bicep is NOT a separate provisioning engine like Terraform. It is a **syntax layer on top of native ARM**. There is no separate "Bicep provider," no third-party state file, no separate execution engine — when you run a Bicep deployment, you are talking **directly** to Azure Resource Manager, the same first-party control plane that the Portal and `az` CLI use.

---

## Bicep vs Terraform — Conceptual Comparison

| Concept | Terraform | Bicep |
|---|---|---|
| Language type | Provider-agnostic (multi-cloud) DSL (HCL) | Azure-only DSL |
| Underlying engine | Terraform Core + `azurerm` provider (third-party translation layer) | Native ARM — no translation layer, compiles straight to ARM JSON |
| State tracking | Separate `terraform.tfstate` file (local or remote backend) you must manage | **No separate state file** — ARM itself tracks deployment history server-side (see file 06) |
| Drift detection | Compares `.tf` config against `.tfstate` | Compares `.bicep` (via compiled JSON) against **live Azure resource state directly** (`what-if`) |
| Multi-cloud | ✅ Yes (AWS, GCP, Azure, etc., via different providers) | ❌ No — Azure (and Azure-adjacent: Azure Stack, Azure Arc) only |
| Resource coverage | Depends on provider release cadence (slight lag from native Azure features) | Always in sync with ARM — new Azure features available immediately |
| File extension | `.tf` | `.bicep` |

> Why learn Bicep if you already know Terraform? Because for **Azure-only shops**, Bicep removes an entire layer of moving parts (no state file to lose, no provider version pinning, no remote backend to bootstrap) while staying perfectly in sync with whatever Azure ships. Many enterprises standardize on Bicep specifically for this reason.

---

## Installing Bicep

Bicep ships as an extension of the Azure CLI. If you have `az` CLI installed, you almost certainly already have Bicep, or can install it in one command.

```bash
# Install (or upgrade) the Bicep CLI — bundled and managed via az
az bicep install
az bicep upgrade

# Verify installation and check version
az bicep version
```

> **VS Code Tip:** Install the official **"Bicep" extension** by Microsoft in VS Code. It gives you IntelliSense, auto-formatting, inline documentation on hover, and real-time linting — genuinely the difference between fighting the syntax and flying through it. This is the de facto standard editor for Bicep authoring.

---

## Authenticating to Azure

Just like Terraform's `azurerm` provider, you need to authenticate before you can deploy anything. Bicep deployments run **through the Azure CLI or Azure PowerShell** — there is no separate Bicep-specific authentication mechanism, because Bicep has no provider of its own.

```bash
# Interactive browser-based login (good for local development)
az login

# Login using a Service Principal (the production-grade approach —
# same principle as Terraform: never use a personal/root account for automation)
az login --service-principal \
  --username <app-id> \
  --password <client-secret> \
  --tenant <tenant-id>

# If you have multiple subscriptions, select the one you want to target
az account set --subscription "<subscription-id>"

# Confirm which subscription/tenant you are currently authenticated against
az account show
```

> **Production pattern:** In CI/CD pipelines (GitHub Actions, Azure DevOps), you authenticate using a Service Principal or, even better, **Workload Identity Federation (OIDC)** — which lets the pipeline authenticate to Azure without storing any long-lived secret at all. This is the direct equivalent of how Terraform recommends `ARM_*` environment variables over hardcoded credentials in `terraform.tfvars`.

---

## Anatomy of a Bicep File

A `.bicep` file is built from a small set of top-level declarations. Here they all are, at a glance, before we use them:

```
targetScope   →  declares what level this file deploys to (resourceGroup, subscription, etc.)
param         →  input value (equivalent to Terraform's "variable")
var           →  computed/internal value (equivalent to Terraform's "local")
resource      →  an actual Azure resource to create/manage
module        →  a reference to another .bicep file (equivalent to Terraform's "module")
output        →  a value exposed after deployment (equivalent to Terraform's "output")
```

There is no `provider` block (Azure is the only target, so there's nothing to configure), and no `terraform { }`-style settings block. This is one of Bicep's biggest simplifications over Terraform.

---

## Your First Resource: Resource Group + Storage Account

### `main.bicep`

```bicep
// targetScope tells Azure: "this template deploys resources INTO a resource group"
// This is the default if omitted, but stating it explicitly is good practice.
targetScope = 'resourceGroup'

// ─── Resource Group ──────────────────────────────────────────────────────────
// NOTE: You normally CANNOT create a Resource Group from within a
// resourceGroup-scoped deployment, because you must already be "inside" one.
// To create a Resource Group itself, you deploy at SUBSCRIPTION scope instead.
// We'll cover that properly in file 02 (scopes). For now, assume the RG
// "example-resource" already exists, and we deploy resources INTO it.

// ─── Storage Account ──────────────────────────────────────────────────────────
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  // "storageAccount" is the Bicep SYMBOLIC NAME — used internally within this
  // file so other resources/outputs can reference it.
  // It is NOT the actual Azure resource name.
  // Syntax to reference it: <symbolicName>.<property>
  // Example: storageAccount.name, storageAccount.id, storageAccount.properties.primaryEndpoints

  // 'Microsoft.Storage/storageAccounts' → the Azure Resource Provider + Resource Type
  // '@2023-01-01'                        → the ARM API version for this resource type
  //   (API versions matter a lot in Bicep — different versions expose different
  //    properties. Always check the latest stable version in Microsoft's docs.)

  name     = 'mystorageacc12345'
  // Real Azure name. Same uniqueness rules as Terraform:
  // - lowercase letters and numbers only
  // - no spaces, hyphens, or underscores
  // - 3 to 24 characters
  // - must be globally unique across ALL of Azure

  location = resourceGroup().location
  // resourceGroup() is a built-in FUNCTION that returns an object describing
  // the resource group this deployment is targeting — including .location,
  // .name, .id, .tags. Using it here means the storage account automatically
  // inherits the same region as its resource group — no hardcoding.

  sku: {
    name: 'Standard_GRS'
    // Combines what Terraform splits into two fields (account_tier +
    // account_replication_type) into a single SKU name:
    //   Standard_LRS, Standard_GRS, Standard_ZRS, Premium_LRS, etc.
  }

  kind: 'StorageV2'
  // The storage account "kind" — StorageV2 is the modern general-purpose v2 kind,
  // supporting blobs, files, queues, tables, and tiering (Hot/Cool/Archive).

  tags: {
    environment: 'staging'
  }
}

// ─── Output ───────────────────────────────────────────────────────────────────
output storageAccountName string = storageAccount.name
output storageAccountId string = storageAccount.id
```

### Key Syntax Differences from Terraform — Side by Side

| | Terraform (HCL) | Bicep |
|---|---|---|
| Block separator | `=` then `{ }` | `=` then `{ }` (same!) |
| Assignment inside block | `key = value` | `key: value` (colon, not equals) — **but** top-level `resource`/`param`/`var`/`output` declarations use `=` |
| Resource declaration | `resource "type" "name" { }` | `resource name 'type@apiVersion' = { }` |
| String quotes | Double `"..."` only | Single `'...'` (idiomatic) — double quotes also work but single is the convention |
| Comments | `#` or `//` | `//` or `/* */` (no `#`) |
| Implicit dependency | `azurerm_x.y.attr` | `symbolicName.property` (identical concept) |

> Notice: inside the curly braces of a `resource` block, Bicep uses `key: value` (colon), mirroring JSON — because that's literally what it compiles to. This trips up almost everyone coming from Terraform's `key = value` HCL style. Get this muscle memory early.

---

## Deploying Your Bicep File

Unlike Terraform's `init` → `plan` → `apply` → `destroy` lifecycle, Bicep deployment is driven entirely through Azure CLI (or PowerShell) commands, because — again — there is no separate Bicep engine. The CLI compiles your `.bicep` to JSON, then calls the ARM `deployments` REST API.

```bash
# Step 1: (Optional but recommended) Compile/build the Bicep file to ARM JSON
# without deploying anything — useful to sanity check the compiled output
# or to hand the JSON off to someone who needs raw ARM templates.
az bicep build --file main.bicep
# → produces main.json in the same directory

# Step 2: Lint / validate the Bicep syntax and semantics
az bicep build --file main.bicep --stdout > /dev/null
# (a failed build = validation failure; there's no separate "validate" subcommand
#  the way Terraform has "terraform validate" — build IS the validation)

# Step 3: Preview what WILL happen — Bicep's equivalent of "terraform plan"
az deployment group what-if \
  --resource-group example-resource \
  --template-file main.bicep

# Step 4: Actually deploy
az deployment group create \
  --resource-group example-resource \
  --template-file main.bicep \
  --name my-first-deployment
  # --name gives this specific deployment a tracked identifier —
  # visible later in "Deployments" under the Resource Group in the Portal
```

> Every `az deployment ... create` run is recorded by ARM as a numbered **deployment** under that scope (visible in Portal → Resource Group → Deployments). This deployment history is part of how Bicep achieves state-tracking-like behavior *without* a state file — covered fully in file 06.

```bash
# Clean up — there's no "bicep destroy". You either:
# (a) delete the resource group itself (destroys everything inside it), or
# (b) delete individual resources via az CLI / Portal, or
# (c) use a Deployment Stack with "deny-delete" / "delete" action (file 06)
az group delete --name example-resource --yes --no-wait
```

---

## Why There's No `terraform init` Equivalent

Terraform's `init` step exists to **download provider plugins** — third-party binaries that translate HCL into provider-specific API calls. Bicep has **no providers to download**, because the Bicep CLI itself already knows how to compile directly to ARM JSON, and ARM JSON is natively understood by Azure. This is the single biggest structural simplification Bicep offers over Terraform — there's nothing to initialize, nothing to version-pin at the provider level, and nothing that can drift out of sync between your local plugin cache and what's deployed.

The only thing resembling provider versioning in Bicep is the **API version** suffix on each resource type (`@2023-01-01`), which you specify *per resource*, inline, every time.

---

## Quick Glossary Before You Continue

| Term | Meaning |
|---|---|
| ARM | Azure Resource Manager — Azure's control plane / REST API that all tools (Portal, CLI, Bicep, Terraform) ultimately talk to |
| ARM Template | The native JSON IaC format that Bicep compiles into |
| Resource Provider | The Azure service namespace owning a resource type, e.g. `Microsoft.Storage`, `Microsoft.Network`, `Microsoft.Compute` |
| API Version | A dated version string (`@2023-01-01`) pinning which version of a resource provider's schema you're using |
| Symbolic Name | The internal Bicep-only identifier for a resource/module (like Terraform's local resource name) |
| Deployment | A single tracked `az deployment ... create` execution, recorded server-side by ARM |
| Scope | The level at which a deployment happens: resource group, subscription, management group, or tenant (file 02) |

The next file builds on this exact storage account example to introduce **parameters, variables, and Bicep's full type system** — the direct equivalent of Terraform's input/output/local variables and type constraints, but adapted to Bicep's decorator-based validation model.