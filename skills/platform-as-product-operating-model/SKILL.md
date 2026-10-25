---
name: platform-as-product-operating-model
description: Establish the organizational operating model for a platform engineering team — treating the platform as a product with no deadline instead of a project with a sunset, defining team purpose against Conway's Law, mapping the existing SDLC/artifact life cycle before building anything, pitching platform capabilities with a value proposition, avoiding the "ivory tower" trap, and fostering a culture of continuous change. Use this when the user asks "how do we set up a platform engineering team," "should this be a project or a product," "how do we get buy-in for a platform initiative," "why isn't anyone using the platform we built," "what's our platform team's purpose/charter," or "how do we decide if we even need a platform." Distinct from idp-architecture-design (the technical architecture) and platform-engineering-success-metrics (the KPIs to track) — this skill is about the team, mandate, and process work that has to happen alongside the engineering.
---

# Platform-as-a-Product Operating Model

A platform is an organizational commitment before it's a technical one. Most failed platform
initiatives fail here — not in the architecture — because they were funded, staffed, and governed like
a project with an end date, on something that structurally has none.

## Project vs. product: pick the right mental model up front

| | Project | Product (platform-as-a-product) |
|---|---|---|
| Lifetime | Ends at a deadline, then maintenance-only | Never finished — continuously evolving |
| Investment pattern | Rising-sun excitement, then sundown | Sustained, ongoing |
| Funding argument | Fixed budget for fixed scope | Must justify itself continuously with real numbers |
| Risk of stalling | Low investment kills it slowly via bit-rot | Low investment kills it via user abandonment |

Running a platform as a project is close to a death sentence: it accrues security patches, feature
requests, and integration debt at open-source-ecosystem speed, but a project's incentive structure
withdraws people and budget the moment the deadline passes. If the environment your platform
runs in changes faster than your organization's project cadence — and for a Kubernetes/cloud-native
stack, it does — a platform has to be funded and staffed like a living product, indefinitely.

## Do you actually need a platform?

Don't skip this — building a platform you don't need adds an abstraction layer without a
corresponding payoff. Signs you're **not** ready:

- Only monolithic applications
- No dedicated development team of your own
- Your infra/DevOps/SysAdmin team is already overworked or siloed (a platform needs slack capacity
  to build, not just run)
- Applications are simple enough to run anywhere
- No budget for training teams on new tooling
- You mostly run commercial off-the-shelf software

Signs you **are** ready:

- Multiple infrastructure environments or a multi-cloud strategy
- Real requirements for security, compliance, or deep observability into environments
- A development team continuously overloaded with non-value work
- A curious infra/DevOps team already taking first unguided steps toward platform-like tooling
- A microservice architecture that needs cross-component orchestration
- A stated organizational goal to optimize cost, transparency, quality, or security across IT

Require *multiple* of these signals before committing — "someone heard about it at a conference" is
not a foundation for the investment a platform-as-a-product requires.

## Defining team purpose against Conway's Law

> "Organizations which design systems are constrained to produce designs which are copies of the
> communication structures of these organizations." — Melvin Conway, 1967

A platform's entire point is often to disrupt the organizational status quo — harmonizing specialist
teams' capabilities into one experience instead of copying existing silo boundaries into the platform's
architecture. Expect this to be political: gaining users and influence inside an org is inseparable from
who has the best relationships and the most visible wins. Counter this by defining your team's purpose
and ideology *before* you're forced into the shape your org chart would otherwise impose on you — a
written charter is what lets you hold a line when organizational pressure pushes you toward copying
existing silos instead of unifying them.

Use a structured purpose exercise (e.g., a Platform Engineering Purpose Canvas) to write down, and
revisit, what the platform is *for* — this is what keeps direction stable through reorgs and political
churn, and what stops "a nice platform with fantastic features" from being mistaken for a strategy.

## Map the existing SDLC before you build anything

Before proposing a platform capability, understand the *actual* end-to-end artifact life cycle in your
organization — not the version any single engineer believes is true. Nobody, including long-tenured
engineers, reliably knows the full path from idea to production in a large org; everyone lives in their
own bubble.

Two ways to learn it:

1. **Forensic reconstruction** — pick a real recent feature and trace every ticket, commit, pipeline
   run, test report, and incident report end to end.
2. **Live experiment** — have a team ship a small, low-risk feature (e.g., behind a feature flag) and
   observe every step, tool, and handoff it goes through in real time.

Capture, per step: who was involved, which tools were used (including inconsistent tooling across
environments), which tasks were manual vs. automated, cross-team dependencies, and time spent per
step and waiting between steps. This produces a visualization of the artifact life cycle you can point
to in every subsequent conversation about where the platform should intervene — and it routinely
surfaces requirements you'd otherwise miss (SSO integration, supply-chain security gates, audit
logging, SLA commitments) that every new platform capability must also satisfy.

