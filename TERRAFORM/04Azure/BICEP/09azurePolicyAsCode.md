# Azure Policy as Code via Bicep

Azure Policy lets you enforce rules across your environment — "every storage account must use HTTPS-only," "no resources outside these three regions," "all VMs must have a specific tag." This file covers authoring Policy **definitions** and **assignments** as Bicep resources — a category where Bicep has a genuine structural advantage over Terraform, because Policy rule logic is itself written in a JSON-shaped expression language that maps almost line-for-line onto Bicep's own `object`/array syntax, with none of HCL's translation overhead.

```
Concepts covered:
  Policy definition vs assignment       →  the rule itself vs "where/how it's enforced"
  Policy effects                          →  Deny, Audit, Append, Modify, DeployIfNotExists
  The "policyRule" object                  →  if/then logic, expressed as nested objects
  Initiative (Policy Set) definitions       →  grouping multiple policies together
  Assignment scope and exemptions            →  applying policy at the right level
```

---

## Policy Definition vs Policy Assignment — The Core Distinction

This mirrors the relationship between a function definition and a function call:

```
Policy Definition   →  THE RULE ITSELF: "storage accounts must disable public blob access"
                        (written once, reusable)
        │
        ▼
Policy Assignment    →  "APPLY that rule HERE" — at a specific management group,
                         subscription, or resource group
```

A single definition can be assigned at many different scopes, with different parameter values each time — exactly analogous to how a single Bicep `module` (file 03) can be called multiple times with different parameters.

---

## Anatomy of a Policy Definition

```bicep
targetScope = 'subscription'   // policy definitions are typically created at subscription
                                 // or management group scope, never resourceGroup scope

@description('Denies creation of storage accounts that allow public blob access')
param policyName string = 'deny-public-blob-access'

resource policyDefinition 'Microsoft.Authorization/policyDefinitions@2021-06-01' = {
  name: policyName
  properties: {
    displayName: 'Deny Public Blob Access on Storage Accounts'
    description: 'This policy denies the creation or update of storage accounts that have allowPublicAccess set to true.'
    policyType: 'Custom'
    mode: 'Indexed'   // 'Indexed' = applies only to resource types supporting tags/location;
                       // 'All' = applies to every resource type, including ones that don't

    // ── parameters: the policy DEFINITION's own parameters, separate from
    // any Bicep param above. These are what someone filling out an
    // ASSIGNMENT later will be prompted to provide.
    parameters: {
      effectType: {
        type: 'String'
        defaultValue: 'Deny'
        allowedValues: [
          'Deny'
          'Audit'
          'Disabled'
        ]
        metadata: {
          displayName: 'Effect'
          description: 'Enable or disable the execution of the policy'
        }
      }
    }

    // ── policyRule: the actual if/then logic — this is the heart of the
    // policy, and it's written as a plain nested object structure,
    // because the Policy engine's rule language IS JSON natively.
    policyRule: {
      if: {
        allOf: [
          {
            field: 'type'
            equals: 'Microsoft.Storage/storageAccounts'
          }
          {
            field: 'Microsoft.Storage/storageAccounts/allowBlobPublicAccess'
            equals: 'true'
          }
        ]
      }
      then: {
        effect: '[parameters(\'effectType\')]'
        // Note: this string is a POLICY-LANGUAGE expression, NOT a Bicep
        // expression — it's the literal ARM Policy function syntax
        // ([parameters('name')]), preserved as-is because the Policy engine
        // evaluates it separately, at enforcement time, not at Bicep
        // compile time. Bicep does not try to interpret or rewrite this
        // string — it passes it through verbatim into the compiled JSON.
      }
    }
  }
}
```

> **Crucial distinction to internalize:** the `policyRule` object's *contents* (the `if`/`then`, `field`, `equals`, and especially the `[parameters('effectType')]` bracket-syntax) are written in **Azure Policy's own expression language**, not Bicep's expression language from file 05. Bicep is just acting as a convenient, typo-checked **container** for this JSON-shaped payload — you still need to learn Policy's own `field`, `value`, `count`, and function syntax separately. Bicep does not abstract Policy's rule language away; it only removes the JSON authoring friction (comments, trailing commas, etc.) around it.

---

## Common Policy Effects

| Effect | Behavior |
|---|---|
| `Deny` | Blocks the resource creation/update entirely if the condition matches |
| `Audit` | Allows the operation, but flags it as non-compliant in Policy compliance reports |
| `Append` | Adds a field/value to the request before it's processed (e.g., force-adding a tag) |
| `Modify` | Similar to `Append`, but can also alter/remove existing fields, and supports remediation of already-existing resources |
| `DeployIfNotExists` | If a related resource is missing (e.g., a diagnostic setting), automatically deploys one |
| `Disabled` | Definition exists but enforces nothing — useful as a safe default during rollout/testing |

```bicep
// Example: an "Append" effect that force-adds an environment tag
policyRule: {
  if: {
    field: 'tags.environment'
    exists: 'false'
  }
  then: {
    effect: 'append'
    details: [
      {
        field: 'tags.environment'
        value: 'unmanaged'
      }
    ]
  }
}
```

---

## Policy Assignment — Applying a Definition at a Scope

