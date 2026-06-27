# Bicep Linter & Static Analysis

Every IaC language needs a way to catch mistakes before they reach the cloud — Terraform leans on `terraform validate`, `terraform fmt -check`, and third-party tools like `tflint`/`checkov`/`tfsec`. Bicep bakes static analysis **directly into the compiler itself**: every `az bicep build` (and therefore every `az deployment ... create` and `what-if`) automatically runs the linter as part of compilation — there's no separate tool to install or invoke.

```
Concepts covered:
  Built-in linter                          →  always-on, part of the compiler itself
  bicepconfig.json rule configuration         →  enabling/disabling/changing severity per rule
  Common built-in rules                         →  the ones you'll hit constantly
  PSRule for Azure                                →  deeper governance-style static analysis, Bicep-aware
  Custom rules / analyzers                          →  what's possible, what isn't
  Formatting (bicep format)                           →  Bicep's "terraform fmt" equivalent
```

---

## The Built-In Linter — Always On, No Separate Command

Recall from file 00: there is no separate `terraform validate`-style command in Bicep, because `az bicep build` **is** the validation step. The linter rides along with that same compile step, automatically, every time.

```bash
# This single command does ALL of: syntax check, type check, AND lint
az bicep build --file main.bicep

# Equivalent — building with linting diagnostics printed to stdout, no
# JSON file written (handy for quick checks in CI without littering the
# repo with .json output)
az bicep build --file main.bicep --stdout > /dev/null
```

```hcl
# Terraform requires a SEPARATE explicit command for each concern:
terraform validate    # syntax + internal consistency check
terraform fmt -check   # formatting check (separate from validate entirely)
# Linting (tflint) and security scanning (tfsec/checkov) are THIRD-PARTY
# tools, not part of Terraform core at all
```

| | Terraform | Bicep |
|---|---|---|
| Syntax/type validation | `terraform validate` (separate command) | Automatic — part of `az bicep build` / any deployment command |
| Linting | Third-party (`tflint`) — separate install, separate config file, separate CI step | Built into the compiler — configured via `bicepconfig.json`, zero extra install |
| Formatting check | `terraform fmt -check` (separate command) | `az bicep format --files main.bicep` (also separate, but no plugin/install needed) |

> This is a recurring theme across this whole curriculum (you've seen it with `init`, with state, with policy's `jsonencode`): wherever Terraform needed to bolt on extra tooling to compensate for being a generic, multi-provider DSL, Bicep's tighter, single-platform scope let Microsoft fold the same capability **directly into the core toolchain**.

---

## Linter Severity Levels

Every rule reports at one of three severities, and a build's exit code/CI behavior depends on which severities are present:

```
error    →  blocks compilation entirely; the build FAILS, deployment cannot proceed
warning  →  printed to console, build SUCCEEDS anyway — informational, not blocking
info     →  lowest-priority notice, easy to miss unless you're looking
```

---

## Configuring Rules via `bicepconfig.json`

File 08 introduced `bicepconfig.json` for module registry aliasing. Its other major job is **linter configuration** — enabling, disabling, or re-leveling individual rules, project-wide.

```json
{
  "analyzers": {
    "core": {
      "enabled": true,
      "rules": {
        "no-hardcoded-env-urls": {
          "level": "error"
        },
        "no-unused-params": {
          "level": "warning"
        },
        "secure-parameter-default": {
          "level": "error"
        },
        "no-unused-vars": {
          "level": "warning"
        },
        "prefer-interpolation": {
          "level": "info"
        }
      }
    }
  }
}
```

> This file should be **checked into source control** at your repository root (or, for monorepos, at the root of each Bicep project) — exactly like a `.tflint.hcl` config or a `.tfsec` ignore file would be for Terraform. Every developer and every CI run picks up the same rule configuration automatically, since the Bicep CLI walks up the directory tree from wherever it's invoked looking for this file.

---

## Common Built-In Rules Worth Knowing

### `no-hardcoded-env-urls`

Flags hardcoded Azure endpoint URLs (e.g., `https://management.azure.com`) instead of using environment-aware functions — important for templates that might target Azure Government or other sovereign clouds, not just public Azure.

```bicep
// ❌ Flagged
var endpoint = 'https://management.azure.com'

// ✅ Preferred — resolves correctly regardless of which Azure cloud this
// deployment targets
var endpoint = environment().resourceManager
```

### `secure-parameter-default`

Flags a `@secure()` parameter that has a **hardcoded, non-empty default value** — because that default would itself be a leaked secret sitting in source control, defeating the entire purpose of `@secure()`.

```bicep
// ❌ Flagged — defeats the purpose of @secure() entirely
@secure()
param adminPassword string = 'P@ssw0rd123!'

// ✅ Correct — no default; caller MUST supply it explicitly, every time
@secure()
param adminPassword string
```

### `no-unused-params` / `no-unused-vars`

Direct equivalent of `tflint`'s unused variable detection — flags a `param` or `var` that's declared but never referenced anywhere in the file. Usually a sign of leftover cruft from refactoring, occasionally a sign of a typo (you meant to use it but referenced the wrong name elsewhere).

### `prefer-unquoted-property-names`

Bicep allows both `name: value` and `'name': value` for object property keys. The linter nudges toward the unquoted form when the key is a valid identifier, for consistency.

