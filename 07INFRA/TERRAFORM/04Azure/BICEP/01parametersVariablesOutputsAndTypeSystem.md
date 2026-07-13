# Bicep Parameters, Variables & the Type System

Bicep has the same three conceptual "kinds" of values as Terraform, just renamed and reshaped slightly:

| Concept | Terraform name | Bicep name | Purpose |
|---|---|---|---|
| Input | `variable` | `param` | Values supplied from outside the file to parameterize a deployment |
| Internal/computed | `local` | `var` | Values computed once, reused internally, never supplied externally |
| Output | `output` | `output` | Values exposed after deployment completes |

There is no separate "output variables" file convention forced on you the way Terraform nudges `outputs.tf` — but as you'll see in file 02, splitting into `main.bicep`, with parameters and outputs inline (or via modules), is still the idiomatic structure.

---

## Parameters (`param`) — Bicep's Input Variables

### Defining a Parameter

```bicep
param environment string = 'staging'
//    ^name        ^type   ^default value (optional)
```

Compare directly to Terraform:

```hcl
variable "environment" {
  type    = string
  default = "staging"
}
```

Bicep collapses the whole block into a single line: `param <name> <type> = <default>`. No `description` is required to declare a parameter (though you should always add one — see decorators below).

### Using a Parameter

Reference parameters with **no prefix at all** — just the bare name. This is a sharp departure from Terraform's `var.<name>`.

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     = 'mystorageacc12345'
  location = resourceGroup().location
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
  tags: {
    environment: environment   // bare reference — no "param." prefix
  }
}
```

> **Common mistake for Terraform users:** writing `param.environment` or `parameters.environment`. Bicep parameters are referenced by their **bare name only**, exactly like a local variable in a general-purpose programming language. There is no namespacing prefix.

---

## Decorators — Bicep's Validation & Metadata System

Terraform expresses constraints through the `type` field and (separately) `validation` blocks inside a `variable {}`. Bicep instead uses **decorators** — `@`-prefixed annotations placed directly above a `param` declaration. This is conceptually similar to decorators/annotations in languages like Python or Java.

```
Decorators covered:
  @description(...)     →  human-readable explanation (always use this)
  @allowed([...])        →  restrict to an explicit set of values (enum-like)
  @minLength(n) / @maxLength(n)   →  string/array length constraints
  @minValue(n) / @maxValue(n)     →  numeric range constraints
  @secure()               →  marks a parameter as sensitive (string or object)
  @metadata({...})        →  arbitrary structured metadata
```

### `@description` — Always Use This

```bicep
@description('The deployment environment name. Used for tagging and naming.')
param environment string = 'staging'
```

### `@allowed` — Restrict to a Fixed Set of Values

The direct equivalent of Terraform's `validation { condition = contains([...], var.x) }`, but built into the type system itself — Bicep rejects invalid values **at compile/validation time**, before any API call is made.

```bicep
@description('The environment type — must be one of the predefined stages.')
@allowed([
  'dev'
  'staging'
  'prod'
])
param environment string = 'staging'
```

If someone passes `environment = 'production'` (a typo for `'prod'`), the deployment fails immediately during validation — never reaching Azure's API. This is strictly more proactive than Terraform's `precondition` blocks (file 05 in the Terraform set), which only fire at plan/apply time for that specific resource.

### `@minLength` / `@maxLength` — Length Constraints

Works on both `string` and `array` parameters.

```bicep
@description('Storage account name — must follow Azure global naming rules.')
@minLength(3)
@maxLength(24)
param storageAccountName string

@description('List of allowed Azure regions for this deployment.')
@minLength(1)   // at least one region must be provided
param allowedLocations array = [
  'westeurope'
  'northeurope'
  'eastus'
]
```

### `@minValue` / `@maxValue` — Numeric Range Constraints

```bicep
@description('OS disk size in GB — must be within Azure-supported managed disk range.')
@minValue(20)
@maxValue(4095)
param storageDiskSizeGb int = 80
```

### `@secure()` — Marking Sensitive Parameters

The direct equivalent of Terraform's `sensitive = true`. A `@secure()` parameter's value is **never** written to deployment logs or the deployment history that ARM stores (covered in file 06) — Terraform's `sensitive` flag, by contrast, only masks console/log output; the raw value still sits in plaintext inside `terraform.tfstate` unless you take extra precautions. This is one area where Bicep's "no state file" model is meaningfully *safer* by default.

```bicep
@description('Administrator password for the VM.')
@secure()
param adminPassword string