```bicep
targetScope = 'subscription'

@description('Resource ID of the policy definition to assign')
param policyDefinitionId string

resource policyAssignment 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'deny-public-blob-assignment'
  properties: {
    displayName: 'Deny Public Blob Access — Production Subscription'
    policyDefinitionId: policyDefinitionId
    parameters: {
      effectType: {
        value: 'Deny'
      }
    }

    // "notScopes" excludes specific child scopes from this assignment —
    // e.g., a legacy resource group that's grandfathered in during migration
    notScopes: [
      resourceId('Microsoft.Resources/resourceGroups', 'legacy-resources')
    ]
  }

  // Assignments that use Append/Modify/DeployIfNotExists effects often
  // need a Managed Identity to actually perform the remediation action
  identity: {
    type: 'SystemAssigned'
  }
  location: 'westeurope'   // required when identity is SystemAssigned
}
```

### Tying Definition and Assignment Together in One Template

```bicep
targetScope = 'subscription'

resource policyDefinition 'Microsoft.Authorization/policyDefinitions@2021-06-01' = {
  name: 'deny-public-blob-access'
  properties: {
    displayName: 'Deny Public Blob Access on Storage Accounts'
    policyType: 'Custom'
    mode: 'Indexed'
    policyRule: {
      if: {
        allOf: [
          { field: 'type', equals: 'Microsoft.Storage/storageAccounts' }
          { field: 'Microsoft.Storage/storageAccounts/allowBlobPublicAccess', equals: 'true' }
        ]
      }
      then: {
        effect: 'Deny'
      }
    }
  }
}

// Implicit dependency, exactly like any other Bicep resource reference (file 02)
resource policyAssignment 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'deny-public-blob-assignment'
  properties: {
    displayName: 'Deny Public Blob Access — Subscription Wide'
    policyDefinitionId: policyDefinition.id   // implicit dependency
  }
}
```

---

## Policy Initiatives (Policy Set Definitions) — Grouping Multiple Policies

An **Initiative** bundles several Policy definitions into one assignable unit — useful for compliance frameworks ("CIS Benchmark," "ISO 27001") that genuinely require dozens of individual rules enforced together.

```bicep
resource policyInitiative 'Microsoft.Authorization/policySetDefinitions@2021-06-01' = {
  name: 'storage-security-baseline'
  properties: {
    displayName: 'Storage Security Baseline'
    policyType: 'Custom'

    policyDefinitions: [
      {
        policyDefinitionId: policyDefinition.id
        // Each entry can independently override the definition's parameters
        parameters: {
          effectType: { value: 'Deny' }
        }
      }
      {
        policyDefinitionId: resourceId('Microsoft.Authorization/policyDefinitions', 'require-https-storage')
        parameters: {}
      }
    ]
  }
}

resource initiativeAssignment 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'storage-baseline-assignment'
  properties: {
    displayName: 'Storage Security Baseline — All Subscriptions'
    policyDefinitionId: policyInitiative.id   // note: same property name, points at the INITIATIVE this time
  }
}
```

---

## Bicep vs Terraform for Policy — Where Bicep's Advantage Really Comes From

```hcl
# Terraform equivalent of the policy definition above
resource "azurerm_policy_definition" "example" {
  name         = "deny-public-blob-access"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Deny Public Blob Access on Storage Accounts"

  policy_rule = jsonencode({
    if = {
      allOf = [
        { field = "type", equals = "Microsoft.Storage/storageAccounts" },
        { field = "Microsoft.Storage/storageAccounts/allowBlobPublicAccess", equals = "true" }
      ]
    }
    then = {
      effect = "Deny"
    }
  })
}
```

Notice the `jsonencode({...})` wrapper Terraform requires — because HCL has no native way to express "this whole nested object is actually meant to become a literal JSON string for the underlying API field," it has to explicitly serialize it. Bicep needs **no such wrapper at all** — the `policyRule` property is declared as a plain nested object, full stop, because Bicep's underlying compiled output already *is* JSON. There's no serialization step to think about, no escaping, no `jsonencode`/`jsondecode` round-tripping.

| | Terraform | Bicep |
|---|---|---|
| Expressing the policy rule body | Nested HCL object, wrapped in `jsonencode(...)` | Nested Bicep object, used directly — no wrapper |
| IDE/linting support for the rule's internal structure | Generic JSON-in-a-string — limited tooling visibility into the nested shape | Full Bicep IntelliSense sees the nested structure as real Bicep objects (though it can't validate Policy-specific semantics like `[parameters(...)]` strings) |
| Risk of subtle escaping bugs | Slightly higher — string interpolation inside a `jsonencode` block, especially with quotes, is a classic gotcha | None — no string serialization involved at all |

This is the clearest illustration yet of the underlying theme from file 06: wherever an Azure-native concept is *already* JSON-shaped, Bicep simply uses it as-is, while Terraform must bridge the gap with explicit (de)serialization.

The next file covers **CI/CD pipeline integration** — wiring `what-if`, validation, and deployment into GitHub Actions and Azure DevOps pipelines, including environment promotion patterns and approval gates, directly parallel to how a Terraform pipeline runs `plan` on PR and `apply` on merge.