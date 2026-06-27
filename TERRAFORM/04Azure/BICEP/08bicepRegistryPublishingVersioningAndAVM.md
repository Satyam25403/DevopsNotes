# The Bicep Registry: Publishing, Versioning & Azure Verified Modules

File 03 briefly introduced consuming modules from a registry (`br:` and `br/public:` references). This file covers the **full publishing workflow** — setting up your own private registry, the actual publish/consume cycle end-to-end, versioning conventions, and a deep look at the **Azure Verified Modules (AVM)** initiative, which is Bicep's answer to the curated, community-vetted modules you'd find on the public Terraform Registry.

```
Concepts covered:
  Why a registry at all                    →  the reuse problem, same motivation as Terraform Registry
  OCI artifacts                              →  what a Bicep module actually IS once published
  Setting up a private registry (ACR)         →  the only piece of new infrastructure required
  The publish/consume cycle                    →  az bicep publish, br: references, version pinning
  Module aliasing via bicepconfig.json           →  shortening long registry references
  Azure Verified Modules (AVM)                     →  the public, Microsoft-curated equivalent
```

---

## Why a Registry? The Same Problem Terraform Solves Differently

Recall from file 03: modules are how you achieve reuse in Bicep. But a **local path** module reference (`module storage 'modules/storageAccount.bicep' = {...}`) only works within a single repository — it can't be shared across teams, repos, or organizations without copy-pasting the file. Terraform solves this with the public Terraform Registry (and private registries) hosting versioned, pullable modules by name. Bicep solves the identical problem with the **Bicep Registry** — conceptually the same idea, but built on a different, very deliberate piece of existing infrastructure: **OCI (Open Container Initiative) artifact registries** — the same technology that stores Docker container images.

> **Why piggyback on container registries specifically?** Because Azure Container Registry (ACR) already existed, was already battle-tested for versioned artifact storage with access control, and already supported the OCI artifact spec generally (not just container images). Rather than building an entirely new artifact storage system from scratch the way HashiCorp built a bespoke Terraform Registry, Microsoft simply taught the Bicep CLI to push/pull Bicep files **as OCI artifacts** into infrastructure that already existed. A published Bicep module isn't stored in some Bicep-specific format — it's stored as a generic OCI artifact, the same primitive Docker images use.

---

## Setting Up a Private Registry (Azure Container Registry)

Unlike Terraform, where the public Registry is ready to use immediately and a private one requires a paid Terraform Cloud/Enterprise tier (or self-hosted alternatives), Bicep's private registry option is **just a standard ACR instance** — cheap, fully self-managed, no special "module registry" SKU or feature flag required.

```bash
# Create a resource group to hold the registry (do this once, like the
# Terraform set's tfstate bootstrap storage account)
az group create --name platform-shared-rg --location westeurope

# Create the Azure Container Registry — Basic SKU is sufficient for module storage
az acr create \
  --resource-group platform-shared-rg \
  --name myorgbicepmodules \
  --sku Basic

# Authenticate the Bicep/Azure CLI against this registry before publishing
az acr login --name myorgbicepmodules
```

> Just like the Terraform set's state-storage bootstrap script (file 01 of that set), this registry must be created **once, manually or via a separate bootstrap pipeline**, before any module can be published to it — you cannot have a Bicep template provision the very registry that template's own modules will later be pulled from. This is the same "chicken and egg" principle as remote state backend provisioning.

---

## Publishing a Module

```bicep
// modules/storageAccount.bicep — same file from file 03, now ready to publish

@description('Name of the storage account')
@minLength(3)
@maxLength(24)
param storageAccountName string

@description('Azure region for the storage account')
param location string

@description('Replication SKU')
@allowed(['Standard_LRS', 'Standard_GRS', 'Standard_ZRS'])
param replicationType string = 'Standard_GRS'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : storageAccountName
  location : location
  sku: { name: replicationType }
  kind: 'StorageV2'
}

output storageAccountId string = storageAccount.id
output storageAccountName string = storageAccount.name
```

