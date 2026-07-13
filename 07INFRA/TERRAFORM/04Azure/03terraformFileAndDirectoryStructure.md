# Terraform File Structure & Dependencies

## The Monolithic `main.tf` (Starting Point)

When learning Terraform, it's common to dump everything into a single `main.tf`. Here's what that looks like for a typical Azure setup:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.8.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "tfstate-day04"
    storage_account_name = "day0417691"
    container_name       = "tfstate"
    key                  = "dev.terraform.tfstate"
  }
  required_version = ">= 1.9.0"
}

provider "azurerm" {
  features {}
}

variable "environment" {
  type        = string
  description = "Environment type"
  default     = "staging"
}

locals {
  common_tags = {
    environment = "dev"
    lob         = "banking"
    stage       = "alpha"
  }
}

resource "azurerm_resource_group" "example" {
  name     = "example-resource"
  location = "West Europe"
}

resource "azurerm_storage_account" "account" {
  name                     = "mystorageacc12345"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  tags = {
    environment = var.environment
  }
}

output "storage_account_name" {
  value = azurerm_storage_account.account.name
}
```

This works, but it gets unwieldy fast. The solution is to **split it into purpose-specific files**.

---

## Recommended File Structure

```
terraform-project/
├── provider.tf          # Terraform block + provider config
├── backend.tf           # Remote state backend config
├── variables.tf         # Input variable declarations
├── terraform.tfvars     # Variable values (⚠️ never commit secrets to git)
├── locals.tf            # Local variable definitions
├── main.tf              # Core resources
├── outputs.tf           # Output values
├── .terraform/          # Provider plugins (auto-generated — do not edit)
├── terraform.tfstate    # State file (auto-generated — do not edit)
└── .terraform.lock.hcl  # Dependency lock file (auto-generated — do not edit)
```

> You can also go further and create **a separate `.tf` file per resource** (e.g. `storage_account.tf`, `resource_group.tf`) — useful when the project grows large.

---

## Breaking It Down: File by File

### `provider.tf`
Terraform core settings and provider requirements.

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

### `backend.tf`
Remote state backend configuration. Kept separate because it almost never changes after initial setup.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-day04"
    storage_account_name = "day0417691"
    container_name       = "tfstate"
    key                  = "dev.terraform.tfstate"
  }
}
```

---

### `locals.tf`
Locally computed values reused across the configuration.

```hcl
locals {
  common_tags = {
    environment = "dev"
    lob         = "banking"
    stage       = "alpha"
  }
}
```

---

### `variables.tf`
Input variable declarations — types, descriptions, and defaults.

```hcl
variable "environment" {
  type        = string
  description = "Environment type"
  default     = "staging"
}
```

Or supply values via `terraform.tfvars`:

```hcl
environment = "production"
```

---

### `main.tf` (resources)
The actual infrastructure resources. Optionally split further into one file per resource type.

```hcl
resource "azurerm_resource_group" "example" {
  name     = "example-resource"
  location = "West Europe"
}

resource "azurerm_storage_account" "account" {
  name                     = "mystorageacc12345"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  tags = {
    environment = var.environment
  }
}
```

---

### `outputs.tf`
Values to expose after `terraform apply`.

```hcl
output "storage_account_name" {
  value = azurerm_storage_account.account.name
}
```

---

## Why Split Files?

Some files like `backend.tf` and `provider.tf` almost **never change** once set up. Splitting them out means:

- Git reviews are scoped and focused — a reviewer only needs to look at what actually changed
- Junior team members can safely edit `main.tf` or `variables.tf` without touching critical config
- Easier to reuse and maintain across projects

---

## File Load Order & Dependencies

By default, Terraform loads all `.tf`, `.tfvars`, and `.tfvars.json` files **in alphabetical order**. This means if you leave things to chance, Terraform might try to create a resource before its dependency exists.

To handle this, Terraform supports two types of dependencies:

---

## Implicit Dependencies

Terraform **automatically detects** a dependency when one resource references an attribute of another. No extra syntax needed — Terraform builds its own dependency graph from these references and always creates resources in the right order.

### Example

```hcl
resource "azurerm_resource_group" "example" {
  name     = "example-resource"
  location = "West Europe"
}

resource "azurerm_storage_account" "account" {
  name                     = "mystorageacc12345"
  resource_group_name      = azurerm_resource_group.example.name      # implicit dependency
  location                 = azurerm_resource_group.example.location   # implicit dependency
  account_tier             = "Standard"
  account_replication_type = "GRS"

  tags = {
    environment = "dev"
  }
}
```

Here, `azurerm_storage_account.account` references `azurerm_resource_group.example.name` and `azurerm_resource_group.example.location`. Terraform sees these references and knows it must create the Resource Group **first**, before the Storage Account — automatically, without you having to say so explicitly.

This is the **preferred approach**. Use it whenever possible.

---

## Explicit Dependencies (`depends_on`)

Sometimes two resources are related but don't reference each other's attributes directly — Terraform has no way to infer the dependency on its own. In these cases, use `depends_on` to declare it manually.

> ⚠️ **Avoid explicit dependencies as much as you can.** They make your config harder to read and maintain. Reach for `depends_on` only when there is no attribute reference that can encode the relationship implicitly.

### Example

Imagine a Storage Account that needs a certain Role Assignment to exist before it's useful, but doesn't reference it directly in any attribute:

```hcl
resource "azurerm_resource_group" "example" {
  name     = "example-resource"
  location = "West Europe"
}

resource "azurerm_role_assignment" "storage_contributor" {
  scope                = azurerm_resource_group.example.id
  role_definition_name = "Storage Account Contributor"
  principal_id         = "00000000-0000-0000-0000-000000000000"
}

resource "azurerm_storage_account" "account" {
  name                     = "mystorageacc12345"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "GRS"

  tags = {
    environment = "dev"
  }

  # No attribute reference to azurerm_role_assignment.storage_contributor exists above,
  # so Terraform wouldn't know to wait for it. We declare the dependency explicitly.
  depends_on = [azurerm_role_assignment.storage_contributor]
}
```

Terraform will now ensure the Role Assignment is fully created before it attempts to create the Storage Account — even though there's no direct attribute reference between them.

---

## Summary: Implicit vs Explicit

| | Implicit | Explicit (`depends_on`) |
|---|---|---|
| How declared | Via attribute reference (`resource_type.name.attribute`) | Via `depends_on = [...]` |
| Terraform detects automatically? | ✅ Yes | ❌ No — you must declare it |
| When to use | Whenever a resource uses another's attribute | When resources are related but share no direct attribute reference |
| Recommended? | ✅ Always prefer this | ⚠️ Only as a last resort |