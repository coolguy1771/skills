---
name: reliability-slos-and-disaster-recovery
description: Define reliability targets with SLIs/SLOs/SLAs and error budgets, architect for high availability and fault tolerance (load balancer + multi-AZ + auto-scaling group self-healing), and choose a disaster recovery strategy (backup-and-restore, pilot light, warm standby, multi-site active-active) using RTO/RPO. Use this when the user asks "what SLA should we promise customers," "what's the difference between an SLO and an SLA," "how do I build a self-healing architecture," "HA vs fault tolerance," "what DR strategy do we need," "how do I calculate an error budget," or is deciding how much redundancy/cost is justified by an uptime target. This is reliability-engineering methodology and hands-on ALB/ASG self-healing setup, not an AWS service catalog — pair with an AWS architecture reference for exhaustive service-by-service HA/DR options.
---

# Reliability Engineering: SLOs and Disaster Recovery

Speed without reliability is a liability. This skill covers the two things that turn "the pipeline
deploys fast" into "the system stays up": a precise, negotiated definition of what "reliable"
means (so you're not chasing an impossible, unhelpful "100% uptime"), and the architectural
patterns that actually deliver it.

## The SLI → SLO → SLA hierarchy

Each layer builds on and constrains the one below it — measurement, then internal target,
then external promise:

- **SLI (Service-Level Indicator)** — the raw, quantifiable measurement of what the service
  actually does: request latency, error rate, throughput. Pick SLIs that genuinely reflect user
  experience; a wrong SLI means every target built on it is meaningless.
- **SLO (Service-Level Objective)** — your internal target for an SLI over a period, e.g. "99.9%
  of API calls complete in under 200ms this month." This is the number engineering actually
  manages toward.
- **SLA (Service-Level Agreement)** — the external, contractual promise to customers, with
  consequences for missing it. Always set the SLA *more lenient* than the SLO (e.g., internal
  SLO 99.9%, external SLA 99.5%) — this margin is your buffer against normal variance,
  never promise externally what you haven't first proven internally.

### Error budgets: the practical output of an SLO

A 99.9% uptime SLO implies roughly 43 minutes of acceptable downtime per month — that's
your **error budget**. While budget remains, ship features and take reasonable risks. As the
budget depletes, freeze risky releases and redirect effort to stability. This converts "reliability"
from a vague aspiration into a number that drives concrete release decisions, and gives teams
a principled way to say no to a risky deploy without it being a political argument.

## High availability vs. fault tolerance (these are not the same target)

- **High Availability (HA)** — minimizes downtime through redundancy; accepts a brief,
  recoverable interruption. Like a hospital backup generator: lights flicker, then power
  resumes. Achieved with a load balancer across ≥2 instances, multiple Availability Zones,
  and managed multi-AZ database failover (typically 60–120 seconds of failover time, during
  which requests may briefly error).
- **Fault Tolerance (FT)** — no user-visible interruption at all, even during a component
  failure. Like a multi-engine plane losing one engine mid-flight. Requires a **stateless
  application design** (no session data on the local instance — state lives in shared storage
  like a cache or database) so any request can be transparently retried on any other
  instance with no loss of context. FT is materially more expensive and complex than HA;
  reserve it for workloads where even a 60-second blip is unacceptable.

Know which one your SLO actually requires before over- (or under-) building the
architecture — most workloads need HA, not FT.

### Building a self-healing architecture (ALB + ASG)

The canonical AWS pattern for HA + basic FT-like recovery:

1. **Launch template** — the reusable EC2 instance blueprint (AMI, instance type, security
   group, user-data bootstrap script). Always use launch templates, not the deprecated
   launch configurations.
2. **Application Load Balancer (ALB)** — the single entry point; health-checks each target
   and stops routing to unhealthy ones within seconds. Deploy across ≥2 Availability Zones
   so the load balancer itself isn't a single point of failure.
3. **Auto-Scaling Group (ASG)** — maintains a desired instance count, replacing any
   terminated/unhealthy instance automatically. Set `min`/`desired`/`max` deliberately (e.g.,
   2/2/4) and attach a target-tracking scaling policy (e.g., hold average CPU at 50%) paired
   with a CloudWatch-style alarm for visibility.

Verify it actually self-heals by manually terminating one instance and confirming: (a) the ALB
stops routing to it within seconds, (b) the ASG launches a replacement within minutes, and
(c) the application stayed reachable throughout via the ALB's DNS name, served by the
remaining healthy instance. If instances stay "unhealthy" for several minutes, SSH in and
`curl localhost` first — a missing inbound HTTP rule on the security group or a failed
user-data script are the two most common causes, not the load balancer itself.

## Disaster recovery: surviving a whole region going down

HA/FT protect against a single server or AZ failing. DR protects against something far bigger
— an entire region outage. Two numbers drive every DR decision:

- **RTO (Recovery Time Objective)** — how long can you be down before it's unacceptable?
  Directly drives infrastructure cost: a 15-minute RTO requires warm/pre-provisioned
  capacity standing by; a 24-hour RTO doesn't.
- **RPO (Recovery Point Objective)** — how much data can you afford to lose? Drives backup
  frequency and replication strategy: a near-zero RPO (financial transactions) needs
  synchronous replication; a 24-hour RPO (regenerable analytics data) is fine with nightly
  backups.

| Strategy | RTO | RPO | Cost | How it works |
|---|---|---|---|---|
| **Backup & restore** | Hours | Hours (backup interval) | Lowest | Regular snapshots to another region; manually provision + restore during a disaster |
| **Pilot light** | Minutes | Minutes | Low-moderate | Core (e.g., database) replicated and running; app servers off, started up on failover |
| **Warm standby** | Minutes | Seconds-minutes | Moderate-high | Scaled-down but fully functional environment always running; scale up and redirect traffic on failover |
| **Multi-site active-active** | Near-zero | Near-zero | Highest | Full stack running in ≥2 regions simultaneously, traffic distributed (e.g., via Route 53 weighted/latency routing); a failing region is automatically dropped from rotation |

Pick the strategy the RTO/RPO actually demands, not the most impressive-sounding one —
an internal reporting tool doesn't need active-active, and a financial transaction system can't
survive on nightly backups.

**Route 53's role in active-active**: health checks continuously probe each region's endpoint;
failover routing policies designate primary/secondary so traffic shifts automatically when the
primary fails its health check, while weighted/latency-based routing distributes load across
both regions in a true active-active setup — all without manual DNS intervention.

**Test the DR plan, not just the document.** A DR plan that has never been executed is a
document, not a capability — schedule at least annual DR drills that simulate a real regional
failure end to end. Drills surface gaps in runbooks and undocumented dependencies long
before a real disaster does.

## Common pitfalls

- **Promising an SLA before establishing an internal SLO and reliable SLI measurement** —
  this is promising customers something you haven't proven you can hit; it will eventually
  be violated.
- **Chasing "100% uptime"** as a literal goal — it's practically impossible and provides no
  guidance on where to actually invest; an error budget is the actionable version of a
  reliability target.
- **Confusing HA with FT** — building only HA (brief failover blips) when the actual
  requirement is zero-interruption FT, or over-engineering FT-level stateless design for a
  workload that would have been fine with simple redundancy.
- **ASGs alone treated as instant failover** — replacement instances take minutes to launch
  and initialize; ASGs give eventual self-healing, not zero-gap continuity, unless paired with
  a genuinely stateless design and enough standing capacity.
- **Choosing a DR strategy without RTO/RPO numbers first** — "let's do warm standby, it
  sounds robust" without the cost/recovery-time math backing it up leads to either
  overspending or a false sense of security.
- **Never drilling the DR plan** — untested runbooks reliably fail in exactly the ways nobody
  anticipated, precisely when the pressure is highest.

## Checklist

- [ ] SLIs are chosen because they reflect actual user experience, not just what's easy to measure
- [ ] SLO is set first, measured against real production data, before any external SLA is promised
- [ ] The SLA leaves a deliberate buffer below the SLO
- [ ] Error budget consumption is tracked and actually influences release-risk decisions
- [ ] The HA vs. FT requirement is explicit before architecture is built, not assumed
- [ ] Load balancer, compute, and (if applicable) database span at least two Availability Zones
- [ ] Self-healing (ASG or equivalent) has been tested by actually terminating an instance, not just configured
- [ ] RTO and RPO are defined numbers, not vague intentions, before a DR strategy is chosen
- [ ] The DR strategy matches what the RTO/RPO actually requires, not the most available budget
- [ ] DR drills are scheduled at least annually and their findings feed back into the runbook