```bash
# Publish this file to the registry as a versioned artifact
az bicep publish \
  --file modules/storageAccount.bicep \
  --target br:myorgbicepmodules.azurecr.io/bicep/modules/storage-account:v1.0.0
```

```
br:                                       → the registry reference scheme ("Bicep Registry")
myorgbicepmodules.azurecr.io               → the registry's login server (just an ACR hostname)
bicep/modules/storage-account               → the artifact's repository path WITHIN the registry
                                              (entirely your own convention — organize however you like)
:v1.0.0                                      → the version tag
```

> Note the structural similarity to a Docker image reference (`myregistry.azurecr.io/myapp:v1.0.0`) — this is not a coincidence; it's the same underlying OCI tagging scheme, because it genuinely is the same kind of artifact storage mechanism.

---

## Consuming a Published Module

```bicep
// main.bicep — consuming the published module from ANY repo, ANY team,
// as long as they have read access to the registry

module storage 'br:myorgbicepmodules.azurecr.io/bicep/modules/storage-account:v1.0.0' = {
  name: 'storageDeployment'
  params: {
    storageAccountName: 'mystorageacc12345'
    location: resourceGroup().location
    replicationType: 'Standard_LRS'
  }
}
```

```hcl
# Terraform equivalent — pulling from a registry by source address + version constraint
module "storage" {
  source  = "app.terraform.io/myorg/storage-account/azurerm"
  version = "1.0.0"

  storage_account_name = "mystorageacc12345"
  location              = var.location
  replication_type       = "LRS"
}
```

| | Terraform Registry | Bicep Registry |
|---|---|---|
| Underlying storage | HashiCorp's own bespoke registry service (or Terraform Enterprise for private) | Any OCI-compliant registry (ACR is the standard choice on Azure) |
| Reference scheme | `<host>/<namespace>/<name>/<provider>` | `br:<registryHost>/<repositoryPath>:<tag>` |
| Versioning | Semantic version constraints (`~> 1.0`, `>= 1.2.0, < 2.0.0`) in the `version` argument | Exact tag pinning only — Bicep has **no version range/constraint syntax** for modules; you always pin one exact tag |
| Public option | terraform Registry (registry.terraform.io) — free, hosted by HashiCorp | `br/public:` — free, hosted by Microsoft (the AVM modules, below) |
| Private option | Terraform Cloud/Enterprise (paid tiers), or self-hosted alternatives | Any private ACR instance — no special paid product required at all |

> **Important asymmetry to internalize:** Terraform lets you express a *range* of acceptable versions (`~> 1.0.0` meaning "any 1.0.x"), and `terraform init` resolves the best match. Bicep modules are **always pinned to one exact tag** — there is no concept of "give me the latest 1.x." If you want to track updates, you must manually bump the tag in your `module` reference yourself, the same way you'd bump a Docker image tag in a Kubernetes manifest. This is a deliberate design choice favoring reproducibility over convenience.

---

## Module Aliases — Shortening Long Registry References With `bicepconfig.json`

Typing the full `br:myorgbicepmodules.azurecr.io/bicep/modules/storage-account:v1.0.0` every time is verbose. A `bicepconfig.json` file, placed at your project root, lets you define short **aliases** for registries — Bicep automatically discovers this file by walking up from wherever you run `az deployment ... create`.

### `bicepconfig.json`

```json
{
  "moduleAliases": {
    "br": {
      "platform": {
        "registry": "myorgbicepmodules.azurecr.io",
        "modulePath": "bicep/modules"
      }
    }
  }
}
```

### Using the Alias

```bicep
// Instead of the full br:myorgbicepmodules.azurecr.io/bicep/modules/storage-account:v1.0.0
module storage 'br/platform:storage-account:v1.0.0' = {
  name: 'storageDeployment'
  params: {
    storageAccountName: 'mystorageacc12345'
    location: resourceGroup().location
  }
}
```

