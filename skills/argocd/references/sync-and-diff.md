# Sync waves, hooks, and diffs that never converge

## Ordering: phases, then waves

Argo CD applies resources in this precedence order:

1. **Phase** — `PreSync`, `Sync`, `PostSync` (plus `Skip` and `SyncFail`)
2. **Wave** — lowest number first, may be negative
3. **Kind** — namespaces first, then other built-ins, then custom resources
4. **Name** — alphabetical

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

Default wave is `0`, so a negative wave runs before everything unannotated —
the idiomatic place for CRDs, namespaces, and database migrations.

**Argo waits for each wave to become healthy before starting the next.** A
resource that never reports Healthy stalls the sync at that wave indefinitely,
and the app sits `Progressing` with no error. That is usually a workload with no
health check that Argo cannot assess, or a genuinely failing one.

Waves are global within an Application, not per-kind. Two resources with the same
wave sync concurrently.

## Hooks

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

| Phase | Runs |
|---|---|
| `PreSync` | before the main sync — migrations, backups |
| `Sync` | alongside the main sync |
| `PostSync` | after everything is healthy — smoke tests, notifications |
| `SyncFail` | only when the sync fails — cleanup, alerting |
| `Skip` | never applied; a way to keep a manifest in Git unmanaged |
| `PostDelete` | after the Application and its resources are deleted |

Delete policies: `HookSucceeded`, `HookFailed`, `BeforeHookCreation` (the
default). Without a delete policy, hook Jobs **accumulate** — one per sync,
forever. `BeforeHookCreation` deletes the previous instance, which keeps exactly
one around for debugging.

A failing `PreSync` hook aborts the whole sync, which is the point: a failed
migration should stop the deploy.

## Diffs that never converge

The signature: an Application flaps `OutOfSync` immediately after every sync, on
a resource nobody is editing. Something in-cluster mutates the object after Argo
applies it.

```bash
argocd app diff <app>          # see exactly which field
```

Usual culprits: an HPA writing `spec.replicas`, a mutating webhook injecting a
sidecar, the API server defaulting a field, a `caBundle` injected by cert-manager.

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
    - group: ""
      kind: Service
      jqPathExpressions:
        - '.spec.ports[] | select(.name == "http") | .nodePort'
    - group: apps
      kind: Deployment
      managedFieldsManagers:
        - kube-controller-manager
```

`managedFieldsManagers` is the cleanest option when a known controller owns the
field: it ignores whatever that manager set, without you enumerating paths.

`ignoreDifferences` only affects the diff: a sync still applies the full
manifest, overwriting the field the controller set. Add
`RespectIgnoreDifferences=true` to `syncOptions` to make sync leave ignored
fields as they are in the cluster.

## Sync options

```yaml
spec:
  syncPolicy:
    syncOptions:
      - CreateNamespace=true          # create spec.destination.namespace
      - ServerSideApply=true          # avoids the last-applied annotation size limit
      - ApplyOutOfSyncOnly=true       # only touch drifted resources; faster on big apps
      - PruneLast=true                # delete removed resources after the rest is healthy
      - PrunePropagationPolicy=foreground
      - RespectIgnoreDifferences=true
      - Validate=false                # skip client-side schema validation
      - SkipDryRunOnMissingResource=true   # for CRs whose CRD arrives in the same sync
```

`ServerSideApply=true` is the fix for `metadata.annotations: Too long: must have
at most 262144 bytes` — the client-side `last-applied-configuration` annotation
overflowing on large CRDs.

`SkipDryRunOnMissingResource=true` handles the bootstrap case where a CR and its
CRD are in the same Application: the dry-run fails because the kind does not
exist yet.

Per-resource, the same options work as an annotation:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Prune=false,Delete=false
```

`Prune=false` protects a resource from pruning; `Delete=false` protects it from
cascading deletion when the Application is deleted. Both are appropriate for
PVCs and other stateful objects.

## Resource tracking

Argo CD v3 tracks ownership with the `argocd.argoproj.io/tracking-id`
annotation by default. Installs upgraded from v2, or configured explicitly, may
still use the `app.kubernetes.io/instance` label, which **collides with other
tools** that use the same standard label and is truncated at 63 characters.
Check which one is set in `argocd-cm`:

```yaml
data:
  application.resourceTrackingMethod: annotation   # label | annotation | annotation+label
```

Switching methods causes a one-time re-adoption of every resource. Do it during
a quiet window, not mid-incident.

## Pruning

```bash
argocd app sync <app> --prune
argocd app sync <app> --prune --dry-run    # see what would be deleted first
```

Resources Argo will not prune without help: anything annotated `Prune=false`,
anything it does not track, and cluster-scoped resources the AppProject forbids.
