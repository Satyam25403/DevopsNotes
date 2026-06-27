# The Bicep ↔ Terraform Migration Playbook

This file is for two audiences: teams with an existing Terraform Azure estate evaluating or executing a move to Bicep, and teams doing the reverse. Migrations between IaC tools are rare to do perfectly, common to do partially, and the practical reality for most organizations is **long-term coexistence** rather than a clean cutover. This file covers the realistic strategy, the tooling that exists today, and the gotchas that bite teams who underestimate the state/deployment-history reconciliation problem.

```
Concepts covered:
  Why migrate at all                       →  realistic motivations, and the case for NOT migrating
  The fundamental asymmetry                  →  state file vs deployment history — what doesn't translate
  Tooling: aztfexport, decompile, etc.         →  what exists, what it actually does, where it falls short
  A phased, low-risk migration strategy          →  strangler-fig pattern applied to IaC
  Coexistence patterns                              →  running both tools against the same subscription safely
  Reverse direction: Bicep → Terraform                →  the same problem, mirrored
```

---

## Why Migrate At All? (And the Case for Not Migrating)

Legitimate reasons teams move **Terraform → Bicep**:
- Azure-only shop, no multi-cloud need on the horizon — removing the state/backend/provider-version management overhead (files 00, 06) is a genuine simplification
- Wanting to stay in lockstep with the newest Azure features the moment they ship, without waiting on `azurerm` provider release cadence
- Wanting native ARM-level governance integration (Policy, Deployment Stacks) without a translation layer

Legitimate reasons teams move **Bicep → Terraform**, or simply stay on Terraform:
- Genuine multi-cloud or multi-tool estate (AWS + Azure + GCP) where one consistent tool/workflow across clouds has real organizational value
- Existing investment in Terraform Cloud/Enterprise workflows, policy-as-code via Sentinel/OPA, or a mature internal module registry already built around Terraform
- A larger hiring pool already fluent in HCL versus Bicep specifically

> **The honest recommendation for most teams:** migrating a large, working Terraform estate to Bicep purely for the sake of "using the native tool" is rarely worth the risk and effort unless one of the concrete motivations above is genuinely present and significant. **New** Azure-only projects are a much lower-risk place to start with Bicep than migrating existing, working infrastructure.

---

## The Fundamental Asymmetry: State File vs Deployment History

This is the single biggest practical obstacle, and it's worth understanding precisely *why* it's hard, not just *that* it's hard.

```
Terraform's tfstate:    Explicit, file-based, EXACTLY enumerates every resource
                         this configuration owns, by resource address.

ARM deployment history (file 06):  A log of past deployment OPERATIONS,
                                     not a clean enumerated ownership list.
                                     "Ownership" of a resource by a particular
                                     Bicep template is, at best, INFERRED —
                                     there's no single authoritative file
                                     saying "this Bicep template manages
                                     exactly these 47 resources."
```

When you migrate Terraform → Bicep, you are moving from a tool with an **explicit, queryable ownership ledger** to a tool whose closest equivalent (Deployment Stacks, file 06) is opt-in and must be deliberately set up — it is not automatically reconstructed from the resources' mere existence in Azure. **Nothing automatically tells Bicep "these existing resources, created previously by Terraform, are the ones this new Bicep template should treat as already-existing and manage going forward."** You have to establish that relationship yourself, resource by resource, the same way Terraform's own `terraform import` requires explicit, resource-by-resource declaration.

---

## Tooling Landscape

### `aztfexport` — Reverse-Engineering Terraform FROM Existing Azure Resources

Microsoft's `aztfexport` tool (formerly `aztfy`) scans a resource group (or subscription) of **already-existing** resources and generates matching Terraform configuration + a populated state file — solving the *opposite* direction's import problem (bringing un-managed or differently-managed resources under Terraform management) extremely well.

```bash
# Export an entire resource group's existing resources into Terraform
# configuration + a real, populated terraform.tfstate
aztfexport resource-group example-resource
```

