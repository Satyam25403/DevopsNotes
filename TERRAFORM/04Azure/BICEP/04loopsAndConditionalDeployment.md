# Bicep Loops & Conditional Deployment

Terraform has two distinct meta-arguments for multiplying resources: `count` (index-based) and `for_each` (key-based), plus a separate `count = condition ? 1 : 0` idiom for conditional creation. Bicep **unifies all of this into a single mechanism**: the `for` expression, applicable directly on the `resource`, `module`, or even individual properties/outputs — plus a dedicated `if` keyword for conditionals. There is no Bicep equivalent of a separate `count` vs `for_each` distinction; everything is `for`.

```
Mechanisms covered:
  for (array)              →  Terraform count-like behavior (index-based, ordered)
  for (array, with index)   →  access both item AND its position
  for (object/map)           →  Terraform for_each-like behavior (key-based)
  if                          →  conditional resource/module deployment
  for + if combined           →  conditional iteration
```

---

## `for` Over an Array — Index-Based Iteration

The most direct equivalent of Terraform's `count`, but driven by iterating an actual array rather than a bare number.

```bicep
param storageAccountNames array = [
  'satyamstorage01'
  'shivamstorage02'
  'satyamshivamstorage03'
]

resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for name in storageAccountNames: {
  name     : name
  location : resourceGroup().location
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
  tags: {
    environment: 'staging'
  }
}]
```

```hcl
# Terraform equivalent using count
resource "azurerm_storage_account" "example" {
  count = length(var.storage_account_names)

  name                     = var.storage_account_names[count.index]
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
}
```

This creates three storage accounts — one per array element. Notice the syntax shape: `[for <item> in <collection>: { ...resource body... }]` — the **entire resource body becomes the expression after the colon**, wrapped in square brackets, because the whole thing now evaluates to an *array of resources* rather than a single resource.

---

## `for` With Index — Accessing Both Item and Position

When you need the loop's numeric position (Terraform's `count.index`), add a second loop variable:

```bicep
resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for (name, i) in storageAccountNames: {
  name     : name
  location : resourceGroup().location
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
  tags: {
    environment: 'staging'
    sequenceNumber: string(i)    // i = 0, 1, 2... — direct equivalent of count.index
  }
}]
```

```hcl
# Terraform equivalent — count.index gives the same positional access
tags = {
  environment     = "staging"
  sequence_number = tostring(count.index)
}
```

---

## `for` Over an Object/Map — Key-Based Iteration

The direct equivalent of Terraform's `for_each` over a `map`. Bicep iterates an `object`'s key-value pairs using a different loop signature — `for key in items(...)` — because plain `for ... in <object>` iterates *values only* by default. To get keys AND values together (the normal `for_each` use case), wrap the object in the built-in `items()` function.

```bicep
param storageAccountConfig object = {
  satyamstorage01: {
    sku: 'Standard_LRS'
    tags: { team: 'platform' }
  }
  shivamstorage02: {
    sku: 'Standard_GRS'
    tags: { team: 'data' }
  }
}

resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for item in items(storageAccountConfig): {
  // items(obj) returns an array of { key: ..., value: ... } objects —
  // "item.key" is the original object key, "item.value" is its value
  name     : item.key
  location : resourceGroup().location
  sku: {
    name: item.value.sku
  }
  kind: 'StorageV2'
  tags: item.value.tags
}]
```

```hcl
# Terraform equivalent using for_each over a map
variable "storage_account_config" {
  type = map(object({
    sku  = string
    tags = map(string)
  }))
}

resource "azurerm_storage_account" "example" {
  for_each = var.storage_account_config

  name                     = each.key
  account_tier             = "Standard"
  account_replication_type = split("_", each.value.sku)[1]
  tags                      = each.value.tags
}
```

| | Terraform `for_each` | Bicep `for ... in items(...)` |
|---|---|---|
| Iterator access | `each.key` / `each.value` | `item.key` / `item.value` (names you choose in `for <name> in ...`) |
| Input type | `set(T)` or `map(T)` | `object` (any shape) |
| Tracked by | Key (stable across additions/removals) | Compiled into an ARM `copy` loop tracked by the **array index of the `items()` result**, not by a string key — see note below |

> **Subtle but important difference from Terraform:** Terraform's `for_each` genuinely tracks state **by key** — removing one map entry only affects that one resource, leaving siblings untouched, because the state file stores each instance under its key. Bicep, lacking a persistent state file, instead relies on ARM's deployment **incremental mode** (file 06) plus resource name stability to achieve similar "don't touch unrelated instances" behavior — but the underlying mechanism is "did this resource's name and properties change between deployments," not "is this key still present in a state map." In practice this behaves similarly for well-named resources, but it's worth understanding the mechanism is genuinely different.

---

## `if` — Conditional Resource Deployment

Terraform achieves conditional resource creation as a side effect of `count` (`count = var.create_storage ? 1 : 0`) — there's no dedicated "if" keyword. Bicep gives conditionals their **own first-class keyword**, separate from looping, which is cleaner to read and reason about.

```bicep
@description('Whether to deploy a storage account at all')
param deployStorage bool = true

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = if (deployStorage) {
  name     : 'mystorageacc12345'
  location : resourceGroup().location
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
}
```

```hcl
# Terraform equivalent — conditional creation via count
variable "deploy_storage" {
  type    = bool
  default = true
}

resource "azurerm_storage_account" "example" {
  count = var.deploy_storage ? 1 : 0

  name                     = "mystorageacc12345"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
}
```