**The artifact life cycle doesn't end at initial delivery.** It includes ongoing operations, releases,
maintenance, and eventual replacement/retirement — scope your platform's ambition accordingly, not
just as an onboarding accelerator.

## Instrument life cycle events, don't just observe them once

Once you understand the life cycle, make it observable continuously: log every phase transition
(requirement accepted → implementation started → PR opened → artifact built → security scan
finished → test complete → deployed → problem detected → resolved → retired) as a structured event
with metadata (timestamp, repo, pipeline, artifact, status). The CNCF's CDEvents spec is a reasonable
starting vocabulary rather than inventing your own from scratch.

This instrumentation directly produces DORA-style metrics (lead time, deployment frequency, change
failure rate, time to restore) as a side effect — see `platform-engineering-success-metrics` for how to
use them — and gives you compliance evidence (did this artifact skip a required security scan?) and
tool-agnostic interoperability (Jenkins and GitHub Actions emitting the same event shape).

## Pitching a platform capability: the value-proposition template

Don't propose a technical solution in isolation — pitch it the way a product manager would, backed by
the SDLC data you gathered:

```
Proposal: <what you're building>
Value proposition: <e.g., improve DORA metrics, reduce cognitive load, scale best practices>
KPIs and impacted teams:
  - Improve <specific metric> by <%> by <mechanism>, eliminating <specific manual toil> for <team>.
  - ...
High-level user journeys and collaboration:
  As a <role>, I <action> ... (one line per team whose workflow changes)
```

Anchoring the pitch in hard, measured numbers (not "this will be great") is what gets sustained
executive sponsorship for something that, by design, never has a completion date.

## Avoid the ivory tower: be customer zero

The platform team must run its own platform the same way it asks users to — "drink your own
champagne." If the platform team ships golden paths and self-service capabilities it doesn't itself use
to deploy and operate the platform, you get the predictable outcome: "We spent months building this
platform. Devs hate it!" Feel the same operational pain your users feel; it's the single best forcing
function for building something people actually want to use.

## Fostering a culture of continuous change

A platform team's job includes leading cultural change, not just shipping features:

- **Create safe environments for experimentation** — sandbox environments and pilot projects for
  users, and equivalent space for your own platform engineers to evaluate new technology.
- **Celebrate documented failure alongside success.** Treat a tested-and-rejected technology
  evaluation as a valid outcome worth recording, not a wasted cycle — it prevents the next person
  from re-running the same experiment.
- **Evolve iteratively, not just in big-bang releases.** Feature toggles and canary rollout to a subset
  of users let you validate changes safely; continuously refactor, revisit previously "unsolvable"
  technical debt, and update dependencies rather than batching everything into rare large migrations.
- **Have an explicit deprecation strategy.** Communicate impact and timelines clearly when retiring
  a capability — this is also how you learn how tightly coupled users have become to something you
  assumed was modular.
- **Consider sustainability as a design input**, not an afterthought: reduce (do you need five-nines
  globally distributed infrastructure for this?), reuse compute rather than retiring it early, rightsize
  instead of over-provisioning headroom, and factor a vendor's sustainability commitments into
  platform/vendor selection. Cost reduction and carbon reduction are usually the same lever.

## Common pitfalls

- **Funding a platform like a project.** Guarantees a sunset the moment budget pressure appears,
  regardless of how much value the platform is producing.
- **Skipping the "do we need this" checklist** because someone senior asked for a platform after a
  conference talk — not because multiple real signals point to the need.
- **Designing the platform to match the org chart** instead of a deliberately chosen purpose —
  reproducing Conway's Law instead of using the platform to counteract it.
- **Building from assumption instead of a mapped SDLC.** Guarantees you'll miss cross-team
  requirements (SSO, audit, SLAs) that surface only after launch, as complaints.
- **Ivory-tower delivery** — mandating best practices from the top without the platform team itself
  living by them day to day.
- **No pitch, or a pitch without numbers.** "This will be great for developers" doesn't survive a
  budget cycle; a value proposition with named KPIs and impacted teams does.

## Checklist

- [ ] The platform is funded and staffed as an ongoing product, not a time-boxed project
- [ ] At least three "ready for a platform" signals are documented before committing to build one
- [ ] A written team purpose/charter exists and is used to push back on org-chart-shaped scope creep
- [ ] The current end-to-end artifact life cycle has been mapped (forensically or via live experiment),
      not assumed
- [ ] Life cycle events are instrumented consistently across tools, feeding lead-time/failure-rate data
- [ ] Every proposed platform capability has a written value proposition with named KPIs and
      impacted-team user journeys
- [ ] The platform team consumes its own platform (customer zero) before mandating it for others
- [ ] A deprecation strategy and sandboxed experimentation space exist for both users and the
      platform team itself
