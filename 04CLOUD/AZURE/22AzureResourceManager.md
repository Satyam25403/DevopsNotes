# Azure Resource Manager (ARM)
## (analogous to AWS CloudFormation)

Azure Resource Manager is the deployment and management layer for Azure. Every action you take — whether via the Portal, CLI, SDK, or Terraform — goes through ARM. It handles authentication, authorization, resource lifecycle, and declarative deployments via **ARM templates** (JSON) or **Bicep** (a cleaner DSL that compiles to ARM JSON).

---

## What ARM Actually Does

```
You (Portal / CLI / SDK / Terraform / Bicep)
          ↓
    ARM Endpoint (management.azure.com)
          ↓
    ┌─────────────────────────────────────┐
    │  Authentication (Entra ID token)    │
    │  Authorization (RBAC check)         │
    │  Throttling / rate limiting         │
    │  Deployment orchestration           │
    │  Resource lifecycle management      │
    └─────────────────────────────────────┘
          ↓
    Resource Providers
    (Microsoft.Compute, Microsoft.Storage, Microsoft.Network, ...)
          ↓
    Actual Azure resources
```

Every Azure resource has a **resource type** owned by a **resource provider**:

| Resource | Resource Provider | Resource Type |
|----------|------------------|---------------|
| Virtual Machine | `Microsoft.Compute` | `virtualMachines` |
| Storage Account | `Microsoft.Storage` | `storageAccounts` |
| VNet | `Microsoft.Network` | `virtualNetworks` |
| Function App | `Microsoft.Web` | `sites` |
| AKS Cluster | `Microsoft.ContainerService` | `managedClusters` |
| Key Vault | `Microsoft.KeyVault` | `vaults` |

---

## Resource IDs

Every resource has a globally unique Resource ID — the full path through the ARM hierarchy:

```
/subscriptions/{subscriptionId}
  /resourceGroups/{resourceGroupName}
    /providers/{resourceProvider}
      /{resourceType}
        /{resourceName}
```

**Example:**
```
/subscriptions/12345678-1234-1234-1234-123456789abc
  /resourceGroups/myapp-prod-rg
    /providers/Microsoft.Compute
      /virtualMachines
        /myVM
```

Child resources append further:
```
/subscriptions/.../providers/Microsoft.Compute/virtualMachines/myVM
  /extensions/customScript
```

---

## ARM Templates

ARM templates are JSON documents that declare the desired state of Azure resources. ARM compares the template against the current state and makes only the necessary changes (**idempotent** — safe to redeploy).

### Template Structure

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",

  "parameters": {
    "appName": {
      "type": "string",
      "minLength": 3,
      "maxLength": 24,
      "metadata": { "description": "Name of the application" }
    },
    "environment": {
      "type": "string",
      "allowedValues": ["dev", "staging", "prod"],
      "defaultValue": "dev"
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    },
    "adminPassword": {
      "type": "securestring"    // never logged or shown
    }
  },

  "variables": {
    "storageAccountName": "[concat(parameters('appName'), 'storage')]",
    "appServicePlanName": "[concat(parameters('appName'), '-plan')]",
    "tags": {
      "Environment": "[parameters('environment')]",
      "ManagedBy": "ARM"
    }
  },

  "resources": [
    // resource definitions go here
  ],

  "outputs": {
    "storageAccountId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]"
    },
    "webAppUrl": {
      "type": "string",
      "value": "[concat('https://', reference(parameters('appName')).defaultHostName)]"
    }
  }
}
```

---

### Defining Resources

```json
"resources": [
  {
    "type": "Microsoft.Storage/storageAccounts",
    "apiVersion": "2023-01-01",
    "name": "[variables('storageAccountName')]",
    "location": "[parameters('location')]",
    "tags": "[variables('tags')]",
    "sku": {
      "name": "Standard_LRS"
    },
    "kind": "StorageV2",
    "properties": {
      "supportsHttpsTrafficOnly": true,
      "minimumTlsVersion": "TLS1_2",
      "allowBlobPublicAccess": false
    }
  },
  {
    "type": "Microsoft.Web/serverfarms",
    "apiVersion": "2022-03-01",
    "name": "[variables('appServicePlanName')]",
    "location": "[parameters('location')]",
    "tags": "[variables('tags')]",
    "sku": {
      "name": "B1",
      "tier": "Basic"
    },
    "kind": "linux",
    "properties": {
      "reserved": true
    }
  },
  {
    "type": "Microsoft.Web/sites",
    "apiVersion": "2022-03-01",
    "name": "[parameters('appName')]",
    "location": "[parameters('location')]",
    "tags": "[variables('tags')]",
    "dependsOn": [
      "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]"
    ],
    "identity": {
      "type": "SystemAssigned"
    },
    "properties": {
      "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('appServicePlanName'))]",
      "siteConfig": {
        "linuxFxVersion": "NODE|20-lts",
        "appSettings": [
          {
            "name": "NODE_ENV",
            "value": "[parameters('environment')]"
          },
          {
            "name": "STORAGE_ACCOUNT_NAME",
            "value": "[variables('storageAccountName')]"
          }
        ]
      },
      "httpsOnly": true
    }
  }
]
```

---

### ARM Template Functions

ARM templates have a rich set of built-in functions:

```json
// String functions
"[concat('prefix-', parameters('name'), '-suffix')]"
"[toLower(parameters('name'))]"
"[toUpper(parameters('env'))]"
"[format('{0}-{1}', parameters('appName'), parameters('env'))]"
"[replace(parameters('name'), '-', '')]"
"[substring(parameters('name'), 0, 8)]"
"[length(parameters('name'))]"