> **Important consuming difference:** because Terraform's conditional trick uses `count`, the resulting resource is **always indexed** even when only zero or one instance exists — you must reference it as `azurerm_storage_account.example[0].name`, with brackets, even though there's conceptually only one. Bicep's `if` keyword does **not** introduce indexing at all — a conditionally-created resource is referenced exactly like a normal, unconditional one: `storageAccount.name`. This is a meaningful ergonomic win; you don't have to remember "oh right, this one has `count` so it needs `[0]`."

### Real-World Example: Environment-Gated Resources

```bicep
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

// Only deploy a secondary/DR storage account in production
resource drStorageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = if (environment == 'prod') {
  name     : 'mystorageaccdr12345'
  location : 'northeurope'   // DR region, different from primary
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
}
```

---

## Combining `for` and `if` — Conditional Iteration

You can filter which elements of a loop actually produce a resource, by adding `if (...)` **inside** the loop body, right after the colon:

```bicep
param environments array = [
  { name: 'dev', deploy: true }
  { name: 'staging', deploy: true }
  { name: 'prod', deploy: false }   // not ready yet — skip this one
]

resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for env in environments: if (env.deploy) {
  name     : 'storage${env.name}'
  location : resourceGroup().location
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
}]
```

This creates storage accounts only for `dev` and `staging` — `prod` is skipped entirely because `env.deploy` is `false`. This is the closest Bicep equivalent of Terraform's `for_each = { for k, v in var.environments : k => v if v.deploy }` (a `for` expression with a filter, feeding into `for_each`) — but Bicep does it inline, in one step, without needing a separate filtering expression upstream.

---

## Looping Over Properties (Not Whole Resources)

Sometimes you don't need multiple *resources* — you need multiple **nested blocks inside one resource** (Terraform's `dynamic` block territory, file 06 of the Terraform set). Bicep uses the exact same `for` syntax, just targeting a property instead of the whole resource:

```bicep
param nsgRules array = [
  { name: 'allow-http', priority: 100, port: '80' }
  { name: 'allow-https', priority: 110, port: '443' }
]

resource nsg 'Microsoft.Network/networkSecurityGroups@2023-09-01' = {
  name     : 'example-nsg'
  location : resourceGroup().location
  properties: {
    securityRules: [for rule in nsgRules: {
      name: rule.name
      properties: {
        priority: rule.priority
        direction: 'Inbound'
        access: 'Allow'
        protocol: 'Tcp'
        sourcePortRange: '*'
        destinationPortRange: rule.port
        sourceAddressPrefix: '*'
        destinationAddressPrefix: '*'
      }
    }]
  }
}
```

```hcl
# Terraform equivalent — the "dynamic" block
dynamic "security_rule" {
  for_each = local.nsg_rules
  content {
    name                    = security_rule.value.name
    priority                = security_rule.value.priority
    direction               = "Inbound"
    access                   = "Allow"
    protocol                 = "Tcp"
    source_port_range        = "*"
    destination_port_range   = security_rule.value.port
    source_address_prefix    = "*"
    destination_address_prefix = "*"
  }
}
```

> **This is a meaningful simplification over Terraform.** Terraform needs an *entirely separate keyword* (`dynamic`) with its own block-scoped iterator naming convention (`<block_label>.key` / `<block_label>.value`, distinct from a resource's own `each.key`/`each.value`) specifically because HCL's nested blocks are not first-class array values. In Bicep, **everything is already JSON-shaped data** — a list of security rules is just an `array` of `object`, full stop. The exact same `for` syntax that loops over resources also loops over a plain property value, because there's no structural distinction between "a nested block" and "an array property" in the underlying JSON. One mechanism, used everywhere.

---

## Looping Over Outputs

```bicep
resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for name in storageAccountNames: {
  name     : name
  location : resourceGroup().location
  sku: { name: 'Standard_GRS' }
  kind: 'StorageV2'
}]

// Collect the .id of every instance created by the loop above
output allStorageAccountIds array = [for i in range(0, length(storageAccountNames)): storageAccounts[i].id]

// Simpler — just map directly over the resource's resulting array
output allStorageAccountNames array = [for sa in storageAccounts: sa.name]
```

> `storageAccounts` (the loop-created resource) behaves like an **array of resource references** once declared with `[for ...]`. You can index into it (`storageAccounts[0].id`) or iterate over it again in an output, exactly like any other array.

---

## `count` and `for_each` vs Bicep `for` — Full Comparison Table

| Concept | Terraform | Bicep |
|---|---|---|
| Multiply a resource, index-based | `count = N` + `[count.index]` | `[for (item, i) in array: {...}]` |
| Multiply a resource, key-based | `for_each = map_or_set` + `each.key`/`each.value` | `[for item in items(object): {...}]` |
| Conditional creation | `count = condition ? 1 : 0` (creates indexing side effect) | `if (condition) {...}` (no indexing side effect) |
| Conditional + iteration combined | `for_each` fed by a filtering `for` expression upstream | `[for item in array: if (condition) {...}]` — inline, one step |
| Repeated nested blocks inside ONE resource | Separate `dynamic "block" { for_each = ... content {...} }` keyword | Same `for` syntax, applied directly to the property |
| Referencing a conditionally-created resource | Always indexed: `resource.example[0].attr` | Never indexed: `resource.attr` |

The next file covers **Bicep functions and expressions** — string interpolation, the conditional (ternary) operator, and the rich built-in function library (`uniqueString()`, `resourceId()`, `union()`, and more) — the direct equivalent of the Terraform expressions lesson.