# Azure Entra ID & RBAC (Identity and Access Management)
## (analogous to AWS IAM)

Azure Entra ID (formerly Azure Active Directory / Azure AD) is Microsoft's cloud-based identity platform. Combined with Azure RBAC (Role-Based Access Control), it controls **who** can access your Azure resources and **what** they can do.

---

## Core Concepts

### Tenant
- Your organization's dedicated instance of Entra ID.
- Created automatically when you sign up for Azure.
- All users, groups, and app registrations live here.

### Users
- Represent individual identities (people or service accounts).
- Can have **portal access** (username + password) and/or **programmatic access** (service principals, managed identities).

### Groups
- Collections of users. Assign roles to the group; all members inherit them.
- Groups **can be nested** (unlike AWS IAM groups).
- Synced from on-premises Active Directory via **Entra Connect**.

### Service Principals (analogous to IAM Roles for services/applications)
- An identity for an **application or service** to authenticate with Azure.
- Created via **App Registrations** in Entra ID.
- Used when your app needs to call Azure APIs.

### Managed Identities (analogous to IAM Instance Profiles / EC2 Roles)
- An identity automatically managed by Azure, attached to a resource (VM, App Service, etc.).
- No credentials to manage — Azure handles token issuance.
- **System-assigned**: tied to a single resource; deleted when the resource is deleted.
- **User-assigned**: standalone identity reusable across multiple resources.

> **Prefer Managed Identities over Service Principals** whenever your code runs inside Azure — no secrets, no rotation headaches.

---

## Azure RBAC (Role-Based Access Control)

RBAC is how you assign permissions in Azure. Every permission assignment has three parts:

```
Security Principal + Role Definition + Scope = Role Assignment
```

| Component | Description |
|-----------|-------------|
| **Security Principal** | Who: user, group, service principal, or managed identity |
| **Role Definition** | What: a set of allowed actions (e.g., read VMs, write storage) |
| **Scope** | Where: management group, subscription, resource group, or single resource |

---

## Built-in Roles

| Role | Description | AWS Equivalent |
|------|-------------|----------------|
| **Owner** | Full access + manage access | AdministratorAccess |
| **Contributor** | Full access except manage access | PowerUserAccess |
| **Reader** | View resources only | ReadOnlyAccess |
| **User Access Administrator** | Manage access only | IAMFullAccess |
| **Virtual Machine Contributor** | Manage VMs, not networking | AmazonEC2FullAccess |
| **Storage Blob Data Contributor** | Read/write blob storage | AmazonS3FullAccess |

---

## Assigning Roles via CLI

```bash
# Assign Contributor role to a user on a resource group
az role assignment create \
  --assignee user@example.com \
  --role Contributor \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG

# Assign role to a managed identity
az role assignment create \
  --assignee <managed-identity-principal-id> \
  --role "Storage Blob Data Reader" \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage

# List role assignments
az role assignment list --resource-group myRG --output table
```

---

## Custom Roles

```json
{
  "Name": "Custom VM Reader",
  "Description": "Can read VMs but not modify them",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/instanceView/read"
  ],
  "NotActions": [],
  "AssignableScopes": ["/subscriptions/<sub-id>"]
}
```

```bash
az role definition create --role-definition custom-role.json
```

---

## App Registrations & Service Principals

```bash
# Create an App Registration (generates a service principal)
az ad app create --display-name "my-app"

# Create service principal from app registration
az ad sp create --id <appId>

# Assign a role to the service principal
az role assignment create \
  --assignee <appId> \
  --role Contributor \
  --scope /subscriptions/<sub-id>

# Create client secret for the service principal
az ad app credential reset --id <appId>
```

---

## Managed Identities

```bash
# Enable system-assigned managed identity on a VM
az vm identity assign --name myVM --resource-group myRG

# Create a user-assigned managed identity
az identity create --name myIdentity --resource-group myRG

# Assign user-assigned identity to a VM
az vm identity assign \
  --name myVM \
  --resource-group myRG \
  --identities myIdentity
```

---

## Policies (analogous to AWS Service Control Policies / Config Rules)

**Azure Policy** enforces organizational standards and compliance across all resources.

```bash
# List built-in policies
az policy definition list --query "[?policyType=='BuiltIn'].{Name:displayName}" --output table

# Assign a policy to a resource group
az policy assignment create \
  --name "require-tags" \
  --policy "require-tag-and-its-value" \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG
```

---

## Key Differences from AWS IAM

| Feature | AWS IAM | Azure Entra ID + RBAC |
|---------|---------|----------------------|
| Identity service | IAM | Entra ID (formerly Azure AD) |
| Permission model | Policy-based JSON | Role assignments (RBAC) |
| Scope hierarchy | Account > resource | Management Group > Subscription > RG > Resource |
| Instance identity | IAM Instance Profile | Managed Identity |
| App identity | IAM Role (assumed) | Service Principal / Managed Identity |
| Nested groups | ❌ | ✅ |
| MFA | IAM MFA | Entra ID Conditional Access |