// Resource functions
"[resourceGroup().name]"              // current resource group name
"[resourceGroup().location]"          // current resource group location
"[resourceGroup().id]"                // full resource group resource ID
"[subscription().subscriptionId]"     // current subscription ID
"[subscription().tenantId]"           // current tenant ID

// Reference functions (get properties of another resource)
"[reference(resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))).primaryEndpoints.blob]"
"[listKeys(resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName')), '2023-01-01').keys[0].value]"

// Resource ID construction
"[resourceId('Microsoft.Network/virtualNetworks/subnets', 'myVNet', 'appSubnet')]"

// Conditional
"[if(equals(parameters('environment'), 'prod'), 'Standard_LRS', 'Standard_LRS')]"

// Array functions
"[first(parameters('allowedIPs'))]"
"[last(parameters('allowedIPs'))]"
"[union(variables('defaultTags'), parameters('extraTags'))]"
```

---

### Conditional Resources

```json
{
  "type": "Microsoft.Insights/components",
  "apiVersion": "2020-02-02",
  "name": "[concat(parameters('appName'), '-insights')]",
  "location": "[parameters('location')]",
  "condition": "[equals(parameters('environment'), 'prod')]",
  "kind": "web",
  "properties": {
    "Application_Type": "web"
  }
}
```

---

### Copy Loops (create multiple resources)

```json
{
  "type": "Microsoft.Storage/storageAccounts",
  "apiVersion": "2023-01-01",
  "name": "[concat('storage', copyIndex())]",
  "location": "[parameters('location')]",
  "sku": { "name": "Standard_LRS" },
  "kind": "StorageV2",
  "copy": {
    "name": "storageCopy",
    "count": 3,
    "mode": "Parallel"    // or "Serial" for sequential
  },
  "properties": {}
}

// Iterate over an array of values
{
  "type": "Microsoft.Storage/storageAccounts",
  "apiVersion": "2023-01-01",
  "name": "[parameters('storageNames')[copyIndex()]]",
  "location": "[parameters('location')]",
  "sku": { "name": "Standard_LRS" },
  "kind": "StorageV2",
  "copy": {
    "name": "storageCopy",
    "count": "[length(parameters('storageNames'))]"
  },
  "properties": {}
}
```

---

### Linked Templates (modular deployments)

Break large templates into reusable modules stored in a Storage Account or template spec:

```json
{
  "type": "Microsoft.Resources/deployments",
  "apiVersion": "2022-09-01",
  "name": "networkDeployment",
  "properties": {
    "mode": "Incremental",
    "templateLink": {
      "uri": "https://mystorage.blob.core.windows.net/templates/network.json",
      "contentVersion": "1.0.0.0"
    },
    "parameters": {
      "vnetName": { "value": "[parameters('vnetName')]" },
      "location": { "value": "[parameters('location')]" }
    }
  }
}
```

---

## Bicep (recommended over raw ARM JSON)

Bicep is Azure's domain-specific language for IaC. It compiles to ARM JSON — so it has the same capabilities with far cleaner syntax. No schemas, no `[concat()]` noise, native type-safety and IDE support.

```bash
# Install Bicep (bundled with Azure CLI 2.20+)
az bicep install
az bicep version

# Compile Bicep to ARM JSON
az bicep build --file main.bicep

# Decompile ARM JSON to Bicep (useful for migrating existing templates)
az bicep decompile --file existing.json
```

---

### Bicep Syntax

```bicep
// Parameters
@description('Name of the application')
@minLength(3)
@maxLength(24)
param appName string

