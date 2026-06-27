# Capstone: A Complete Multi-Tier Application Stack

This file ties together every concept from files 00–10 into one realistic, production-shaped deployment: a Resource Group, a Virtual Network with multiple subnets, a Network Security Group with looped rules, a Storage Account with a looped set of containers, a Key Vault holding a generated secret, and a Virtual Machine — orchestrated through a `main.bicep` that calls multiple modules, deployed at subscription scope, with environment-driven conditional logic throughout.

```
Concepts deliberately exercised in this capstone:
  Scopes (file 02)            →  subscription-scoped orchestrator creating its own resource group
  Modules (file 03)             →  one module per logical tier, called from main.bicep
  Loops (file 04)                 →  looped subnets, NSG rules, storage containers
  Conditions (file 04)             →  DR storage account only in prod
  Type system (file 01)             →  UDTs for strongly-typed network config
  Expressions/functions (file 05)    →  uniqueString(), ternaries, safe-access
  Nested resources (file 07)          →  blob containers nested under the storage account
  Deployment Stacks (file 06)          →  the recommended way to manage this whole stack
```

---

## Project Structure

```
infra/
├── main.bicep                  # subscription-scoped orchestrator
├── modules/
│   ├── network.bicep             # VNet, subnets, NSG + rules
│   ├── storage.bicep              # storage account + containers + (conditional) DR account
│   ├── keyvault.bicep               # Key Vault + generated secret
│   └── virtualMachine.bicep          # NIC + VM
└── params/
    ├── dev.bicepparam
    └── prod.bicepparam
```

---

## Shared Types — `modules/network.bicep` Uses a User-Defined Type

Recall file 01's User-Defined Types — here's where they genuinely pay off, giving the subnet list compile-time-checked structure instead of a loosely-typed `array`.

```bicep
// Declared once, used by both main.bicep (to type its own param) and
// network.bicep (to type ITS param) — UDTs can be repeated across files
// since Bicep has no shared "types.bicep" import mechanism in classic
// usage; each file declares what it needs. (Bicep DOES support importing
// user-defined types across files via the "import" keyword in newer
// versions — mentioned here, not required for this capstone.)
type subnetConfigType = {
  name: string
  addressPrefix: string
}
```

---

## `modules/network.bicep` — VNet, Subnets (Looped), NSG (Looped Rules)

```bicep
type subnetConfigType = {
  name: string
  addressPrefix: string
}

@description('Environment name, used in resource naming')
param environment string

@description('Azure region')
param location string

@description('VNet address space')
param vnetAddressSpace string = '10.0.0.0/16'

@description('Subnets to create within the VNet')
param subnets subnetConfigType[] = [
  { name: 'frontend', addressPrefix: '10.0.1.0/24' }
  { name: 'backend', addressPrefix: '10.0.2.0/24' }
]

@description('Inbound NSG rules to apply')
param nsgRules array = [
  { name: 'allow-https', priority: 100, port: '443' }
  { name: 'allow-ssh', priority: 110, port: '22' }
]

resource nsg 'Microsoft.Network/networkSecurityGroups@2023-09-01' = {
  name     : '${environment}-nsg'
  location : location
  properties: {
    // Looping over a property (file 04) — generates one securityRule
    // block per entry in nsgRules, without hand-writing each one
    securityRules: [for rule in nsgRules: {
      name: rule.name
      properties: {
        priority                   : rule.priority
        direction                   : 'Inbound'
        access                       : 'Allow'
        protocol                       : 'Tcp'
        sourcePortRange                 : '*'
        destinationPortRange             : rule.port
        sourceAddressPrefix                : '*'
        destinationAddressPrefix            : '*'
      }
    }]
  }
}

resource vnet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name     : '${environment}-vnet'
  location : location
  properties: {
    addressSpace: {
      addressPrefixes: [vnetAddressSpace]
    }
  }
}

// Sibling/top-level child resources (file 07) — looped, so nesting
// inside "vnet" directly isn't an option (you cannot "for" inside a
// nested declaration)
resource subnetResources 'Microsoft.Network/virtualNetworks/subnets@2023-09-01' = [for subnet in subnets: {
  name   : subnet.name
  parent : vnet
  properties: {
    addressPrefix       : subnet.addressPrefix
    networkSecurityGroup: {
      id: nsg.id   // every subnet shares the same NSG in this example
    }
  }
}]

output vnetId string = vnet.id
output nsgId string = nsg.id
// Looping over an output (file 04) — collect every subnet's id into an array
output subnetIds array = [for i in range(0, length(subnets)): subnetResources[i].id]
// A keyed lookup map is often more USEFUL than a bare array for consumers —
// built with a "for" expression producing an object instead of an array
output subnetIdsByName object = toObject(subnetResources, sub => sub.name, sub => sub.id)
```

