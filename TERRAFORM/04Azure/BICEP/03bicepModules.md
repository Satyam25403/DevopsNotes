# Bicep Modules

A **module** in Bicep is a reference to another `.bicep` (or published registry) file, deployed as a **nested deployment**. This is conceptually identical to Terraform's `module` blocks, but because Bicep has no implicit multi-file merging (file 02), modules are the **only** way to split logic across files in Bicep — there's no equivalent of Terraform's "just drop more `.tf` files in the same folder."

```
Terraform:  module "name" { source = "./path" ... }
Bicep:      module name 'path/to/file.bicep' = { ... }
```

---

## Why Use Modules?

- **Reusability** — write a storage account module once, call it for dev/staging/prod with different parameters
- **Encapsulation** — hide internal complexity behind a clean parameter interface
- **Crossing scopes deliberately** — as seen in file 02, a module can target a different scope than its parent (subscription parent → resource-group-scoped module)
- **Team boundaries** — networking team owns `modules/network.bicep`, app team owns `modules/app.bicep`, both consumed from a shared `main.bicep`
- **Testability** — a module is a complete, independently deployable unit; you can `az deployment group create` directly against a module file for isolated testing

---

## Anatomy of a Module File

A module file is just... a normal Bicep file. There is no special syntax inside the file itself that marks it as "a module" — any `.bicep` file can be deployed standalone OR referenced as a module from another file. What makes it "a module" is purely how it's *consumed*.

### `modules/storageAccount.bicep`

```bicep
@description('Name of the storage account')
@minLength(3)
@maxLength(24)
param storageAccountName string

@description('Azure region for the storage account')
param location string

@description('Replication SKU')
@allowed([
  'Standard_LRS'
  'Standard_GRS'
  'Standard_ZRS'
])
param replicationType string = 'Standard_GRS'

@description('Tags to apply')
param tags object = {}

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : storageAccountName
  location : location
  sku: {
    name: replicationType
  }
  kind: 'StorageV2'
  tags: tags
}

// Outputs are how a module "returns" data to whatever called it
output storageAccountId string = storageAccount.id
output storageAccountName string = storageAccount.name
output primaryBlobEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

Notice: this file has **no `targetScope` declaration**, so it defaults to `resourceGroup`. It's a completely ordinary, self-contained Bicep template — you could deploy it directly:

```bash
# Deploying the module file STANDALONE, with no parent — perfectly valid
az deployment group create \
  --resource-group example-resource \
  --template-file modules/storageAccount.bicep \
  --parameters storageAccountName=mystorageacc12345 location=westeurope
```

---

## Calling a Module From a Parent File

### `main.bicep`

```bicep
targetScope = 'resourceGroup'

@description('Environment name')
param environment string = 'staging'

@description('Azure region')
param location string = resourceGroup().location

var commonTags = {
  environment: environment
  lob: 'banking'
}

// ── Module Declaration ────────────────────────────────────────────────────
module storage 'modules/storageAccount.bicep' = {
  // "storage" is the symbolic name for THIS MODULE CALL — used to reference
  // its outputs elsewhere in main.bicep. It is NOT the module file's name.

  name : 'storageDeployment'
  // REQUIRED. This becomes the name of the nested ARM deployment, visible
  // in the Portal under Resource Group → Deployments, as a child of the
  // parent deployment. Equivalent in spirit to Terraform showing module
  // resources nested under "module.storage.azurerm_storage_account...".

  params: {
    // These map directly onto the param declarations inside storageAccount.bicep.
    // Bicep validates these against the module's actual parameter types
    // AT COMPILE TIME — a typo'd parameter name, or wrong type, fails before
    // any deployment is attempted. This is stronger guarantee than Terraform
    // gives you when passing variables into a module block.
    storageAccountName : 'mystorageacc12345'
    location            : location
    replicationType      : environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'
    tags                 : commonTags
  }
}

// ── Consuming the Module's Outputs ───────────────────────────────────────
// Syntax: <moduleSymbolicName>.outputs.<outputName>
output storageAccountName string = storage.outputs.storageAccountName
output blobEndpoint string = storage.outputs.primaryBlobEndpoint
```

### Side-by-Side With Terraform

```hcl
module "storage" {
  source              = "./modules/storage_account"
  storage_account_name = "mystorageacc12345"
  location             = var.location
  replication_type     = var.environment == "prod" ? "Standard_GRS" : "Standard_LRS"
  tags                  = local.common_tags
}

output "storage_account_name" {
  value = module.storage.storage_account_name
}
```

| | Terraform | Bicep |
|---|---|---|
| Declaration | `module "name" { source = "./path" }` | `module name 'path/file.bicep' = { }` |
| Passing values in | Top-level keys directly in the block | Nested under a `params: { }` object |
| Reading outputs | `module.<name>.<output>` | `<symbolicName>.outputs.<output>` |
| Source location | Local path, Git URL, or Terraform Registry | Local path, or **Bicep/ARM Template Registry** (private OCI-compliant registry) |

---

## Module Source Locations

### Local Path (Most Common)

```bicep
module storage 'modules/storageAccount.bicep' = { ... }
module storage '../shared-modules/storageAccount.bicep' = { ... }
```

### Public/Private Bicep Registry (Equivalent to Terraform Registry Modules)

Bicep supports publishing modules to a container registry (Azure Container Registry, configured as a Bicep module registry) and consuming them by reference — directly analogous to pulling a module from the public Terraform Registry or a private module registry.

```bicep
module storage 'br:myregistry.azurecr.io/bicep/modules/storage-account:v1.2.0' = {
  name: 'storageDeployment'
  params: {
    storageAccountName: 'mystorageacc12345'
    location: location
  }
}
```

```bash
# Publishing a module to a registry — the "push" half of this workflow
az bicep publish \
  --file modules/storageAccount.bicep \
  --target br:myregistry.azurecr.io/bicep/modules/storage-account:v1.2.0
