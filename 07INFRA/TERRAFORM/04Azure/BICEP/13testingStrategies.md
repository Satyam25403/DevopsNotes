# Testing Bicep Deployments

Terraform's testing story has matured into a dedicated `terraform test` framework (HCL-native `.tftest.hcl` files) plus long-standing third-party options like Terratest. Bicep's testing story is younger and leans on a combination of: a native `assert` statement for in-template validation, `what-if` for pre-deployment behavioral checks, and **Pester** (PowerShell's testing framework) for genuine automated test suites that exercise compiled templates. This file covers all three layers.

```
Concepts covered:
  assert statement                     →  in-template invariant checks, Bicep-native
  Module-level testing via what-if       →  behavioral "did this do what I expected" checks
  Pester-based test suites                 →  the closest equivalent to "terraform test" / Terratest
  Testing modules in isolation               →  parameter boundary testing, mocking patterns
  Test pyramid for IaC                         →  where each technique fits
```

---

## The `assert` Statement — Bicep-Native Invariant Checking

Bicep supports a top-level `assert` keyword (similar in spirit to the `precondition`/`postcondition` blocks from the Terraform set's meta-arguments lesson) that validates an expression **at compile/preflight time**, independent of any specific resource. Think of it as a lightweight, template-wide sanity check, distinct from a `lifecycle.precondition` which is scoped to one resource.

```bicep
@description('Environment name')
param environment string

@description('VM size — must be appropriately sized for production')
param vmSize string = 'Standard_DS1_v2'

// assert: a standalone, named boolean check. If it evaluates to false,
// the deployment fails BEFORE any resource is touched — same spirit as
// Terraform's precondition, but not tied to any one resource block.
assert isProductionSizedCorrectly = environment != 'prod' || vmSize != 'Standard_DS1_v2'
// Reads as: "if this is prod, vmSize must NOT be the smallest size"
```

```hcl
# Terraform's closest equivalent — a resource-scoped precondition
resource "azurerm_resource_group" "example" {
  # ...
  lifecycle {
    precondition {
      condition     = var.environment != "prod" || var.vm_size != "Standard_DS1_v2"
      error_message = "Production environments must not use the smallest VM size."
    }
  }
}
```

> **Key difference:** Terraform's `precondition` lives *inside* a specific resource's `lifecycle` block — it's conceptually "a guard on this particular resource's creation." Bicep's `assert` is **resource-independent** — a free-floating template-wide invariant, checked once, regardless of how many resources reference the values involved. This makes `assert` better suited for cross-cutting rules that don't naturally belong to any single resource (like the production-sizing rule above, which isn't really "about" any one resource — it's about the deployment's overall intent).

> **Current limitation worth knowing:** unlike Terraform's `precondition`, Bicep's `assert` does not (as of recent stable versions) support a custom `error_message`-style string — failures report the assertion's name and expression, which is less immediately readable to someone unfamiliar with the codebase. Name your assertions descriptively (as shown above: `isProductionSizedCorrectly`, not `check1`) to partially compensate.

---

## `what-if` as a Behavioral Test

File 06 introduced `what-if` as Bicep's `terraform plan` equivalent. It also doubles as a lightweight **test** — especially in CI, where you can assert on its *output*, not just eyeball it:

```bash
# Capture what-if output as JSON, then assert on its structure in a script —
# e.g., "fail the build if what-if proposes ANY delete, when this PR
# shouldn't be deleting anything"
az deployment group what-if \
  --resource-group example-resource \
  --template-file main.bicep \
  --parameters main.bicepparam \
  --result-format FullResourcePayloads \
  --no-pretty-print > whatif-output.json

# A simple guard script (conceptual — actual parsing depends on your CI's
# scripting language) checking for unexpected deletes
jq '.changes[] | select(.changeType == "Delete")' whatif-output.json
# If this returns anything, and the PR wasn't supposed to delete resources,
# fail the build here.
```

This pattern — parsing structured plan/what-if output and asserting on it programmatically, rather than just reading it — is exactly how more rigorous Terraform pipelines use `terraform plan -out=tfplan` + `terraform show -json tfplan` to build automated guard rails (e.g., "fail if any resource matching `*-prod-*` would be destroyed"). The technique transfers directly; only the JSON shape differs.

---

## Pester — Genuine Automated Test Suites for Bicep

**Pester** is PowerShell's native testing framework — broadly the same role RSpec plays for Ruby or pytest plays for Python. Because the Azure CLI/PowerShell ecosystem is the natural home for Bicep tooling, Pester has become the de facto standard for writing real, repeatable, assertion-based test suites against Bicep templates — filling the space Terratest or `terraform test` fill for Terraform.

### What Pester Tests Actually Check

Pester cannot (and shouldn't try to) replace a real deployment for verifying live Azure behavior — that's what a `dev`-environment deployment + manual or integration testing is for. What Pester tests around Bicep are well-suited for:

- **Compiled JSON structure** — "does this template, once compiled, actually produce a resource of type X with property Y set correctly"
- **Parameter validation behavior** — "does supplying an invalid `environment` value actually fail, the way `@allowed` promises it will"
- **Module contract stability** — "does this module's output shape still match what consumers expect" (a regression test against breaking changes)

### Example Test File — `main.tests.ps1`

```powershell
# main.tests.ps1 — run via: Invoke-Pester ./main.tests.ps1

BeforeAll {
    # Compile the Bicep file once, before any tests run — equivalent
    # conceptually to Terratest's pattern of running "terraform init" and
    # "terraform plan" once in a test setup phase
    az bicep build --file ./main.bicep --outfile ./main.test.json
    $script:compiledTemplate = Get-Content ./main.test.json -Raw | ConvertFrom-Json
}

Describe "Storage Account Module" {

    It "Declares a storage account resource" {
        $resourceTypes = $compiledTemplate.resources.type
        $resourceTypes | Should -Contain "Microsoft.Storage/storageAccounts"
    }

    It "Uses GRS replication for the storage account SKU" {
        $storageResource = $compiledTemplate.resources |
            Where-Object { $_.type -eq "Microsoft.Storage/storageAccounts" }

        $storageResource.sku.name | Should -Be "Standard_GRS"
    }

    It "Enforces HTTPS-only traffic" {
        $storageResource = $compiledTemplate.resources |
            Where-Object { $_.type -eq "Microsoft.Storage/storageAccounts" }

        $storageResource.properties.supportsHttpsTrafficOnly | Should -Be $true
    }

    It "Declares the expected output names" {
        $compiledTemplate.outputs.PSObject.Properties.Name |
            Should -Contain "storageAccountName"
    }
}

Describe "Parameter Validation" {

    It "Rejects an invalid environment value at deployment time" {
        # This invokes an ACTUAL what-if against Azure, specifically to
        # confirm the @allowed decorator (file 01) truly blocks bad input —
        # a genuine integration-style check, not just a static JSON assertion
        { az deployment group what-if `
            --resource-group test-rg `
            --template-file ./main.bicep `
            --parameters environment=production-typo `
          } | Should -Throw
    }
}
```

```bash
# Run the suite — typically wired into the CI pipeline (file 10) as a step
# that runs BEFORE "what-if", catching structural regressions even faster
Invoke-Pester ./main.tests.ps1 -Output Detailed
```

### Side-by-Side With Terraform's Native Test Framework

```hcl
# main.tftest.hcl — Terraform's own native test format
run "storage_account_uses_grs_in_prod" {
  command = plan

  variables {
    environment = "prod"
  }

  assert {
    condition     = azurerm_storage_account.account.account_replication_type == "GRS"
    error_message = "Production storage accounts must use GRS replication."
  }
}
```

| | Terraform (`terraform test`) | Bicep (Pester) |
|---|---|---|
| Native to the IaC tool itself? | ✅ Yes — `.tftest.hcl` is a first-party Terraform file format | ❌ No — Pester is a general PowerShell framework, not Bicep-specific; Bicep brings no native test file format of its own |
| What's being asserted against | Plan or apply output, directly via `command = plan` / `command = apply` | The **compiled JSON structure**, or live `what-if`/deployment behavior, inspected manually via PowerShell objects |
| Mocking provider calls | Supported via `mock_provider` blocks — genuinely test logic without touching real infrastructure | Not directly supported — Bicep has no provider layer to mock (file 00); testing realistic *behavior* generally requires a real (often a disposable, sandboxed) Azure resource group |

> **The honest tradeoff to communicate:** Terraform's native test framework can mock provider responses, letting you test complex conditional logic with zero real cloud calls. Bicep's Pester-based approach is **structurally weaker here** — because there's no provider abstraction to intercept, genuinely testing "does this `if (environment == 'prod')` branch produce the resource I expect" either means asserting against the **compiled JSON** (cheap, fast, no Azure calls — shown above) or running an actual `what-if`/deployment against a real sandbox resource group (slower, costs nothing, but does require live Azure access). Most mature Bicep test suites lean heavily on the compiled-JSON-assertion style specifically to avoid needing live Azure calls for every single test.

---

## Testing Modules in Isolation

Just like a Terraform module should be testable on its own, independent of whatever root module eventually consumes it, a Bicep module file is — recall file 03 — already a fully independent, standalone-deployable template. This makes isolated module testing straightforward: point Pester (or a raw `az bicep build`) directly at the module file, never the orchestrating `main.bicep`.

```powershell
Describe "Network Module in Isolation" {

    BeforeAll {
        az bicep build --file ./modules/network.bicep --outfile ./network.test.json
        $script:networkTemplate = Get-Content ./network.test.json -Raw | ConvertFrom-Json
    }

    It "Creates one subnet per entry in the subnets parameter" {
        # Compile-time structural check: confirm the COPY LOOP exists in
        # the compiled JSON for the subnets resource
        $subnetResource = $networkTemplate.resources |
            Where-Object { $_.type -eq "Microsoft.Network/virtualNetworks/subnets" }

        $subnetResource.copy.count | Should -Not -BeNullOrEmpty
        # (the exact JSON shape of a "for" loop, once compiled, is a "copy"
        # block — a detail worth knowing if you ever need to assert on
        # loop behavior structurally)
    }
}
```

### Boundary/Parameter Testing — Confirming Decorators Actually Work

Decorators (file 01) are only useful if they genuinely block bad input. A focused test suite should specifically probe the *edges* of every `@minValue`/`@maxValue`/`@allowed`/`@minLength` constraint — exactly the same "test the boundaries, not just the happy path" discipline you'd apply to a Terraform `variable` block's `validation` rule.

```powershell
Describe "VM Module Parameter Boundaries" {

    It "Rejects a disk size below the minimum" {
        { az deployment group what-if `
            --resource-group test-rg `
            --template-file ./modules/virtualMachine.bicep `
            --parameters storageDiskSizeGb=10 `
          } | Should -Throw   # @minValue(20) should block this
    }

    It "Accepts a disk size at exactly the minimum boundary" {
        { az deployment group what-if `
            --resource-group test-rg `
            --template-file ./modules/virtualMachine.bicep `
            --parameters storageDiskSizeGb=20 `
          } | Should -Not -Throw   # the boundary value itself must be ALLOWED, not rejected
    }
}
```

---

## The IaC Test Pyramid — Where Each Technique Fits

```
        ▲  Slow, expensive, highest confidence
        │
        │   Full live deployment to a sandbox RG + manual/integration verification
        │   (rarely automated per-PR; usually a periodic/scheduled pipeline run)
        │
        │   Pester tests calling REAL what-if / deployment commands
        │   (file 10's PR pipeline stage — moderate cost, real Azure validation)
        │
        │   Pester tests asserting on COMPILED JSON structure
        │   (fast, free, no Azure calls — run on every commit, even locally)
        │
        │   assert statements + @allowed/@minValue/etc decorators
        │   (instantaneous — enforced by the compiler itself, file 12's linter included)
        ▼  Fast, cheap, runs constantly
```

| Layer | Terraform Equivalent | Bicep Equivalent | When it runs |
|---|---|---|---|
| Compile-time invariants | `variable` `validation` blocks, `precondition` | `@allowed`/`@minValue`/etc decorators, `assert` | Every build, instantly |
| Structural/unit tests | `terraform test` with `command = plan`, mocked providers | Pester against compiled JSON | Every commit (CI, fast) |
| Behavioral/integration tests | `terraform test` with `command = apply` against real infra, or Terratest | Pester calling real `what-if`/deployment against a sandbox RG | PR pipeline (file 10), moderate cost |
| Full end-to-end verification | Scheduled/periodic full `apply` + manual or scripted verification | Scheduled deployment to a long-lived dev/staging environment | Periodic, not per-PR |

The next file is the **Bicep ↔ Terraform migration playbook** — for teams that already have a Terraform Azure estate and are evaluating (or actively executing) a move to Bicep, or vice versa: resource mapping strategy, state-to-deployment-history reconciliation, the `aztfexport`/`tf2bicep`-style tooling landscape, and a realistic phased migration plan.