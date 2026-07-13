# Bicep File Structure, Deployment Scopes & Dependencies

## The Monolithic `main.bicep` (Starting Point)

Just like Terraform's single sprawling `main.tf`, it's common to start Bicep learning with everything crammed into one file:

```bicep
targetScope = 'resourceGroup'

@description('The environment type')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'staging'

@description('Allowed Azure regions')
param allowedLocations array = [
  'westeurope'
  'northeurope'
  'eastus'
]

var commonTags = {
  environment: environment
  lob: 'banking'
  stage: 'alpha'
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : 'mystorageacc12345'
  location : allowedLocations[2]
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
  tags: commonTags
}

output storageAccountName string = storageAccount.name
```

This works for a demo. It does not work for a real project. The fix is the same philosophy as Terraform: **split by purpose**, not by resource type alone.

---

## Recommended File Structure

```
bicep-project/
├── main.bicep            # Orchestration: declares scope, calls modules, top-level resources
├── main.bicepparam        # Parameter VALUES for a given environment (⚠️ never commit secrets)
├── modules/
│   ├── storageAccount.bicep
│   ├── network.bicep
│   └── virtualMachine.bicep
├── .bicep/                # (no such directory actually exists — see note below)
```

> **Important structural difference from Terraform:** Bicep has **no `provider.tf`-style file** (no provider to configure), **no `backend.tf`-style file** (no separate state backend to configure — file 06 explains why), and **no auto-generated hidden directories** like `.terraform/` or a `terraform.tfstate` file sitting in your project. A Bicep project, at rest, is just your `.bicep` source files — nothing else gets generated locally unless you explicitly run `az bicep build` to produce a `.json` file.

A more realistic structure separates parameters per environment and groups resources into modules (modules are covered in depth in file 03):

```
bicep-project/
├── main.bicep
├── params/
│   ├── dev.bicepparam
│   ├── staging.bicepparam
│   └── prod.bicepparam
├── modules/
│   ├── storageAccount.bicep
│   ├── network.bicep
│   └── virtualMachine.bicep
└── README.md
```

```bash
# Deploy the dev environment
az deployment group create \
  --resource-group dev-resources \
  --template-file main.bicep \
  --parameters params/dev.bicepparam

# Deploy production — same template, different parameter file
az deployment group create \
  --resource-group prod-resources \
  --template-file main.bicep \
  --parameters params/prod.bicepparam
```

> This mirrors Terraform's pattern of one `.tfvars` per environment (`dev.tfvars`, `prod.tfvars`) reused against the same `.tf` configuration — same philosophy, different file extension.

---

## Deployment Scopes — A Concept Terraform Doesn't Have

This is the biggest *new* idea in this file. Terraform's `azurerm` provider always operates within whatever subscription/credentials context you authenticated with — there's no equivalent "scope" keyword in HCL itself. Bicep, because it talks **directly** to ARM, must explicitly declare **which level of the Azure resource hierarchy** a deployment targets — because ARM itself organizes everything into a strict hierarchy:

```
Tenant (Azure AD / Entra ID boundary)
   │
   ▼
Management Group(s)        →  organizes multiple subscriptions
   │
   ▼
Subscription(s)             →  billing + access boundary
   │
   ▼
Resource Group(s)            →  logical container for related resources
   │
   ▼
Resource(s)                  →  the actual VM, storage account, etc.
```

Every Bicep file declares its `targetScope` at the top — and that determines which built-in functions are available, which resource types can be deployed, and what `az deployment <scope> create` subcommand you must use.

### The Four Scopes

```bicep
targetScope = 'resourceGroup'      // default if omitted — most common
targetScope = 'subscription'       // can create resource groups, subscription-level policies/RBAC
targetScope = 'managementGroup'    // can create subscriptions (with right permissions), MG-level policy
targetScope = 'tenant'              // rare — top-of-hierarchy operations, e.g. tenant-wide policy
```

| Scope | Can deploy | CLI command | Common use case |
|---|---|---|---|
| `resourceGroup` (default) | Any resource inside a resource group | `az deployment group create` | 95% of day-to-day work — storage accounts, VMs, networking |
| `subscription` | Resource groups themselves, subscription-level RBAC/policy | `az deployment sub create` | Bootstrapping new resource groups, subscription policy assignment |
| `managementGroup` | Subscriptions, management-group-level policy | `az deployment mg create` | Platform/landing-zone teams provisioning whole subscriptions |
| `tenant` | Tenant-wide policy and management group structure | `az deployment tenant create` | Rare — enterprise governance teams only |

### Example: Creating a Resource Group Itself (Subscription Scope)

Recall from file 00 that you **cannot** create a Resource Group from a `resourceGroup`-scoped deployment — because that scope assumes you're already inside one. To create the Resource Group itself, you deploy at `subscription` scope:

```bicep
targetScope = 'subscription'

@description('Name of the resource group to create')
param resourceGroupName string = 'example-resource'

@description('Azure region for the resource group')
param location string = 'westeurope'

resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name     : resourceGroupName
  location : location
  tags: {
    environment: 'staging'
  }
}

output resourceGroupId string = rg.id
```

```bash
# Note: subscription-scoped deployments require --location (where deployment
# METADATA is stored — not where resources go; resources go where YOU specify)
az deployment sub create \
  --location westeurope \
  --template-file main.bicep
```

### Deploying Resources *Into* a Newly Created Resource Group, in One Template

A subscription-scoped template can create the resource group **and** deploy resources into it, by nesting a **module** (covered fully in file 03) with its own `scope` property:

