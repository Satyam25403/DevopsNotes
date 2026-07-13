# Terraform Expressions

Expressions are how Terraform computes values — referencing variables, transforming data, making decisions, and iterating over collections. They appear on the right-hand side of `=` in almost every argument.

```
Types of expressions covered:
  References          →  var.*, local.*, resource.*, data.*
  String interpolation / templates  →  "${...}"
  Conditional (ternary)  →  condition ? true_val : false_val
  for expression         →  transform/filter collections
  Splat expression       →  [*] shorthand for collecting attributes
  dynamic blocks         →  generate repeated nested blocks programmatically
  Type conversion        →  tolist(), toset(), tomap(), tonumber(), tostring()
  Built-in functions     →  length(), contains(), element(), join(), lookup(), merge(), ...
```

---

## File Structure

```
terraform-project/
├── provider.tf
├── backend.tf
├── variables.tf
├── locals.tf
├── main.tf
└── outputs.tf
```

---

## `provider.tf`

```hcl
# You can have many provider blocks (e.g. multiple aliased azurerm instances).
# "hashicorp/azurerm" is short for registry.terraform.io/hashicorp/azurerm

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.8.0"
    }
  }
  required_version = ">= 1.9.0"
}

provider "azurerm" {
  features {}
}
```

---

## `backend.tf`

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-day04"   # or: -backend-config="resource_group_name=<value>"
    storage_account_name = "day0417691"       # or: -backend-config="storage_account_name=<value>"
    container_name       = "tfstate"          # or: -backend-config="container_name=<value>"
    key                  = "dev.terraform.tfstate"  # or: -backend-config="key=<value>"
  }
}
```

---

## `variables.tf`

```hcl
# String — used in conditional expression and string interpolation
variable "environment" {
  type        = string
  description = "Environment name (e.g., dev, prod, staging)"
  default     = "dev"
}

# List — used in splat and index access
variable "account_names" {
  type        = list(string)
  description = "List of storage account names"
  default     = ["satyam", "shivam", "satyamshivam"]
}
```

---

## `locals.tf`

```hcl
locals {
  # A map of objects — iterated in main.tf using a dynamic block
  nsg_rules = {
    "allow_http" = {
      priority               = 100
      destination_port_range = "80"
      description            = "Allow HTTP"
    },
    "allow_https" = {
      priority               = 110
      destination_port_range = "443"
      description            = "Allow HTTPS"
    }
  }
}
```

---

## `main.tf` — Expressions in Action

### Reference Expressions

The most fundamental expression — reading a value from somewhere else.

```hcl
# Syntax:
#   var.<name>          → input variable
#   local.<name>        → local value
#   <type>.<name>.<attr>  → resource attribute
#   data.<type>.<name>.<attr>  → data source attribute

resource "azurerm_resource_group" "rg" {
  name     = "day10-rg"
  location = "westus2"
}
```

---

### String Interpolation & Template Expressions

Embed expressions inside strings using `${}`. Works anywhere a string is expected.

```hcl
# Simple variable interpolation
name = "${var.environment}-resources"   # → "dev-resources"

# Multiple values in one string
name = "${var.environment}-${var.location}-rg"

# Expression inside interpolation
name = "${var.environment == "prod" ? "production" : var.environment}-rg"
```

**Heredoc (multi-line strings):**
```hcl
description = <<-EOT
  This resource group belongs to the ${var.environment} environment.
  Managed by Terraform.
EOT
```

---

### Conditional (Ternary) Expression

```
condition ? value_if_true : value_if_false
```

Works exactly like a ternary operator in most languages. Condition must evaluate to `bool`.

```hcl
resource "azurerm_network_security_group" "example" {
  # If environment is "dev", name it "dev-nsg", otherwise "stage-nsg"
  name                = var.environment == "dev" ? "dev-nsg" : "stage-nsg"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}
```

**More examples:**
```hcl
# Toggle replication type based on environment
account_replication_type = var.environment == "prod" ? "GRS" : "LRS"

# Conditional count
count = var.environment == "prod" ? 3 : 1