@description('Deployment environment')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

param location string = resourceGroup().location

@secure()
param adminPassword string

// Variables
var storageAccountName = '${appName}storage'
var appServicePlanName = '${appName}-plan'
var tags = {
  Environment: environment
  ManagedBy: 'Bicep'
}

// Resources
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  tags: tags
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}

resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: appServicePlanName
  location: location
  tags: tags
  sku: {
    name: 'B1'
    tier: 'Basic'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

resource webApp 'Microsoft.Web/sites@2022-03-01' = {
  name: appName
  location: location
  tags: tags
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id    // implicit dependsOn — no explicit declaration needed
    siteConfig: {
      linuxFxVersion: 'NODE|20-lts'
      appSettings: [
        { name: 'NODE_ENV', value: environment }
        { name: 'STORAGE_ACCOUNT', value: storageAccount.name }
        { name: 'STORAGE_BLOB_ENDPOINT', value: storageAccount.properties.primaryEndpoints.blob }
      ]
    }
    httpsOnly: true
  }
}

// Outputs
output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
output storageAccountId string = storageAccount.id
output webAppPrincipalId string = webApp.identity.principalId
```

---

### Bicep: Conditionals

```bicep
// Conditionally create a resource
resource appInsights 'Microsoft.Insights/components@2020-02-02' = if (environment == 'prod') {
  name: '${appName}-insights'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
}

// Conditional property value
var skuName = environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'
```

---

### Bicep: Loops

```bicep
// Create multiple storage accounts from an array
param storageNames array = ['data', 'backup', 'archive']

resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for name in storageNames: {
  name: '${appName}${name}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {}
}]

// Loop with index
resource subnets 'Microsoft.Network/virtualNetworks/subnets@2023-04-01' = [for (subnet, i) in subnetsConfig: {
  name: subnet.name
  parent: vnet
  properties: {
    addressPrefix: subnet.cidr
  }
}]

// Output array from loop
output storageIds array = [for i in range(0, length(storageNames)): storageAccounts[i].id]
```

---

### Bicep: Modules (the recommended way to structure large deployments)

Split your infrastructure into reusable module files:

```
infra/
├── main.bicep              ← orchestrates everything
├── modules/
│   ├── network.bicep       ← VNet, subnets, NSGs
│   ├── storage.bicep       ← storage accounts
│   ├── app.bicep           ← App Service, Function Apps
│   └── monitoring.bicep    ← Log Analytics, App Insights
```

```bicep
// modules/network.bicep
param location string
param vnetName string
param addressPrefix string = '10.0.0.0/16'

resource vnet 'Microsoft.Network/virtualNetworks@2023-04-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [addressPrefix]
    }
    subnets: [
      { name: 'webSubnet',  properties: { addressPrefix: '10.0.1.0/24' } }
      { name: 'appSubnet',  properties: { addressPrefix: '10.0.2.0/24' } }
      { name: 'dataSubnet', properties: { addressPrefix: '10.0.3.0/24' } }
    ]
  }
}

output vnetId string = vnet.id
output appSubnetId string = vnet.properties.subnets[1].id
```

```bicep
// main.bicep — consumes modules
param appName string
param environment string = 'prod'
param location string = resourceGroup().location

module network './modules/network.bicep' = {
  name: 'networkDeployment'
  params: {
    location: location
    vnetName: '${appName}-vnet'
  }
}

module app './modules/app.bicep' = {
  name: 'appDeployment'
  params: {
    location: location
    appName: appName
    subnetId: network.outputs.appSubnetId   // pass output from one module to another
  }
}

output appUrl string = app.outputs.webAppUrl
```

---

### Bicep: Existing Resources (reference without deploying)

```bicep
// Reference an existing Key Vault (not deploying it, just referencing it)
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' existing = {
  name: 'myKeyVault'
  scope: resourceGroup('mySharedRG')   // can reference across resource groups
}

// Now use it
resource secret 'Microsoft.KeyVault/vaults/secrets@2023-02-01' = {
  parent: keyVault
  name: 'db-password'
  properties: {
    value: adminPassword
  }
}

// Read a Key Vault secret into a parameter
resource kvRef 'Microsoft.KeyVault/vaults@2023-02-01' existing = {
  name: 'myKeyVault'
}