> `toObject()` is a built-in function not yet covered in file 05 — it converts an array into a keyed object using two lambda-style expressions (key selector, value selector), giving consumers a `{ frontend: '...', backend: '...' }` shape instead of forcing them to know array index positions. Worth knowing for exactly this "give callers a name-based lookup, not an index-based one" pattern.

---

## `modules/storage.bicep` — Storage Account, Looped Containers, Conditional DR Account

```bicep
@description('Environment name')
param environment string

@description('Azure region')
param location string

@description('Names of blob containers to create')
param containerNames array = ['raw-data', 'processed-data']

@description('Whether this is production (gates DR account creation)')
param isProduction bool = false

@description('DR region — only used if isProduction is true')
param drLocation string = 'northeurope'

var storageAccountName = 'st${environment}${uniqueString(resourceGroup().id)}'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name     : storageAccountName
  location : location
  sku: {
    // Conditional expression (file 05) driving SKU choice by environment
    name: isProduction ? 'Standard_GRS' : 'Standard_LRS'
  }
  kind: 'StorageV2'

  // Nested child resource (file 07) — fixed, single instance, so nesting
  // (rather than sibling+parent) is the cleaner choice here
  resource blobService 'blobServices@2023-01-01' = {
    name: 'default'
    properties: {
      deleteRetentionPolicy: {
        enabled: true
        days: 7
      }
    }
  }

  tags: {
    environment: environment
  }
}

// Looped containers (file 04) need sibling syntax (file 07) since "for"
// cannot target something already nested two levels deep cleanly here
resource containers 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = [for name in containerNames: {
  name   : name
  parent : storageAccount::blobService
  properties: {
    publicAccess: 'None'
  }
}]

// Conditional resource (file 04) — only deployed when isProduction is true.
// No indexing required to reference it later, unlike Terraform's count-based
// conditional pattern.
resource drStorageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = if (isProduction) {
  name     : 'st${environment}dr${uniqueString(resourceGroup().id, 'dr')}'
  location : drLocation
  sku: {
    name: 'Standard_GRS'
  }
  kind: 'StorageV2'
  tags: {
    environment: environment
    purpose: 'disaster-recovery'
  }
}

output storageAccountId string = storageAccount.id
output storageAccountName string = storageAccount.name
// Safe-access (file 05): drStorageAccount might not exist (isProduction=false),
// so ".?id" avoids an error, falling back to null via "??"
output drStorageAccountId string = isProduction ? drStorageAccount.id : ''
```

---

## `modules/keyvault.bicep` — Key Vault With a Generated Secret

```bicep
@description('Environment name')
param environment string

@description('Azure region')
param location string

@secure()
@description('Administrator password to store as a secret — passed in securely, never defaulted')
param adminPassword string

var keyVaultName = 'kv-${environment}-${uniqueString(resourceGroup().id)}'

resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name     : take(keyVaultName, 24)   // Key Vault names are capped at 24 chars —
                                         // take() (file 05) truncates safely
  location : location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId   // deployment context function (file 05)
    enableRbacAuthorization: true        // modern access model — prefer this over
                                            // legacy access policies
  }

  // Nested secret resource (file 07)
  resource adminPasswordSecret 'secrets@2023-07-01' = {
    name: 'vm-admin-password'
    properties: {
      value: adminPassword   // the @secure() value flows straight through —
                                // never logged, never written to deployment
                                // history in plaintext (file 01)
    }
  }
}

output keyVaultId string = keyVault.id
output adminPasswordSecretUri string = keyVault::adminPasswordSecret.properties.secretUri
```

---

## `modules/virtualMachine.bicep` — NIC + VM, Using a Strongly-Typed Config Object

