# Azure CLI (Command Line Interface)

Azure CLI is a cross-platform tool for managing Azure resources from your terminal. It's the Azure equivalent of AWS CLI — perfect for scripting, automation, and moving faster than clicking through the Azure Portal.

---

## What You Can Do with Azure CLI

- **Manage Azure resources**: VMs, storage accounts, App Services, AKS clusters, and more
- **Automate workflows**: Shell scripts or CI/CD pipelines to deploy infrastructure
- **Query and filter data**: JSON output with JMESPath queries (same as AWS CLI)
- **Switch subscriptions and tenants**: Work across multiple Azure accounts with ease

---

## 1. Installation

### Linux
```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### Windows (PowerShell)
```powershell
winget install -e --id Microsoft.AzureCLI
```

### macOS
```bash
brew install azure-cli
```

### Verify Installation
```bash
az version
```

---

## 2. Authentication

```bash
# Interactive browser login
az login

# Service principal login (for CI/CD)
az login --service-principal \
  --username <appId> \
  --password <password> \
  --tenant <tenantId>

# Managed Identity login (inside Azure VMs/containers)
az login --identity
```

---

## 3. Configuration

```bash
# Set default subscription
az account set --subscription "My Subscription"

# List all subscriptions
az account list --output table

# Set default resource group and location
az configure --defaults group=myResourceGroup location=eastus

# View current defaults
az configure --list-defaults
```

---

## 4. Core Concepts: Subscriptions, Resource Groups, Locations

| Concept | Description |
|---------|-------------|
| **Tenant** | Your Azure Active Directory (Entra ID) organization |
| **Subscription** | Billing + resource boundary (analogous to an AWS account) |
| **Resource Group** | Logical container for related resources (no direct AWS equivalent — AWS uses tags/accounts) |
| **Location/Region** | Geographic data center (e.g., `eastus`, `westeurope`) |

> **Resource Groups are unique to Azure.** Every resource must live in one. Think of it as a folder that you can delete all at once — deleting a resource group deletes everything in it.

---

## 5. Common Commands

### Resource Groups
```bash
az group create --name myRG --location eastus
az group list --output table
az group delete --name myRG --yes
```

### Virtual Machines (analogous to EC2 in AWS)
```bash
az vm create \
  --resource-group myRG \
  --name myVM \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys

az vm start --resource-group myRG --name myVM
az vm stop --resource-group myRG --name myVM
az vm list --output table
```

### Storage
```bash
az storage account create \
  --name mystorageacct \
  --resource-group myRG \
  --location eastus \
  --sku Standard_LRS
```

---

## 6. Output Formats

```bash
az vm list --output json        # Full JSON (default)
az vm list --output table       # Human-readable table
az vm list --output tsv         # Tab-separated (for scripting)
az vm list --output yaml        # YAML
```

---

## 7. JMESPath Queries (same syntax as AWS CLI)

```bash
# Get just VM names
az vm list --query "[].name" --output tsv

# Get name + location pairs
az vm list --query "[].{Name:name, Location:location}" --output table

# Filter by location
az vm list --query "[?location=='eastus'].name" --output tsv
```

---

## 8. Azure Cloud Shell (analogous to AWS CloudShell)

Azure Cloud Shell is a browser-based shell pre-authenticated with your Azure account.

- Launch from the Azure Portal (terminal icon) or shell.azure.com
- Comes with Azure CLI, PowerShell, Python, Node.js, kubectl, Helm, Terraform pre-installed
- 5 GB persistent storage (mounted Azure file share)
- Supports both **Bash** and **PowerShell**

---

## 9. Extensions

Azure CLI supports installable extensions (like plugins):

```bash
az extension add --name aks-preview
az extension add --name containerapp
az extension list --output table
az extension update --name aks-preview
```

---

## Cheatsheet

| Task | Command |
|------|---------|
| Login | `az login` |
| Set subscription | `az account set --subscription "name"` |
| Create resource group | `az group create -n myRG -l eastus` |
| List resources in group | `az resource list -g myRG --output table` |
| Get help for any command | `az vm --help` |
| Interactive mode | `az interactive` |