---
name: terraform-advanced-module-composition
description: Design, structure, and version reusable Terraform modules beyond the basics — when to extract a module vs. keep resources inline, folder-per-environment vs. tfvars-per-environment root module layouts, local vs. Git vs. Terraform Registry module sourcing with proper `ref`/`version` pinning, and the count/for_each/dynamic-block/provider-alias/lifecycle meta-arguments that make modules composable. Use this when the user asks "should this be a module," "how do I structure my Terraform root module," "how do I version a module from a Git repo/registry," "how do I repeat a block of resources across regions," "why is my for_each/count causing resources to get destroyed and recreated," or wants to review/refactor a Terraform module's interface. Distinct from terraform-iac-workflow-design (basic module variables/outputs) — this goes into module design judgment, iteration meta-arguments, and multi-source versioning strategy.
---

# Terraform Advanced Module Composition

Terraform's only unit of encapsulation is the folder. There's no function keyword, no class —
just `module "x" { source = "..." }` pointing at another directory. That simplicity hides a real
design discipline: deciding *when* a module earns its abstraction, how to structure a root
module across environments, how to iterate a module safely, and how to pin its version so an
upstream change doesn't silently break every consumer.

## When to extract a module (and when not to)

Treat this like deciding whether to extract a function in any other language — value must
exceed the abstraction cost:

- **One resource, one module** — almost always wrong. You've added an indirection layer
  (`module.foo.thing` instead of `resource.thing`) without hiding meaningful complexity. Just
  reference the resource directly.
- **A few tightly coupled resources with one clear interface** — the sweet spot. Example: a
  security group plus its rules, or a Key Vault plus its access policy plus a secret. The module
  trades one or two extra input variables for hiding several resource blocks the consumer
  shouldn't have to think about individually.
- **A repeating pattern across N instances** (e.g., the same frontend+backend+configmap trio
  repeated per microservice, or a full regional deployment repeated per region) — extract the
  repeated unit into a module and let the *caller* iterate the module with `count`/`for_each`,
  rather than iterating every resource inside it. Each module instance then only has to reason
  about a single copy of itself; the complexity of "how many" lives entirely at the call site.

