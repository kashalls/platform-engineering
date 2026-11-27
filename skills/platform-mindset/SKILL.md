---
name: platform-mindset
description: Apply product thinking to internal platforms. Use when deciding whether to build, buy, or leave alone; when designing golden paths, paved roads, and self-service abstractions; when weighing guardrails against gates; when a platform team is becoming a ticket queue or a bottleneck; when defining platform APIs and contracts; or when measuring whether a platform is actually helping.
---

# Platform mindset

A platform is a **product with internal users**. Almost every platform failure
traces back to forgetting that: teams build what is technically interesting,
mandate its use, and are then surprised that users route around it.

## The central test

> Would developers choose this if it were optional?

If yes, you have a platform. If no, you have a tax, and mandating it converts
resentment into shadow infrastructure. This is the question to return to whenever
a design decision is contested.

## Golden paths, not golden cages

A **golden path** is the well-supported, documented, obvious way to do a common
thing. It is not the only way.

- Make it **easy**, not mandatory. Adoption is your metric, not compliance.
- Cover the **common case completely**, not every case partially. A path that
  handles 80% of services end-to-end beats one that handles 100% of services
  halfway.
- Provide an **escape hatch**, and make using it a conversation rather than a
  violation. Teams that leave the path are showing you where it is too narrow.
- **Do not break the escape hatch to force adoption.** That is the fastest way to
  lose credibility with the teams whose problems are hardest.

The signal that a golden path is working: teams pick it without being told,
because it is the shortest route to production.

## Guardrails over gates

| | Gate | Guardrail |
|---|---|---|
| Timing | Blocks, waits for a human | Prevents or corrects automatically |
| Scaling | Queue grows with demand | Cost is fixed |
| Failure mode | People route around it | People notice at the right moment |
| Example | Manual architecture review | Admission policy rejecting privileged pods |

Prefer, in order:

1. **Impossible** — the platform will not let it happen (policy engine, type
   system, API that does not expose it).
2. **Automatic** — it self-corrects (GitOps drift correction, autoscaling).
3. **Visible** — it is obvious and attributable (dashboards, cost showback).
4. **Documented** — a human must remember. The weakest control; treat it as a
   last resort, not a first one.

Every manual review step is a bottleneck that scales with the number of teams,
not with the size of the platform team. That arithmetic is the whole argument.

## Self-service means all the way

Self-service that ends in "then file a ticket" is not self-service. A real
self-service capability includes:

- **Provisioning** — get the thing without a human.
- **Configuration** — change it without a human.
- **Observation** — see its state, health, and cost without a human.
- **Debugging** — diagnose it without a human.
- **Deletion** — remove it without a human, and stop paying for it.

The last two are where most internal platforms stop, which is exactly why the
platform team ends up as a support desk. If users cannot debug their own
workload, every incident is your incident.

## The abstraction trap

Abstractions must be **honest about what they hide**.

- A leaky abstraction that hides 90% and clearly exposes the rest is fine.
- An abstraction that hides everything until it breaks, then requires full
  knowledge of the underlying system **plus** the abstraction, is worse than
  nothing.
- Wrapping Kubernetes so nobody needs to learn it means that when it fails, the
  user has to learn both.

Test: when this abstraction fails, what does the user see, and can they act on
it? If the answer is "an error from three layers down", the abstraction is not
finished.

Prefer abstractions that **compose** and can be opened. Generate a manifest the
user can read and, in a crisis, edit. A black box has no crisis mode.

## Build, buy, or leave alone

| Question | If yes |
|---|---|
| Is this our competitive differentiator? | Build |
| Does a good managed option exist at our scale? | Buy |
| Is the pain real, measured, and recurring? | Solve it |
| Are we building it because it is interesting? | Stop |
| Will we still staff this in three years? | Required for build |

The most under-used option is **leave alone**. Not every inconsistency needs a
platform. Three teams doing something slightly differently is often cheaper than
the abstraction that unifies them, plus its maintenance, plus its migration.

Build only what you will operate. An internal tool with no owner is technical
debt with a dashboard.

## Measuring a platform

Measure **outcomes for users**, not platform activity.

Good:

- Lead time from commit to production
- Change failure rate, and time to restore
- Time for a new service to reach production (hours, not weeks)
- Time for a new engineer's first deploy
- **Voluntary adoption rate** of golden paths
- Cost per unit of workload

Bad:

- Tickets closed (rewards being a queue)
- Number of features shipped by the platform team
- Cluster count, resource count (activity, not value)
- Mandated adoption percentage (measures the mandate, not the platform)

The DORA four — deployment frequency, lead time, change failure rate, time to
restore — are the standard outcome set, and they measure the whole delivery
system rather than your team's output. That is the point.

Ask users directly and regularly: *what is the most annoying part of shipping
here?* The answer is your backlog, and it is usually not what you were working on.

## Anti-patterns

- **The ticket queue.** Every request needs a human. Automate the top three
  request types and the queue halves.
- **The mandate.** Adoption enforced rather than earned. Produces compliance
  theatre and shadow infrastructure.
- **The abstraction nobody asked for.** Built for an imagined user. Find three
  real teams with the same real problem first.
- **The platform team as gatekeeper.** Approving instead of enabling. Convert
  every approval into a policy or a default.
- **Version zero forever.** Never deprecating, so every experiment is supported
  in perpetuity, and the team is fully occupied by its own history.
- **Documentation as the product.** A wiki explaining how to do it manually is
  not a platform.
- **Ignoring the exit.** No path off the platform means teams with genuinely
  different needs are stuck fighting you.

## Practical starting points

- **Publish the paved road** and what it guarantees. If it is not written down,
  it is not a contract.
- **Version your platform APIs** and deprecate on a published schedule with a
  migration path. Internal does not mean exempt.
- **Default to secure and cheap**, allow expensive and permissive with a
  deliberate opt-in that is visible and attributable.
- **Give every resource an owner label** and make unowned resources visible.
  Cost and cleanup both depend on it.
- **Run your own platform.** If the team does not deploy through the golden path,
  it will not notice that it is broken.
- **Treat the first day as the product.** Time from "I have a repo" to "it is
  serving traffic" is the single most honest measure of a platform.