```bicep
type vmImageConfigType = {
  publisher: string
  offer: string
  sku: string
  version: string
}

@description('Environment name')
param environment string

@description('Azure region')
param location string

@description('Subnet resource ID to attach the VM NIC to')
param subnetId string

@description('VM size')
@allowed(['Standard_DS1_v2', 'Standard_DS2_v2', 'Standard_DS3_v2'])
param vmSize string = 'Standard_DS1_v2'

@description('VM image configuration')
param imageConfig vmImageConfigType = {
  publisher: 'Canonical'
  offer    : '0001-com-ubuntu-server-jammy'
  sku       : '22_04-lts'
  version    : 'latest'
}

@secure()
@description('Administrator password')
param adminPassword string

resource nic 'Microsoft.Network/networkInterfaces@2023-09-01' = {
  name     : '${environment}-nic'
  location : location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: { id: subnetId }
          privateIPAllocationMethod: 'Dynamic'
        }
      }
    ]
  }
}

resource vm 'Microsoft.Compute/virtualMachines@2023-09-01' = {
  name     : '${environment}-vm'
  location : location
  properties: {
    hardwareProfile: {
      vmSize: vmSize
    }
    networkProfile: {
      networkInterfaces: [
        { id: nic.id }
      ]
    }
    storageProfile: {
      imageReference: imageConfig   // the whole UDT-typed object, passed straight
                                       // through — field names happen to match
                                       // ARM's expected shape exactly
      osDisk: {
        createOption: 'FromImage'
        managedDisk: {
          storageAccountType: 'Standard_LRS'
        }
      }
    }
    osProfile: {
      computerName : 'hostname'
      adminUsername: 'azureadmin'
      adminPassword: adminPassword
      linuxConfiguration: {
        disablePasswordAuthentication: false
      }
    }
  }
}

output vmId string = vm.id
output nicId string = nic.id
```

---

## `main.bicep` — The Subscription-Scoped Orchestrator

```bicep
targetScope = 'subscription'

@description('Environment name — drives naming and conditional logic throughout')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

@description('Azure region')
param location string = 'westeurope'

@secure()
@description('Administrator password for the VM and Key Vault secret')
param adminPassword string

@description('Subnets to create')
param subnets array = [
  { name: 'frontend', addressPrefix: '10.0.1.0/24' }
  { name: 'backend', addressPrefix: '10.0.2.0/24' }
]

var isProduction = environment == 'prod'
var resourceGroupName = '${environment}-resources'

// ── Resource Group (subscription scope creates it; file 02) ────────────────
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name     : resourceGroupName
  location : location
  tags: {
    environment: environment
  }
}

// ── Network Module (crosses scope via "scope: rg"; file 02 + file 03) ──────
module network 'modules/network.bicep' = {
  name : 'networkDeployment'
  scope: rg
  params: {
    environment: environment
    location   : location
    subnets    : subnets
  }
}

// ── Storage Module — note the conditional DR logic flows in via param ──────
module storage 'modules/storage.bicep' = {
  name : 'storageDeployment'
  scope: rg
  params: {
    environment  : environment
    location      : location
    isProduction   : isProduction
  }
}

// ── Key Vault Module ─────────────────────────────────────────────────────────
module keyvault 'modules/keyvault.bicep' = {
  name : 'keyVaultDeployment'
  scope: rg
  params: {
    environment   : environment
    location       : location
    adminPassword   : adminPassword
  }
}

// ── Virtual Machine Module — consumes the network module's output ──────────
module vm 'modules/virtualMachine.bicep' = {
  name : 'vmDeployment'
  scope: rg
  params: {
    environment   : environment
    location       : location
    subnetId        : network.outputs.subnetIdsByName.frontend   // cross-module
                                                                     // reference, via
                                                                     // outputs (file 03)
    adminPassword    : adminPassword
  }
}

// ── Top-Level Outputs — surfacing the most useful values from every tier ───
output resourceGroupName string = rg.name
output vnetId string = network.outputs.vnetId
output storageAccountName string = storage.outputs.storageAccountName
output keyVaultId string = keyvault.outputs.keyVaultId
output vmId string = vm.outputs.vmId
```

---

