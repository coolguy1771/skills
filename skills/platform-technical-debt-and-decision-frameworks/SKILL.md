---
name: platform-technical-debt-and-decision-frameworks
description: Make and document deliberate platform-engineering trade-off decisions — a technical-debt assessment questionnaire for consciously accepting debt, a build-vs-buy decision framework, guidance on avoiding over-engineering, a practical refactor-vs-rewrite decision guide with real-world examples (Helm 2→3, cgroups v1→v2), and how to write architectural decision records (ADRs) that survive team turnover. Use this when the user asks "should we take on this technical debt," "should we build this ourselves or buy/adopt an existing tool," "is it time to rewrite this component or just refactor it," "how do we document why we made this architectural decision," or "are we over-engineering this." Distinct from terraform-iac-workflow-design and terraform-advanced-module-composition (which cover IaC-specific tooling decisions) — this skill is the general decision-and-documentation discipline for any platform component choice.
---

# Platform Technical Debt and Decision Frameworks

No platform ships debt-free, and no product decision is permanent. The skill isn't avoiding technical
debt — it's taking it on *consciously*, documenting *why*, and having a repeatable way to decide when
to refactor versus rewrite versus replace.

## Taking on technical debt consciously

Before accepting a shortcoming in a solution (rather than treating "solve it perfectly" and "ship it
now" as mutually exclusive), run this assessment as a team — technical decisions made unilaterally
tend to be revisited unilaterally, too, usually at a worse time:

1. What problems are you solving today?
2. What problems do you *not* have (yet)? — this is your explicit non-goals list.
3. What will this cost in time to build or adopt?
4. What will this cost to run?
5. What's necessary to maintain it?
6. What's the expected return?
7. Can the team sustain it as-is? If not, can they upskill, and is there time? If it's OSS, how strong is
   the community, and how does the team join it?
8. What will it take for new team members to get up to speed on it?
9. When will you know it's time to scale it?
10. How will you scale it?
11. What are the deprecation criteria for this solution?

Treat the answers like an accounting ledger you can review later — technical debt isn't usually "the
wrong decision in retrospect," it's the natural cost of a decision that made sense at the time and has
since been outgrown. Revisit this ledger on a cadence, not just when something breaks.

## Avoiding over-engineering

Over-engineering is building a maximally robust, fully-featured system to solve problems that don't
exist yet. Questions 1 and 2 above are your primary defense — explicitly defining goals *and*
non-goals keeps scope sane. Concretely, for any component with unbounded-sounding requirements
(a common trap: observability/data-retention systems), ask targeted questions instead of designing for
infinite scale by default:

- How much of this data is actually useful?
- For how long is it worth keeping?
- How long are we legally/contractually required to keep it?
- Who actually needs access to it?

If there's no use case for retention beyond a defined window, build a system that handles a bounded
amount of data and purges on schedule — not an infinitely scalable one "just in case." Data
aggregation (rolling seconds-granularity telemetry up into days/weeks/months) is often the middle
ground when full purging feels too aggressive.

This isn't an argument against ever planning ahead — a composable, modular architecture that allows
swapping a component later is different from building unused capacity into every component now.
The general rule: **solve today's problems and roadmap tomorrow's**, don't build tomorrow's solution
today.

## Build vs. buy: a decision framework

"Buy" includes adopting a free/open-source tool, not just a paid purchase — the real axis is "build it
ourselves" vs. "adopt something that already exists." This decision is more sociotechnical than
technical: engineers are naturally biased toward building because it aligns with identity and interest,
which is exactly why it needs a deliberate, group-run process rather than an individual's gut call.

Run the decision as a group discussion, explicitly surfacing:

- Does an existing open-source or commercial solution already solve this well enough?
- What's the real integration cost of adopting it vs. the real maintenance cost of building it?
- Does the team have (or can it acquire) the skill to build and sustain a custom solution?
- What's the deprecation/exit cost if the chosen path (build or buy) turns out wrong?

Revisit the same framework when deciding whether to **rewrite** an existing component — the
build-vs-buy conversation applies again: if an open-source or affordable solution exists now that
didn't when the component was first built, a rewrite might reduce to "swap in an integration" instead
of "write a bespoke replacement."

## Refactor vs. rewrite: a practical guide

**Refactor when:**
- You need to improve efficiency, security, or add/change an interface incrementally
- The codebase can reach the desired state without discarding it
- An external dependency changed in a way that requires adaptation, not replacement (see the Helm
  2→3 example below) — this is usually lower effort and higher ROI than a rewrite

**Rewrite when:**
- Continuing to use the current approach would require bolting on a disproportionately complex
  helper (e.g., adopting a full service-mesh sidecar proxy just to support one outlier service written
  in a different language may cost more than rewriting that one service)
- A licensing change removes your ability to get security updates or legally continue using the
  component as-is
- The component is genuinely unmaintained and blocking needed functionality or performance gains
  — but note this is the *rare* case; if a component is only somewhat behind, refactoring is usually
  still the better ROI

**Worked example — refactor (Helm 2 → Helm 3 CRD handling):** Helm 2's `crd-install` hook could
silently fail to register new CRDs on upgrade, forcing full chart teardown-and-reinstall when a CRD
changed — teams built custom bash-script workarounds (e.g., splitting CRDs into separately-ordered
charts, as Istio did) to compensate. Helm 3 replaced the hook with a dedicated `crds/` directory that
Helm processes before the rest of the chart, which was a small annotation-level change per chart but
let most teams delete their custom tooling entirely. Lesson: a small upstream design fix can retire a
large amount of downstream workaround debt — that's a refactor, not a rewrite, even though the
blast radius (every chart using CRDs) was wide.

**Worked example — rewrite with ripple effects (cgroups v1 → v2):** The Linux kernel's cgroups
rewrite (performance-driven) is generally superior but breaks legacy applications (older Java/Node.js
memory-querying assumptions), causing OOM crashes under the new default. Kubernetes still supports
v1 for compatibility, but treating that as a permanent option is a mistake — v1 is far more likely to be
deprecated than dual-maintained indefinitely. The actual decision facing consumers wasn't "rewrite or
not" (the kernel project already decided that) but "refactor cluster configuration to pin v1 a while
longer" vs. "modernize the legacy applications now" — and staying on v1 forgoes newer
Kubernetes workload-management features that only work with v2. Lesson: an upstream rewrite forces
your own build-vs-buy-style decision downstream, and deferring it has a growing opportunity cost,
not just a static risk.

**After a rewrite/replacement**: define and test a transition plan *before* cutting over — messy
transitions extend how long you have to run both the old and new component in parallel, at double
the operational cost. The transition plan itself becomes a template for the next migration, so
documenting it pays off beyond this one event.

## Architectural Decision Records (ADRs)

An ADR captures not just *what* was decided but *why* — critical because the "why" is what future
owners need and what documentation of the current state alone can't provide.

**Why bother:**
- Onboarding — new team members and users understand system usage and data flow faster from
  documentation than from reverse-engineering code.
- Security/compliance — audits require architecture documentation as a matter of course.
- Forward planning — you can't decide where a platform should go next without a clear record of
  where it currently is and why it got there.

**What a good ADR captures, per decision:**
- Was there a limiting factor that forced a less-than-optimal solution?
- What bar would a replacement need to clear to be worth adopting?
- What were the known risks and mitigations at decision time?
- What were the goals *and* non-goals, and why?
- What impact does this component have on others (and vice versa)?

Recording these turns institutional knowledge into a legacy the team can build on, instead of "legacy
software" nobody understands well enough to change safely. Keep ADRs alongside other required
documentation (architecture diagrams, transition plans) as a standing artifact, not a one-time
compliance exercise — the ongoing maintenance is itself a form of debt, but one with a strongly
positive return over the platform's lifetime.

## Common pitfalls

- **Taking on technical debt without running the assessment questions as a group** — individual,
  undocumented debt decisions are the ones nobody can explain (or defend) two years later.
- **Designing for infinite scale on data/observability systems** by default instead of asking the four
  targeted retention questions first.
- **Letting "engineers like to build things" drive build-vs-buy** instead of running it as a deliberate,
  biased-aware group decision.
- **Rewriting when a refactor would do** — check whether the *actual* blocker is an external
  dependency change (refactor territory) before assuming a ground-up rewrite is required.
- **Cutting over without a tested transition plan**, extending the double-running cost of old and new
  components and losing the chance to create a reusable migration template.
- **Treating ADRs as a one-time deliverable** rather than a standing practice — a platform with no
  ADR trail forces every future architect to reverse-engineer intent from code and tickets.

## Checklist

- [ ] Every consciously-accepted piece of technical debt has documented answers to the 11 assessment
      questions, not just a verbal agreement
- [ ] Unbounded-sounding requirements (especially observability/data retention) have explicit
      goals/non-goals before implementation starts
- [ ] Build-vs-buy decisions are made as a group discussion with the bias risk named explicitly, not
      resolved by whoever wants to build it most
- [ ] Before proposing a rewrite, the team has checked whether a refactor or an available
      open-source/commercial adoption already solves the problem
- [ ] Every rewrite/replacement has a written, tested transition plan before cutover
- [ ] Every significant architectural decision has a corresponding ADR capturing the why, not just the
      what, and ADRs are kept current as a standing practice
