---
name: platform-security-and-compliance-framework
description: Scope and reason about platform-level security and compliance — reconciling "security to the left" and Zero Trust with developer self-service, running threat-modeling sessions with the Threat Modeling Manifesto's four questions, using OWASP Top 10 and the Kubernetes-specific OWASP Top 10 as a scoping baseline, mapping which compliance standard (PCI DSS, HIPAA, GDPR, FedRAMP, DPDPA, DGA) applies to your platform, classifying and protecting data across its three states (at rest, in motion, in use), and generating/using SBOMs for supply-chain risk. Use this when the user asks "how secure does our platform need to be," "how do we balance security with developer self-service," "what compliance framework applies to us," "how do we run a threat model for our platform," "what's our security scope for an IDP," or "how do we track our software supply chain." Distinct from k8s-enterprise-secrets-management, k8s-enterprise-policy-admission-control, and k8s-enterprise-multi-tenancy-rbac (which cover the Kubernetes-level implementation mechanics) — this skill is the scoping, threat-modeling, and compliance-framework layer that decides what those mechanisms need to enforce.
---

# Platform Security and Compliance Framework

Security and developer self-service pull in opposite directions by default: the more the platform
enforces on developers' behalf, the more it feels like a barrier that kills adoption. This skill is about
resolving that tension deliberately, not about the specific Kubernetes controls that implement the
decision — see the `k8s-enterprise-*` security skills for that layer.

## Security to the left, reconciled with Zero Trust

"Shift left" means testing security early and often across the whole delivery flow (dev team → source
control → CI/CD → application), not just at the application/runtime layer where it's already too late
to prevent, only to detect. But full "shift left" ownership by the platform team runs counter to
self-service — if the platform owns the entire security story end to end, it necessarily restricts what
developers can do unilaterally.

The resolution is **Zero Trust applied with a deliberately scoped platform responsibility**: the
platform doesn't own the application's security posture, but it provides secure scaffolding and
fail-safes so a developer *doesn't have to* be a security expert to stay compliant:

- Supply a pre-hardened, trusted base image (e.g., `python:slim` from a vetted registry) as the
  default rather than requiring each team to source and harden their own.
- Push checks as far left as is practical (CI-time Dockerfile/image-source scanning) — but recognize
  that writing this kind of check yourself is often out of scope for a platform team; a policy engine
  is usually the more maintainable investment.
- Use **admission-webhook policy as the Zero Trust fail-safe**, not the primary control — reject Pod
  definitions using untrusted image sources at the cluster boundary, accepting that this is "less far
  left than ideal" but catches what upstream process failed to catch. It's reasonable to relax this for
  prototyping environments and enforce it strictly only for production.
- Require signed commits where the org's security posture demands non-repudiation — but recognize
  this is often outside a platform team's actual authority (it's usually a source-control-org policy, not
  a platform-enforced one) and coordinate with the security team rather than trying to own it.

**"Security is good; security theater is not."** The skill here is knowing when you've done enough —
there's no universal formula. An environment for space-shuttle or nuclear-reactor-adjacent work
justifies an air-gapped, no-public-internet posture; an internal tool for low-risk static content does
not, and over-applying maximum-security patterns elsewhere just adds cognitive load and adoption
friction without a matching risk reduction.

## Scoping security: narrow first, then expand

1. Determine the **minimum level the platform must enforce**.
2. Determine the **minimum level the platform must support** (for teams that need more than the
   floor).
3. Resist the urge to design for the maximum threat model up front — it inflates scope and often
   obscures the actual near-term risks ("can't see the forest for the trees").

Use published baselines to scope rather than starting from a blank page:

- **OWASP Top 10** (application-level): Broken Access Control, Cryptographic Failures, Injection,
  Insecure Design, Security Misconfiguration, Vulnerable/Outdated Components, Identification and
  Authentication Failures, Software and Data Integrity Failures, Security Logging and Monitoring
  Failures, Server-Side Request Forgery.
- **OWASP Kubernetes Top 10** (platform-level): Insecure Workload Configurations, Supply Chain
  Vulnerabilities, Overly Permissive RBAC, Lack of Centralized Policy Enforcement, Inadequate
  Logging/Monitoring, Broken Authentication Mechanisms, Missing Network Segmentation, Secrets
  Management Failures, Misconfigured Cluster Components, Outdated/Vulnerable Kubernetes
  Components.

Treat both lists as a **bare-minimum starting scope**, not a comprehensive security program — they're
a checklist to make sure you haven't missed an obvious category, not a substitute for threat modeling
your specific system.

## Threat modeling with the Threat Modeling Manifesto's four questions

Run this as a collaborative session with platform stakeholders (not a solo document exercise):

1. **What are we working on?** (e.g., "how users authenticate to the IDP")
2. **What can go wrong?** (walk each relevant OWASP/K8s-Top-10 category against the answer to #1 —
   expect a many-to-one fan-out here)
3. **What are we going to do about it?**
4. **Did we do a good enough job?** — if the honest answer isn't a clear yes, loop back to question 1
   for the gap that remains.

