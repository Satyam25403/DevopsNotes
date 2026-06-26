# Terraform Resource Meta-Arguments

Meta-arguments are special arguments available on **every** resource block regardless of resource type. They control how Terraform creates, updates, and manages resources — not what the resource is, but how it behaves.

```
depends_on  →  explicit dependency ordering
count       →  create N copies of a resource
for_each    →  create one resource per element in a set/map
provider    →  pin a resource to a specific provider instance
lifecycle   →  control create/update/destroy behaviour
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
variable "environment" {
  type        = string
  description = "The environment type"
  default     = "staging"
}

variable "storage_disk" {
  type        = number
  description = "OS disk size in GB"
  default     = 80
}

variable "is_delete" {
  type        = bool
  description = "Whether to delete the OS disk on VM termination"
  default     = true
}

variable "allowed_locations" {
  type        = list(string)
  description = "List of allowed Azure regions"
  default     = ["West Europe", "North Europe", "East US"]
}

variable "resource_tags" {
  type        = map(string)
  description = "Tags to apply to resources"
  default     = {
    "environment" = "staging"
    "managed_by"  = "terraform"
    "department"  = "devops"
  }
}

variable "network_config" {
  type        = tuple([string, string, number])
  description = "Network configuration: (VNet address space, subnet address, subnet mask)"
  default     = ["10.0.0.0/16", "10.0.2.0", 24]
}

variable "allowed_vm_sizes" {
  type        = list(string)
  description = "Allowed VM sizes"
  default     = ["Standard_DS1_v2", "Standard_DS2_v2", "Standard_DS3_v2"]
}

variable "vm_config" {
  type = object({
    size      = string
    publisher = string
    offer     = string
    sku       = string
    version   = string
  })
  description = "Virtual machine image and size configuration"
  default = {
    size      = "Standard_DS1_v2"
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }
}

# set(string) — unique strings only, duplicates automatically dropped
variable "storage_account_name" {
  type    = set(string)
  default = ["satyam", "shivam"]
}
```

---

## `locals.tf`

```hcl
locals {
  common_tags = {
    environment = var.environment
    lob         = "banking"
    stage       = "alpha"
  }
}
```

---

## `main.tf` — Meta-Arguments in Action

### `depends_on` — Explicit Dependency

Forces Terraform to create/destroy resources in a specific order even when no attribute reference exists between them. Use only when implicit dependency isn't possible.

```hcl
resource "azurerm_resource_group" "example" {
  name     = "${var.environment}-resources"
  location = var.allowed_locations[2]   # "East US"
}

resource "azurerm_role_assignment" "rg_contributor" {
  scope                = azurerm_resource_group.example.id
  role_definition_name = "Contributor"
  principal_id         = "00000000-0000-0000-0000-000000000000"
}

resource "azurerm_storage_account" "manual_dep" {
  name                     = "satdepexample"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  # No attribute references azurerm_role_assignment above,
  # so Terraform wouldn't wait for it without this explicit declaration.
  depends_on = [azurerm_role_assignment.rg_contributor]
}
```

> ⚠️ Prefer implicit dependencies (attribute references) over `depends_on` wherever possible. Explicit dependencies make configs harder to read and maintain.

---

### `count` — Create N Copies of a Resource

`count` is the default meta-argument for creating multiple identical (or near-identical) copies of a resource. Each copy is indexed starting from `0`.

```hcl
resource "azurerm_storage_account" "count_example" {
  count = length(var.storage_account_name)   # creates as many accounts as there are names in the set

  name                     = tolist(var.storage_account_name)[count.index]   # access by index
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  tags = {
    environment = "staging"
  }
}
```

**Limitation of `count`:** resources are tracked by index (`[0]`, `[1]`...). If you remove an element from the middle of the list, Terraform shifts all subsequent indices and **recreates** those resources — which is destructive and unexpected. This is why `for_each` is generally preferred.

---

### `for_each` — Create One Resource Per Element

`for_each` iterates over a **set** or **map** and creates one resource instance per element. Resources are tracked by their key/value — not by index — so adding or removing elements doesn't affect other instances.

```hcl
resource "azurerm_storage_account" "example" {
  for_each = var.storage_account_name   # set(string) — works directly with for_each

  # each.key   → the element itself (for sets, key == value)
  # each.value → the element itself (for sets, key == value)
  name = each.value

  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  tags = {
    environment = "staging"
  }
}
```