```bicep
targetScope = 'subscription'

param resourceGroupName string = 'example-resource'
param location string = 'westeurope'

resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name     : resourceGroupName
  location : location
}

// A module deploying INTO the resource group we just created above.
// "scope" tells this module: "run as if you were resourceGroup-scoped,
// targeting THIS specific resource group" — even though the parent
// template itself is subscription-scoped.
module storage 'modules/storageAccount.bicep' = {
  name  : 'storageDeployment'
  scope : rg   // <-- scope override; this is how cross-scope deployment works
  params: {
    storageAccountName: 'mystorageacc12345'
    location: location
  }
}
```

> The `scope` property on a `module` block is how Bicep crosses scope boundaries within a single deployment — a subscription-scoped parent orchestrating a resource-group-scoped child. There is no equivalent concept in Terraform, because Terraform's provider model treats "where" something is created as just another resource argument (`resource_group_name = ...`), not as a structural property of the deployment itself.

---

## File Load Order — Simpler Than Terraform

Terraform loads every `.tf` file in a directory alphabetically and merges them into one logical configuration — which is *why* file-splitting works at all (`provider.tf`, `variables.tf`, `main.tf` all just get concatenated conceptually).

**Bicep does not work this way.** Each `.bicep` file is an **independent, self-contained template**. There is no implicit merging of multiple `.bicep` files sitting in the same directory. If you want to split logic across files, you do it **explicitly**, via the `module` keyword (file 03) — never by just dropping multiple `.bicep` files next to each other and hoping Bicep notices.

```
Terraform:  *.tf files in a folder  →  implicitly merged into ONE configuration
Bicep:      *.bicep files            →  each is independent; combine via explicit "module" calls only
```

This is a fundamental mental model shift: in Terraform, splitting `main.tf` into `provider.tf` + `variables.tf` + `main.tf` changes *nothing* about execution — it's purely organizational. In Bicep, splitting `main.bicep` into `main.bicep` + `modules/storageAccount.bicep` **does** change execution — it creates an actual nested deployment (file 03 covers exactly what that means and why it matters).

---

## Implicit Dependencies — Identical Philosophy to Terraform

Just like Terraform, Bicep automatically detects a dependency whenever one resource references a **symbolic name** of another. No extra syntax required.

```bicep
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name     : 'example-resource'
  location : 'westeurope'
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : 'mystorageacc12345'
  location : rg.location     // implicit dependency — Bicep waits for "rg" first
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
}
```

> Wait — didn't we just say you can't create a resource group AND a storage account in the same `resourceGroup`-scoped file? Correct. This particular example only works at `subscription` scope (or by treating `rg` as a `module` with its own scope, as shown above). The implicit-dependency *mechanism* itself, though, works identically regardless of scope: reference a symbolic name's property, and Bicep orders the deployment correctly, automatically.

Because `storageAccount` references `rg.location`, the compiled ARM JSON automatically includes a `dependsOn` entry pointing at the resource group — you never had to type `dependsOn` yourself. This is **exactly** the same philosophy as Terraform's implicit dependency graph from attribute references.

---

## Explicit Dependencies (`dependsOn`)

When two resources are related but share no attribute reference Bicep can detect, declare the relationship manually with `dependsOn` — Bicep's direct equivalent of Terraform's `depends_on` meta-argument.

```bicep
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name     : 'example-resource'
  location : 'westeurope'
}

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name      : guid(rg.id, 'storage-contributor')   // guid() generates a deterministic unique name
  scope     : rg.id
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '17d1049b-9a84-46fb-8f53-869881c3d3ab')
    principalId: '00000000-0000-0000-0000-000000000000'
  }
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : 'mystorageacc12345'
  location : rg.location
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'

  // No property above references "roleAssignment" directly,
  // so Bicep would not know to wait for it without this explicit declaration.
  dependsOn: [
    roleAssignment
  ]
}
```

> ⚠️ Same advice as Terraform: **avoid `dependsOn` whenever an implicit reference can express the same relationship.** It makes the dependency graph harder to read at a glance and is easy to leave stale (pointing at a resource that no longer matters) as code evolves. Reach for it only when no attribute reference exists to encode the relationship naturally.

---

## Summary: Implicit vs Explicit (Same Table, Bicep Syntax)

| | Implicit | Explicit (`dependsOn`) |
|---|---|---|
| How declared | Reference a symbolic name's property (`rg.location`, `storageAccount.id`) | `dependsOn: [ symbolicName1, symbolicName2 ]` |
| Bicep detects automatically? | ✅ Yes | ❌ No — you must declare it |
| Compiles to | `dependsOn` array in the ARM JSON (generated for you) | Same `dependsOn` array, but written by hand |
| When to use | Whenever a resource uses another's property | When resources are related but share no direct property reference |
| Recommended? | ✅ Always prefer this | ⚠️ Only as a last resort |

---

## Cross-Scope Dependency Note

One subtlety with no Terraform parallel: a resource at `resourceGroup` scope **cannot** implicitly or explicitly depend on something at a *different* scope (e.g., a subscription-level resource) unless that dependency is threaded through a `module` boundary with an explicit `scope` property, as shown earlier. Dependencies only flow naturally **within the same scope** — crossing scope boundaries always requires going through a `module`.

The next file is the natural continuation of this scope-crossing pattern: **Modules** — Bicep's mechanism for splitting deployments across files, reusing logic, and crossing scope boundaries deliberately, the direct (if more powerful) equivalent of Terraform's `module` blocks.