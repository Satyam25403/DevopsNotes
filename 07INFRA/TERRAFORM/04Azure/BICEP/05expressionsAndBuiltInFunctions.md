# Bicep Expressions & Built-in Functions

Expressions are how Bicep computes values — referencing parameters, transforming data, making decisions. Compared to Terraform's HCL expressions, Bicep's are generally **simpler to write** (no `${}` wrapping needed almost anywhere) but rely on a **much larger built-in function library**, because Bicep has no third-party providers to lean on for resource-specific computed values — everything has to come from core functions.

```
Types of expressions covered:
  References               →  bare param/var names, resource.property, module.outputs.x
  String interpolation      →  '${...}' inside single-quoted strings
  Conditional (ternary)      →  condition ? trueVal : falseVal
  Built-in functions          →  string, array, object, logical, deployment, resource functions
  Type conversion              →  string(), int(), bool(), array(), json()
```

---

## Reference Expressions — The Foundation

```bicep
// Syntax:
//   <bareName>                       → param or var (no prefix, ever)
//   <symbolicName>.<property>         → resource attribute
//   <moduleSymbolicName>.outputs.<x>  → module output
//   <functionName>()                  → built-in function (often scope-aware, e.g. resourceGroup())

resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name     : 'day10-rg'
  location : 'westus2'
}
```

Recall from file 01: **Bicep has no `var.` or `local.` prefix** the way Terraform does. This is worth repeating because it's the single most common point of confusion for people coming from HCL.

---

## String Interpolation — `${}` Inside Single-Quoted Strings

Bicep uses the **same `${}` syntax** as Terraform for embedding expressions inside strings — but only inside **single-quoted** strings (Bicep's idiomatic string style).

```bicep
// Simple variable interpolation
var rgPrefix = '${environment}-resources'    // → "dev-resources"

// Multiple values in one string
var fullName = '${environment}-${location}-rg'

// Expression inside interpolation
var displayEnv = '${environment == 'prod' ? 'production' : environment}-rg'
```

> Unlike Terraform — where interpolation works inside ALL string contexts including resource argument values directly (`name = "${var.x}-rg"`) — Bicep behaves identically here too. The syntax is genuinely the same `${}` token; the main practical difference is just Bicep's preference for single quotes as the default string delimiter.

### Multi-line Strings

```bicep
var description = '''
  This resource group belongs to the ${environment} environment.
  Managed by Bicep.
'''
```

Bicep's triple-single-quote (`'''`) multi-line string is the equivalent of Terraform's heredoc (`<<-EOT ... EOT`) syntax.

---

## Conditional (Ternary) Expression

Identical concept and **identical syntax** to Terraform — same `condition ? trueVal : falseVal` shape, inherited from the same C-family lineage.

```bicep
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-09-01' = {
  name     : environment == 'dev' ? 'dev-nsg' : 'stage-nsg'
  location : rg.location
}
```

**More examples:**

```bicep
// Toggle replication SKU based on environment
var replicationSku = environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'

// Conditional instance count, fed into a "for" loop's range
var instanceCount = environment == 'prod' ? 3 : 1

// Nested ternary (same readability caveat as Terraform — use sparingly)
var vmSize = environment == 'prod'
  ? 'Standard_DS3_v2'
  : (environment == 'staging' ? 'Standard_DS2_v2' : 'Standard_DS1_v2')
```

---

## Built-in Functions — The Backbone of Bicep

Because Bicep has no provider ecosystem to compute provider-specific derived values for you, **almost everything dynamic flows through built-in functions**. This is a much larger and more load-bearing function library than Terraform's — worth learning thoroughly.

### Deployment Context Functions — No Terraform Equivalent

These read information about the **current deployment's scope itself** — something Terraform has no need for, because in Terraform "where am I deploying" is just whatever `resource_group_name` argument you typed, not a property of the execution context.

```bicep
resourceGroup()             // → object with .name, .location, .id, .tags of the CURRENT resource group
resourceGroup().location     // → e.g. "westeurope" — extremely common, seen throughout these files
resourceGroup().name         // → e.g. "example-resource"

subscription()               // → object with .subscriptionId, .tenantId, .displayName
subscription().subscriptionId

deployment()                  // → metadata about the deployment itself: .name, .properties

tenant()                      // → object with .tenantId, .displayName (tenant-scope deployments)

managementGroup()             // → object describing the current management group (mg-scope deployments)
```

### `resourceId()` — Constructing Resource IDs Manually

A function with **no Terraform equivalent need**, because Terraform always gives you a real resource's `.id` attribute directly from its own state. Bicep needs this function specifically for referencing resources **outside** the current template/module (e.g., a resource in another resource group, or a built-in role definition):

```bicep
// Construct the ARM resource ID for a role definition (Contributor role's well-known GUID)
resourceId('Microsoft.Authorization/roleDefinitions', 'b24988ac-6180-42a0-ab88-20f7382dd24c')