@description('Service principal client secret.')
@secure()
param clientSecret string
```

> `@secure()` can only be applied to `string` or `object` typed parameters. There is no default value allowed on a `@secure()` parameter without an explicit default expression — Azure discourages hardcoding secrets as defaults entirely.

### Combining Decorators

Decorators stack — apply as many as relevant, each on its own line, directly above the `param`:

```bicep
@description('Number of VM instances to deploy in this environment.')
@minValue(1)
@maxValue(10)
param instanceCount int = 2
```

---

## The Full Bicep Type System

| Bicep Type | Category | Terraform Equivalent | Notes |
|---|---|---|---|
| `string` | Primitive | `string` | Identical concept |
| `int` | Primitive | `number` | Bicep splits Terraform's single `number` into `int` (whole numbers only) |
| `bool` | Primitive | `bool` | Identical concept |
| `array` | Collection | `list(any)` / `set(any)` | **No separate `set` type** — Bicep arrays allow duplicates, like Terraform `list`. Deduplication must be done manually with the `union()` function. |
| `object` | Structural | `object({...})` / `map(string)` | Used for BOTH free-form maps AND strongly-typed structured objects — see below |
| `array` of typed items | Structural | `list(object({...}))` | e.g. `param items array` holding a list of objects |

### Important Structural Difference: No `set`, No `map(T)`, No `tuple`

This is the single biggest type-system departure from Terraform:

- **No `set` type.** Bicep has only `array`. If you need uniqueness, you call the `union()` function manually, or simply rely on `for_each`-style iteration logic (covered in file 04) and accept potential duplicates being your own responsibility.
- **No `map(string)`-style typed map.** Bicep's `object` type serves double duty — it can be a loosely-typed bag of key-value pairs (like a Terraform `map(string)`) OR a strongly-typed structure with named, individually-typed fields (like Terraform's `object({...})`). Context (and optional User-Defined Types — see below) determines which.
- **No `tuple` type.** A fixed-position, mixed-type ordered sequence simply doesn't exist as a primitive in classic Bicep parameter declarations. You model that need with an `object` (named fields) instead — which is usually *more* readable than a positional tuple anyway.

### `object` as a Loose Map (Terraform `map(string)` equivalent)

```bicep
@description('Tags to apply to all resources.')
param resourceTags object = {
  environment: 'staging'
  managedBy: 'bicep'
  department: 'devops'
}
```

Access by key using dot notation or bracket notation:

```bicep
tags: {
  environment: resourceTags.environment      // dot notation
  managedBy: resourceTags['managedBy']        // bracket notation — both work
}
```

### `object` as a Strongly-Typed Structure (Terraform `object({...})` equivalent)

```bicep
@description('Virtual machine image and size configuration.')
param vmConfig object = {
  size: 'Standard_DS1_v2'
  publisher: 'Canonical'
  offer: '0001-com-ubuntu-server-jammy'
  sku: '22_04-lts'
  version: 'latest'
}
```

```bicep
storageImageReference: {
  publisher: vmConfig.publisher
  offer: vmConfig.offer
  sku: vmConfig.sku
  version: vmConfig.version
}
```

> Notice: declared as plain `object`, Bicep does **not** enforce which fields must exist or what type each field is — unlike Terraform's `object({ size = string, publisher = string, ... })`, which is strictly validated at plan time. To get that same strictness in Bicep, you need **User-Defined Types** (next section) — a more modern feature.

### Arrays — Ordered, Allows Duplicates

```bicep
@description('Allowed Azure regions, in priority order.')
param allowedLocations array = [
  'westeurope'
  'northeurope'
  'eastus'
]
```

Index access works exactly like you'd expect:

```bicep
location: allowedLocations[2]   // → "eastus"
```

### Arrays of Objects — Bicep's `tuple`/structured-list Equivalent

When you need an ordered collection of structured items (Terraform might reach for `list(object({...}))`), Bicep just nests an `object` shape inside an `array`:

```bicep
@description('Network configuration entries — one object per subnet.')
param subnetConfigs array = [
  {
    name: 'frontend'
    addressPrefix: '10.0.1.0/24'
  }
  {
    name: 'backend'
    addressPrefix: '10.0.2.0/24'
  }
]
```

```bicep
// Access first subnet's name
subnetConfigs[0].name        // → "frontend"
subnetConfigs[1].addressPrefix  // → "10.0.2.0/24"
```

---

## User-Defined Types (UDTs) — Strict Structural Typing

Newer Bicep versions (1.9+) support declaring **named, reusable, strictly-typed shapes** with the `type` keyword — finally giving Bicep the same compile-time strictness Terraform's `object({...})` has always had, plus reusability across files.

```bicep
// Define a reusable, strictly-typed shape
type vmConfigType = {
  size: string
  publisher: string
  offer: string
  sku: string
  version: string
}

