# Writing manifests that do not page you

## Resources

```yaml
resources:
  requests: {cpu: 100m, memory: 256Mi}   # what schedules
  limits:   {memory: 256Mi}              # what kills
```

- **Always set memory requests and limits, and set them equal.** Memory is
  incompressible: exceeding the limit means OOMKill, so a gap between request and
  limit only buys you an unpredictable death.
- **Set CPU requests. Consider omitting CPU limits.** CPU is compressible —
  hitting the limit throttles, adding latency with no visible error. For
  latency-sensitive services the throttling is usually worse than the noisy
  neighbor it prevents.
- Derive numbers from observed usage (p95 for requests, peak for memory limits),
  not from guesses.

## Probes

```yaml
startupProbe:                       # protects slow starts
  httpGet: {path: /healthz, port: http}
  failureThreshold: 30
  periodSeconds: 5                  # allows 150s to start
livenessProbe:                      # is the process wedged
  httpGet: {path: /healthz, port: http}
  periodSeconds: 10
readinessProbe:                     # should it get traffic
  httpGet: {path: /ready, port: http}
  periodSeconds: 5
```

- **Liveness must not check dependencies.** A liveness probe that touches the
  database turns a database blip into a full restart storm.
- **Readiness should check dependencies.** That is what it is for.
- **Use `startupProbe` instead of a large `initialDelaySeconds`.** The startup
  probe holds off liveness until the app is up; a long liveness delay leaves the
  app unmonitored for that entire period.
- The effective timeout is `failureThreshold × periodSeconds`.
- `/healthz` and `/ready` should be **different endpoints**. Pointing both at one
  handler collapses the distinction that makes the pair useful.

## Rollouts

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate: {maxSurge: 1, maxUnavailable: 0}
```

`maxUnavailable: 0` keeps full capacity during a deploy, at the cost of needing
room for one extra pod. Set `minReadySeconds` so a pod must stay Ready before the
rollout continues, and `progressDeadlineSeconds` so a wedged rollout fails
loudly instead of hanging.

```bash
kubectl rollout status deployment/<name> -n <ns> --timeout=5m
kubectl rollout history deployment/<name> -n <ns>
kubectl rollout undo deployment/<name> -n <ns> --to-revision=<n>
```

`kubectl rollout restart` re-rolls pods by stamping a pod-template annotation —
the correct way to pick up a changed Secret or ConfigMap, since neither triggers
a rollout on its own.

## Graceful shutdown

```yaml
terminationGracePeriodSeconds: 60
lifecycle:
  preStop:
    exec: {command: ["sleep", "5"]}
```

Endpoint removal and SIGTERM happen **concurrently**, not in order. A pod can
receive new connections for a moment after SIGTERM, which shows up as 502s during
every deploy. The `preStop` sleep gives the dataplane time to stop routing before
the app begins shutting down. The app must also handle SIGTERM by draining rather
than exiting immediately.

## Disruption budgets

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2          # or maxUnavailable: 1
  selector:
    matchLabels: {app: web}
```

PDBs constrain **voluntary** disruptions only — drains and evictions, not crashes
or node failures. `minAvailable` equal to the replica count means
`disruptionsAllowed: 0`, which blocks node drains indefinitely. Prefer
`maxUnavailable: 1` for a service with several replicas.

## Security context

```yaml
securityContext:                   # pod level
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  seccompProfile: {type: RuntimeDefault}
containers:
  - securityContext:               # container level, wins on conflict
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities: {drop: ["ALL"]}
```

`readOnlyRootFilesystem: true` requires an `emptyDir` for every path the app
writes — usually `/tmp`. `fsGroup` triggers a recursive chown of every volume,
which is slow on large ones; `fsGroupChangePolicy: OnRootMismatch` avoids
repeating it on each start.

## Config and secrets

Mount config as files rather than env vars where you can: mounted ConfigMaps and
Secrets update in place (with kubelet sync delay), while env vars are fixed for
the pod's lifetime. Neither restarts pods on change — use `rollout restart`, or a
hash annotation on the pod template so the change itself triggers a rollout.

`envFrom` on a ConfigMap silently skips keys that are not valid env var names.