// Construct a resource ID for something in a DIFFERENT resource group
resourceId('other-rg', 'Microsoft.Network/virtualNetworks', 'shared-vnet')

// Subscription-level resource ID construction
subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'b24988ac-6180-42a0-ab88-20f7382dd24c')

// Tenant-level resource ID construction
tenantResourceId('Microsoft.Management/managementGroups', 'platform-mg')
```

### `uniqueString()` — Deterministic Pseudo-Random Naming

Storage account names must be globally unique. Terraform users typically solve this with `random_string`/`random_id` **resources** (a stateful resource that, once created, locks in its value forever in the state file). Bicep instead offers a **pure, stateless function** that deterministically derives a short hash from whatever seed values you give it — same inputs always produce the same output, with no state needed to "remember" the value:

```bicep
// Deterministic — running this twice with the SAME resourceGroup().id
// always produces the SAME string. No state file needed to "remember" it.
var uniqueSuffix = uniqueString(resourceGroup().id)
// → e.g. "x7k2m9p1qz3vt" (13-character deterministic hash)

var storageAccountName = 'st${uniqueSuffix}'
// → "stx7k2m9p1qz3vt" — guaranteed unique to this resource group, every time
```

```hcl
# Terraform's nearest equivalent — a STATEFUL resource, not a pure function
resource "random_string" "suffix" {
  length  = 13
  special = false
  upper   = false
}

resource "azurerm_storage_account" "example" {
  name = "st${random_string.suffix.result}"
  # ...
}
```

> This is a genuinely elegant Bicep advantage: because `uniqueString()` is a pure function of its inputs (not a stateful resource), there's nothing to "lose" if state goes away — recompiling the same template with the same resource group always regenerates the exact same name. Terraform's `random_string` resource, by contrast, generates its value **once** and then stores it in state forever; if you lose state, you lose the ability to regenerate that exact same value deterministically.

### `guid()` — Deterministic GUID Generation

```bicep
// Generates a deterministic GUID from the given seed values — same inputs,
// same GUID, every time. Commonly used for role assignment names, which
// Azure requires to be GUIDs.
guid(resourceGroup().id, 'storage-contributor-role')
```

---

## String Functions

```bicep
toUpper('dev')                    // → "DEV"
toLower('PROD')                   // → "prod"
trim('  dev  ')                   // → "dev"
replace('dev-rg', '-', '_')       // → "dev_rg"
substring('staging', 0, 3)        // → "sta"
concat('dev', '-', 'rg')          // → "dev-rg"  (also works on arrays! see below)
split('a,b,c', ',')               // → ["a", "b", "c"]
join(['a', 'b', 'c'], ', ')       // → "a, b, c"
startsWith('storage01', 'storage') // → true
endsWith('storage01', '01')        // → true
length('staging')                  // → 7 (also works on arrays/objects — see below)
```

> `concat()` is **overloaded**: applied to strings it concatenates text; applied to arrays it concatenates/merges arrays into one. This dual behavior has no direct single-function Terraform parallel (Terraform separates string concatenation, handled via `${}`/`format()`, from array concatenation, handled via the `concat()` function which ONLY works on lists) — Bicep's `concat()` does both based on argument type.

---

## Array & Collection Functions

```bicep
length(allowedLocations)               // → 3 (count of items)
contains(allowedLocations, 'eastus')   // → true
first(allowedLocations)                // → "westeurope" (first element)
last(allowedLocations)                  // → "eastus" (last element)
take(allowedLocations, 2)               // → ["westeurope", "northeurope"] (first N elements)
skip(allowedLocations, 1)               // → ["northeurope", "eastus"] (all but first N)
union(['a','b'], ['b','c'])             // → ["a","b","c"] — DEDUPLICATES, Bicep's closest "set" behavior
intersection(['a','b','c'], ['b','c','d'])  // → ["b","c"]
range(0, 5)                              // → [0, 1, 2, 3, 4] — generates a numeric array, used heavily with "for"
flatten([['a','b'], ['c']])              // → ["a","b","c"]
sort(['c','a','b'])                       // → ["a","b","c"]
```

> Recall from file 01: Bicep has no native `set` type. `union()` is how you actually get set-like deduplication behavior when you need it — the functional equivalent of Terraform's automatic deduplication when a value is typed as `set(string)`.

---

## Object Functions

```bicep
// items(obj) — convert an object into an array of {key, value} pairs
// (seen heavily in file 04's for_each-equivalent loops)
items(resourceTags)
// → [{ key: 'environment', value: 'staging' }, { key: 'managedBy', value: 'bicep' }, ...]