```bicep
// ❌ Flagged (unnecessary quotes)
tags: {
  'environment': 'prod'
}

// ✅ Preferred
tags: {
  environment: 'prod'
}
```

### `use-stable-resource-identifiers`

Flags resources whose `name` property is built from something that could change between deployments in a way that causes unintended resource **replacement** (e.g., basing a name on `utcNow()`, which is different every single run) — a subtle but real footgun, since changing a resource's `name` in ARM almost always means delete-and-recreate, not update-in-place.

```bicep
// ❌ Flagged — utcNow() changes every deployment, so the storage account
// name changes every deployment, so ARM deletes and recreates it EVERY TIME
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'st${utcNow()}'
  // ...
}

// ✅ Correct — uniqueString(resourceGroup().id) is stable across deployments
// (file 05) — same inputs always produce the same output
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'st${uniqueString(resourceGroup().id)}'
  // ...
}
```

> This single rule prevents a class of bug that, in Terraform, would only surface the hard way — as an unexpected `-/+ destroy and re-create` in a `terraform plan` output someone wasn't carefully reading. Bicep catches the *root cause* (a non-deterministic name expression) statically, before any plan/what-if is even run.

---

## PSRule for Azure — Deeper Governance-Style Analysis

The built-in linter focuses on Bicep *language* correctness — syntax, types, obvious footguns. For deeper, opinionated **governance and Well-Architected-Framework-style** checks ("does this storage account enforce HTTPS-only," "does this VM have boot diagnostics enabled," "is this Key Vault using RBAC instead of legacy access policies"), the community/Microsoft-maintained **PSRule for Azure** tool is the standard add-on — conceptually filling the same space `checkov` or `tfsec` fill for Terraform.

```bash
# Install as a PowerShell module
Install-Module -Name PSRule.Rules.Azure -Scope CurrentUser

# Run against a compiled Bicep template (PSRule analyzes the compiled
# ARM JSON, not the .bicep source directly — recall file 00, the JSON
# IS the real deployable artifact)
az bicep build --file main.bicep
Assert-PSRule -InputPath ./main.json -Module PSRule.Rules.Azure
```

```hcl
# Terraform's closest equivalent invocation, using checkov
checkov --directory . --framework terraform
```

| | `tfsec` / `checkov` (Terraform) | PSRule for Azure (Bicep) |
|---|---|---|
| Scope | Security + best-practice misconfigurations across the `.tf` source | Security + Well-Architected Framework checks, against compiled ARM JSON |
| Maintained by | Open-source community / Bridgecrew (checkov) | Microsoft + open-source community |
| Typical CI placement | Same stage as `terraform plan`, before apply | Same stage as `what-if`, before deployment |

---

## Custom Rules / Analyzers — What's Possible

Unlike `tflint`, which supports a genuine plugin ecosystem for writing arbitrary custom rules in Go, Bicep's **core linter rule set is not currently end-user-extensible** in the same way — you configure severity and on/off state for Microsoft's predefined rule set (via `bicepconfig.json`), but you cannot author a brand-new linter rule yourself and load it into `az bicep build`.

For genuinely custom organizational policy ("every resource must have a `costCenter` tag," "no resource may be deployed outside these two regions"), the idiomatic Bicep answer is **not** a custom linter rule — it's one of:

1. **`precondition`-equivalent logic** via Bicep's `assert` statement (covered fully in the next file on testing)
2. **Azure Policy** (file 09) — enforced server-side, at the ARM level, regardless of what tool deployed the resource
3. **A wrapper module** that hardcodes/validates the organizational constraint as part of its own parameter decorators (file 01)

> This is a meaningful contrast with Terraform's `tflint` plugin model, and worth remembering: if you're tempted to "write a custom Bicep lint rule," redirect that energy toward Azure Policy instead — it enforces the rule at the control-plane level, catching violations regardless of whether someone deployed via Bicep, the Portal, raw ARM JSON, or even Terraform's own `azurerm` provider hitting the same subscription.

---

## Formatting — `bicep format`

```bash
# Format a file in place — Bicep's "terraform fmt" equivalent
az bicep format --files main.bicep

# Format every .bicep file in a directory tree
az bicep format --files **/*.bicep

# Check formatting without writing changes (useful for a CI gate)
az bicep format --files main.bicep --outdir /tmp/formatted-check
# (compare output against the original to detect drift — there is no
# single built-in "--check" flag the way "terraform fmt -check" has;
# diffing against a formatted copy is the standard CI workaround)
```

The VS Code Bicep extension (file 00) also formats automatically on save, by default — most teams never invoke `az bicep format` manually at all outside of a CI guard-rail step.

---

## Linter — Quick Reference

| Concern | Terraform | Bicep |
|---|---|---|
| Syntax/semantic validation | `terraform validate` | `az bicep build` (automatic, always runs) |
| Linting | `tflint` (third-party, separate install + config) | Built-in (`bicepconfig.json` for config) |
| Security/governance scanning | `tfsec` / `checkov` (third-party) | PSRule for Azure (Microsoft-maintained, against compiled JSON) |
| Formatting | `terraform fmt` | `az bicep format` |
| Custom rule authoring | Plugin system (Go-based, genuinely extensible) | Not extensible at the linter level — push custom org rules to Azure Policy instead |

The next file covers **testing Bicep deployments** in depth — the `assert` statement mentioned above, Pester-based test patterns for validating compiled templates, and strategies for testing modules in isolation before they're consumed downstream.