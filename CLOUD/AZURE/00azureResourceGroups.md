# Azure Resource Groups
## (analogous to AWS — no direct equivalent; closest are AWS tags, CloudFormation stacks, or AWS Organizations OUs)

A Resource Group is a **logical container** for related Azure resources. Every resource in Azure — a VM, a storage account, a Key Vault, a Function App — must live in exactly one resource group. It is the fundamental unit of lifecycle management, access control, and cost tracking in Azure.

> The closest AWS concept is a **CloudFormation Stack** (grouped resources with shared lifecycle) combined with **resource tags** (for filtering/billing). But Azure Resource Groups are more deeply integrated — RBAC, cost analysis, and deletion are all native to the group, not bolted on.

---

## What Resource Groups Are For

| Purpose | How Resource Groups Help |
|---------|--------------------------|
| **Lifecycle management** | Delete a resource group → deletes every resource in it, atomically |
| **Access control** | Assign RBAC roles at the group level; all resources inherit them |
| **Cost tracking** | Filter Azure Cost Analysis by resource group |
| **Deployment scope** | ARM/Bicep templates deploy to a resource group by default |
| **Grouping by environment** | One RG per environment (`myapp-dev`, `myapp-staging`, `myapp-prod`) |
| **Grouping by workload** | One RG per service (`payments-rg`, `auth-rg`, `data-rg`) |

---

## Core Rules

- Every resource belongs to **exactly one** resource group.
- A resource group itself lives in a **region** (metadata storage), but can contain resources from **any region**.
- Deleting a resource group **permanently deletes all resources** inside it — there is no recycle bin.
- Resource groups **cannot be nested**.
- Moving a resource between resource groups is supported but has per-service caveats (see below).

---

## Creating and Managing Resource Groups

```bash
# Create a resource group
az group create \
  --name myapp-prod \
  --location eastus

# List all resource groups
az group list --output table

# Show details of a resource group
az group show --name myapp-prod

# Check if a resource group exists
az group exists --name myapp-prod

# Update tags on a resource group
az group update \
  --name myapp-prod \
  --tags Environment=Production Team=Backend CostCenter=CC-1042

# Delete a resource group (and ALL resources in it — irreversible)
az group delete --name myapp-prod --yes --no-wait
```

---

## Listing Resources in a Group

```bash
# List all resources in a group
az resource list \
  --resource-group myapp-prod \
  --output table

# Filter by resource type
az resource list \
  --resource-group myapp-prod \
  --resource-type Microsoft.Compute/virtualMachines \
  --output table

# List with useful columns (name, type, location)
az resource list \
  --resource-group myapp-prod \
  --query "[].{Name:name, Type:type, Location:location}" \
  --output table
```

---

## Tagging

Tags are key-value pairs on a resource group (and individual resources) used for cost allocation, automation, and governance. Unlike grouping strategy, tags are flexible — a resource can have up to 50 tags.

```bash
# Tag a resource group
az group update \
  --name myapp-prod \
  --tags \
    Environment=Production \
    Team=Backend \
    CostCenter=CC-1042 \
    ManagedBy=Terraform

# Tag an individual resource (overrides existing tags)
az resource tag \
  --resource-group myapp-prod \
  --name myVM \
  --resource-type Microsoft.Compute/virtualMachines \
  --tags Environment=Production Owner=alice@company.com

# Query all resources with a specific tag across the subscription
az resource list \
  --tag Environment=Production \
  --output table

# List resource groups by tag
az group list \
  --tag CostCenter=CC-1042 \
  --output table
```

---

## Locks (prevent accidental deletion or modification)

Resource locks are applied to a resource group (or individual resource) and block destructive operations regardless of RBAC permissions — even subscription owners are blocked.

| Lock Type | Effect |
|-----------|--------|
| **CanNotDelete** | Read and modify allowed; delete blocked |
| **ReadOnly** | Read allowed; all modifications and deletions blocked |

```bash
# Apply a delete lock to a production resource group
az lock create \
  --name no-delete \
  --resource-group myapp-prod \
  --lock-type CanNotDelete \
  --notes "Production environment — requires change request to delete"

# Apply a read-only lock (use carefully — breaks many operations)
az lock create \
  --name read-only \
  --resource-group myapp-prod \
  --lock-type ReadOnly

# List locks on a resource group
az lock list \
  --resource-group myapp-prod \
  --output table

# Remove a lock (required before deletion)
az lock delete \
  --name no-delete \
  --resource-group myapp-prod
```

> Locks are inherited by all resources in the group. To delete a locked resource group, you must first remove the lock.

---

## Role Assignments at Resource Group Scope

RBAC assignments at the resource group level apply to all resources within the group. This is the most common scope for granting access.

```bash
# Grant a user Contributor access to a resource group
az role assignment create \
  --assignee alice@company.com \
  --role Contributor \
  --resource-group myapp-prod

# Grant a managed identity Reader access
az role assignment create \
  --assignee <managed-identity-principal-id> \
  --role Reader \
  --resource-group myapp-prod

# List all role assignments in a resource group
az role assignment list \
  --resource-group myapp-prod \
  --output table

# Remove a role assignment
az role assignment delete \
  --assignee alice@company.com \
  --role Contributor \
  --resource-group myapp-prod
```

---

## Moving Resources Between Resource Groups

Resources can be moved between groups (or subscriptions) without redeployment. The resource keeps its Azure resource ID structure updated, but its configuration and data are unchanged.