@description('Virtual machine image and size configuration.')
param vmConfig vmConfigType = {
  size: 'Standard_DS1_v2'
  publisher: 'Canonical'
  offer: '0001-com-ubuntu-server-jammy'
  sku: '22_04-lts'
  version: 'latest'
}
```

Now, unlike a bare `object` parameter, Bicep will **reject at compile time** any value for `vmConfig` that is missing a required field, has an extra unexpected field, or has the wrong type for any field — exactly matching Terraform's `object({...})` strictness.

### Union Types — Bicep's Closest Equivalent to `@allowed` as a Type

```bicep
type environmentType = 'dev' | 'staging' | 'prod'

@description('The deployment environment.')
param environment environmentType = 'staging'
```

This is functionally similar to `@allowed(['dev','staging','prod'])` on a plain `string`, but defines the constraint as a reusable named type rather than a one-off decorator — preferred when the same allowed-set is reused across multiple parameters or multiple files.

---

## Variables (`var`) — Bicep's Local Values

`var` is the direct equivalent of Terraform's `locals {}` block — computed once, used internally, never settable from outside the file.

### Defining Variables

```bicep
var commonTags = {
  environment: environment   // references the "environment" param — allowed!
  lob: 'banking'
  stage: 'alpha'
}
```

Unlike `param`, a `var` declaration has **no decorators** and **no default value syntax** — its value is *always* the expression on the right-hand side; there's nothing to override from outside.

### Using Variables

Reference with the bare name — same pattern as parameters, no prefix:

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : storageAccountName
  location : resourceGroup().location
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
  tags: commonTags   // pass the whole var directly
}
```

> **Crucial difference from Terraform:** in HCL, `var.x` (input) and `local.x` (computed) are visually distinguished by prefix, so you can tell at a glance which kind of value you're looking at. In Bicep, **both `param` and `var` are referenced identically — bare name, no prefix.** This means Bicep relies entirely on the *declaration* (`param` vs `var` keyword) to tell you which kind a name is; you cannot tell from a usage site alone. Get in the habit of checking the top of the file when reading unfamiliar Bicep code.

---

## Outputs — Exposing Values After Deployment

Identical concept to Terraform, with a small syntactic difference: Bicep outputs **must declare a type**, just like parameters.

```bicep
output storageAccountName string = storageAccount.name
output storageAccountId string = storageAccount.id
output primaryBlobEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

```hcl
# Terraform equivalent — type is inferred, not required
output "storage_account_name" {
  value = azurerm_storage_account.account.name
}
```

### Viewing Outputs After Deployment

```bash
# Outputs are returned in the JSON response of the deployment command itself
az deployment group create \
  --resource-group example-resource \
  --template-file main.bicep \
  --query properties.outputs

# Query a previously completed deployment's outputs by deployment name
az deployment group show \
  --resource-group example-resource \
  --name my-first-deployment \
  --query properties.outputs
```

Unlike Terraform, there is no persistent local `terraform output` command you can run any time — because there's no local state file holding the last-known outputs. You either capture the outputs at deploy time, or query Azure's deployment history for a *specific named deployment* after the fact (file 06 covers this deployment-history model in depth).

---

## Supplying Parameter Values — The Bicep Parameter File

Terraform offers five ways to supply variable values (default, `.tfvars`, `.tfvars.json`, env vars, `-var` flag). Bicep's equivalents:

### 1. Default Value in the Parameter Declaration

```bicep
param environment string = 'staging'   // used if nothing else overrides it
```

### 2. Bicep Parameters File (`.bicepparam`) — The Modern, Recommended Approach

```bicep
// main.bicepparam
using 'main.bicep'   // ties this parameter file to a specific Bicep template

param environment = 'production'
param storageAccountName = 'mystorageaccprod01'
param vmConfig = {
  size: 'Standard_DS3_v2'
  publisher: 'Canonical'
  offer: '0001-com-ubuntu-server-jammy'
  sku: '22_04-lts'
  version: 'latest'
}
```

```bash
az deployment group create \
  --resource-group example-resource \
  --parameters main.bicepparam
