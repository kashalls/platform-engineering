---
name: argocd
description: Troubleshoot and author Argo CD. Use when an Application is OutOfSync, Degraded, Missing, Unknown, or stuck Progressing; when a sync fails, hangs, or never prunes; when self-heal or auto-sync is not behaving; when diffs never converge because of mutating controllers; or when designing app-of-apps, ApplicationSets, sync waves, hooks, ignoreDifferences, projects, and RBAC.
---

# Argo CD

Argo CD compares **desired** (Git/Helm/Kustomize output) against **live**
(cluster) and reports two independent axes. Read both before acting:

- **Sync status** — does live match desired? `Synced` / `OutOfSync` / `Unknown`
- **Health status** — is the thing working? `Healthy` / `Progressing` /
  `Degraded` / `Suspended` / `Missing` / `Unknown`

They are orthogonal, and the combination is the diagnosis:

| Sync | Health | Meaning |
|---|---|---|
| `Synced` | `Degraded` | Argo applied exactly what Git says; the manifest itself is wrong or the app is broken. **Not an Argo problem.** |
| `OutOfSync` | `Healthy` | Working, but drifted from Git, or a controller mutates the object after apply. |
| `Synced` | `Progressing` | Rollout in flight. Wait, then investigate if it never settles. |
| `OutOfSync` | `Missing` | Never applied — usually a failing sync or a pruned/blocked resource. |
| `Unknown` | `Unknown` | Repo unreachable, manifests fail to render, or cluster unreachable. |

App health is the **worst** status among its immediate children, prioritized
`Healthy → Suspended → Progressing → Missing → Degraded → Unknown`.

## Triage

```bash
argocd app list                                  # everything, both axes
argocd app get <app>                             # per-resource breakdown
argocd app get <app> --refresh                   # force re-compare against Git
argocd app diff <app>                            # exactly what differs
argocd app history <app>                         # what was deployed when
argocd app resources <app>                       # the resource tree
```

`argocd app get` lists each resource with its own sync and health status. The one
red row is the answer — do not debug at the Application level when a single
Deployment is the cause.

Without the CLI, everything is on the CR:

```bash
kubectl get applications -A
kubectl get application <app> -n argocd -o jsonpath='{.status.conditions}' | jq
kubectl get application <app> -n argocd -o jsonpath='{.status.operationState}' | jq
kubectl get application <app> -n argocd -o jsonpath='{.status.sync.status}{"\t"}{.status.health.status}'
```

`status.operationState.message` carries the actual failure text from the last
sync. It is the single most useful field on the object.

Controller logs, when the object is not enough:

```bash
kubectl logs -n argocd statefulset/argocd-application-controller --tail=200
kubectl logs -n argocd deploy/argocd-repo-server --tail=200     # render failures
```

Note the kinds: in the upstream install the application controller is a
**StatefulSet** (it shards by cluster), while every other component is a
Deployment. `deploy/argocd-application-controller` silently matches nothing.

Render errors (bad Helm values, kustomize build failure, missing path) come from
**repo-server**, not the application controller.

## Sync

```bash
argocd app sync <app>
argocd app sync <app> --prune                    # allow deleting removed resources
argocd app sync <app> --dry-run
argocd app sync <app> --resource apps:Deployment:web    # one resource only
argocd app sync <app> --local ./path             # sync from local files, skip Git
argocd app wait <app> --health --timeout 600
```

Verified against CLI v3.5.1. Useful flags: `--server-side` (server-side apply),
`--replace` (create/replace instead of apply, for immutable-field conflicts),
`--force`, `--retry-limit`, `--async`.

**`--prune` is not the default.** Deleting a manifest from Git leaves the live
object in place and the app `OutOfSync` forever until a prune-enabled sync runs.

`argocd app diff` **exits non-zero when a diff exists** (`--exit-code` defaults to
true), which is what makes it usable directly in CI.

## Auto-sync and self-heal

```yaml
spec:
  syncPolicy:
    automated:
      prune: true          # delete resources removed from Git
      selfHeal: true       # revert manual cluster changes
      allowEmpty: false    # refuse to sync an empty manifest set
```

All three default to **false/off**, including inside `automated`. An `automated`
block without `prune` and `selfHeal` still syncs every change Git makes, new or
modified; it just never deletes resources removed from Git, and never reverts
changes made directly in the cluster.

`selfHeal: true` reverts `kubectl edit` — including emergency fixes during an
incident. Know whether it is on before you hand-patch anything.

`allowEmpty: false` is a guardrail against a repo misconfiguration rendering zero
manifests and pruning an entire application.

## Gotchas

- **`Synced` + `Degraded` is not an Argo bug.** Argo did its job; the workload is
  broken. Debug the workload.
- **Perpetual `OutOfSync` on a resource nobody edits** means a controller mutates
  it after apply — HPA on `replicas`, a mutating webhook, a defaulted field. Fix
  with `ignoreDifferences`, not by re-syncing.
- **Deleting an Application deletes its resources** when the app has the default
  `resources-finalizer.argocd.argoproj.io` finalizer. Remove the finalizer first
  to orphan them deliberately.
- **`app-of-apps` deletion cascades.** Deleting the parent deletes every child app
  and everything they manage.
- **Namespaced vs cluster-scoped resources need AppProject permission.** A denied
  resource shows as a sync error naming the project, not an RBAC error.
- **Repo-server caches manifests.** A fixed Git commit that still renders the old
  output needs `argocd app get <app> --refresh` (or hard-refresh) to re-render.
- **`Unknown` sync status usually means repo-server cannot render**, not that the
  cluster is down. Check repo-server logs first.

## References

| File | Covers |
|---|---|
| [references/sync-and-diff.md](references/sync-and-diff.md) | Sync waves, hooks, ignoreDifferences, drift that never converges |
| [references/applications.md](references/applications.md) | Application spec, app-of-apps, ApplicationSets, sources |
| [references/projects-and-rbac.md](references/projects-and-rbac.md) | AppProjects, restrictions, RBAC, SSO, multi-tenancy |