// In parameters file — pull secret at deploy time (not stored in template)
// main.bicepparam:
// param adminPassword = getSecret('<sub-id>', 'mySharedRG', 'myKeyVault', 'admin-password')
```

---

## Deploying Templates

### Deploy Scope

ARM templates can deploy at four scopes:

| Scope | CLI Command | Use Case |
|-------|-------------|----------|
| **Resource Group** | `az deployment group create` | Most resources (default) |
| **Subscription** | `az deployment sub create` | Resource groups, policies, role assignments |
| **Management Group** | `az deployment mg create` | Policies across multiple subscriptions |
| **Tenant** | `az deployment tenant create` | Tenant-wide management groups |

---

### Resource Group Deployments (most common)

```bash
# Deploy a Bicep file
az deployment group create \
  --resource-group myapp-prod-rg \
  --template-file main.bicep \
  --parameters appName=myapp environment=prod

# Deploy with a parameters file
az deployment group create \
  --resource-group myapp-prod-rg \
  --template-file main.bicep \
  --parameters @main.prod.bicepparam

# Preview changes before deploying (what-if — like terraform plan)
az deployment group what-if \
  --resource-group myapp-prod-rg \
  --template-file main.bicep \
  --parameters appName=myapp environment=prod

# Deploy an ARM JSON template
az deployment group create \
  --resource-group myapp-prod-rg \
  --template-file azuredeploy.json \
  --parameters @azuredeploy.parameters.json

# Deploy from a URL (e.g., GitHub raw or blob storage)
az deployment group create \
  --resource-group myapp-prod-rg \
  --template-uri https://raw.githubusercontent.com/myorg/myrepo/main/infra/main.json \
  --parameters appName=myapp
```

---

### Parameters Files

```json
// azuredeploy.parameters.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "appName": { "value": "myapp" },
    "environment": { "value": "prod" },
    "adminPassword": {
      "reference": {
        "keyVault": {
          "id": "/subscriptions/<sub-id>/resourceGroups/shared-rg/providers/Microsoft.KeyVault/vaults/myKeyVault"
        },
        "secretName": "admin-password"
      }
    }
  }
}
```

```bicep
// main.prod.bicepparam (Bicep parameters file — cleaner syntax)
using './main.bicep'

param appName = 'myapp'
param environment = 'prod'
param adminPassword = getSecret('<sub-id>', 'shared-rg', 'myKeyVault', 'admin-password')
```

---

### Deployment Modes

```bash
# Incremental (default) — only add/update resources in template; don't touch others
az deployment group create \
  --resource-group myapp-prod-rg \
  --template-file main.bicep \
  --mode Incremental

# Complete — delete any resource in the RG not in the template (dangerous!)
az deployment group create \
  --resource-group myapp-prod-rg \
  --template-file main.bicep \
  --mode Complete
```

> **Always use Incremental** in production. Complete mode will delete resources not in the template — including databases, key vaults, or anything added outside the template.

---

### Subscription-Level Deployments

Use subscription scope for creating resource groups, assigning policies, or setting up role assignments across an entire subscription.

```bicep
// sub-level.bicep — must set targetScope
targetScope = 'subscription'

param location string = 'eastus'
param environment string

// Create resource groups
resource appRG 'Microsoft.Resources/resourceGroups@2022-09-01' = {
  name: 'myapp-${environment}-rg'
  location: location
  tags: {
    Environment: environment
  }
}

resource sharedRG 'Microsoft.Resources/resourceGroups@2022-09-01' = {
  name: 'shared-${environment}-rg'
  location: location
}

// Deploy a module into the newly created RG
module network './modules/network.bicep' = {
  name: 'networkDeployment'
  scope: appRG          // scope to the resource group created above
  params: {
    location: location
    vnetName: 'myapp-vnet'
  }
}

// Assign a policy at subscription level
resource noPublicIPPolicy 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'deny-public-ip'
  properties: {
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/83a86a26-fd1f-447c-b59d-e51f44264114'
    displayName: 'Deny Public IPs'
  }
}
```

```bash
az deployment sub create \
  --location eastus \
  --template-file sub-level.bicep \
  --parameters environment=prod
```

---

### Managing Deployments

```bash
# List deployments in a resource group
az deployment group list \
  --resource-group myapp-prod-rg \
  --output table

# Show a specific deployment (inputs, outputs, status)
az deployment group show \
  --resource-group myapp-prod-rg \
  --name main \
  --query "{Status:properties.provisioningState, Outputs:properties.outputs}"

# Show deployment operations (individual resource results — useful for debugging failures)
az deployment group operation list \
  --resource-group myapp-prod-rg \
  --name main \
  --output table

# Cancel a running deployment
az deployment group cancel \
  --resource-group myapp-prod-rg \
  --name main

# Delete a deployment record (does NOT delete the resources)
az deployment group delete \
  --resource-group myapp-prod-rg \
  --name main