Two module archetypes to recognize:
- **Scenario-driven module** — narrow interface, does one specific thing ("give me a standard
  NSG"). Easy to consume, easy to reason about.
- **Framework module** — broad interface with many levers (often accepting complex object
  types as inputs) to support many use cases. More powerful, more expensive to maintain and to
  learn. Don't reach for this shape until you have multiple real consumers with genuinely
  different needs — premature framework modules are worse than copy-paste.

Avoid over-typing module inputs as bare strings "because it's simpler" — use `bool`, `number`,
and structured object types where the value actually has that shape. It removes a whole class
of caller mistakes and stringly-typed casting inside the module.

## Root module layout: pick one, know the tradeoff

**Folder per environment** — `terraform/{dev,test,prod}/` each with its own `main.tf`,
`versions.tf`, `variables.tf`, `terraform.tfvars`, fully independent:

```
terraform/
  dev/   {main.tf, versions.tf, variables.tf, terraform.tfvars}
  test/  {main.tf, versions.tf, variables.tf, terraform.tfvars}
  prod/  {main.tf, versions.tf, variables.tf, terraform.tfvars}
```

Maximum isolation — each environment can run a different Terraform version, provider version,
or even structurally different resources. Good fit for teams not yet running a disciplined
GitFlow/branch-promotion process. Downside: drift between environments' *code*, not just their
config, is easy to introduce by accident (someone edits `prod/main.tf` and forgets `dev/main.tf`).

**Shared module + tfvars per environment** — one `modules/solution/` module, one root module,
environment differences live entirely in `.tfvars`:

```
terraform/
  modules/solution/ {main.tf, versions.tf, variables.tf}
  env/               {dev.tfvars, test.tfvars, prod.tfvars}
  main.tf
  versions.tf
  variables.tf
```

Enforces structural parity between environments — you can't accidentally give prod a resource
dev doesn't have. Pairs naturally with a GitFlow-style branch-promotion pipeline. Costs some
flexibility: a genuinely environment-specific resource (e.g., prod-only WAF) needs a
conditional (`count = var.enable_waf ? 1 : 0`) rather than just omitting it.

Pick based on how much your environments are *allowed* to structurally diverge, not on which
looks cleaner in a demo.

## Module sourcing and version pinning

| Source | Syntax | Pinning |
|---|---|---|
| Local | `source = "../modules/rando"` | N/A — same commit as the caller |
| Git | `source = "git::ssh://git@host/org/repo//path"` | Append `?ref=<tag>` — **never omit this** |
| Terraform Registry | `source = "aztfmod/caf/azurerm"` | `version = "~> 5.5.0"` |

```hcl
# Local — for module reuse within a single project/repo (e.g., a per-region deployment unit)
module "app_region" {
  source     = "../modules/app-region"
  for_each   = local.regions
  region     = each.key
  node_count = each.value.node_count
}

# Git — for cross-repo reuse within an org, without publishing to a registry
module "network" {
  source = "git::ssh://git@ssh.dev.azure.com/v3/myorg/myproject/mymodules//network?ref=v2.3.0"
}

# Terraform Registry — for public, versioned modules
module "caf" {
  source  = "aztfmod/caf/azurerm"
  version = "~> 5.5.0"
}
```

**Referencing a Git module without `?ref=`** pulls the default branch's latest commit on every
`init` — the single most common cause of "it worked yesterday, it's broken today" in a team
using shared modules. Always pin to a tag; bump the tag deliberately when you want the update.
The Terraform Registry forces version pinning by convention (registry modules require a
`version` constraint), which is one reason to prefer it over a bare Git source when publishing
something broadly reusable.

## Iteration meta-arguments: count vs. for_each vs. dynamic vs. for-expression

Four different tools for four different jobs — mixing them up is the most common source of
"why did Terraform want to destroy and recreate everything" surprises.

**`count`** — index-based, use only for an anonymous, append-only collection where you don't
care *which* instance is which, only *how many*:

```hcl
resource "aws_instance" "node" {
  count = var.node_count
  # ...
}
```

Danger: removing or reordering an item in the middle of a list shifts every subsequent index,
which Terraform reads as "destroy this instance, create a different one" for every shifted
item. Only safe when the underlying list is treated as append-only (add/remove strictly at the
end) or when instances are truly interchangeable.

**`for_each`** — key-based, use whenever items have a natural unique identifier (a region name,
an environment name, a map key):

```hcl
locals {
  regions = {
    westus = { node_count = 5 }
    eastus = { node_count = 8 }
  }
}

module "regional_deployment" {
  for_each   = local.regions
  source     = "../modules/app-region"
  node_count = each.value.node_count
}
```

Removing `eastus` from the map removes exactly that instance — no index-shift blast radius on
`westus`. This is the correct default for multi-region/multi-account fan-out; reach for `count`
only when there's genuinely no natural key.

**`for` expressions** — not attached to a resource/module/data block; used to transform
collections in-memory (list→list, map→map, list→map):

```hcl
# list -> list
region_names = [for r in var.regions : upper(r.name)]

# map -> list of objects (flatten)
region_array = [for k, v in var.regions : { region = k, node_count = v.node_count }]

# list -> map (needed before you can for_each over a plain list)
foo_map = { for idx, el in local.foo_list : el => idx }
```

`for_each` requires a map (or a set of strings) — if your data is a list of objects, convert it
with a `for` expression first rather than falling back to `count`.

**Dynamic blocks** — for repeating a *nested block* inside a single resource, driven by a
collection, instead of hand-writing N copies:

```hcl
locals {
  geo_regions = { westus = 0, eastus = 1 }
}

resource "azurerm_cosmosdb_account" "db" {
  name                = "cosmos-foobar"
  location            = azurerm_resource_group.foobar.location
  resource_group_name = azurerm_resource_group.foobar.name
  offer_type          = "Standard"
  kind                = "MongoDB"

  dynamic "geo_location" {
    for_each = local.geo_regions
    content {
      location          = geo_location.key
      failover_priority = geo_location.value
    }
  }
}
```

**Splat expressions** (`resource.name[*].attr`) — the concise way to pull one attribute off
every instance of a `count`/`for_each` resource without writing a full `for` expression:
`all_ids = aws_instance.node[*].id`.

## Meta-arguments that control resource behavior, not iteration

**`provider` (aliasing)** — required whenever a single root module targets more than one
region/account/subscription with the same provider:

```hcl
provider "aws" {
  region = "us-east-1"
}
provider "aws" {
  alias  = "secondary"
  region = "us-west-1"
}

resource "aws_instance" "bar" {
  provider = aws.secondary
}
```
Resources with no explicit `provider` use the unaliased (default) provider block. AWS/GCP use
this for cross-region; Azure typically only for cross-subscription (rarer).

**`depends_on`** — for *implicit* dependencies Terraform's graph can't see. Classic case: an
Azure Key Vault access policy must exist before a secret can be written, but there's no
attribute reference linking them, so Terraform will try to create both in parallel and the
secret creation will fail with a race:

```hcl
resource "azurerm_key_vault_secret" "foo" {
  name         = "foo"
  value        = "bar"
  key_vault_id = azurerm_key_vault.top_secret.id
  depends_on   = [azurerm_key_vault_access_policy.terraform]
}
```
Use `depends_on` sparingly and only for genuine implicit dependencies — overusing it (a common
symptom of code-generated/imported configs) adds bloat and hides the real dependency graph.

**`lifecycle`** block — four levers for edge cases:
- `create_before_destroy = true` — build the replacement before tearing down the original
  (certs, security groups — anything where a gap would be an outage).
- `prevent_destroy = true` — a second gate an operator must deliberately remove before
  `destroy` can touch this resource; use on anything stateful or foundational.
- `ignore_changes = [tags]` — tell Terraform to stop fighting another actor (a tagging policy
  engine, a human) that mutates specific attributes outside of Terraform.
- `replace_triggered_by = [...]` — force a replacement when a *related* resource changes, even
  though there's no attribute-level dependency Terraform would otherwise notice.

## Common pitfalls

- **Bare Git module source with no `?ref=`** — every `init` can silently pull a newer, possibly
  breaking, version of the module. Always pin a tag.
- **`count` on a list that isn't append-only** — inserting/removing from the middle triggers a
  cascade of destroy/recreate on every subsequent index. Use `for_each` with a map instead.
- **One-resource modules** — added indirection with no complexity actually hidden; inline it.
- **Framework modules built before there are 2+ real consumers** — speculative generality adds
  object-typed inputs nobody asked for yet and is harder to walk back than to add later.
- **Stringly-typed module inputs** — using `string` for booleans/numbers "to keep it simple"
  just moves the type-casting into every consumer and every future maintainer's head.
- **Missing `depends_on` for implicit dependencies** — resources that reference each other only
  through side effects (like the Key Vault access-policy case) will race unless the dependency
  is declared explicitly.

## Checklist

- [ ] Every module either hides 2+ genuinely coupled resources or encapsulates a pattern meant
      to be repeated — not a 1:1 wrapper around a single resource
- [ ] Root module layout choice (folder-per-env vs. shared-module+tfvars) matches how much
      environments are allowed to structurally diverge
- [ ] Every Git-sourced module pins `?ref=<tag>`; every registry module pins a `version` constraint
- [ ] Iteration uses `for_each` (keyed) by default; `count` only for genuinely anonymous,
      append-only collections
- [ ] Nested repeating blocks inside a single resource use `dynamic`, not hand-copied blocks
- [ ] Multi-region/multi-account root modules alias each non-default provider context explicitly
- [ ] Implicit dependencies (no attribute reference, but a real ordering requirement) are
      declared with `depends_on`
- [ ] Stateful/foundational resources carry `lifecycle { prevent_destroy = true }`