# Nested ternary (use sparingly — hard to read)
vm_size = var.environment == "prod" ? "Standard_DS3_v2" : (var.environment == "staging" ? "Standard_DS2_v2" : "Standard_DS1_v2")
```

---

### `dynamic` Blocks

Used to **generate repeated nested blocks programmatically** from a collection — instead of writing each block manually.

Without `dynamic`, adding 10 NSG rules would need 10 hard-coded `security_rule {}` blocks. With `dynamic`, you drive them from a local or variable.

> ⚠️ Never overuse dynamic blocks — only use where the number of nested blocks is variable or driven by data. Static configurations are always clearer.

```hcl
resource "azurerm_network_security_group" "example" {
  name                = var.environment == "dev" ? "dev-nsg" : "stage-nsg"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  # dynamic "<nested_block_name>" iterates over a collection and
  # generates one nested block per element
  dynamic "security_rule" {
    for_each = local.nsg_rules   # map — iterates over each key-value pair

    content {
      # Inside dynamic, use <block_label>.key and <block_label>.value
      # (unlike for_each on resources where you use each.key / each.value)
      name                       = security_rule.key             # "allow_http", "allow_https"
      priority                   = security_rule.value.priority  # 100, 110
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = security_rule.value.destination_port_range  # "80", "443"
      source_address_prefix      = "*"
      destination_address_prefix = "*"
      description                = security_rule.value.description
    }
  }
}
```

**How it differs from `for_each` on a resource:**

| | `for_each` on resource | `dynamic` block |
|---|---|---|
| Scope | Creates multiple **resource instances** | Creates multiple **nested blocks** within one resource |
| Iterator | `each.key` / `each.value` | `<block_label>.key` / `<block_label>.value` |
| Use case | Multiple storage accounts, VMs, etc. | Multiple security rules, ingress rules, tags, etc. |

---

## `outputs.tf` — Output Expressions

### Basic Reference Output

```hcl
output "resource_group_name" {
  value = azurerm_resource_group.rg.name   # simple attribute reference
}

output "env" {
  value = var.environment
}
```

---

### `for` Expression

Transforms or filters a collection and returns a new collection. Think of it as a map/filter operation.

```
# Returns a list
[ for <item> in <collection> : <transform> ]

# Returns a list with filter
[ for <item> in <collection> : <transform> if <condition> ]

# Returns a map
{ for <key>, <val> in <map> : <new_key> => <new_val> }
```

```hcl
# Collect just the description field from each rule in the map
output "demo" {
  value = [for rule in local.nsg_rules : rule.description]
  # → ["Allow HTTP", "Allow HTTPS"]
}

# Iterate over for_each resource instances and collect names
output "storage_name" {
  value = [for i in azurerm_storage_account.example : i.name]
}

# for with a filter — only rules with priority < 110
output "low_priority_rules" {
  value = [for k, v in local.nsg_rules : k if v.priority < 110]
  # → ["allow_http"]
}

# Transform a list to uppercase (using built-in function)
output "upper_accounts" {
  value = [for name in var.account_names : upper(name)]
  # → ["SATYAM", "SHIVAM", "SATYAMSHIVAM"]
}

# Return a map — key → port mapping
output "rule_ports" {
  value = { for k, v in local.nsg_rules : k => v.destination_port_range }
  # → { "allow_http" = "80", "allow_https" = "443" }
}
```

---

### Splat Expression `[*]`

A shorthand for collecting one attribute across all instances of a `count`-based resource or list. More concise than a `for` expression for simple collection.

```hcl
# Splat on a list variable — collect element at index 1
output "splat" {
  value = var.account_names[1]   # → "shivam" (index access, not splat)
}

# Splat on count-based resource — collect all names
output "all_rg_names" {
  value = azurerm_resource_group.rg[*].name
}

# Splat on a list of objects
output "all_locations" {
  value = azurerm_resource_group.rg[*].location
}
```

> **Splat `[*]` vs `for` expression:**
> - `[*]` — works on `count`-based resources and lists; concise single-attribute collection
> - `for` — works on `for_each`-based resources (keyed maps); supports transformation, filtering, and map output
> - `[*]` does **not** work directly on maps (like `local.nsg_rules`) because maps don't have a numeric index

```hcl
# This does NOT work — nsg_rules is a map, not a list:
# value = local.nsg_rules[*]   ← invalid