```

---

## Template Specs (versioned, shareable templates)

Template Specs store ARM/Bicep templates in Azure itself — versioned, access-controlled, and shareable across the organization without external storage.

```bash
# Create a template spec from a Bicep file
az ts create \
  --resource-group shared-rg \
  --name webapp-template \
  --version "1.0.0" \
  --template-file main.bicep \
  --display-name "Standard Web App" \
  --description "App Service + Storage + App Insights"

# List template specs
az ts list --resource-group shared-rg --output table

# List versions of a template spec
az ts list --resource-group shared-rg --name webapp-template --output table

# Deploy from a template spec
az deployment group create \
  --resource-group myapp-prod-rg \
  --template-spec /subscriptions/<sub-id>/resourceGroups/shared-rg/providers/Microsoft.Resources/templateSpecs/webapp-template/versions/1.0.0 \
  --parameters appName=myapp environment=prod
```

Reference a template spec as a Bicep module:

```bicep
module webApp 'ts:shared-rg/webapp-template:1.0.0' = {
  name: 'webAppDeployment'
  params: {
    appName: 'myapp'
    environment: 'prod'
  }
}
```

---

## Resource Provider Operations

```bash
# List all registered resource providers in your subscription
az provider list --output table

# Show registration status of a specific provider
az provider show --namespace Microsoft.ContainerService --query registrationState

# Register a provider (required before using it for the first time)
az provider register --namespace Microsoft.ContainerService

# List all resource types for a provider
az provider show \
  --namespace Microsoft.Compute \
  --query "resourceTypes[].{Type:resourceType, Locations:locations}" \
  --output table

# List all available API versions for a resource type
az provider show \
  --namespace Microsoft.Web \
  --query "resourceTypes[?resourceType=='sites'].apiVersions" \
  --output json
```

---

## ARM REST API (direct HTTP calls)

Everything the CLI does goes through the ARM REST API. Useful for automation, custom tooling, or when the CLI doesn't expose a feature yet.

```bash
# Get an access token
TOKEN=$(az account get-access-token --query accessToken -o tsv)
SUB_ID=$(az account show --query id -o tsv)

# List resource groups
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups?api-version=2022-09-01" \
  | jq '.value[].name'

# Get a specific resource
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/myRG/providers/Microsoft.Web/sites/myApp?api-version=2022-03-01" \
  | jq '{name: .name, state: .properties.state, url: .properties.defaultHostName}'

# Deploy a template via REST API
curl -s -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @deployment-body.json \
  "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/myRG/providers/Microsoft.Resources/deployments/myDeployment?api-version=2022-09-01"
```

---

## ARM vs Bicep vs Terraform — When to Use What

| | ARM JSON | Bicep | Terraform |
|--|----------|-------|-----------|
| **Syntax** | Verbose JSON | Clean DSL | HCL |
| **Azure-native** | ✅ Yes | ✅ Yes | ❌ Third-party provider |
| **New resource support** | Same day | Same day | Depends on provider update |
| **Multi-cloud** | ❌ Azure only | ❌ Azure only | ✅ Yes |
| **State file** | No (ARM manages state) | No (ARM manages state) | Yes (must manage .tfstate) |
| **Plan / preview** | `what-if` | `what-if` | `terraform plan` |
| **IDE support** | Limited | Excellent (VS Code ext) | Good |
| **Modularity** | Linked templates | Modules | Modules |
| **Recommended for** | Legacy / generated | New Azure-only IaC | Multi-cloud or existing Terraform teams |

---

## Key Differences from AWS CloudFormation

| Feature | AWS CloudFormation | Azure ARM / Bicep |
|---------|-------------------|-------------------|
| Template format | JSON or YAML | JSON (ARM) or Bicep DSL |
| Cleaner DSL | CDK (TypeScript/Python/etc.) | Bicep |
| State management | CloudFormation manages stacks | ARM manages resources directly — no stack state file |
| Drift detection | CloudFormation drift detection | `az deployment group what-if` shows current vs desired |
| Rollback on failure | Automatic (stack rollback) | Manual or via `--rollback-on-error` flag |
| Modular templates | Nested stacks | Linked templates / Bicep modules / Template Specs |
| Versioned templates | No native versioning | Template Specs (versioned) |
| Deployment scopes | Account / StackSet (multi-account) | RG / Subscription / Management Group / Tenant |
| Preview changes | Change sets | `what-if` |
| Resource import | CloudFormation import | `az deployment group create` with existing resources |
| Cross-account/region | StackSets | Management Group deployments + scope overrides |
| Private template storage | S3 | Template Specs (in Azure) or any URL |