Security to the left extends to the *people*, not just tooling — a threat model should also cover
sociotechnical risk (credential-stealing social engineering, physical device security), since a perfectly
configured admission controller doesn't stop a stolen laptop.

## Mapping the applicable compliance standard(s)

Compliance and security are related but distinct — a system can be secure without being compliant,
and vice versa (e.g., PCI DSS requires physical inspection of hardware, which is orthogonal to
software security posture). Determine applicable frameworks from your industry and customers, not
by defaulting to the most famous one:

| Standard | Locale | Applies to | Notes |
|---|---|---|---|
| PCI DSS | International | Anyone processing/transmitting/storing credit card data | Created by card brands, not government; levels based on transaction volume |
| HIPAA | US | Healthcare data | Set in the 1990s, continuously reinterpreted for modern tech |
| FedRAMP | US | Selling software/services to the US federal government | Moderate/High levels; distinct from state-government requirements |
| GDPR | EU | Personal data handling | Foundational global-influence privacy regulation |
| DGA (Data Governance Act) | EU | Public-sector data use/sharing, data altruism | Fills gaps GDPR leaves in public-sector contexts |
| DPDPA | India | Personal data processing | 2023 law, comparable to but distinct from GDPR |

This is not exhaustive — always check for industry-specific overlays (e.g., a hospital needs HIPAA
*and* whatever generic frameworks apply to its cloud provider relationship).

## Asset protection: data across three states

Your most valuable platform asset is data, and it needs protection differently depending on its state:

**At rest** — classify data first (type, sensitivity, whether physical/regulatory isolation is required),
encrypt at every layer, use salts/hashes deliberately (they hurt human-readability and can inflate
memory usage — don't apply them reflexively), gate access behind a formal review process, and use
redundant backups sized to the cost of losing the data (not one uniform policy for everything).

**Retention policy questions worth asking explicitly, per data category, not once for the whole
system:**
- Do we need it at all? ("Bloat occurs when people assume data is necessary.")
- What's the business case — does it make money, save money, or reduce risk exposure by existing?
- What's the operational use case (troubleshooting, audit trail)?
- What risk does *keeping* it expose us to?

**In motion** — encrypt in transit, minimize how long data sits in memory/caches, avoid widely-open
or privileged ports, log only what's needed about the transaction (never the transacted data itself —
see log sanitization), sanitize inputs against injection, and apply standard MITM-resistant network
practices.

**In use** — apply the same sanitization/transformation discipline as data in motion; this is data being
read, cached, or freshly entering the system (new user registration, new log line).

**Environment isolation** is a cross-cutting control: keep staging/dev/production fully separate,
including separate databases (not shared tables with different access rules) and ideally separate
clusters — a single cluster with network-policy-based separation still shares a single point of failure
(API server, etcd, audit log) across "isolated" tenants.

## Software supply chain: SBOMs

An SBOM (Software Bill of Materials) is a ledger of everything your software depends on and where
those dependencies originate — generated at build time in CI (tools: Syft, and newer vendor-specific
options), in the SPDX format for portability. Treat it as more than a checkbox:

- Cross-reference a newly announced CVE against your SBOMs to immediately know which
  systems/artifacts are affected — this is the actual operational payoff, not just "we have one."
- GitHub exposes SBOMs for any repo via its API in SPDX format without extra tooling.
- SBOM/scanner tooling is still immature — using more than one tool in concert catches gaps any
  single one misses; don't assume one tool's output is complete.

## Common pitfalls

- **The platform trying to own the application's entire security posture.** Kills self-service and
  isn't actually achievable — scope what the platform enforces vs. supports instead.
- **Security theater**: applying maximum-security patterns (air-gapping, exhaustive manual review)
  to workloads that don't warrant it, purely because it "feels safer," at the cost of adoption.
  Threat model instead of defaulting to maximum.
  - **Treating OWASP Top 10 / K8s Top 10 as a finished checklist** rather than a floor — teams that
  stop there haven't actually threat-modeled their specific system.
- **Confusing security with compliance**, or assuming one implies the other — verify both
  independently against your actual regulatory exposure.
- **Not classifying data before deciding on retention/encryption/backup policy** — applying one
  uniform policy to all data either under-protects sensitive data or wastes resources over-protecting
  everything else.
- **A single Kubernetes cluster presented as equivalent to separate environments.** Namespace/RBAC
  separation is real isolation for many purposes, but the cluster's control plane remains a shared
  single point of failure — call that out explicitly rather than letting stakeholders assume full
  isolation.

## Checklist

- [ ] Security scope is written down as "what the platform enforces" vs. "what it supports" — not left
      implicit
- [ ] OWASP Top 10 and OWASP Kubernetes Top 10 have been walked as a scoping floor, and a real
      threat-modeling session (using the four Manifesto questions) has gone beyond them
- [ ] The applicable compliance standard(s) are identified from actual industry/customer exposure,
      not assumed from the most commonly cited framework
- [ ] Data is classified before retention, encryption, and backup policy are decided — not after
- [ ] Production is isolated from non-production with separate databases and, ideally, separate
      clusters — and any shared-cluster compromise is documented as a residual risk, not ignored
- [ ] An SBOM is generated at build time and actually used to answer "are we affected" during CVE
      response, not just stored
