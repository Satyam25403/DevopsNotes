# Bicep Nested & Child Resources, Resource Naming Conventions

Many Azure resources are organized in a **parent-child hierarchy** at the ARM level itself — a storage account *contains* blob containers, a SQL server *contains* databases, a virtual network *contains* subnets. This file covers how Bicep expresses that hierarchy, the naming rules that come with it, and a subtlety that trips up almost everyone moving from Terraform: **Terraform flattens this hierarchy into separate top-level resources; ARM (and therefore Bicep) does not.**

```
Concepts covered:
  Parent-child resource hierarchy in ARM      →  why it exists, how it differs from Terraform's flat model
  Nested resource syntax (resource inside resource)  →  the most concise approach
  Sibling/top-level syntax with parent property        →  the more flexible alternative
  Compound naming rules                                  →  parent/child name concatenation
  "existing" + child resource patterns                    →  referencing a child of an unmanaged parent
```

---

## The Core Difference From Terraform

Terraform's `azurerm` provider almost always models parent-child Azure relationships as **two separate, flat, top-level resource blocks**, linked only by an attribute reference:

```hcl
resource "azurerm_storage_account" "example" {
  name                     = "mystorageacc12345"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
}

resource "azurerm_storage_container" "example" {
  name                  = "mycontainer"
  storage_account_name  = azurerm_storage_account.example.name   # just an attribute reference
  container_access_type = "private"
}
```

Both blocks sit at the same indentation level, same file structure, same `resource "type" "name"` shape — the parent-child relationship is *purely* expressed through that one attribute reference (`storage_account_name = ...`), exactly like any other implicit dependency.

**ARM, underneath, does not work this way.** A blob container's true ARM resource type is literally `Microsoft.Storage/storageAccounts/blobServices/containers` — a slash-delimited type that encodes its full ancestry. Terraform's provider hides this by exposing it as a flat, independently-named resource type (`azurerm_storage_container`) and translating behind the scenes. Bicep, having no translation layer, exposes this ARM truth directly — and gives you **two syntactic ways** to express it.

---

## Approach 1: Nested Resource Declaration (Most Concise)

Declare the child resource **literally inside** the parent's resource block. This is the most natural-reading syntax and the one you'll see most often in official examples.

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : 'mystorageacc12345'
  location : resourceGroup().location
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'

  // ── Nested child resource: blob service settings ──────────────────────
  resource blobService 'blobServices@2023-01-01' = {
    name: 'default'
    // Note: the type here is JUST 'blobServices@2023-01-01' — no need to
    // repeat 'Microsoft.Storage/storageAccounts/' since Bicep already
    // knows the parent type from context.

    properties: {
      deleteRetentionPolicy: {
        enabled: true
        days: 7
      }
    }

    // ── Doubly-nested child: a container, inside the blob service, inside the account ──
    resource container 'containers@2023-01-01' = {
      name: 'mycontainer'
      properties: {
        publicAccess: 'None'
      }
    }
  }
}

// Referencing a nested resource from OUTSIDE the parent block uses
// dot-chained symbolic names: parentName::childName (double-colon syntax)
output containerId string = storageAccount::blobService::container.id
```

> **The `::` (double-colon) accessor** is unique to Bicep and has no Terraform equivalent, precisely because Terraform never nests resource blocks inside one another in the first place. Use it whenever you need to reference a nested resource's properties from somewhere outside its immediate parent block (e.g., in an output, or from a sibling resource).

### Why Nesting Is Often Preferred

- Visually communicates the real ARM hierarchy — a reader instantly sees "this container belongs to this storage account," with no need to trace an attribute reference
- Bicep automatically handles the compound naming and dependency ordering for you — zero risk of forgetting an implicit dependency
- Matches how the Portal and ARM templates themselves think about these resources

---

## Approach 2: Sibling/Top-Level Declaration With `parent` Property

Sometimes nesting is inconvenient — e.g., the child resource is large enough to deserve its own file/module, or you're looping over many children with `for` (file 04) and nesting would force you to also loop the entire parent unnecessarily. In these cases, declare the child as a **top-level resource**, and explicitly point at its parent using the `parent` property.

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : 'mystorageacc12345'
  location : resourceGroup().location
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
}

resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  // Note: here the FULL type path IS required, including the parent segment
  // ('storageAccounts/blobServices'), because this declaration is no longer
  // nested inside the parent — Bicep has no surrounding context to infer it from.

  name   : 'default'
  parent : storageAccount   // <-- THIS is what establishes the hierarchy
  properties: {
    deleteRetentionPolicy: {
      enabled: true
      days: 7
    }
  }
}

resource container 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  name   : 'mycontainer'
  parent : blobService
  properties: {
    publicAccess: 'None'
  }
}

// Referencing from outside is now simpler — just the bare symbolic name,
// no "::" chain needed, because it's already a top-level declaration
output containerId string = container.id
```