> This tool is genuinely excellent for the **Bicep/manual → Terraform** direction, or for bringing entirely unmanaged resources under Terraform. It does **not** directly help with Terraform → Bicep, since it generates Terraform, not Bicep — but it's worth knowing about, since "migrate everything to one tool" projects often discover mixed-ownership resource groups (some Terraform-managed, some manually created, some Bicep-managed) and need exactly this kind of reverse-engineering capability regardless of which direction the overall migration is going.

### `az bicep decompile` — ARM JSON → Bicep, Mechanically

If you have existing ARM JSON templates (which Terraform's `azurerm` provider produces internally, even though you never see them directly), Bicep ships a built-in decompiler:

```bash
az bicep decompile --file existing-template.json
# → produces existing-template.bicep
```

This is **not** a Terraform → Bicep tool directly — it converts raw ARM JSON to Bicep. The practical path for Terraform → Bicep migration is therefore usually:

```
1. Identify the live resources Terraform currently manages (terraform state list)
        │
        ▼
2. EITHER: manually author equivalent Bicep (recommended for anything beyond
   trivial resources — gives you a chance to apply this curriculum's idioms,
   modules, UDTs, etc., rather than inheriting verbose decompiled output)
        │
   OR: export the resource group's ARM template from the Portal/CLI
   ("az group export"), then "az bicep decompile" it as a faster-but-messier
   starting point, to be cleaned up by hand afterward
        │
        ▼
3. Bring the new Bicep template's resources under management WITHOUT
   recreating them — deploy in Incremental mode (file 06) against the
   EXISTING resource group; ARM treats a Bicep-declared resource that
   already exists, with a matching name, as "update in place," not
   "create new" — functionally similar to Terraform's import, but without
   a dedicated "import" command; the deployment itself IS the import,
   as long as names/properties line up correctly
        │
        ▼
4. Verify with what-if FIRST (file 06) — confirm it shows "Modify" or
   "NoChange" for these resources, NEVER "Delete" + "Create" — a mismatch
   here means your Bicep declaration doesn't actually match the live
   resource closely enough, and deploying as-is would destructively replace it
        │
        ▼
5. Only once Bicep deployment is confirmed safe (via what-if), decommission
   the OLD Terraform configuration: remove the resources from Terraform
   state WITHOUT destroying them (terraform state rm <address>) — this
   is the critical step that hands off ownership without anyone's tool
   trying to delete the real infrastructure
```

```bash
# Step 5 in detail — removing a resource from Terraform's bookkeeping
# WITHOUT touching the real Azure resource at all
terraform state rm azurerm_storage_account.example
# The storage account still exists in Azure, completely untouched.
# Terraform simply "forgets" it was ever managing it.
# From this point forward, ONLY the new Bicep template/Deployment Stack
# should be considered authoritative for this resource.
```

> **The single most dangerous failure mode in this entire migration**: deploying the new Bicep template *before* confirming via `what-if` that it recognizes the existing resource as "already there," and/or forgetting to `terraform state rm` afterward — leaving **two tools simultaneously believing they own the same resource**. If both later run independently (e.g., a stale Terraform pipeline still scheduled to run nightly), you risk a destructive fight over the same resource's properties, or an outright delete-recreate cycle. Treat the handoff as a single atomic checklist, not two independent tasks done on different days.

---

## A Phased, Low-Risk Migration Strategy

Borrowing directly from the "strangler fig pattern" used in application migrations: **never attempt a single big-bang cutover.** Migrate resource-group-by-resource-group, or even resource-type-by-resource-type within a single resource group, validating each slice thoroughly before moving to the next.

```
Phase 1: New resources ONLY are built in Bicep going forward.
         Existing Terraform-managed resources are left entirely alone.
         (Lowest risk possible — zero migration of existing infrastructure,
         purely a "going forward" policy change.)

Phase 2: Pick the LOWEST-RISK existing resource group (e.g., a sandbox
         or non-production environment) to migrate first.
         Run the full export → author → what-if → state rm checklist above.
         Let it run for at least one full deployment cycle in its new
         Bicep-managed form before touching anything else.

Phase 3: Migrate remaining non-production environments, applying lessons
         learned from Phase 2's first migration.

Phase 4: Migrate production — ONLY after the same pattern has been proven
         repeatedly in lower environments. Schedule a maintenance window;
         treat this exactly as carefully as you would any other
         production infrastructure change with real blast radius.

Phase 5: Decommission Terraform tooling/pipelines/state backend for the
         fully-migrated estate, ONLY once you're confident no resource
         is still dual-owned.
```

---

## Coexistence Patterns — Running Both Tools Long-Term

Many organizations never complete a full migration, and that's a perfectly legitimate end state, not just a "still in progress" one. The key discipline for safe long-term coexistence:

| Rule | Why |
|---|---|
| **Never let both tools manage the same resource group**, even temporarily | The dual-ownership risk described above — pick one tool per resource group/scope, permanently |
| Document which tool owns which scope, visibly (README, a tagging convention, a wiki page) | New team members and new automation need to know, at a glance, which tool to touch for a given resource group |
| If resources in one tool's scope need to reference resources in the other's scope, use `data` blocks (Terraform) or `existing` (Bicep, file 03) — never duplicate management | Cross-tool references should always be read-only lookups, never attempts by both tools to manage the same object |
| Standardize naming/tagging conventions across BOTH tools | Makes it trivially easy to audit, at a glance in the Portal, which tool "owns" any given resource — even without checking source control |

```bicep
// Bicep referencing a resource that Terraform manages elsewhere —
// read-only, exactly like file 03's "existing" pattern for shared infrastructure
resource terraformManagedVnet 'Microsoft.Network/virtualNetworks@2023-09-01' existing = {
  name: 'shared-platform-vnet'
  scope: resourceGroup('platform-shared-rg')   // a DIFFERENT resource group,
                                                   // managed by the OTHER tool
}
```

```hcl
# Terraform referencing a resource that Bicep manages elsewhere — same
# read-only principle, via a data source
data "azurerm_virtual_network" "bicep_managed_vnet" {
  name                = "shared-platform-vnet"
  resource_group_name = "platform-shared-rg"
}
```

---

## The Reverse Direction: Bicep → Terraform

The exact same asymmetry applies, mirrored. Moving from Bicep to Terraform means going from ARM's implicit, history-based tracking to Terraform's explicit state file — and here, `aztfexport` (introduced above) is **directly and fully applicable**, since it was built precisely for "bring existing Azure resources, regardless of how they got there, under Terraform management."

```bash
# This works identically whether the existing resources were created by
# Bicep, the Portal, raw ARM templates, or even manually via az CLI —
# aztfexport doesn't care HOW a resource was created, only that it exists
aztfexport resource-group example-resource
```

The phased strategy, the dual-ownership warning, and the coexistence patterns above all apply identically in this direction — only the specific tooling commands change (`aztfexport` instead of `az bicep decompile` + manual authoring; `terraform import`/the `import` block instead of `Incremental`-mode redeployment).

---

## Migration — Quick Reference

| Concern | Terraform → Bicep | Bicep → Terraform |
|---|---|---|
| Reverse-engineer existing resources into the NEW tool's config | Manual authoring, or `az group export` + `az bicep decompile` (messy starting point) | `aztfexport` (mature, well-maintained, handles state generation automatically) |
| Bring existing resources under the new tool's management, without recreating them | Deploy in `Incremental` mode; verify via `what-if` shows no Delete+Create | `terraform import` (CLI command or HCL `import` block) |
| Release ownership from the OLD tool, without destroying resources | `terraform state rm <address>` | No direct equivalent needed — Bicep has no state to "forget"; simply stop deploying that template/Deployment Stack |
| Risk of dual ownership if done carelessly | High — same as the reverse | High — same as the reverse |

The next file covers **multi-subscription and landing-zone patterns** — how the scope concept from file 02 scales up to genuinely enterprise-wide deployments: management group hierarchies, hub-and-spoke networking provisioned via Bicep, and the Azure Landing Zone (ALZ) reference architecture's relationship to Bicep specifically.