# Terraform Data Types — Practical Demonstration

A hands-on walkthrough of all Terraform variable type constraints in use, provisioning a full Azure VM stack: Resource Group → VNet → Subnet → NIC → VM.

---

## File Structure

```
terraform-project/
├── provider.tf
├── backend.tf
├── locals.tf
├── variables.tf
├── main.tf
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

Backend config values can also be passed dynamically at `terraform init` time using `-backend-config` flags instead of hardcoding them here — useful for keeping environment-specific values out of source control.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-day04"    # or: -backend-config="resource_group_name=<value>"
    storage_account_name = "day0417691"        # or: -backend-config="storage_account_name=<value>"
    container_name       = "tfstate"           # or: -backend-config="container_name=<value>"
    key                  = "dev.terraform.tfstate"  # or: -backend-config="key=<value>"
  }
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

## `variables.tf` — All Type Constraints Explained

```hcl
# ── Primitive: string ────────────────────────────────────────────────────────
variable "environment" {
  type        = string
  description = "The environment type"
  default     = "staging"
}

# ── Primitive: number ────────────────────────────────────────────────────────
variable "storage_disk" {
  type        = number
  description = "OS disk size in GB"
  default     = 80
}

# ── Primitive: bool ──────────────────────────────────────────────────────────
variable "is_delete" {
  type        = bool
  description = "Whether to delete the OS disk when the VM is terminated"
  default     = true
}

# ── Collection: list(string) ─────────────────────────────────────────────────
# An ordered collection of the same data type. Allows duplicates.
# Elements are accessed by index: var.allowed_locations[0], [1], [2] ...
variable "allowed_locations" {
  type        = list(string)
  description = "List of allowed Azure regions"
  default     = ["West Europe", "North Europe", "East US"]
}

# ── Collection: list(string) — VM sizes ──────────────────────────────────────
variable "allowed_vm_sizes" {
  type        = list(string)
  description = "Allowed VM sizes"
  default     = ["Standard_DS1_v2", "Standard_DS2_v2", "Standard_DS3_v2"]
}

# ── Collection: map(string) ──────────────────────────────────────────────────
# Key-value pairs, all values of the same type.
# Elements are accessed by key: var.resource_tags["environment"]
variable "resource_tags" {
  type        = map(string)
  description = "Tags to apply to resources"
  default     = {
    "environment" = "staging"
    "managed_by"  = "terraform"
    "department"  = "devops"
  }
}

# ── Collection: set(string) ──────────────────────────────────────────────────
# Like list, but UNORDERED and NO duplicates — duplicates are silently dropped.
# Cannot be accessed by index. Consumed via for_each to iterate over unique elements: consumed as whole
variable "allowed_ports" {
  type        = set(string)
  description = "Set of allowed inbound ports — duplicates automatically removed"
  default     = ["22", "80", "443"]
}

# ── Structural: tuple([string, string, number]) ───────────────────────────────
# An ordered sequence of elements with DIFFERENT data types.
# Unlike list, each position has its own type.
# Elements are accessed by index: element(var.network_config, 0), (1), (2)
variable "network_config" {
  type        = tuple([string, string, number])
  description = "Network configuration: (VNet address space, subnet address, subnet mask)"
  default     = ["10.0.0.0/16", "10.0.2.0", 24]
}

