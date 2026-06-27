# Bicep "State" Management: Deployment History, What-If & Deployment Stacks

This file directly answers the question every Terraform engineer asks first about Bicep: **if there's no `terraform.tfstate`, how does Bicep know what it's managing — and how do you safely tear things down?**

The short answer: **Bicep doesn't need a state file, because ARM itself already tracks every deployment server-side.** What Terraform must manage explicitly (a state file, a remote backend, locking, drift detection logic) is, for Bicep, simply a built-in capability of the Azure control plane itself.

---

## Recap: Why Terraform Needs State At All

Terraform talks to many different clouds through many different providers. None of those clouds were designed with Terraform in mind — AWS, GCP, and Azure don't natively track "which resources does this particular Terraform configuration own." So Terraform must **invent its own bookkeeping layer** (the state file) to remember: *what did I create, with what IDs, and what were its last-known attribute values.*

```
Terraform's problem:  Multiple unrelated clouds, none state-aware  →  MUST invent own state file
Bicep's situation:     Single cloud (Azure), natively state-aware   →  CAN piggyback on ARM's own tracking
```

ARM was built from day one to track every deployment that has ever targeted a given scope (resource group, subscription, etc.) — this is the same history that powers the Portal's "Deployments" blade you've seen referenced throughout these files. Bicep doesn't add a new state mechanism on top — it simply **uses the state-tracking Azure already had**.

---

## Deployment History — ARM's Native "State"

Every `az deployment group create` (or `sub`/`mg`/`tenant` variant) call is recorded as a **named, numbered, timestamped deployment object**, server-side, by ARM. You never see a local file for this — query it directly:

```bash
# List all deployments ever run against this resource group
az deployment group list --resource-group example-resource --output table

# Show full details of a SPECIFIC deployment by name
az deployment group show \
  --resource-group example-resource \
  --name my-first-deployment

# See exactly what changes a specific deployment made (its compiled template + parameters)
az deployment group show \
  --resource-group example-resource \
  --name my-first-deployment \
  --query properties.template
```

