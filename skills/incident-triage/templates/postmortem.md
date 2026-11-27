# Postmortem: <short descriptive title>

| | |
|---|---|
| **Date** | YYYY-MM-DD |
| **Severity** | SEV<n> |
| **Duration** | <detection → resolution>, <total user impact> |
| **Author** | <name> |
| **Status** | Draft / In review / Final |
| **Incident channel** | <link> |

## Summary

Two or three sentences a stakeholder can read. What broke, who was affected, how
long, and what fixed it. No jargon, no cause analysis.

> Between 10:02 and 10:38 UTC, approximately 30% of checkout requests failed with
> HTTP 500. The cause was a configuration change deployed at 09:58 that pointed
> the payment client at a decommissioned endpoint. Service was restored by rolling
> back the release.

## Impact

- **Users affected**: <number or percentage, and which segment>
- **Duration of user impact**: <start → end>
- **Requests/transactions failed**: <number>
- **Revenue / SLA / data**: <if applicable, or "none">
- **Error budget consumed**: <percentage of the period's budget>

State impact in user terms. "Pods restarted" is not impact.

## Timeline

All times UTC. Include detection, every significant action, and every point where
understanding changed — including the wrong turns.

| Time | Event |
|---|---|
| 09:58 | Release `v2.4.1` deployed to production |
| 10:02 | Error rate begins climbing; no alert yet |
| 10:07 | `HighErrorRate` alert fires; on-call paged |
| 10:09 | Incident declared SEV2, lead assigned |
| 10:14 | First hypothesis (database saturation) investigated and ruled out |
| 10:22 | Correlation with the 09:58 deploy identified |
| 10:31 | Rollback started |
| 10:38 | Error rate returns to baseline |
| 11:08 | Monitoring period complete; incident resolved |

Two derived numbers worth calling out explicitly:

- **Time to detect**: 10:02 → 10:07 (5 min)
- **Time to mitigate**: 10:07 → 10:38 (31 min)

## Root cause

What actually happened, mechanically. Follow the chain past the first anomaly
until you reach something that *changed*.

> The payment client reads its endpoint from a ConfigMap. A refactor in `v2.4.1`
> renamed the key from `payment_url` to `paymentUrl`, but the ConfigMap in the
> production overlay was not updated. The client's config loader treats a missing
> key as an empty string rather than an error, so it fell back to a hardcoded
> default pointing at an endpoint decommissioned in March.

## Contributing factors

Everything that made this worse than it needed to be:

- The config loader silently defaults instead of failing fast on a missing key.
- Staging uses a different overlay, so the rename was exercised only where it was
  correct.
- No canary or progressive rollout; the change went to 100% at once.
- The alert threshold required a 5-minute sustained error rate, delaying detection.

## The four questions

| Question | Answer | Action |
|---|---|---|
| What made it **possible**? | Silent default on a missing config key | AI-1 |
| What made it **hard to detect**? | Alert needed 5 min sustained; no deploy-correlated alerting | AI-2 |
| What made it **hard to diagnose**? | No config validation surface; first hypothesis consumed 8 min | AI-3 |
| What made it **hard to fix**? | Nothing — rollback worked as designed | — |

## What went well

Genuinely useful, not a morale exercise — these are the practices to protect.

- Rollback procedure worked first time and was well understood.
- Comms cadence held; stakeholders did not need to ask for updates.

## What went badly

- Eight minutes spent on a hypothesis with no supporting evidence.
- The deploy that caused it was not visible on the dashboard used for triage.

## Where we got lucky

The uncomfortable section, and often the most valuable one.

- The decommissioned endpoint returned errors rather than accepting and silently
  dropping payments. Had it accepted them, this would have been a data incident
  discovered days later.

## Action items

Every item has an owner, a due date, and a ticket. No item is "investigate X".

| # | Action | Type | Owner | Due | Ticket |
|---|---|---|---|---|---|
| AI-1 | Config loader fails fast on missing required keys | Prevent | | | |
| AI-2 | Alert on error-rate delta within 2 min of a deploy | Detect | | | |
| AI-3 | Surface recent deploys on the primary triage dashboard | Diagnose | | | |
| AI-4 | Progressive rollout with automatic rollback for this service | Mitigate | | | |

Types: Prevent (make it impossible) > Mitigate (make it automatic) > Detect
(make it obvious) > Document (make it survivable).

If every action item is "Document", say explicitly that the risk was accepted.

## Lessons

One or two sentences worth carrying to other services. If nothing here
generalizes, say so — not every incident teaches something new.