> The `parent` property does two things simultaneously: it establishes the **implicit dependency** (Bicep waits for `storageAccount` before creating `blobService`), AND it tells Bicep how to construct the **compound resource name** correctly behind the scenes (next section) — you do not need to manually concatenate `'${storageAccount.name}/default'` yourself.

### Looping Over Children — Why Sibling Syntax Matters

```bicep
param containerNames array = [
  'raw-data'
  'processed-data'
  'archive'
]

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : 'mystorageacc12345'
  location : resourceGroup().location
  sku: { name: 'Standard_GRS' }
  kind: 'StorageV2'
}

resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  name   : 'default'
  parent : storageAccount
}

// Looping a NESTED resource isn't possible — "for" can only be applied at
// the top level of a resource/module declaration. So whenever you need
// MULTIPLE children, sibling syntax with "parent" is mandatory, not optional.
resource containers 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = [for name in containerNames: {
  name   : name
  parent : blobService
  properties: {
    publicAccess: 'None'
  }
}]
```

```hcl
# Terraform equivalent — for_each on the flat child resource type
resource "azurerm_storage_container" "example" {
  for_each              = toset(var.container_names)
  name                  = each.value
  storage_account_name  = azurerm_storage_account.example.name
  container_access_type = "private"
}
```

---

## Compound Naming Rules

When you use either nested or sibling+`parent` syntax, Bicep automatically builds the correct **fully-qualified ARM name** for you — but it's important to understand what's actually happening underneath, especially when you need to construct such a name manually (e.g., referencing an `existing` child resource, next section).

A child resource's true ARM name is always `<parentName>/<childName>`, slash-delimited, all the way up the hierarchy:

```
storageAccount.name        → "mystorageacc12345"
blobService.name (compiled) → "mystorageacc12345/default"
container.name (compiled)    → "mystorageacc12345/default/mycontainer"
```

You never type the slashes yourself when using `parent` or nesting — Bicep computes this for you. But you'll see this exact slash-delimited shape if you ever inspect the compiled ARM JSON (`az bicep build`), or if you need to reference an existing child resource manually:

```bicep
// Referencing an EXISTING (not managed by this template) nested resource
// requires manually constructing the compound name — there's no "parent"
// shortcut available on an "existing" declaration in every Bicep version,
// so know this pattern as a fallback:
resource existingContainer 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' existing = {
  name: 'mystorageacc12345/default/mycontainer'   // full compound path, manually written
}
```

A cleaner alternative — nest the `existing` declaration inside an `existing` parent, letting Bicep compute the compound name for you, exactly like it does for managed resources:

```bicep
resource existingStorageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: 'mystorageacc12345'

  resource existingBlobService 'blobServices@2023-01-01' existing = {
    name: 'default'

    resource existingContainer 'containers@2023-01-01' existing = {
      name: 'mycontainer'
    }
  }
}

output containerId string = existingStorageAccount::existingBlobService::existingContainer.id
```

---

## Resource Naming Best Practices

These apply regardless of nesting approach, and mirror good Terraform naming hygiene:

```bicep
// ── GOOD: descriptive symbolic names, environment baked into the actual Azure name ──
resource appStorageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'st${environment}${uniqueString(resourceGroup().id)}'
  // → "stproductionx7k2m9p1qz3v" — guaranteed unique, environment-identifiable,
  //    within Azure's 24-character limit if "environment" is kept short
  location: resourceGroup().location
  sku: { name: 'Standard_GRS' }
  kind: 'StorageV2'
}

// ── AVOID: symbolic name that's just a copy of the resource type — adds no information ──
resource storageaccountsstorageaccount 'Microsoft.Storage/storageAccounts@2023-01-01' = { ... }
```

| Naming layer | Rule of thumb | Example |
|---|---|---|
| Symbolic name (Bicep-internal) | Describe the resource's *purpose* in this template, not its type | `appStorageAccount`, not `sa1` or `storageAccount2` |
| Actual Azure `name` property | Follow the specific resource type's Azure naming rules (length, charset) — often combine environment + `uniqueString()` | `'st${environment}${uniqueString(resourceGroup().id)}'` |
| Module deployment `name` | Make it traceable in deployment history (file 06) — include what it deploys | `name: 'storageDeployment-${environment}'` |

---

## Quick Reference: When to Nest vs When to Use `parent`

| Scenario | Recommended approach |
|---|---|
| Single, fixed child resource (e.g., one blob service config block) | Nested declaration — most readable |
| Multiple children needing `for` iteration | Sibling + `parent` — `for` cannot be applied inside a nested block |
| Child resource large/complex enough to deserve its own module file | Sibling + `parent`, likely promoted into its own module entirely |
| Referencing a child of a resource you don't manage | `existing`, nested or with manually-built compound name |

The next file covers **publishing and consuming modules through the Bicep Registry** in full depth — how `az bicep publish` actually works, versioning conventions, private Azure Container Registry setup, and the public Azure Verified Modules (AVM) ecosystem briefly introduced in file 03.