```

> `.bicepparam` is the direct, purpose-built equivalent of Terraform's `terraform.tfvars`, with one major advantage: the `using` statement **binds it to a specific template file**, and Bicep validates parameter names/types against that template at compile time — so a typo'd parameter name fails immediately, rather than silently being ignored (a real footgun that can happen with loosely-matched `.tfvars`).

### 3. JSON Parameters File (Legacy ARM-style — still supported)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": { "value": "production" },
    "storageAccountName": { "value": "mystorageaccprod01" }
  }
}
```

```bash
az deployment group create \
  --resource-group example-resource \
  --template-file main.bicep \
  --parameters @params.json
```

### 4. Inline on the Command Line — Equivalent to Terraform's `-var`

```bash
az deployment group create \
  --resource-group example-resource \
  --template-file main.bicep \
  --parameters environment=production storageAccountName=mystorageaccprod01
```

### 5. Reference a Key Vault Secret Directly From a Parameters File

This is a Bicep-native capability with **no direct Terraform equivalent in the variables system itself** (Terraform achieves something similar via the separate `azurerm_key_vault_secret` data source). A `.bicepparam` file can pull a `@secure()` parameter's value straight out of Key Vault at deployment time, so the secret never appears in any file at all:

```bicep
using 'main.bicep'

param adminPassword = getSecret('my-subscription-id', 'my-rg', 'my-keyvault', 'vm-admin-password')
```

### Parameter Value Precedence

When the same parameter is supplied in more than one way, **command-line `--parameters` always wins** over a parameters file, which always wins over the in-template default:

| Priority | Source |
|---|---|
| 1 (lowest) | Default value inside `param` declaration |
| 2 | `.bicepparam` or JSON parameters file |
| 3 (highest) | Inline `--parameters key=value` on the CLI |

Same overriding philosophy as Terraform — explicit, closer-to-execution-time values win.

---

## Side-by-Side Recap: Same Concept, New Syntax

```bicep
// ── Parameters (inputs) ──────────────────────────────────────────────────────
@description('The environment type')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'staging'

@description('OS disk size in GB')
@minValue(20)
@maxValue(4095)
param storageDiskSizeGb int = 80

@description('Whether to enable soft delete')
param isDelete bool = true

@description('List of allowed Azure regions')
param allowedLocations array = [
  'westeurope'
  'northeurope'
  'eastus'
]

@description('Tags to apply to resources')
param resourceTags object = {
  environment: 'staging'
  managedBy: 'bicep'
  department: 'devops'
}

@description('VM image and size configuration')
param vmConfig object = {
  size: 'Standard_DS1_v2'
  publisher: 'Canonical'
  offer: '0001-com-ubuntu-server-jammy'
  sku: '22_04-lts'
  version: 'latest'
}

// ── Variables (internal/computed) ────────────────────────────────────────────
var commonTags = {
  environment: environment
  lob: 'banking'
  stage: 'alpha'
}

// ── Resource using everything above ─────────────────────────────────────────
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : 'mystorageacc12345'
  location : allowedLocations[2]            // index access → "eastus"
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
  tags: commonTags
}

// ── Outputs ───────────────────────────────────────────────────────────────────
output storageAccountName string = storageAccount.name
output appliedTags object = commonTags
```

---

## Type System — Quick Reference

| Type | Category | Allows Duplicates | Ordered | Access Pattern | Terraform Equivalent |
|---|---|---|---|---|---|
| `string` | Primitive | — | — | directly | `string` |
| `int` | Primitive | — | — | directly | `number` |
| `bool` | Primitive | — | — | directly | `bool` |
| `array` | Collection | ✅ yes (no native `set`) | ✅ yes | `name[index]` | `list(T)` (closest) |
| `object` (loose) | Structural | n/a | ❌ no | `name.key` / `name['key']` | `map(string)` |
| `object` (typed via UDT) | Structural | n/a | ❌ no | `name.field` | `object({...})` |
| User-Defined `type` (union) | Structural | n/a | n/a | used as a type annotation | `@allowed` validation |
| `array` of `object` | Structural | ✅ yes | ✅ yes | `name[index].field` | `list(object({...}))` |

The next file covers Bicep's **file/directory structure, deployment scopes (resourceGroup / subscription / managementGroup / tenant), and dependency management** — the equivalent of the Terraform file-structure-and-dependencies lesson, plus a concept Terraform doesn't have an exact analogue for: **scope**.