# ── Structural: object({...}) ─────────────────────────────────────────────────
# A named collection of attributes, each with its own type.
# Fields are accessed by name: var.vm_config.sku, var.vm_config.version
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
```

---

## `main.tf` — Resources Using Every Variable Type

### Resource Group
```hcl
resource "azurerm_resource_group" "example" {
  name     = "${var.environment}-resources"   # string interpolation
  location = var.allowed_locations[2]         # list access by index → "East US"
}
```

### Virtual Network
```hcl
resource "azurerm_virtual_network" "main" {
  name                = "${var.environment}-network"
  address_space       = [element(var.network_config, 0)]   # tuple index 0 → "10.0.0.0/16"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
}
```

> `element(list, index)` is a built-in Terraform function — equivalent to `var.network_config[0]` but safer for dynamic index access.

### Subnet
```hcl
resource "azurerm_subnet" "internal" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.main.name

  # Concatenating different tuple fields: "10.0.2.0" + "/" + 24 → "10.0.2.0/24"
  address_prefixes = ["${element(var.network_config, 1)}/${element(var.network_config, 2)}"]
}
```

### Network Interface
```hcl
resource "azurerm_network_interface" "main" {
  name                = "${var.environment}-nic"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name

  ip_configuration {
    name                          = "testconfiguration1"
    subnet_id                     = azurerm_subnet.internal.id
    private_ip_address_allocation = "Dynamic"
  }
}
```

### Virtual Machine
```hcl
resource "azurerm_virtual_machine" "main" {
  name                  = "${var.environment}-vm"
  location              = azurerm_resource_group.example.location
  resource_group_name   = azurerm_resource_group.example.name
  network_interface_ids = [azurerm_network_interface.main.id]
  vm_size               = var.allowed_vm_sizes[0]         # list access → "Standard_DS1_v2"

  delete_os_disk_on_termination = var.is_delete           # bool variable

  storage_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = var.vm_config.sku                         # object field access → "22_04-lts"
    version   = var.vm_config.version                     # object field access → "latest"
  }

  storage_os_disk {
    name              = "myosdisk1"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Standard_LRS"
    disk_size_gb      = var.storage_disk                  # number variable → 80
  }

  os_profile {
    computer_name  = "hostname"
    admin_username = "testadmin"
    admin_password = "Password1234!"
  }

  os_profile_linux_config {
    disable_password_authentication = false
  }

  tags = {
    environment = var.resource_tags["environment"]        # map access by key → "staging"
    managed_by  = var.resource_tags["managed_by"]         # map access by key → "terraform"
    department  = var.resource_tags["department"]         # map access by key → "devops"
  }
}
```

### Network Security Group — `set` with `for_each`

`set` cannot be accessed by index. The correct way to consume it is `for_each` — Terraform creates **one resource instance per element** in the set:

```hcl
resource "azurerm_network_security_group" "main" {
  name                = "${var.environment}-nsg"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
}

resource "azurerm_network_security_rule" "allow_ports" {
  for_each = var.allowed_ports    # set access — iterates over each unique port

  name                        = "allow-port-${each.value}"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = each.value        # current element in iteration
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.example.name
  network_security_group_name = azurerm_network_security_group.main.name
}
```

This creates **three** `azurerm_network_security_rule` instances — one for port `22`, one for `80`, one for `443` — without repeating the resource block.

---

## Type Constraints — Quick Reference

| Type | Category | Allows Mixed Types | Ordered | Access Pattern | Example |
|------|----------|--------------------|---------|----------------|---------|
| `string` | Primitive | — | — | directly | `var.environment` |
| `number` | Primitive | — | — | directly | `var.storage_disk` |
| `bool` | Primitive | — | — | directly | `var.is_delete` |
| `list(T)` | Collection | ❌ same type only | ✅ yes | `var.name[index]` | `var.allowed_locations[2]` |
| `set(T)` | Collection | ❌ same type only | ❌ no | iteration only | — |
| `map(T)` | Collection | ❌ same type only | ❌ no | `var.name["key"]` | `var.resource_tags["environment"]` |
| `tuple([T...])` | Structural | ✅ yes | ✅ yes | `element(var.name, index)` | `element(var.network_config, 2)` |
| `object({...})` | Structural | ✅ yes | ❌ no | `var.name.field` | `var.vm_config.sku` |
| `any` | — | ✅ yes | — | depends on value | — |

---

## Dependency Chain in This Configuration

All five resources are linked through **implicit dependencies** — each references attributes of the one before it:

```
azurerm_resource_group
        │
        ▼
azurerm_virtual_network  (references resource group name + location)
        │
        ▼
azurerm_subnet           (references resource group name + vnet name)
        │
        ▼
azurerm_network_interface  (references resource group name + location + subnet id)
        │
        ▼
azurerm_virtual_machine  (references resource group name + location + NIC id)

azurerm_resource_group
        │
        ▼
azurerm_network_security_group   (references resource group name + location)
        │
        ▼
azurerm_network_security_rule    (references resource group name + NSG name — one instance per set element)
```

Terraform builds this graph automatically from the attribute references — no `depends_on` needed.