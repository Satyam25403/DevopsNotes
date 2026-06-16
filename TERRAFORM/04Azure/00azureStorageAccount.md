# Azure Storage Account Creation

## Basic Provider Setup

**main.tf:**
```hcl
terraform {
  # Specifies the providers that Terraform needs to download and use
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"   # Official Azure provider maintained by HashiCorp
      version = "~> 4.8.0"            # Uses version 4.8.x (allows patch updates but not major/minor breaking updates)
    }
  }

  # Specifies the Terraform CLI version required to run this configuration
  required_version = ">= 1.9.8"
}

provider "azurerm" {
  # Provider configuration block for Azure Resource Manager (ARM)
  # In real projects, we do not use a root/owner user account for provisioning.
  # Instead, we create a Service Principal (or Managed Identity), assign the required IAM role
  # (like Contributor or a custom role) at Subscription/Resource Group level,
  # and use those credentials to authenticate Terraform.
  # This follows the principle of least privilege.
  features {}
}
```

## Resource Group and Storage Account Example

```hcl
resource "azurerm_resource_group" "example" {
  # "example" is the local Terraform resource name.
  # It is not the actual Azure Resource Group name.
  # Terraform uses this name internally so that other resources can reference it.
  # Syntax: <resource_type>.<local_name>.<attribute>
  # Example: azurerm_resource_group.example.name

  name     = "example-resource"  # Actual name of the Resource Group created in Azure
  location = "West Europe"       # Azure region where the Resource Group will be deployed
}

resource "azurerm_storage_account" "account" {
  # "account" is the Terraform local name used for internal references.
  # The actual storage account name is defined by the "name" attribute below.

  # Storage account names must be globally unique across all Azure subscriptions.
  # Rules:
  # - Only lowercase letters and numbers are allowed
  # - No spaces, hyphens, or underscores
  # - Length must be between 3 and 24 characters

  name = "mystorageacc12345"

  # Implicit dependency:
  # Terraform understands that the Storage Account depends on the Resource Group
  # because it references attributes of azurerm_resource_group.example.
  # Therefore, Terraform automatically creates the Resource Group first.

  resource_group_name = azurerm_resource_group.example.name

  # Another implicit dependency.
  # The Storage Account gets the same Azure region as the Resource Group.

  location = azurerm_resource_group.example.location

  # Defines the performance tier of the Storage Account.
  # Standard = HDD-backed storage (cheaper)
  # Premium = SSD-backed storage (higher performance)

  account_tier = "Standard"

  # Defines how Azure replicates storage data.
  # GRS (Geo-Redundant Storage):
  # - Stores 3 copies in the primary region
  # - Replicates another 3 copies to a secondary region
  # - Provides protection against regional outages

  account_replication_type = "GRS"

  # Tags are key-value pairs used for organizing and managing Azure resources.
  # Commonly used for environment, owner, project, cost allocation, etc.

  tags = {
    environment = "staging"
  }
}
```

> **Note on implicit dependencies:** whenever a resource block references an attribute of another resource (e.g. `azurerm_resource_group.example.name`), Terraform automatically builds a dependency graph and creates/updates resources in the correct order — no explicit `depends_on` is needed for these cases.

## Azure Authentication Variables

**variables.tf:**
```hcl
# Azure Subscription ID
variable "subscription_id" {
  description = "Azure Subscription ID used by Terraform"
  type        = string
  sensitive   = true
}

# Azure Tenant ID (Azure Entra ID tenant)
variable "tenant_id" {
  description = "Azure Tenant ID"
  type        = string
  sensitive   = true
}

# Service Principal Application (Client) ID
variable "client_id" {
  description = "Azure Service Principal Client ID"
  type        = string
  sensitive   = true
}

# Service Principal Secret (Password)
variable "client_secret" {
  description = "Azure Service Principal Client Secret"
  type        = string
  sensitive   = true
}
```

**terraform.tfvars** (⚠️ never commit to git):
```hcl
subscription_id = "your-subscription-id"
tenant_id       = "your-tenant-id"
client_id       = "your-service-principal-client-id"
client_secret   = "your-service-principal-secret"
```

## Steps to Provision Azure Infrastructure

```bash
# Initialize the infrastructure — just like .git/, this creates a .terraform/ directory
# This directory holds the provider binary that translates Terraform code into
# the API calls/language that the cloud provider (Azure) understands
terraform init

# Run syntax and configuration validations
terraform validate

# Show what Terraform WILL do when you apply — preview the plan
terraform plan

# More precisely, to see only the resources that will be created:
terraform plan | grep "will be created"

# Apply changes without an interactive yes/no prompt
terraform apply --auto-approve
```

The errors/successes shown in the console are the results of REST API calls made to the cloud provider.

Every time `terraform apply` is run, Terraform compares the **desired state** (your `.tf` configuration) with the **actual state** (what currently exists in the cloud / in `terraform.tfstate`), and only applies the difference.

```bash
# Clean up — destroys all resources tracked in the current state
terraform destroy
```

## Industry Standard for Secrets (Recommended)

In real production environments, we usually avoid `terraform.tfvars` for secrets. Instead, we use one of:

- Azure DevOps Variable Groups
- GitHub Actions Secrets
- Azure Key Vault
- Environment variables

**Example using environment variables:**
```bash
export ARM_SUBSCRIPTION_ID="xxxx-xxxx-xxxx"
export ARM_TENANT_ID="xxxx-xxxx-xxxx"
export ARM_CLIENT_ID="xxxx-xxxx-xxxx"
export ARM_CLIENT_SECRET="xxxxxxxx"
```

When these `ARM_*` environment variables are set, Terraform's AzureRM provider automatically picks them up — so the provider block can simply be:

```hcl
provider "azurerm" {
  features {}
}
```

This keeps credentials out of source control entirely while still letting Terraform authenticate against Azure.