> Each deployment record includes the fully compiled ARM JSON template that was used, the parameter values supplied, the outputs produced, and a timestamped success/failure status. This is genuinely richer audit information than a bare `terraform.tfstate` snapshot gives you out of the box — Terraform's state only holds the *current* resolved attributes, not a full history of every past apply's template+params, unless you separately version your state backend (e.g., Azure Blob Storage soft-delete/versioning, as covered in the Terraform set's state-management file).

### Deployment Modes — `Incremental` vs `Complete`

This is the ARM-native concept that does the heavy lifting Terraform's state-diffing otherwise handles. Every deployment runs in one of two modes:

```bicep
// Mode is set via the CLI/PowerShell command, NOT inside the .bicep file itself
```

```bash
# Incremental (DEFAULT): only adds/updates resources defined in the template.
# Resources that exist in the resource group but are NOT in this template
# are left completely untouched.
az deployment group create \
  --resource-group example-resource \
  --template-file main.bicep \
  --mode Incremental

# Complete: adds/updates resources defined in the template, AND DELETES
# any resource in the resource group that is NOT defined in the template.
# This is the mode that gives ARM genuinely Terraform-like "this is the
# full desired state, make reality match it exactly" semantics.
az deployment group create \
  --resource-group example-resource \
  --template-file main.bicep \
  --mode Complete
```

| | `Incremental` (default) | `Complete` |
|---|---|---|
| Resources in template | Created/updated | Created/updated |
| Resources in resource group but NOT in template | Left alone | **Deleted** |
| Terraform analogy | Closest to: only ever running `terraform apply` and never noticing/removing manually-created drift | Closest to: `terraform apply` with a config that is treated as the **complete** ground truth — anything not declared is drift to be removed |
| Risk | Orphaned resources can accumulate silently | ⚠️ A typo or incomplete template can **delete real, intentional resources** that simply weren't included this run |

> **Use `Complete` mode with extreme caution**, and almost always pair it with `what-if` (next section) run first. A resource group with even one resource created manually through the Portal — completely intentionally — will be **deleted** the moment someone runs a `Complete`-mode deployment that doesn't happen to declare it.

---

## `what-if` — Bicep's `terraform plan`

This is the most direct, practical Terraform analogy in the whole Bicep toolchain. `what-if` compiles your Bicep, compares it against **live Azure resource state** (not a cached state file — the actual current resources, queried live), and shows exactly what would change, without changing anything.

```bash
az deployment group what-if \
  --resource-group example-resource \
  --template-file main.bicep \
  --parameters main.bicepparam
```

Sample output interpretation (conceptually — actual CLI output uses color-coded symbols):

```
+ Create   → resource doesn't exist yet, will be created
~ Modify   → resource exists, some properties will change
- Delete   → (Complete mode only) resource exists but isn't in the template, will be deleted
= NoChange → resource exists and matches the template exactly, nothing to do
! NoEffect → a property change was requested but won't actually take effect (e.g. immutable property)
```

```hcl
# Terraform equivalent
terraform plan
# +  create
# ~  update in-place
# -  destroy
# -/+ destroy and re-create
```

| | `terraform plan` | `az deployment ... what-if` |
|---|---|---|
| Compares against | Local/remote `.tfstate` file | **Live Azure resources, queried directly** — no intermediate file at all |
| Can go stale | ✅ Yes — if state drifts from reality between plan and apply, or if state file itself is stale/corrupted | ❌ No — every `what-if` run re-queries live Azure state fresh, every single time |
| Detects manual/out-of-band changes | Only on next `refresh`/`plan`, and only for resources Terraform manages | Always — because it's never comparing against a stale snapshot in the first place |

> This is arguably **Bicep's single biggest practical advantage** over Terraform's state model: because there's no cached snapshot to go stale, `what-if` can never lie to you about the current state of the world the way a corrupted or out-of-sync `.tfstate` file occasionally can.

---

## So... Is There Really Nothing to Lose? The Real Tradeoffs

Bicep's "no state file" model isn't strictly superior in every dimension — it trades away a few things Terraform state gives you for free:

| Capability | Terraform (via state file) | Bicep (via ARM deployment history) |
|---|---|---|
| Know exactly which resources "belong" to this config, with no ambiguity | ✅ Explicit — state file IS the list | ⚠️ Implicit — inferred from "what's in this template" + deployment mode; less explicit ownership tracking, especially across multiple overlapping deployments targeting the same resource group |
| Safely import a pre-existing, manually-created resource into management | `terraform import` | `az resource list` + write matching Bicep, then deploy in `Incremental` mode (ARM treats a matching resource as "already exists, just update it") — no formal "import" command, but works in practice for most resource types |
| Track resources spanning MULTIPLE unrelated templates as one logical unit | One state file can span many `.tf` files freely | Each deployment record is independent; you need a higher-level construct to unify them — this is exactly what **Deployment Stacks** (next section) solve |
| Lock against concurrent modification | Native state locking (file 01 of the Terraform set) | ARM serializes deployments to the same scope at the API level, but there is no separate explicit "lock" primitive you interact with directly |

---

## Deployment Stacks — Bicep's Closest True Equivalent to a "Managed State Boundary"

**Deployment Stacks** are a newer Azure feature built specifically to close the gap above — giving Bicep something that behaves much more like Terraform's *explicit, persistent, queryable* notion of "this stack of resources is one managed unit," beyond what raw deployment history alone provides.

```bash
# Create a Deployment Stack — this is now a persistent, named, queryable object
# in Azure, distinct from a one-off deployment record
az stack group create \
  --name my-app-stack \
  --resource-group example-resource \
  --template-file main.bicep \
  --parameters main.bicepparam \
  --deny-settings-mode none \
  --action-on-unmanage deleteResources
```

```bash
# List all resources currently managed by this stack — directly analogous
# to "terraform state list"
az stack group show --name my-app-stack --resource-group example-resource

# Update the stack — analogous to "terraform apply" against existing state
az stack group create \
  --name my-app-stack \
  --resource-group example-resource \
  --template-file main.bicep \
  --action-on-unmanage deleteResources

# Delete the ENTIRE stack and everything it manages — directly analogous
# to "terraform destroy"
az stack group delete \
  --name my-app-stack \
  --resource-group example-resource \
  --action-on-unmanage deleteResources
```

### `--action-on-unmanage` — Controlling What Happens to Removed Resources

This flag controls behavior the moment a resource is **removed from the template** (not the whole stack — just one resource dropping out of the next update) — directly comparable to deciding what happens when you delete a `resource` block from a `.tf` file:

```bash
--action-on-unmanage deleteResources     # delete the resource from Azure entirely (matches Terraform's default behavior when a resource block is removed)
--action-on-unmanage detachResources       # leave the resource in Azure, just stop tracking/managing it in this stack
```

### `--deny-settings-mode` — Bicep's Closest Equivalent to `prevent_destroy`

Recall Terraform's `lifecycle { prevent_destroy = true }` from the meta-arguments lesson. Deployment Stacks offer something structurally similar but enforced by **Azure RBAC itself**, not just a flag inside your template that anyone could delete:

```bash
az stack group create \
  --name my-app-stack \
  --resource-group example-resource \
  --template-file main.bicep \
  --deny-settings-mode denyDelete \
  --deny-settings-excluded-actions "Microsoft.Storage/storageAccounts/listKeys/action"
```

```
denyDelete       → blocks delete operations on managed resources, from ANYONE,
                    via an actual Azure Policy-like deny assignment — not just
                    a flag inside a file someone could edit out
denyWriteAndDelete → blocks both modification AND deletion
none               → no extra protection beyond normal RBAC (default)
```

> **Meaningful difference from Terraform's `prevent_destroy`:** Terraform's protection lives *inside your `.tf` file* — remove the `lifecycle` block, run `apply`, and the protection is gone (the Terraform set's meta-arguments lesson calls this out explicitly: "treat it as a speed bump, not a lock"). A Deployment Stack's `deny-settings-mode` is enforced as an **actual Azure-level deny assignment** sitting on the resources themselves — someone would need explicit RBAC permission over the *stack* (not just the template file) to remove that protection. This is a genuinely stronger guarantee.

---

## Putting It All Together — The Full Bicep "State" Picture

```
                    az deployment group create
                              │
                              ▼
              ┌────────────────────────────────┐
              │   ARM Deployment History         │   ← every run recorded, queryable forever,
              │   (always present, automatic)    │     full template + params + outputs retained
              └────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                                 ▼
   az deployment ... what-if          az stack group create
   (ad-hoc "plan" check,                (persistent, named, RBAC-protected
    no persistent object created)        management boundary — closest
                                          analogue to terraform.tfstate +
                                          terraform destroy combined)
```

| Terraform Concept | Closest Bicep Equivalent | Where Covered |
|---|---|---|
| `terraform.tfstate` | ARM's native deployment history (automatic, no setup) | This file |
| Remote backend (S3/Azure Blob) | N/A — there's no file to relocate; ARM already IS the remote, shared source of truth | This file |
| `terraform plan` | `az deployment ... what-if` | This file |
| `terraform apply` | `az deployment group create` (`Incremental` mode) | File 00, this file |
| `terraform apply` with a config treated as complete ground truth | `az deployment group create --mode Complete` | This file |
| `terraform destroy` | `az group delete` (whole RG) or `az stack group delete` (scoped to a stack) | File 00, this file |
| `lifecycle { prevent_destroy = true }` | Deployment Stack `--deny-settings-mode denyDelete` | This file |
| `terraform state list` | `az stack group show` | This file |
| `terraform import` | Deploy matching Bicep in `Incremental` mode against an existing resource (ARM updates in place rather than erroring) | This file |
| State locking | ARM's own request serialization at the API level (implicit, no separate lock primitive) | This file |

---

## Summary: The Big Mental Shift

If there is one idea to walk away with from this entire Bicep curriculum, it's this: **Terraform builds its own abstractions on top of clouds that weren't designed for it — providers, state files, backends, locking. Bicep simply uses the abstractions Azure already had**, because Bicep was built by the same team that owns ARM, specifically to be a thin, ergonomic layer over a control plane that was *already* tracking everything Bicep needs. Every "where's the Bicep equivalent of X" question in this curriculum has essentially the same underlying answer: **ARM was already doing that — Bicep just exposes it.**

This is also precisely why Bicep has no multi-cloud ambitions, and never will: the entire design is a bet that being *perfectly* native to one platform beats being *adequately* abstracted across many.