## Deploying the Capstone

```bash
# 1. Preview everything with what-if (file 06) — ALWAYS do this before a
#    subscription-scoped deployment, since it can create entire resource groups
az deployment sub what-if \
  --location westeurope \
  --template-file infra/main.bicep \
  --parameters infra/params/prod.bicepparam

# 2. Deploy as a Deployment Stack (file 06) — giving this whole multi-tier
#    stack a persistent, named, RBAC-protectable management boundary,
#    rather than a one-off untracked deployment
az stack sub create \
  --name app-stack-prod \
  --location westeurope \
  --template-file infra/main.bicep \
  --parameters infra/params/prod.bicepparam \
  --deny-settings-mode denyDelete \
  --action-on-unmanage deleteResources
```

```bicep
// infra/params/prod.bicepparam
using '../main.bicep'

param environment = 'prod'
param location = 'westeurope'
param subnets = [
  { name: 'frontend', addressPrefix: '10.0.1.0/24' }
  { name: 'backend', addressPrefix: '10.0.2.0/24' }
  { name: 'data', addressPrefix: '10.0.3.0/24' }   // prod gets an extra subnet —
                                                       // demonstrates that promotion
                                                       // (file 10) doesn't mean
                                                       // IDENTICAL parameters, just
                                                       // the same template
]
param adminPassword = getSecret('<subscription-id>', 'platform-shared-rg', 'bootstrap-kv', 'capstone-admin-password')
```

---

## Dependency Graph of the Full Capstone

```
main.bicep (subscription scope)
   │
   ├── resourceGroup "rg"
   │       │
   │       ├── module "network"     (depends on rg via scope)
   │       │       ├── nsg
   │       │       ├── vnet
   │       │       └── subnets[]    (loop; depends on vnet + nsg)
   │       │
   │       ├── module "storage"     (depends on rg via scope)
   │       │       ├── storageAccount
   │       │       │       └── blobService (nested)
   │       │       │               └── containers[]  (loop; sibling+parent)
   │       │       └── drStorageAccount (conditional — prod only)
   │       │
   │       ├── module "keyvault"    (depends on rg via scope)
   │       │       └── keyVault
   │       │               └── adminPasswordSecret (nested)
   │       │
   │       └── module "vm"          (depends on rg via scope, AND on
   │               │                  network.outputs.subnetIdsByName —
   │               │                  cross-module dependency)
   │               ├── nic
   │               └── vm
```

Every arrow in this graph is either an **implicit dependency** (file 02) — a symbolic name or module output referenced directly — or a **scope relationship** (file 02) — a module's `scope: rg` property. There is not a single `dependsOn` anywhere in this entire capstone, which is exactly the outcome good Bicep (and good Terraform) authoring aims for: let references do the work, reach for explicit dependency declarations only when truly nothing else can express the relationship.

---

## What This Capstone Demonstrates, Concept by Concept

| File | Concept | Where it appears in the capstone |
|---|---|---|
| 00 | Resource + provider-free deployment model | Every resource block, no provider config anywhere |
| 01 | Decorators, `@secure()`, UDTs | `adminPassword`, `vmConfigType`, `subnetConfigType` |
| 02 | Scopes, `scope:` on modules, implicit dependencies | `targetScope = 'subscription'`, every `scope: rg` |
| 03 | Modules, cross-module outputs | `network.outputs.subnetIdsByName.frontend` consumed by the VM module |
| 04 | Loops, conditionals, combined | Subnets, NSG rules, containers (loops); `drStorageAccount` (conditional) |
| 05 | Functions, ternaries, safe-access | `uniqueString()`, `take()`, `toObject()`, `isProduction ? ... : ...` |
| 06 | Deployment Stacks, `what-if` | Final deployment commands |
| 07 | Nested + sibling child resources | `blobService`/`containers` (both styles shown deliberately) |
| 08–10 | Registry, Policy, CI/CD | Referenced conceptually — this capstone is what those pipelines would deploy |

This concludes the from-zero-to-hero Bicep curriculum, structured to mirror your Terraform set's progressive, example-driven teaching style while surfacing every place Bicep's ARM-native design diverges — sometimes simplifies, sometimes demands new concepts entirely — from the Terraform mental model you already have.