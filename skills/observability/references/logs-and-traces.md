# Logs and traces

## Structured logging

Emit JSON (or logfmt), one event per line, with a stable set of keys:

```json
{"ts":"2026-08-21T10:15:30Z","level":"error","msg":"payment failed",
 "service":"checkout","trace_id":"4bf92f...","user_id":"u_123",
 "order_id":"o_456","err":"upstream timeout","duration_ms":5031}
```

Rules that make logs queryable later:

- **One event per line.** Multi-line stack traces need a collector-side
  multiline rule; without one, every line becomes a separate unrelated "log".
- **Consistent key names across services.** `user_id` in one service and `userId`
  in another makes cross-service queries impossible.
- **Include `trace_id`** on every line. This is what connects logs to traces and
  is the single highest-value field in the record.
- **Log the error, not a message about the error.** `err` as a field, not
  interpolated into `msg`.
- **Never log secrets, tokens, full request bodies, or PII** you have not decided
  to retain. Logs are copied to more places than you expect and are hard to
  redact retroactively.
- **`msg` should be a constant string**, with the variable parts in fields. That
  makes "count of this event type" a simple aggregation.

## LogQL (Loki)

A LogQL query is a **stream selector** followed by optional filters and parsers.

```logql
# Stream selector: label matchers, always required
{namespace="prod", app="checkout"}

# Line filters: |= contains, != not contains, |~ regex, !~ not regex
{namespace="prod"} |= "error" != "healthcheck"

# Parse structured fields, then filter on them
{namespace="prod"} | json | level="error" | duration_ms > 1000
{namespace="prod"} | logfmt | status >= 500

# Extract fields from unstructured lines
{namespace="prod"} | pattern "<_> <method> <path> <status> <_>" | status="500"

# Metrics from logs
sum by (level) (rate({namespace="prod"} | json [5m]))
sum(count_over_time({namespace="prod"} |= "error" [5m]))

# Quantile over an extracted numeric field
quantile_over_time(0.99, {namespace="prod"} | json | unwrap duration_ms [5m])
```

**The stream selector is the performance-critical part.** Loki indexes only the
stream labels; everything after the selector is a brute-force scan of matching
chunks. `{namespace="prod"} |= "error"` over a day can read enormous volumes.
Narrow by labels and by time first, always.

**Do not put high-cardinality values in Loki stream labels** either. A label per
`trace_id` creates a stream per request, which is worse for Loki than high
cardinality is for Prometheus. Keep those as fields inside the line, extracted at
query time with `| json`.

## Log levels that mean something

| Level | Use | Should it alert? |
|---|---|---|
| `error` | A request failed and a human may need to act | Rate-based alert, yes |
| `warn` | Recovered, degraded, or deprecated | No |
| `info` | Significant state changes — startup, config, shutdown | No |
| `debug` | Off in production | No |

If `error` is used for things nobody will act on, error-rate alerts become
useless and the level stops carrying information. That erosion is the most common
logging failure in practice.

## Sampling

At volume, log everything at `error`, sample the rest:

- Keep **all** errors and slow requests.
- Sample successful, fast requests at 1–10%.
- Sample **per trace**, not per line, so a sampled request keeps a complete story.

Head-based sampling decides at the start (cheap, may miss the interesting
request). Tail-based decides after seeing the whole trace (keeps errors and slow
outliers, needs a collector that buffers). Tail-based is worth the complexity
once volume is real.

## Traces

A trace is a tree of spans. Each span has a name, start and end, and attributes.

```
checkout (240ms)
├── auth.verify (12ms)
├── inventory.reserve (45ms)
└── payment.charge (180ms)
    └── HTTP POST psp.example.com (175ms)   ← the actual cost
```

Instrument with OpenTelemetry rather than a vendor SDK — the collector can then
export to any backend and you avoid re-instrumenting on a migration.

What makes traces useful:

- **Propagate context across every boundary.** A broken propagation chain turns
  one trace into several disconnected ones. `traceparent` (W3C) is the standard
  header.
- **Span attributes are the right place for high cardinality** — user IDs, order
  IDs, feature flags. This is what metrics cannot do.
- **Record exceptions on the span**, and set span status to error.
- **Name spans by operation, not by instance.** `GET /users/:id`, never
  `GET /users/12345` — the same normalization rule as metrics.

## Correlating the three

The workflow that makes an incident short:

1. **Metric alert fires** — error ratio above threshold. *Something is wrong.*
2. **Dashboard** narrows it to a service and a time window. *Where.*
3. **Exemplars or a trace search** for a failing request in that window. *Which
   call is slow or failing.*
4. **Logs filtered by that `trace_id`** give the exact error. *Why.*

Each step must hand off to the next. The handoffs are: consistent service labels
between metrics and logs, `trace_id` in log lines, and exemplars (or a link)
from metrics to traces. If a handoff is missing, the chain breaks and people fall
back to grepping, which is where the time goes.

```promql
# Exemplars attach trace IDs to histogram samples; Grafana renders them
# as clickable points on the graph.
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

## Retention and cost

Rough shape of the tradeoff:

- **Metrics** — cheap per series, expensive in aggregate. Keep 13 months of
  downsampled data for capacity planning; raw resolution for weeks.
- **Logs** — expensive per byte. 7–30 days hot is typical; archive to object
  storage if compliance requires more.
- **Traces** — expensive at 100% sampling. Days, not weeks, and sample.

Decide retention per signal deliberately. The default of "keep everything
forever" is how observability bills exceed production infrastructure bills.
