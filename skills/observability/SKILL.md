---
name: observability
description: Design and debug observability. Use when writing PromQL or LogQL queries, when choosing what to measure for a service, when an alert is noisy, flapping, or never fires, when metric cardinality is blowing up Prometheus, when building dashboards or SLOs and error budgets, or when instrumenting an application with metrics, logs, and traces.
---

# Observability

Three signals, three jobs. Using the wrong one is the most common and most
expensive mistake in a platform's telemetry:

| Signal | Answers | Cost driver |
|---|---|---|
| **Metrics** | "Is something wrong, and how much?" | Cardinality (series count) |
| **Logs** | "What exactly happened to this one request?" | Volume (bytes) |
| **Traces** | "Where in the call graph did the time go?" | Sampling rate |

Metrics detect, traces localize, logs explain. Alert on metrics; never alert on
log volume unless you have accepted the cost of doing so.

## What to measure

**RED** — for request-driven services (the default choice):

- **Rate** — requests per second
- **Errors** — failed requests per second
- **Duration** — latency distribution

**USE** — for resources (nodes, disks, pools, queues):

- **Utilization** — percent busy
- **Saturation** — queued work
- **Errors** — error count

A service gets RED. The things it runs on get USE. Most dashboards are bad
because they show neither, and instead show whatever the library exported by
default.

## PromQL that survives contact

Verified against a live Prometheus.

```promql
# Rate by response code (RED: rate + errors)
sum by (code) (rate(apiserver_request_total[5m]))

# p99 latency — aggregate the buckets BEFORE the quantile
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Error ratio
sum(rate(http_requests_total{code=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# CPU throttling ratio, worst namespaces
topk(3, sum by (namespace) (
  rate(container_cpu_cfs_throttled_periods_total[5m])
    / rate(container_cpu_cfs_periods_total[5m])))

# Restarts in the last hour
sum by (namespace) (increase(kube_pod_container_status_restarts_total[1h])) > 0
```

The non-negotiable rules:

- **`rate()` before `sum()`, always.** `rate()` needs the raw counter to detect
  resets. `rate(sum(...))` is wrong and silently produces garbage after any pod
  restart.
- **`sum by (le)` before `histogram_quantile`.** Taking the quantile per-series
  and then averaging gives a number that means nothing.
- **A quantile of a quantile is not a quantile.** You cannot average p99s across
  instances. Aggregate buckets, then compute.
- **Range must cover at least 4 scrape intervals.** `rate(x[1m])` on a 30s scrape
  is fragile; `[5m]` is the safe default.
- **`rate()` for per-second, `increase()` for a count over a window.** Both handle
  counter resets; `delta()` does not and is for gauges.
- **`irate()` is for graphs, not alerts** — it uses the last two points and is far
  too jumpy to threshold on.

## Alerts that are worth waking up for

```yaml
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{code=~"5.."}[5m]))
      / sum(rate(http_requests_total[5m])) > 0.05
  for: 10m
  labels: {severity: page}
  annotations:
    summary: "5xx rate above 5% for 10m"
    runbook_url: "https://runbooks.example.com/high-error-rate"
```

- **`for:` is what kills flapping.** A threshold with no duration pages on a
  single scrape blip.
- **Alert on symptoms, not causes.** "Error rate is high" is worth a page. "CPU is
  at 90%" is not — if nothing is failing, nobody should wake up.
- **Every paging alert needs a runbook link.** If you cannot write the runbook,
  the alert is not actionable and should not page.
- **Ratios, not absolute counts.** "100 errors/sec" means different things at
  1k rps and 10 rps.
- **Beware `absent()` and division by zero.** A service with no traffic gives
  `0/0 = NaN`, which is not `> 0.05`, so the alert silently stops evaluating.
  Guard with `and on() rate(...) > 0` or use `absent_over_time()` separately.

Three tiers, and only one of them wakes a human:

| Tier | Response | Example |
|---|---|---|
| Page | Now, a human is woken | Customer-visible errors, SLO burn |
| Ticket | Next business day | Certificate expiring in 14 days |
| Dashboard only | On demand | Utilization trends |

## Cardinality: the thing that kills Prometheus

Series count = product of all label value counts. One high-cardinality label
multiplies everything.

**Never use as a label:** user ID, request ID, trace ID, email, full URL path
with IDs, timestamp, pod IP, container ID, error message text.

Those belong in logs or traces, which are built to be high-cardinality. Metrics
are not.

```promql
# Which metrics have the most series
topk(10, count by (__name__)({__name__=~".+"}))

# Which label is doing the damage on one metric
count(count by (label_name) (metric_name))
```

Normalize before it becomes a label: `/users/123/orders` becomes
`/users/:id/orders`. Unbounded path labels are the most common self-inflicted
cardinality explosion.

## References

| File | Covers |
|---|---|
| [references/promql.md](references/promql.md) | Query patterns, recording rules, functions, common mistakes |
| [references/logs-and-traces.md](references/logs-and-traces.md) | LogQL, structured logging, sampling, trace correlation |
| [references/slos.md](references/slos.md) | SLIs, SLOs, error budgets, burn-rate alerting |

## Gotchas

- **Counters reset to zero on restart.** Always wrap in `rate`/`increase`; never
  graph a raw `_total`.
- **`histogram_quantile` interpolates within a bucket**, so precision is bounded
  by bucket boundaries. A p99 of 10s when the top bucket is 10s means "at least
  10s" — pick buckets that match your SLO.
- **Staleness**: a series that stops being reported disappears after ~5 minutes,
  and expressions referencing it evaluate to nothing rather than zero. This makes
  "alert when X == 0" fail exactly when the exporter dies.
- **Scrape interval vs range** mismatches produce jagged, misleading graphs.
- **Recording rules do not backfill.** A new rule has no history; do not expect a
  dashboard using it to show last month.
- **Grafana's `$__rate_interval`** exists to avoid the range/step mismatch. Use it
  in dashboards rather than a hardcoded `[5m]`.