```bash
# Move a VM and its disk to a different resource group
az resource move \
  --destination-group myapp-staging \
  --ids \
    /subscriptions/<sub-id>/resourceGroups/myapp-prod/providers/Microsoft.Compute/virtualMachines/myVM \
    /subscriptions/<sub-id>/resourceGroups/myapp-prod/providers/Microsoft.Compute/disks/myDisk

# Validate the move before executing it
az resource invoke-action \
  --action validateMoveResources \
  --ids /subscriptions/<sub-id>/resourceGroups/myapp-prod \
  --request-body '{
    "resources": [
      "/subscriptions/<sub-id>/resourceGroups/myapp-prod/providers/Microsoft.Compute/virtualMachines/myVM"
    ],
    "targetResourceGroup": "/subscriptions/<sub-id>/resourceGroups/myapp-staging"
  }'
```

> Not all resource types support moves. Check the [Azure docs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/move-support-resources) per resource type before moving. Key caveats: VNets with peering cannot be moved; Key Vaults may require additional steps; AKS clusters have restrictions.

---

## Deploying to a Resource Group (ARM / Bicep)

By default, ARM templates and Bicep files deploy at the resource group scope.

```bash
# Deploy a Bicep template to a resource group
az deployment group create \
  --resource-group myapp-prod \
  --template-file main.bicep \
  --parameters environment=prod appName=myapp

# Preview changes before deploying (what-if)
az deployment group what-if \
  --resource-group myapp-prod \
  --template-file main.bicep \
  --parameters environment=prod appName=myapp

# List all deployments in a resource group
az deployment group list \
  --resource-group myapp-prod \
  --output table

# Show deployment details (inputs, outputs, errors)
az deployment group show \
  --resource-group myapp-prod \
  --name main \
  --query "{Status:properties.provisioningState, Outputs:properties.outputs}"
```

---

## Cost Analysis by Resource Group

```bash
# Get cost for a resource group for the current month
az consumption usage list \
  --scope /subscriptions/<sub-id>/resourceGroups/myapp-prod \
  --start-date $(date -u +%Y-%m-01) \
  --end-date $(date -u +%Y-%m-%d) \
  --query "[].{Resource:instanceName, Cost:pretaxCost, Currency:currency}" \
  --output table
```

In the Azure Portal: **Cost Management + Billing → Cost Analysis → Scope → Resource Group** gives a full breakdown by service, resource, and tag.

---

## Resource Group Naming Conventions

There is no enforced convention, but a consistent pattern saves confusion at scale:

```
<workload>-<environment>-rg
<workload>-<component>-<environment>-rg
```

**Examples:**

| Pattern | Example |
|---------|---------|
| By environment | `myapp-dev-rg`, `myapp-staging-rg`, `myapp-prod-rg` |
| By workload + env | `payments-prod-rg`, `auth-prod-rg`, `data-prod-rg` |
| By team | `platform-rg`, `data-engineering-rg` |
| Shared services | `networking-rg`, `security-rg`, `monitoring-rg` |

> Azure Policy can enforce naming conventions across your subscription using `deny` effects on resource group creation.

---

## Common Grouping Strategies

### By Environment (most common for small/medium teams)
```
myapp-dev-rg       → all dev resources
myapp-staging-rg   → all staging resources
myapp-prod-rg      → all prod resources
```

### By Workload / Service (better for microservices)
```
payments-rg        → payment service + its DB + its Function Apps
auth-rg            → identity service + Key Vault + APIM config
data-rg            → Synapse + Data Factory + storage accounts
```

### Shared Services Pattern (landing zone)
```
networking-rg      → VNet, subnets, NSGs, Firewall, Bastion, DNS
security-rg        → Key Vault, Defender policies, Log Analytics
monitoring-rg      → App Insights, alerts, dashboards
identity-rg        → Managed identities, APIM
workload-prod-rg   → The actual application resources
```

---

## Azure Policy on Resource Groups

Policies can be scoped to a resource group to enforce compliance within it.

```bash
# Require a specific tag on all resources in a group
az policy assignment create \
  --name require-environment-tag \
  --scope /subscriptions/<sub-id>/resourceGroups/myapp-prod \
  --policy "871b6d14-10aa-478d-b590-94f262ecfa99"  # built-in: Require tag on resources

# Deny creation of public IPs inside a resource group
az policy assignment create \
  --name deny-public-ip \
  --scope /subscriptions/<sub-id>/resourceGroups/myapp-prod \
  --policy "83a86a26-fd1f-447c-b59d-e51f44264114"  # built-in: Not allowed resource types
  --params '{"listOfResourceTypesNotAllowed": {"value": ["Microsoft.Network/publicIPAddresses"]}}'
```

---

## Key Differences from AWS

| Concept | AWS | Azure Resource Groups |
|---------|-----|-----------------------|
| Closest equivalent | CloudFormation Stack + resource tags | Resource Group |
| Lifecycle grouping | Stack delete removes stack resources | RG delete removes all resources |
| RBAC scope | IAM policies at resource/account level | RBAC at RG level (inherited) |
| Cost grouping | Cost allocation tags | Resource group (first-class cost scope) |
| Nesting | OUs in Organizations | Not supported — flat structure |
| Mandatory grouping | No — resources exist independently | Yes — every resource must be in an RG |
| Deployment scope | Stack (account-level) | Resource group (default) or subscription |
| Delete protection | Stack termination protection | Resource locks (CanNotDelete) |