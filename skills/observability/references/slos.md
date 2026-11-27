# SLIs, SLOs, and error budgets

## The chain

- **SLI** — a measurement of user-visible behavior. *"Fraction of HTTP requests
  that return non-5xx within 300ms."*
- **SLO** — a target for that measurement over a window. *"99.9% over 30 days."*
- **Error budget** — what the SLO permits you to fail. 99.9% over 30 days is
  **43 minutes 12 seconds** of failure.
- **SLA** — a contract with financial consequences. Set it looser than the SLO,
  always.

The point of an error budget is that it converts reliability from an argument
into a number. Budget remaining means ship; budget exhausted means stop shipping
and fix reliability. That rule is the only reason any of this is worth the effort.

## Choosing an SLI

It must be measured **from the user's perspective**. CPU utilization is not an
SLI. Neither is pod restart count.

| Service type | Reasonable SLI |
|---|---|
| Request/response | Fraction of requests that succeed within a latency threshold |
| Pipeline / batch | Fraction of records processed correctly within a freshness target |
| Storage | Fraction of reads/writes that succeed; durability |
| Scheduled job | Fraction of runs that complete on time |

Measure at the point closest to the user that you control — the load balancer or
API gateway, not inside the application, which cannot record the requests it
never received.

## Budget arithmetic

Over a 30-day window:

| SLO | Allowed downtime |
|---|---|
| 99% | 7h 12m |
| 99.5% | 3h 36m |
| 99.9% | 43m 12s |
| 99.95% | 21m 36s |
| 99.99% | 4m 19s |

Each extra nine costs roughly an order of magnitude more. **99.99% is
incompatible with manual intervention** — 4 minutes is less than the time to read
a page and open a laptop. Do not commit to it without automated failover.

Pick the loosest target that keeps users happy. An SLO tighter than users need
spends engineering effort with no return, and burns the budget mechanism's
credibility the first time you ignore a breach.

## Implementing an SLI in PromQL

```promql
# Good-events / total-events, the standard shape
sum(rate(http_requests_total{code!~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# Latency SLI via the bucket ratio — exact, unlike a quantile
sum(rate(http_request_duration_seconds_bucket{le="0.3", code!~"5.."}[5m]))
  / sum(rate(http_request_duration_seconds_count[5m]))
```

Use the bucket-ratio form rather than `histogram_quantile`. It is exact, it is
cheap, and it maps directly onto "fraction of requests meeting the target" — which
is the actual definition. It does require a bucket boundary at your threshold.

Record the SLI as a recording rule so dashboards and alerts share one definition:

```yaml
- record: slo:http_requests:good_ratio_rate5m
  expr: |
    sum(rate(http_requests_total{code!~"5.."}[5m]))
      / sum(rate(http_requests_total[5m]))
```

## Burn-rate alerting

Do not alert when the SLO is already breached — by then it is too late. Alert on
the **rate** at which the budget is being consumed.

Burn rate 1 means you will exhaust the budget exactly at the end of the window.
Burn rate 14.4 means you will exhaust a 30-day budget in about 2 days.

Two windows, two severities:

```yaml
# Fast burn: 2% of a 30d budget in 1h → page
- alert: ErrorBudgetFastBurn
  expr: |
    (1 - slo:http_requests:good_ratio_rate5m) > (14.4 * 0.001)
    and
    (1 - slo:http_requests:good_ratio_rate1h) > (14.4 * 0.001)
  for: 2m
  labels: {severity: page}

# Slow burn: 5% of budget in 6h → ticket
- alert: ErrorBudgetSlowBurn
  expr: |
    (1 - slo:http_requests:good_ratio_rate30m) > (6 * 0.001)
    and
    (1 - slo:http_requests:good_ratio_rate6h) > (6 * 0.001)
  for: 15m
  labels: {severity: ticket}
```

The `and` of a short and a long window is the important part: the long window
confirms the problem is sustained, and the short window makes the alert resolve
quickly once it is fixed. A single-window burn alert either flaps or takes hours
to clear.

`0.001` is `1 - 0.999`, the budget for a 99.9% SLO. Adjust both the multiplier
and the budget together.

## Operating the budget

Agree the policy **before** you need it, in writing:

- **Budget healthy** → ship features at normal pace.
- **Budget below 25%** → reliability work takes priority over features.
- **Budget exhausted** → feature freeze until the budget recovers, with a named
  exception process for security fixes.

An error budget nobody enforces is a dashboard, not a policy. The first time a
breach is waved through without a decision, the mechanism is dead.

Review monthly: was the SLO breached, was the budget spent on planned risk or on
surprises, and is the target still the right one? An SLO that has never been
close to breaching is too loose to be informative.

## Common mistakes

- **Measuring availability by uptime checks.** A synthetic probe every 60s from
  one location says almost nothing about real user experience.
- **Averaging latency.** The mean hides the tail, and the tail is what users
  complain about.
- **One SLO for a whole platform.** Different services have different users and
  different criticality. A shared SLO is either too tight for the batch job or
  too loose for the payment path.
- **Counting planned maintenance as budget-free.** If users are affected, it
  spends budget. Otherwise the number stops describing reality.
- **SLOs on internal components.** Set them on user-visible journeys. Component
  metrics are for debugging, not for commitments.
