---
name: incident-triage
description: Run and write up production incidents. Use when something is broken in production and you need a triage order, when deciding severity or whether to declare an incident, when coordinating roles and comms during an outage, when deciding to mitigate versus diagnose, or when writing a blameless postmortem, RCA, timeline, or corrective actions afterwards.
---

# Incident triage and RCA

Two distinct modes, and confusing them is the most common failure:

- **During**: restore service. Understanding is optional.
- **After**: understand. Speed is optional.

Do not root-cause during an incident, and do not skip root-causing after one.

## First five minutes

1. **Declare.** If you are wondering whether it is an incident, it is. Declaring
   early is cheap; declaring late costs the whole detection window.
2. **Assign a lead.** One person coordinates and does not debug. On a small
   incident this feels like overhead; on a large one its absence is fatal.
3. **Open one channel.** All discussion in one place, timestamped. Side DMs
   destroy the timeline you will need later.
4. **State the impact in user terms.** "Checkout is failing for ~30% of users",
   not "the payment pod is crashlooping". Impact drives severity; symptoms do not.
5. **Ask the only question that matters first:** *what changed?*

```bash
# What changed, in rough order of likelihood
kubectl rollout history deployment/<name> -n <ns>
flux get all -A --status-selector ready=false
argocd app history <app>
git log --oneline --since="4 hours ago"
```

Most incidents are caused by a change. Deploys, config changes, feature flags,
certificate expiry, a dependency's change, or a scheduled job. If something
changed shortly before impact began, that is your first hypothesis.

## Mitigate before you diagnose

The reflex to fix properly is the wrong reflex at 03:00.

| Mitigation | When |
|---|---|
| Roll back | A recent deploy correlates with the start of impact |
| Disable the feature flag | The change was flag-gated |
| Scale up | Saturation, not a bug |
| Fail over / drain | One node, zone, or replica is bad |
| Rate-limit or shed load | Overload, cascading failure |
| Revert the config | Config change correlates |

**Rolling back is not an admission of anything.** It is the fastest known-good
state. Diagnose the bad version at leisure, from the artifacts.

```bash
kubectl rollout undo deployment/<name> -n <ns>
flux suspend kustomization <name> -n <ns>     # stop GitOps fighting your fix
argocd app rollback <app> <history-id>
```

Suspending the GitOps controller first is important: otherwise it reverts your
manual mitigation within a minute and you lose time wondering why.

## Roles

Even for a two-person incident, name these explicitly:

- **Incident lead** — decides, delegates, keeps time. Does not debug.
- **Operations** — the only person changing things. Announces before acting.
- **Communications** — updates stakeholders on a fixed cadence.
- **Scribe** — timestamps events as they happen.

One person can hold several roles. The point is that each is *someone's* job.
"Everyone is debugging and nobody is communicating" is the default failure mode.

## Severity

Define these before you need them. A rough shape:

| Sev | Meaning | Response |
|---|---|---|
| SEV1 | Major user-visible outage, or data loss | Page immediately, all hands, exec comms |
| SEV2 | Significant degradation or a core feature down | Page, dedicated responders |
| SEV3 | Minor or partial impact, workaround exists | Business hours |
| SEV4 | No user impact; internal only | Ticket |

Severity is about **impact**, not cause or effort. Downgrade when impact drops,
and say so explicitly — otherwise the response stays at full intensity for hours.

## Communicating

Fixed cadence beats "when we know more" — silence reads as nothing happening.

```
[10:15] INVESTIGATING — Checkout failing for ~30% of users since 10:02.
        Investigating a deploy at 09:58. Next update 10:30.

[10:30] IDENTIFIED — Caused by the 09:58 release. Rolling back now.
        Next update 10:45.

[10:45] MONITORING — Rollback complete, error rate normal since 10:38.
        Monitoring for 30 minutes. Next update 11:15.

[11:15] RESOLVED — Stable for 30 minutes. Postmortem within 5 business days.
```

States: `INVESTIGATING → IDENTIFIED → MONITORING → RESOLVED`. Every update carries
impact, what is being done, and when the next update comes. Never skip the next-update
time.

## Diagnosis, when mitigation is not obvious

Work outside-in, and let each answer eliminate a layer:

```
Is it everyone or some? ─ some → which segment: region, tenant, version, path?
Did it start sharply?    ─ yes  → a change. no → saturation or a leak.
Is one dependency slow?  ─ yes  → follow it. Do not stop at the first slow thing:
                                  it may be a victim, not a cause.
Does it correlate with   ─ a deploy, a cron, a traffic peak, a cert expiry?
```

The classic trap is **stopping at the first anomaly**. A saturated thread pool is
usually a symptom of a slow downstream call, not the cause. Keep asking "and what
made *that* happen" until the answer is something that changed.

## After: the postmortem

Write it within a week, while memory is fresh. Use
[templates/postmortem.md](templates/postmortem.md).

**Blameless means directed at systems, not at people.** Not "Sam deployed without
testing" but "the pipeline allowed a deploy with no integration test, and the
canary had no automatic rollback". Sam is not the vulnerability; a pipeline that
permits an untested deploy is.

The test: could a competent, well-intentioned person have made this mistake? If
yes — and it nearly always is — the system permitted it, and the system is what
you fix.

Ask instead:

- What made this **possible**?
- What made it **hard to detect**?
- What made it **hard to diagnose**?
- What made it **hard to fix**?

Those four produce four different classes of action item. Most postmortems only
answer the first and ship a code fix, leaving detection and diagnosis exactly as
bad for next time.

## Action items that actually happen

Every action item needs an owner, a due date, and a ticket. Anything else is a
wish.

Prefer, in order:

1. **Make it impossible** — a type, a constraint, a policy that rejects it.
2. **Make it automatic** — automated rollback, a health gate.
3. **Make it obvious** — an alert that fires before users notice.
4. **Make it documented** — a runbook.

"Add a runbook" as the *only* action item means you accepted the failure and
wrote down how to suffer it faster. Sometimes that is the right call — say so
explicitly, rather than by omission.

Cap the list. Five items that ship beat twenty that do not.

## Gotchas

- **The first hypothesis is usually wrong**, and the cost is the time spent
  confirming it. Timebox each hypothesis and say out loud what would disprove it.
- **Correlation with a deploy is strong evidence but not proof.** Sometimes the
  deploy exposed something that had been broken for weeks.
- **Alert fatigue is an incident cause**, not a nuisance. If the page that
  mattered was one of forty, fix the other thirty-nine.
- **Recovery is not resolution.** A service that recovered on its own will do the
  opposite on its own too.
- **Do not change two things at once** while mitigating; you will not know which
  worked, and one may be making it worse.
- **Write the timeline as you go.** Reconstructing it afterwards from memory and
  chat scrollback is the worst part of any postmortem.