> **Why `set` and not `list` with `for_each`?**
> `for_each` does not accept lists because lists can contain duplicates — and Terraform needs a unique key per instance. Use `set(string)` directly, or convert a list with `toset()`:
> ```hcl
> for_each = toset(var.some_list)
> ```

#### `count` vs `for_each` — When to Use Which

| | `count` | `for_each` |
|---|---|---|
| Input type | `number` | `set` or `map` |
| Tracks instances by | Index (`[0]`, `[1]`) | Key (`["satyam"]`, `["shivam"]`) |
| Safe to add/remove elements | ❌ shifts indices, may recreate | ✅ only affects changed element |
| Best for | Identical copies, simple repetition | Named/unique resources |

---

### `provider` — Pin to a Specific Provider Instance

By default, a resource uses the default provider for its type. Use `provider` to explicitly route a resource to an aliased provider — useful for multi-region or multi-account deployments.

```hcl
# Define two provider instances — one per region
provider "azurerm" {
  features {}
  # default instance → used by most resources
}

provider "azurerm" {
  alias    = "west_europe"
  features {}
  # secondary instance pinned to a different region/subscription
}

# This resource uses the default provider
resource "azurerm_resource_group" "primary" {
  name     = "primary-rg"
  location = "East US"
}

# This resource is explicitly pinned to the west_europe provider alias
resource "azurerm_resource_group" "secondary" {
  name     = "secondary-rg"
  location = "West Europe"

  provider = azurerm.west_europe   # overrides the default provider
}
```

---

### `lifecycle` — Control Create / Update / Destroy Behaviour

`lifecycle` lets you override Terraform's default behaviour when a resource needs to change or be destroyed. It is a nested block supported inside any resource.

```
Available lifecycle rules:
  create_before_destroy  →  minimize downtime during replacement
  prevent_destroy        →  guard against accidental deletion
  ignore_changes         →  ignore external drift on specific attributes
  replace_triggered_by   →  force replacement when a dependency changes
  precondition           →  validate inputs before resource is created/updated
  postcondition          →  validate outputs after resource is created/updated
```

---

#### 1. `create_before_destroy`

**Default Terraform behaviour:** destroy old resource → create new one.
**With `create_before_destroy = true`:** create new resource first → destroy old one only after the new one is ready.

Use this for production resources where even a brief gap causes downtime — load balancer rules, DNS records, storage accounts, etc.

```hcl
resource "azurerm_resource_group" "example" {
  name     = "${var.environment}-resources"
  location = var.location

  tags = {
    environment = var.environment
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

> If `var.location` changes, Terraform must replace the resource group. With this rule, it creates the new one in the new location first, then deletes the old one — instead of deleting first and leaving a gap.

---

#### 2. `prevent_destroy`

Blocks **any** `terraform destroy` or `terraform apply` operation that would destroy this resource. Terraform throws a hard error and halts rather than proceeding.

Set this to `true` on critical, stateful resources — production databases, storage accounts holding state, Key Vaults, etc. Set to `false` (or omit entirely) for non-critical or ephemeral resources.

```hcl
resource "azurerm_resource_group" "example" {
  name     = "${var.environment}-resources"
  location = var.location

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false   # false here since this is a demo; set true for prod-critical resources
  }
}
```

> Even with `prevent_destroy = true`, the protection only applies while the `lifecycle` block exists in your `.tf` file. If someone removes the block and runs `apply`, the guard is gone. Treat it as a speed bump, not a lock — combine with remote state access controls for real protection.

---

#### 3. `ignore_changes`

Tells Terraform to **ignore drift** on the listed attributes. When Terraform runs `plan`, it will not flag a difference if those specific fields were changed outside of Terraform (via portal, CLI, autoscaler, etc.) and will not attempt to revert them.

```hcl
resource "azurerm_storage_account" "example" {
  for_each = var.storage_account_name

  name                     = each.value
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  tags = {
    environment = var.environment
  }

  lifecycle {
    ignore_changes = [
      account_replication_type   # if replication type is changed outside Terraform, do NOT revert it
    ]
  }
}
```

You can also use `ignore_changes = all` to ignore every attribute — though this effectively disables Terraform's drift detection on that resource and should be used sparingly.

```hcl
# ignore a specific tag key only
ignore_changes = [ tags["last_modified"] ]

# ignore the entire tags map
ignore_changes = [ tags ]

