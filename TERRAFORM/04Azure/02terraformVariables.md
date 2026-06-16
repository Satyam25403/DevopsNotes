# Terraform Variables

Terraform has three kinds of variables, each serving a different purpose:

| Type | Purpose |
|------|---------|
| **Input** | Values you provide to parameterize your configuration — avoids hardcoding and repetition |
| **Output** | Values Terraform exposes after `apply` — useful for passing data between modules or just inspecting results |
| **Local** | Internal values computed once and reused within a module — not exposed externally |

---

## Variable Type Constraints

| Category | Types |
|----------|-------|
| Primitive | `string`, `number`, `bool` |
| Non-primitive | `list`, `set`, `map`, `object`, `tuple` |
| Unconstrained | `any` — used when type is not specified |

---

## Input Variables

We use input variables because we don't want to hardcode values and don't want to repeat the same values again and again across resources.

### Defining an Input Variable

```hcl
variable "environment" {
  type        = string
  description = "Environment type"
  default     = "staging"
}
```

### Using an Input Variable

Reference input variables with the `var.<name>` syntax:

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
    environment = var.environment   # using input variable
  }
}
```

### Ways to Supply Variable Values

**1. Default value in variable block** (used if nothing else is provided):
```hcl
variable "environment" {
  default = "staging"
}
```

**2. `terraform.tfvars` file** (generally used for secrets — ⚠️ never commit to git):
```hcl
environment = "demo"
```

**3. `terraform.tfvars.json` file:**
```json
{
  "environment": "demo"
}
```

**4. Environment variables** — prefix with `TF_VAR_`:
```bash
export TF_VAR_environment=production
terraform plan
```

**5. `-var` or `-var-file` flag at runtime** (useful for overrides):
```bash
terraform plan -var="environment=dev"
terraform apply -var-file="prod.tfvars"
```

### Variable Precedence

When the same variable is set in multiple places, Terraform resolves it in this order — **lower in the list wins**:

| Priority | Source |
|----------|--------|
| 1 (lowest) | Environment variables (`TF_VAR_*`) |
| 2 | `terraform.tfvars` file |
| 3 | `terraform.tfvars.json` file |
| 4 (highest) | `-var` or `-var-file` on the command line |

So a `-var` flag on the CLI will always override anything set in `.tfvars` or environment variables.

---

## Output Variables

Output variables expose values after `terraform apply` — useful for inspecting resource attributes or passing data to other modules.

### Defining an Output

```hcl
output "storage_account_name" {
  value = azurerm_storage_account.account.name
}
```

### Viewing Outputs

```bash
# Outputs are printed automatically at the end of terraform apply

# Or query them explicitly after apply
terraform output

# Query a specific output
terraform output storage_account_name

# Machine-readable JSON format
terraform output -json
```

---

## Local Variables

Local variables are for values that are used internally within a module but not exposed as inputs or outputs. They're great for grouping repeated values (like common tags) into one place.

### Defining Locals

```hcl
locals {
  common_tags = {
    environment = "dev"
    lob         = "banking"
    stage       = "alpha"
  }
}
```

### Using Locals

Reference locals with the `local.<name>` syntax (no `s` — `local`, not `locals`):

```hcl
resource "azurerm_storage_account" "account" {
  # ...
  tags = {
    environment = local.common_tags.environment
    lob         = local.common_tags.lob
    stage       = local.common_tags.stage
  }
}
```

Or pass the whole map directly:

```hcl
tags = local.common_tags
```

> **Tip:** Locals are evaluated once and reused — unlike input variables, they can reference other Terraform expressions, resource attributes, and even other locals. Use them to avoid duplicating complex expressions across your config.