```

### Public Bicep Registry — Microsoft-Maintained Modules

Microsoft maintains a curated set of **verified modules** (the "Azure Verified Modules," AVM, initiative) at `br/public:`, directly comparable to verified modules on the public Terraform Registry:

```bicep
module storage 'br/public:avm/res/storage/storage-account:0.9.1' = {
  name: 'storageDeployment'
  params: {
    name: 'mystorageacc12345'
    location: location
  }
}
```

---

## Nested Deployments — What's Actually Happening

This is conceptually important and has no clean Terraform parallel. When you call a `module`, Bicep does **not** simply inline the module's resources into the parent's compiled JSON (though older/simpler cases can be optimized this way under the hood). Instead, it typically compiles the module into its own **nested ARM deployment** — a child deployment object, embedded inside the parent template, that ARM executes as a logically separate (but coordinated) deployment.

```
az deployment group create (main.bicep)
   │
   ├── Deployment: "main" (parent)
   │      │
   │      └── Deployment: "storageDeployment" (the module's nested deployment)
   │             │
   │             └── Resource: Microsoft.Storage/storageAccounts/mystorageacc12345
```

This is *why* a module needs its own `name` property (`name: 'storageDeployment'`) — that name identifies the **nested deployment record** itself, separately trackable in the Portal's Deployments view, separately visible in deployment history (file 06), and separately referenceable if you need to inspect what that specific nested deployment did.

> **Practical implication:** if a module deployment fails partway through, you can inspect *that specific nested deployment's* error directly — Azure doesn't just give you one giant opaque failure for the whole `main.bicep` run. This granular, per-module deployment tracking is a structural benefit of how Bicep modules compile, distinct from Terraform where a module's resources are simply part of the single flat state file and plan output.

---

## Looping Over Modules

Modules support the same `for` iteration mechanics covered fully in file 04 — worth a preview here since "deploy N copies of this module" is an extremely common real-world need:

```bicep
param storageAccountNames array = [
  'satyamstorage01'
  'shivamstorage02'
]

module storageAccounts 'modules/storageAccount.bicep' = [for name in storageAccountNames: {
  name : 'storageDeployment-${name}'
  params: {
    storageAccountName : name
    location            : location
  }
}]

// Accessing outputs from a LOOPED module requires array indexing
output firstAccountId string = storageAccounts[0].outputs.storageAccountId
```

File 04 covers the full `for` expression syntax (on resources, modules, properties, and outputs) in depth — this is just enough to show that modules participate in the same looping mechanism as any other resource.

---

## Passing an Existing Resource Into a Module (Cross-File Reference)

Sometimes a module needs to know about a resource that already exists, created outside the current deployment entirely (e.g., a shared Key Vault managed by a platform team). Use the `existing` keyword — Bicep's equivalent of Terraform's **data source** concept, scoped specifically to "a resource I'm not creating, just referencing."

```bicep
// Reference an existing, already-deployed Key Vault — Bicep will NOT
// attempt to create or modify it; this is purely a read-only reference,
// exactly like a Terraform "data" block.
resource existingKeyVault 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: 'shared-platform-kv'
}

module storage 'modules/storageAccount.bicep' = {
  name: 'storageDeployment'
  params: {
    storageAccountName: 'mystorageacc12345'
    location: location
    // Pass a property of the existing resource straight into the module
    keyVaultId: existingKeyVault.id
  }
}
```

```hcl
# Terraform equivalent — the "data" block
data "azurerm_key_vault" "existing" {
  name                = "shared-platform-kv"
  resource_group_name = "platform-rg"
}

module "storage" {
  source       = "./modules/storage_account"
  key_vault_id = data.azurerm_key_vault.existing.id
}
```

> **Key distinction:** Terraform has a separate `data` block keyword entirely. Bicep instead reuses the *same* `resource` keyword, just appending the `existing` modifier at the end of the declaration. Same underlying concept — "read, don't manage" — expressed as a variant of the normal resource syntax rather than a wholly separate block type.

---

## Module Best Practices — Quick Reference

| Practice | Why |
|---|---|
| One module = one logical unit (e.g., "a complete storage layer," not "one storage account property") | Mirrors good Terraform module design — cohesive, not arbitrarily fine-grained |
| Always give modules a `@description` on every parameter | Same discipline as top-level params — modules are often consumed by people who didn't write them |
| Prefer `existing` over re-declaring shared infrastructure in every module | Avoids accidental duplicate management of the same resource from multiple templates |
| Use registry-published modules (`br:` or `br/public:avm/...`) for anything reused across multiple repos/teams | Avoids copy-pasted, drifting module code across projects — same motivation as using the public Terraform Registry |
| Name module deployments (`name: '...'`) meaningfully and uniquely | These names show up in deployment history (file 06) — meaningful names make debugging dramatically easier |

The next file covers **loops (`for` expressions) and conditional deployment** — Bicep's equivalent of Terraform's `count` and `for_each` meta-arguments, applied uniformly across resources, modules, properties, and outputs.