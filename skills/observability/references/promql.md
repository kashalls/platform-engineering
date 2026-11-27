# PromQL

## Types

- **Instant vector** — one sample per series at one instant. `http_requests_total`
- **Range vector** — a window of samples per series. `http_requests_total[5m]`
- **Scalar** — a single number.

Range vectors cannot be graphed directly; they exist to be fed into a function
like `rate()`. `Error executing query: invalid expression type "range vector"` is
almost always a missing `rate()`.

## Counters, gauges, histograms

| Type | Use | Functions |
|---|---|---|
| Counter (`_total`) | Only goes up, resets to 0 on restart | `rate`, `irate`, `increase`, `resets` |
| Gauge | Goes up and down | `delta`, `deriv`, `predict_linear`, `avg_over_time` |
| Histogram (`_bucket`, `_sum`, `_count`) | Distributions | `histogram_quantile` |
| Summary (`_sum`, `_count`, quantile label) | Pre-computed quantiles | **Cannot be aggregated** |

**Summaries cannot be aggregated across instances.** Their quantiles are computed
client-side, so there is no correct way to combine them. If you need a fleet-wide
p99, you need a histogram. This is the main reason to prefer histograms.

## Rate functions

```promql
rate(http_requests_total[5m])       # per-second average over the window
irate(http_requests_total[5m])      # per-second from the LAST TWO points
increase(http_requests_total[1h])   # total increase over the window
```

All three handle counter resets. `delta()` does not — it is for gauges only.

- Use `rate()` for alerts and most graphs.
- Use `irate()` only for high-resolution graphs of volatile signals; it is far too
  jumpy for alerting.
- `increase()` is exactly `rate() * window`, so it can return fractional values.
  `increase(x[1h]) > 0` is fine; `increase(x[1h]) >= 1` may miss real events.

**Always `rate()` before `sum()`:**

```promql
sum(rate(http_requests_total[5m]))          # correct
rate(sum(http_requests_total)[5m:])         # wrong — resets are hidden
```

Summing counters first hides per-series resets, so a pod restart looks like a
massive negative jump and the rate goes to zero or spikes.

## Histograms

```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Per-route p95
histogram_quantile(0.95, sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))

# Fraction of requests under 300ms — often better than a quantile
sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
  / sum(rate(http_request_duration_seconds_count[5m]))
```

Rules:

1. `rate()` the buckets.
2. `sum by (le, ...)` — `le` **must** survive the aggregation.
3. Then `histogram_quantile`.

Accuracy is bounded by bucket boundaries; the function linearly interpolates
inside a bucket. If your SLO is 300ms, you need a bucket boundary at 0.3, or the
number is a guess. Requests above the largest finite bucket land in `+Inf` and
the quantile saturates at that boundary.

The bucket-ratio form (the third query above) is exact, cheap, and usually what
an SLO actually needs.

## Selectors and matching

```promql
metric{job="api", code=~"5..", env!="dev"}
metric{__name__=~"http_.*"}
```

`=~` is fully anchored, so `code=~"5.."` means the whole value matches. A regex
that matches nothing yields an empty result, not an error — which is why typos in
label names produce silently empty dashboards.

Binary operations between vectors match on **all labels by default**:

```promql
# Match only on shared labels you name
sum by (instance) (rate(errors[5m]))
  / on (instance) sum by (instance) (rate(total[5m]))

# Many-to-one: keep labels from the "many" side
rate(errors[5m]) / on (instance) group_left rate(total[5m])

# Ignore a label that differs between the two sides
a / ignoring (code) b
```

`Error: found duplicate series for the match group` means the two sides are not
uniquely matched — you need `on()`/`ignoring()`, or a `group_left`/`group_right`.

## Handling absent and zero

```promql
absent(up{job="api"})                     # 1 if the series does not exist
absent_over_time(up{job="api"}[5m])
(sum(rate(errors[5m])) or vector(0))      # default to 0 when empty
clamp_min(sum(rate(total[5m])), 1)        # avoid divide-by-zero
```

An empty result is not zero. `sum(rate(errors[5m])) > 0` never fires if the
metric has never existed — including when the exporter is down, which is exactly
when you wanted to know. Pair error alerts with an `absent()` or `up == 0` alert.

## Recording rules

Precompute expensive or frequently used expressions:

```yaml
groups:
  - name: api
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
      - record: job:http_errors:ratio_rate5m
        expr: |
          sum by (job) (rate(http_requests_total{code=~"5.."}[5m]))
            / sum by (job) (rate(http_requests_total[5m]))
```

Naming convention: `level:metric:operations` — the aggregation level, the metric,
then what was done to it. It is worth following; it makes rules self-describing.

Rules within a group evaluate **sequentially**, so a rule can use one defined
earlier in the same group. Across groups, evaluation is concurrent and ordering
is not guaranteed.

Recording rules do not backfill. A new rule starts producing data now, with no
history.

## Subqueries

```promql
max_over_time(rate(http_requests_total[5m])[1h:1m])
```

Reads as: evaluate `rate(...[5m])` every 1m over the last 1h, then take the max.
Expensive — prefer a recording rule for anything queried often.

## Debugging a query

Build it up one layer at a time:

1. `metric_name` — does the series exist at all?
2. `metric_name{label="value"}` — do the selectors match?
3. `rate(metric_name[5m])` — is it increasing?
4. `sum by (x) (rate(...))` — does the aggregation keep the labels you need?

Empty results are almost always step 2 (a label typo or a value that does not
exist) or staleness — the series existed but has not been scraped recently.
