---
name: platform-engineering-success-metrics
description: Measure and report platform engineering success using DORA metrics (with elite/high/medium/low benchmark bands), the SPACE framework, the DevEx framework (flow state, cognitive load, feedback loops), adoption-curve KPIs, and cost-based performance metrics (cost per change, cost per project). Use this when the user asks "how do I measure whether our platform is working," "what KPIs should a platform team track," "how do I prove our IDP's ROI," "why do our adoption numbers look flat," "what's the difference between DORA, SPACE, and DevEx," or wants to build a platform adoption dashboard. Distinct from devops-ai-governance-playbook's ROI section (which covers AI-specific tooling ROI) — this skill is about measuring a platform/IDP's overall adoption and developer-experience impact, AI or not.
---

# Platform Engineering Success Metrics

A platform with no evidence of impact loses funding regardless of how well-built it is. This skill
covers the frameworks for proving that impact and reading the numbers correctly — including the
traps that make good platforms look bad and bad platforms look fine.

## Step 1: know what you're already measuring

Before inventing new KPIs, inventory what you already collect (metrics, logs, traces, support tickets,
onboarding requests, PR counts) — most useful platform KPIs are combinations of numbers you
already have, read in the right context, not new instrumentation.

**The critical caveat: raw numbers without maturity context are meaningless or actively misleading.**
A drop in support tickets could mean great documentation and self-service — or that nobody uses the
platform anymore. Always pair a metric with *where the platform is in its life cycle*.

## The adoption curve has a predictable shape

Two numbers in particular follow a hill shape, not a straight line, and reading them as straight lines
produces the wrong conclusion:

- **Onboarding requests**: slow at first (early adopters are cautious), fast during ramp-up, then
  taper as most eligible applications have already migrated.
- **Support requests**: grow with user count initially, but should *decrease toward a floor* over time
  as self-service quality improves — flat or rising support volume alongside flat or falling user growth
  is the actual bad-sign combination, not either number alone.

Segment your dashboard by life-cycle phase (initial adoption / ramp-up / optimization) rather than
plotting one continuous line — cost-per-change and support requests typically rise during initial
adoption, fall steeply during ramp-up, and flatten during optimization; deployment time drops
throughout and also flattens late. A platform that looks "flat" in its optimization phase is succeeding,
not stagnating — flat costs with rising satisfaction and shipped features *is* the win condition for a
mature platform.

## DORA metrics

The four DORA (DevOps Research and Assessment) metrics measure delivery performance and are
the most widely adopted starting point — most orgs already have the raw data to compute them.

| Metric | Definition | How to source it |
|---|---|---|
| Deployment frequency | How often you successfully release to production (or "release," if your org has intermediate stages) | API against your SCM/release system |
| Lead time for changes | Time from commit to production | Track commit → release mapping (harder than it sounds) |
| Change failure rate | % of deployments causing a production failure | Requires tagging failures back to the release that caused them |
| Time to restore service | How long it takes to recover from a production failure | Incident management system: time-to-open to time-to-resolved |

Benchmark bands from DORA's own research (2023 report) — use these to say "we're at X level," not
just "our number is Y":

| | Elite | High | Medium | Low |
|---|---|---|---|---|
| Deployment frequency | On-demand (multiple/day) | Once/day–once/week | Once/week–once/month | Once/week–once/month |
| Change lead time | <1 day | 1 day–1 week | 1 week–1 month | 1 week–1 month |
| Change failure rate | 5% | 10% | 15% | 64% |
| Failed-deployment recovery time | <1 hour | <1 day | 1 day–1 week | 1 month–6 months |

A CNCF project like Keptn can auto-generate several of these (deployment duration, success/failure,
dimensioned by app/environment/version) from namespace annotations rather than requiring
hand-built pipeline instrumentation — worth adopting before building a bespoke collector.

## SPACE framework: beyond delivery-only metrics

DORA measures delivery performance; SPACE (developed by GitHub, Microsoft, and University of
Victoria) measures developer/team productivity more broadly across five dimensions:

