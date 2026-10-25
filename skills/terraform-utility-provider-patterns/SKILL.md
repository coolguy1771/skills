---
name: terraform-utility-provider-patterns
description: Use HashiCorp's utility providers — random, time, null, tls, and external — to solve problems that have nothing to do with any cloud API, such as generating unique resource-name suffixes, creating secrets/passwords Terraform should manage, sequencing dependencies between resources with no native relationship, generating self-signed TLS certificates, or shelling out to an external program as a pseudo data source. Use this when the user asks "how do I generate a random suffix/password in Terraform," "how do I make Terraform wait before creating this resource," "how do I create a self-signed cert in Terraform," "how do I run an external script and use its output as a Terraform value," or is fighting a naming-collision, ordering, or arbitrary-compute problem that a cloud resource block alone can't solve. Distinct from terraform-iac-workflow-design, which only covers cloud-facing providers.
---

# Terraform Utility Provider Patterns

Not every problem in a Terraform configuration is "provision this cloud resource." Naming
collisions, generated secrets, artificial ordering, and one-off computation are common enough
that HashiCorp ships dedicated providers for them. Reaching for these instead of shell scripts
glued on with `local-exec` keeps the logic declarative, visible in `plan`, and tracked in state
like everything else.

## `random` — naming, secrets, and selection

**Unique naming** — avoid collisions on globally-unique cloud resource names (storage accounts,
S3 buckets) without hardcoding a suffix:

```hcl
resource "random_string" "name_suffix" {
  length  = 6
  special = false
  upper   = false
}

resource "azurerm_resource_group" "example" {
  name = "rg-foobar-${random_string.name_suffix.result}"
}
```

**Secrets Terraform should own** — `random_password` marks its result sensitive automatically,
so it won't print in `plan`/`apply` output:

```hcl
resource "random_password" "database" {
  length  = 24
  special = true
}
```

**Long unique identifiers** — `random_uuid` for resources that expect a GUID-shaped value.

**Human-friendly random names** — `random_pet` generates `adjective-noun` names (e.g.
`clever-walrus`) — handy for ephemeral/dev resources where readability beats uniqueness
guarantees.

**Random selection from a fixed list** — `random_integer` picks a bounded number, useful for
picking e.g. an availability zone index:

```hcl
resource "random_integer" "az" {
  min = 0
  max = length(local.azs) - 1
}

locals {
  selected_az = local.azs[random_integer.az.result]
}
```

For selecting *multiple, non-repeating* items from a list (e.g., two AZs for a load balancer),
`random_shuffle` is far less cumbersome than chaining multiple `random_integer` resources and
manually ensuring they don't collide:

```hcl
resource "random_shuffle" "azs" {
  input        = local.azs
  result_count = 2
}

resource "aws_elb" "example" {
  availability_zones = random_shuffle.azs.result
  # ...
}
```

**Stability warning**: every `random_*` resource's value is generated once and then held in
state — it does not change on subsequent applies unless its inputs change or the resource is
tainted/replaced. If a downstream resource depends on a `random_shuffle`/`random_integer`
result and you need to force a new value, you'll need a targeted `terraform apply -replace=...`
rather than expecting it to reroll automatically.

## `time` — timestamps and rotation without drift

**Freezing a timestamp at creation** — `time_static` captures "now" once and holds it in state,
so re-applying doesn't perturb a value meant to represent creation time:

```hcl
resource "time_static" "created" {}
```

Or pin it to a specific known instant with the `rfc3339` attribute rather than "now":

```hcl
resource "time_static" "product_launch" {
  rfc3339 = "2026-05-04T00:00:00Z"
}
```

**Periodic rotation** — `time_rotating` is the tool for "this needs to be regenerated every N
days" (certificate rotation, credential rotation cadence): once `rotation_days` elapses, the
resource is marked for replacement on the next plan, which can in turn trigger replacement of
anything depending on it via `replace_triggered_by`.

```hcl
resource "time_rotating" "certificate" {
  rotation_days = 90
}
```

**Artificial delay** — `time_sleep` inserts a wait between two resources that have no other way
to express "give the platform a moment before the next step," paired with `null_resource` where
you need an anchor with no inherent cloud identity to depend on:

```hcl
resource "null_resource" "previous" {}

resource "time_sleep" "wait_30_seconds" {
  depends_on      = [null_resource.previous]
  create_duration = "30s"
}

resource "null_resource" "next" {
  depends_on = [time_sleep.wait_30_seconds]
}
```

## `null_resource` — an anchor for provisioners and arbitrary triggers