# Use for instead:
output "security_rules_output" {
  value = azurerm_network_security_group.example.security_rule
}
```

---

## Built-in Functions — Commonly Used

Terraform has a large standard library of functions. These are the ones you'll reach for most often.

### String Functions

```hcl
upper("dev")              # → "DEV"
lower("PROD")             # → "prod"
trimspace("  dev  ")      # → "dev"
replace("dev-rg", "-", "_")  # → "dev_rg"
format("%.3s", "staging") # → "sta"  (format like printf)
join(", ", ["a","b","c"]) # → "a, b, c"
split(",", "a,b,c")       # → ["a", "b", "c"]
```

### Collection Functions

```hcl
length(var.account_names)               # → 3
contains(var.account_names, "satyam")   # → true
element(var.account_names, 0)           # → "satyam"  (safe index access)
index(var.account_names, "shivam")      # → 1  (find position of element)
flatten([["a","b"], ["c"]])             # → ["a","b","c"]  (flatten nested lists)
distinct(["a","b","a","c"])             # → ["a","b","c"]  (remove duplicates)
toset(var.account_names)               # → set (removes duplicates, unordered)
tolist(var.some_set)                   # → list (ordered, index-accessible)
```

### Map Functions

```hcl
# lookup(map, key, default) — safe key access with fallback
lookup(var.resource_tags, "environment", "unknown")   # → "staging"
lookup(var.resource_tags, "missing_key", "default")   # → "default"  (no error)

# merge — combine multiple maps (later maps override earlier on key conflict)
merge(
  { environment = "dev", managed_by = "terraform" },
  { environment = "prod", team = "platform" }
)
# → { environment = "prod", managed_by = "terraform", team = "platform" }

# keys / values — extract keys or values from a map as a list
keys(local.nsg_rules)    # → ["allow_http", "allow_https"]
values(local.nsg_rules)  # → [{ priority=100, ... }, { priority=110, ... }]
```

### Numeric Functions

```hcl
max(10, 20, 5)    # → 20
min(10, 20, 5)    # → 5
ceil(1.2)         # → 2
floor(1.9)        # → 1
abs(-5)           # → 5
```

### Type Conversion Functions

```hcl
tostring(42)       # → "42"
tonumber("42")     # → 42
tobool("true")     # → true
tolist(toset(...)) # convert set → list for index access
tomap({...})       # ensure value is treated as map type
```

### Filesystem / Encoding Functions

```hcl
file("./scripts/init.sh")             # read a local file into a string
filebase64("./certs/cert.pem")        # read file and base64-encode it
base64encode("hello")                 # → "aGVsbG8="
jsonencode({ key = "value" })         # → "{\"key\":\"value\"}"
jsondecode("{\"key\":\"value\"}")     # → { key = "value" }
yamlencode({ key = "value" })         # → "key: value\n"
```

---

## Expressions — Quick Reference

| Expression | Syntax | Returns | Example |
|------------|--------|---------|---------|
| Reference | `var.name`, `local.name`, `resource.type.name.attr` | any | `var.environment` |
| Interpolation | `"${expr}"` | string | `"${var.env}-rg"` |
| Conditional | `cond ? a : b` | any | `var.env == "prod" ? "GRS" : "LRS"` |
| for (list) | `[for x in col : expr]` | list | `[for r in local.nsg_rules : r.description]` |
| for (map) | `{for k,v in col : k => expr}` | map | `{for k,v in local.nsg_rules : k => v.priority}` |
| for (filter) | `[for x in col : expr if cond]` | list | `[for k,v in rules : k if v.priority < 110]` |
| Splat | `resource[*].attr` | list | `azurerm_resource_group.rg[*].name` |
| Index | `list[n]`, `map["key"]` | element | `var.account_names[1]` |
| dynamic block | `dynamic "block" { for_each = ... content { } }` | nested blocks | NSG security rules |