| Dimension | What it captures | Example signals |
|---|---|---|
| **Satisfaction & well-being** | Fulfillment, happiness, mental health | Employee satisfaction score (ESS), eNPS, simple surveys |
| **Performance** | Output-oriented | Lead time, incident-to-recovery time, releases, test coverage — DORA metrics fit here |
| **Activity** | Raw activity counts | Commits, PRs, reviews, deployment frequency, story points |
| **Communication & collaboration** | Team alignment and interaction quality | Merge/review times, comment volume, meeting-quality feedback |
| **Efficiency & flow** | How smoothly work moves | DORA metrics, number of hand-offs, developer focus-hours, velocity |

**Compliance warning**: activity metrics (commits, PRs, lines of code) can look like individual
performance surveillance and are restricted or prohibited for employee evaluation in some
jurisdictions. Loop in your data-protection/HR function before rolling out SPACE metrics at the
individual level — use it at team or org level by default.

## DevEx framework: the developer-experience gap DORA and SPACE both miss

Three dimensions, each measurable through both hard metrics and subjective feedback — use both,
since neither alone is a full picture:

| Dimension | Metrics | Subjective feedback questions |
|---|---|---|
| **Flow state** | Reopened commits, change requests, commits/PR | How often do they feel interrupted? Continuously stressed? Level of decision autonomy? |
| **Cognitive load** | Reopened bugs, debugging time, dependency count per component, time-to-solve | Time coding vs. other tasks; technical scope; how often they context-switch |
| **Feedback loop** | Cycle time, deployment frequency, MTTR, bug count, test automation level | Code review effectiveness; team communication quality |

Report these frameworks as a stack, not a menu: DORA (delivery) → SPACE (team productivity,
extends DORA) → DevEx (developer flow/cognitive load, fills the gap both leave). Architects should
understand what each metric actually means before using it to justify a platform change — a metric
you can't explain the direction of causation for is not evidence.

## Cost-based performance metrics

Cost numbers are the most durable way to talk to stakeholders and executives about platform
efficiency — unglamorous, but hard to argue with.

- **Cost per change**: total platform cost ÷ number of changes shipped in a period. Expect this to
  start high with few changes and fall as the platform matures and change volume grows — e.g., going
  from $500/change ($180K/year at 30 changes/month) to $80/change ($28.8K/year at the same
  volume) is the expected trajectory, not a one-time win.
- **Cost per project/product**: use *total shared costs* (team + platform costs, split evenly across
  projects) for simple, low-effort reporting on platform performance. Avoid investing heavily in
  causation-based cost allocation (real infra consumption per project) unless you specifically need
  it — it's expensive to calculate and mostly answers a different question (onboarding efficiency, not
  platform performance).
- **User/product overhead cost**: shared and direct costs minus actual consumed resources. Useful as
  a north-star efficiency number, but expect tension — too low and it looks unconvincing as "real
  value," too high and it's hard to defend internally. Aim for a defensible middle, not a minimum.

## Common pitfalls

- **Reading adoption metrics as straight lines** instead of a life-cycle curve — declaring a mature,
  flat-cost platform "stagnant" when flat costs with rising features/satisfaction is success.
- **Using SPACE activity metrics for individual performance review** without checking local
  employment-law/data-protection constraints first.
- **Reporting a single metric in isolation** (e.g., support tickets down) without the adoption-context
  metric that disambiguates "self-service is working" from "nobody's using this anymore."
- **Skipping DORA because "it doesn't fit our pipeline exactly."** Most orgs redefine "production" as
  "release" or "productive release" to fit multi-stage delivery — that's expected, not a disqualifier.
- **Building a bespoke DORA collector from scratch** before checking whether an existing CNCF
  project (e.g., Keptn) or your CI/CD platform already emits most of what you need.

## Checklist

- [ ] Every reported KPI is paired with the platform's current life-cycle phase, not shown as a bare
      number
- [ ] DORA metrics are tracked with an explicit maturity-band comparison (elite/high/medium/low),
      not just an internal trend line
- [ ] SPACE activity/communication metrics have been cleared with data-protection/HR before any
      individual-level use
- [ ] At least one DevEx dimension (flow, cognitive load, feedback loop) is measured with both a hard
      metric and a subjective survey question
- [ ] Cost per change (or an equivalent cost-based metric) is tracked over time and used in stakeholder
      reporting, not just technical dashboards