`null_resource` has no cloud identity of its own — its entire purpose is to host `provisioner`
blocks (`local-exec`/`remote-exec`) or to act as a dependency anchor, replaced whenever its
`triggers` map changes:

```hcl
resource "null_resource" "deploy_hook" {
  triggers = {
    app_version = var.app_version
  }

  provisioner "local-exec" {
    command = "./scripts/notify-deploy.sh ${var.app_version}"
  }
}
```

Any string change in `triggers` forces replacement, which re-runs the attached provisioners —
this is the mechanism for "run this script again only when X changes," not a cron-like repeat.

## `tls` — certificates without leaving Terraform

Generate a private key, a certificate signing request, and (optionally) a self-signed or
locally-signed certificate, entirely in HCL — useful for dev/test environments or bootstrapping
mTLS material that a proper CA will later replace:

```hcl
resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

output "private_key" {
  value     = tls_private_key.ssh_key.private_key_openssh
  sensitive = true
}
```

Building a full chain — key → CSR → locally-signed cert:

```hcl
resource "tls_private_key" "ca" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "tls_private_key" "leaf" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "tls_cert_request" "leaf" {
  private_key_pem = tls_private_key.leaf.private_key_pem
  subject {
    common_name = "internal.example.com"
  }
}

resource "tls_self_signed_cert" "ca" {
  private_key_pem      = tls_private_key.ca.private_key_pem
  is_ca_certificate    = true
  validity_period_hours = 8760
  allowed_uses          = ["cert_signing", "crl_signing"]
  subject {
    common_name = "Internal Root CA"
  }
}

resource "tls_locally_signed_cert" "leaf" {
  cert_request_pem   = tls_cert_request.leaf.cert_request_pem
  ca_private_key_pem = tls_private_key.ca.private_key_pem
  ca_cert_pem        = tls_self_signed_cert.ca.cert_pem
  validity_period_hours = 720
  allowed_uses = ["key_encipherment", "digital_signature", "server_auth"]
}
```

Never use `tls`-generated self-signed material for a production public-facing certificate — use
it for internal/dev trust chains, mTLS bootstrapping, or test fixtures, and pair it with
`time_rotating` if it needs periodic renewal.

## `external` data source — escape hatch for one-off computation

When a value genuinely can't be produced by any Terraform function or resource, the `external`
data source shells out to a program and reads back JSON:

```hcl
data "external" "example" {
  program = ["python3", "${path.module}/scripts/compute.py"]

  query = {
    input_value = var.some_input
  }
}

output "computed" {
  value = data.external.example.result.output_value
}
```

The external program must read a JSON object from stdin and write a flat JSON object of
string-to-string values to stdout — no nested objects, no other types. Treat this as a last
resort: it's invisible to Terraform's dependency graph beyond its declared inputs, isn't
portable across environments without the same interpreter/script available, and produces no
meaningful `plan` diff — reach for a real provider resource or a `for` expression first, and
only fall back to `external` when neither can express what you need.

## Common pitfalls

- **Expecting `random_*` values to reroll on every apply** — they're stable once created; force
  a new value with `-replace` or by changing an input like `keepers`, not by re-running `apply`.
- **Using `tls` self-signed certs in production** — fine for dev/test/internal trust chains;
  never for a public-facing service that needs real CA trust.
- **Reaching for `external` before checking for a built-in function or provider resource** — it
  bypasses plan visibility and portability that native HCL constructs give you for free.
- **Forgetting `sensitive = true`** on outputs exposing `random_password`/`tls_private_key`
  values — Terraform won't guess that a random string or PEM block should be redacted.
- **Using `null_resource` + `local-exec` as a general substitute for a missing provider
  resource** — it works, but it's untracked, non-idempotent by default, and invisible to `plan`;
  only reach for it when there's truly no resource-based alternative.

## Checklist

- [ ] Globally-unique resource names use a `random_string`/`random_id` suffix instead of a
      manually chosen "hopefully unique" string
- [ ] Generated secrets use `random_password` (or another provider's native secret-generation
      resource) with `sensitive = true` on any output that exposes them
- [ ] Artificial ordering/delay uses `null_resource` + `time_sleep` with explicit `depends_on`,
      not a shell `sleep` buried in a provisioner script
- [ ] Certificates generated via `tls` are scoped to dev/test/internal use, paired with
      `time_rotating` if periodic renewal is required
- [ ] `external` data source use is limited to genuinely irreducible computation, with its
      inputs/outputs kept to flat string maps
- [ ] Sensitive utility-provider outputs (`tls_private_key`, `random_password`) are marked
      `sensitive = true` everywhere they're surfaced
