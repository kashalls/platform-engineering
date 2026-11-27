# Kustomization

A Flux `Kustomization` (`kustomize.toolkit.fluxcd.io/v1`) builds a path from a
source and applies it. It is **not** the same kind as a `kustomization.yaml`
file's `Kustomization` (`kustomize.config.k8s.io/v1beta1`) — same word, different
object, and conflating them causes real confusion when reading manifests.

```bash
flux get kustomizations -A --status-selector ready=false
kubectl get kustomization <name> -n <ns> -o jsonpath='{.status.conditions}' | jq
flux tree kustomization <name> -n <ns>
```

## Failure messages

| Message | Cause |
|---|---|
| `kustomize build failed` | The overlay itself is broken — bad patch target, missing resource, invalid YAML. |
| `path not found` | `spec.path` does not exist in the artifact. Paths are relative to the repo root. |
| `dry-run failed` / `admission webhook denied` | The cluster rejected the manifests. A policy engine or a CRD that is not installed yet. |
| `no matches for kind` | The CRD is missing. Ordering problem — see `dependsOn`. |
| `Health check failed after ...` | Applied fine, but the workload never became Ready. |
| `context deadline exceeded` | `spec.timeout` shorter than the workload takes. |
| `variable substitution failed` | `postBuild.substituteFrom` references a missing ConfigMap/Secret key. |

Reproduce a build failure locally before touching the cluster:

```bash
kustomize build ./path/to/overlay
```

That reproduces `kustomize build failed` exactly and is far faster than a
reconcile loop.

## Health checks and `wait`

By default a Kustomization is Ready when the API server **accepts** the
manifests. That says nothing about whether anything works.

```yaml
spec:
  wait: true          # wait for all applied objects to be Ready
  timeout: 5m
```

or check specific objects:

```yaml
spec:
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: web
      namespace: prod
```

Per the CRD: *"Wait instructs the controller to check the health of all the
reconciled resources. When enabled, the HealthChecks are ignored."* So setting
both is not an error — `wait: true` silently wins and your `healthChecks` list
does nothing. Use `healthChecks` alone when you only care about a few objects and
do not want one slow sidecar blocking readiness.

`wait` defaults to `false`.

**This matters for `dependsOn`,** which waits for Ready. Without `wait: true`,
a dependency is "ready" the instant its YAML applies, so dependents start against
a CRD whose controller has not started. That is the usual cause of intermittent
`no matches for kind`.

## dependsOn

```yaml
spec:
  dependsOn:
    - name: infrastructure
      namespace: flux-system     # required if different from this object's
```

Ordering rules worth internalizing:

- It waits for **Ready**, so it is only as meaningful as the health config above.
- It is not transitive in a useful sense — model the real graph, not a chain.
- A dependency that is suspended blocks its dependents forever, reporting
  "dependency not ready" with no hint that suspension is the cause.
- Circular `dependsOn` deadlocks silently; both sides sit not-ready.

The standard layering is `sources → CRDs/operators → infrastructure → apps`.

## Variable substitution

```yaml
spec:
  postBuild:
    substitute:
      cluster_name: prod
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars
      - kind: Secret
        name: cluster-secrets
        optional: true
```

Substitution happens **after** `kustomize build`, on the rendered YAML, using
`${VAR}` syntax. Consequences:

- Any literal `${...}` in your manifests is a substitution target, including shell
  scripts in ConfigMaps and Prometheus/Grafana templating. Escape as `$${...}`.
- A missing variable fails the whole build unless the source is `optional: true`.
- Substituted values are **not** encrypted at rest in Git; use SOPS for secrets.

## Pruning and drift

```yaml
spec:
  prune: true         # delete objects removed from Git
  force: false        # recreate immutable resources on conflict
```

- **`prune: true` plus deleting the Kustomization deletes every managed object.**
  Suspend first if you want the workloads to survive removing Flux's management.
- Flux tracks its inventory in `status.inventory`; objects it never applied are
  never pruned, so hand-created resources are safe but also uncorrected.
- **Drift correction only covers fields Flux manages.** A field set by another
  controller or a human `kubectl edit` on an unmanaged field persists.
- `force: true` handles immutable-field conflicts (Job selectors, some PVC and
  Service fields) by **deleting and recreating**. On a StatefulSet or a Service
  with a LoadBalancer that is disruptive.

Find drift without changing anything:

```bash
flux diff kustomization <name> -n <ns> --path ./path/to/overlay
```

## Server-side apply conflicts

Flux applies server-side as the field manager `kustomize-controller`. A conflict
error names the competing manager:

```bash
kubectl get <kind> <name> -n <ns> -o jsonpath='{.metadata.managedFields}' | jq -r '.[].manager'
```

A HorizontalPodAutoscaler fighting a hardcoded `replicas` is the classic case.
Fix it by removing `replicas` from Git, not by forcing.