// union() also works on objects — merges them, later object wins on key conflict
union(
  { environment: 'dev', managedBy: 'bicep' },
  { environment: 'prod', team: 'platform' }
)
// → { environment: 'prod', managedBy: 'bicep', team: 'platform' }

contains(resourceTags, 'environment')   // → true (key existence check)

// Safe key access with fallback — Bicep doesn't have a separate "lookup()"
// function; this is achieved via the .? safe-access operator (next section)
```

```hcl
# Terraform's merge() — same union-of-objects concept
merge(
  { environment = "dev", managed_by = "terraform" },
  { environment = "prod", team = "platform" }
)
```

---

## Safe Navigation — The `.?` Operator (No Terraform Equivalent)

Terraform's `lookup(map, key, default)` provides a safe fallback when a key might not exist. Bicep achieves the same safety via the **safe-access operator** `.?`, which short-circuits to `null` instead of throwing an error if a property/key doesn't exist:

```bicep
// If "managedBy" key doesn't exist in resourceTags, this evaluates to null
// instead of crashing the deployment
var managedByOrNull = resourceTags.?managedBy

// Combine with ?? (the "coalesce" operator) for a true lookup()-with-default equivalent
var managedBy = resourceTags.?managedBy ?? 'unknown'
```

```hcl
# Terraform equivalent
lookup(var.resource_tags, "managedBy", "unknown")
```

> `.?` also works for safely drilling into potentially-`null` nested resource properties — e.g. `storageAccount.properties.?networkAcls.?defaultAction` — extremely useful when a property only exists conditionally based on resource configuration.

---

## Numeric Functions

```bicep
max(10, 20, 5)     // → 20  (also accepts an array: max([10, 20, 5]))
min(10, 20, 5)      // → 5
```

> Bicep's numeric function library is intentionally smaller than Terraform's — there is **no built-in `ceil()`, `floor()`, or `abs()`** in classic Bicep expression functions. Rounding/absolute-value logic, if truly needed, is typically pushed into a parameter's allowed value set, a module's input validation, or computed upstream (e.g., in a pipeline script) before being passed in as a plain `int` parameter.

---

## Type Conversion Functions

```bicep
string(42)          // → "42"
int('42')           // → 42
bool('true')        // → true
array('singleValue') // → ["singleValue"] — wraps a scalar into a one-element array
json('{"key":"value"}')      // → parses a JSON string into an object/array
string({ key: 'value' })      // → '{"key":"value"}' — serializes an object/array to a JSON string
base64('hello')                // → "aGVsbG8="
base64ToString('aGVsbG8=')      // → "hello"
dateTimeAdd(utcNow(), 'P1D')     // → current UTC time + 1 day, ISO 8601 format
```

---

## Loading External Content

```bicep
loadTextContent('./scripts/init.sh')          // read a local file into a string
loadFileAsBase64('./certs/cert.pem')           // read file and base64-encode it
loadJsonContent('./config/settings.json')       // read and parse a local JSON file directly into an object
loadYamlContent('./config/settings.yaml')        // read and parse a local YAML file directly into an object
```

> `loadJsonContent()` / `loadYamlContent()` have no precise one-function Terraform equivalent — Terraform typically combines `file()` with `jsondecode()`/`yamldecode()` as two separate steps. Bicep collapses "read + parse" into one function call.

---

## Expressions — Quick Reference

| Expression | Syntax | Returns | Example |
|---|---|---|---|
| Reference | bare name, `resource.property`, `module.outputs.x` | any | `environment`, `rg.location` |
| Interpolation | `'${expr}'` | string | `'${environment}-rg'` |
| Conditional | `cond ? a : b` | any | `environment == 'prod' ? 'GRS' : 'LRS'` |
| Safe access | `value.?property` | any or null | `resourceTags.?managedBy` |
| Coalesce | `value ?? fallback` | any | `resourceTags.?managedBy ?? 'unknown'` |
| for (array) | `[for x in col: expr]` | array | `[for r in nsgRules: r.description]` |
| for (object) | `[for item in items(obj): expr]` | array | `[for item in items(tags): item.key]` |
| Deployment context | `resourceGroup()`, `subscription()`, `deployment()` | object | `resourceGroup().location` |
| Deterministic naming | `uniqueString(...)`, `guid(...)` | string | `uniqueString(resourceGroup().id)` |

The next (and final, for this batch) file covers **Bicep's state model: deployment history, the `what-if` operation, and Deployment Stacks** — directly addressing the question every Terraform engineer asks first about Bicep: *"if there's no state file, how does Bicep know what it's managing, and how do I do safe teardown?"*