This is conceptually similar to how a Terraform private registry's `<host>/<namespace>` prefix is implicitly "your organization's stuff" once you've configured credentials for that host — `bicepconfig.json` just makes that shortcut explicit and visible in source control, rather than relying on CLI-level registry login context alone.

> `bicepconfig.json` also lets you configure linter rule severities, analyzer settings, and experimental feature flags — but module aliasing is by far its most commonly used capability in team settings.

---

## Azure Verified Modules (AVM) — The Public, Curated Equivalent

Microsoft maintains an official initiative — **Azure Verified Modules** — publishing battle-tested, Microsoft-reviewed Bicep (and Terraform!) modules for the most common Azure resource patterns, available to everyone for free at `br/public:`.

```bicep
// Using an AVM-published module directly — no registry setup needed at all,
// "public" is a built-in alias every Bicep installation already knows
module storage 'br/public:avm/res/storage/storage-account:0.9.1' = {
  name: 'storageDeployment'
  params: {
    name: 'mystorageacc12345'
    location: resourceGroup().location
  }
}
```

### Why Use AVM Modules Instead of Writing Your Own?

| Reason | Explanation |
|---|---|
| Encodes Microsoft's own best practices | AVM modules are reviewed by the Azure product teams themselves, not just community contributors |
| Handles edge cases you'd otherwise discover the hard way | Things like correct diagnostic settings wiring, private endpoint support, RBAC role assignment helpers — already built in |
| Consistent interface across resource types | All AVM modules follow the same parameter-naming conventions (`name`, `location`, `tags`, `roleAssignments`, `diagnosticSettings` — predictable across every module) |
| Actively maintained against new API versions | As Azure ships new API versions and features, AVM modules get updated centrally — you benefit without re-deriving the change yourself |

```hcl
# Terraform's closest direct equivalent — verified modules on the public
# Terraform Registry, often (but not always) maintained by HashiCorp partners
# rather than Microsoft directly
module "storage_account" {
  source  = "Azure/avm-res-storage-storageaccount/azurerm"
  version = "0.2.5"
  # ...
}
```

> Notably, the **same AVM initiative publishes both Bicep and Terraform versions** of many modules, with matching parameter philosophies where possible — a useful detail if your organization runs a mixed Bicep/Terraform estate and wants consistent module design language across both.

### Finding AVM Modules

```bash
# Browse the official AVM module index
# (search the web — this is a living catalog, not something to hardcode from memory)
```

Always check the current AVM module registry/index directly rather than relying on a remembered version number — module versions update frequently, and pinning a stale version deliberately (file 06's reproducibility principle) is different from *discovering* what's currently available.

---

## Registry Workflow — End-to-End Summary

```
1. Bootstrap a private ACR (once, manually) — file 00/01 pattern, same as Terraform state storage bootstrap
        │
        ▼
2. Write & test a module LOCALLY (file path reference) until it's stable
        │
        ▼
3. az bicep publish --target br:<registry>/<path>:<version>
        │
        ▼
4. Consumers reference it via br: (full) or br/<alias>: (via bicepconfig.json)
        │
        ▼
5. New version needed? Publish a NEW TAG — never overwrite an existing tag
   (exactly like Docker image immutability conventions)
```

> **Immutability convention:** Just as you should never push a Docker image and overwrite an existing tag like `v1.0.0`, you should never republish a Bicep module artifact under a tag that's already been consumed by anyone. Bump the version (`v1.0.1`, `v1.1.0`) for any change, however small. Consumers pinning an exact tag (recall: no version ranges in Bicep) are relying on that tag being permanently stable.

The next file covers **Azure Policy as Code via Bicep** — a category of resource Terraform can also manage, but one where Bicep's native ARM alignment gives it a structural authoring advantage, since Policy definitions are themselves expressed as ARM-shaped JSON rule logic.