# ignore all attributes (use with extreme caution)
ignore_changes = all
```

---

#### 4. `replace_triggered_by`

Forces Terraform to **destroy and recreate** this resource whenever a referenced resource or attribute changes — even if this resource's own attributes haven't changed.

Useful when a downstream resource depends on something that Terraform can't detect automatically (e.g. a resource group being recreated, a certificate being rotated).

```hcl
resource "azurerm_storage_account" "example" {
  for_each = var.storage_account_name

  name                     = each.value
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  tags = {
    environment = var.environment
  }

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [account_replication_type]

    replace_triggered_by = [
      azurerm_resource_group.example.id   # if the resource group is replaced, replace the storage account too
    ]
  }
}
```

> Without `replace_triggered_by`, if the resource group is destroyed and recreated (new ID), Terraform might try to associate the existing storage account with a resource group that no longer exists — leading to errors. This rule forces a clean replacement cascade.

---

#### 5. Custom Conditions — `precondition` and `postcondition`

Custom conditions let you **validate your own rules** before or after Terraform acts on a resource. When a condition fails, Terraform raises the `error_message` and halts — much more informative than a cryptic API error from Azure.

**`precondition`** — runs **before** the resource is created or updated. Use it to validate inputs.

**`postcondition`** — runs **after** the resource is created or updated. Use it to validate outputs/results.

```hcl
resource "azurerm_resource_group" "example" {
  name     = "${var.environment}-resources"
  location = var.location

  tags = {
    environment = var.environment
  }

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = false

    # precondition: validated BEFORE the resource is created/updated
    # If var.location is not in the allowed list, Terraform errors out immediately
    # instead of sending an invalid location to the Azure API.
    precondition {
      condition     = contains(var.allowed_locations, var.location)
      error_message = "Please enter a valid location! Allowed: ${join(", ", var.allowed_locations)}"
    }
  }
}
```

**`postcondition` example** — validate the result after creation:

```hcl
resource "azurerm_storage_account" "example" {
  for_each = var.storage_account_name

  name                     = each.value
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  lifecycle {
    # postcondition: validated AFTER the resource is created
    # Ensures the storage account ended up in the right resource group
    postcondition {
      condition     = self.resource_group_name == azurerm_resource_group.example.name
      error_message = "Storage account was created in the wrong resource group!"
    }
  }
}
```

> Inside a `postcondition`, use `self.<attribute>` to reference the resource's own attributes after creation. `precondition` uses `var.*`, `local.*`, or other resource references — `self` is not available there since the resource doesn't exist yet.

---

#### `lifecycle` Rules — Full Reference

| Rule | When it runs | What it does | Best used for |
|------|-------------|--------------|---------------|
| `create_before_destroy` | On replacement | Creates new resource before deleting old | Zero-downtime replacements |
| `prevent_destroy` | On destroy/apply | Hard-errors if destroy would occur | Production-critical resources |
| `ignore_changes` | On plan | Skips drift detection for listed attributes | Externally managed fields |
| `replace_triggered_by` | On plan | Forces replacement when dependency changes | Cascading replacements |
| `precondition` | Before create/update | Validates inputs before acting | Input validation (location, naming, etc.) |
| `postcondition` | After create/update | Validates outputs after acting | Output/result validation |

---

## `outputs.tf`

```hcl
# Splat expression [*] — collects the named attribute across all instances
output "rgname" {
  value = azurerm_resource_group.example[*].name
}

# for expression — iterate over for_each instances and extract a field
# 'i' is each storage account instance; i.name extracts the name attribute
output "storage_name" {
  value = [for i in azurerm_storage_account.example : i.name]
}
```

> **Splat `[*]` vs `for` expression:**
> - `[*]` works on `count`-based resources (indexed) — collects one attribute across all instances
> - `for` expression works on `for_each`-based resources (keyed) — lets you transform and filter values during iteration

---

## Meta-Arguments — Summary

| Meta-Argument | Purpose | Scope |
|---------------|---------|-------|
| `depends_on` | Explicit ordering when no attribute reference exists | Any resource/module |
| `count` | Create N indexed copies | Any resource |
| `for_each` | Create one instance per set/map element, tracked by key | Any resource/module |
| `provider` | Pin resource to a specific aliased provider instance | Any resource |
| `lifecycle` | Override default create/update/destroy behaviour | Any resource |