---
name: flux
description: Troubleshoot and author Flux CD (GitOps Toolkit) v2. Use when a Kustomization or HelmRelease is not ready, stuck, or reconciling the wrong revision; when a GitRepository, OCIRepository, or HelmRepository source fails to fetch or authenticate; when changes in Git are not landing in the cluster; when a HelmRelease is stuck upgrading, retries-exhausted, or in a failed release; or when designing repo structure, dependsOn ordering, variable substitution, drift correction, and multi-tenancy.
---

# Flux CD (v2)

Flux reconciles cluster state from a source. Almost every problem is answered by
walking the chain **backwards** from the symptom to the source:

```
GitRepository / OCIRepository / HelmRepository   ← does the source fetch?
        └── Kustomization                        ← does it build and apply?
                └── HelmRelease                  ← does the chart install?
                        └── workload             ← is the pod healthy?
```

A failure at any level makes everything below it stale, and the object below
usually reports a *misleading* error. Always start at the top of the chain.

## Ground the install

```bash
flux check                    # CRDs, controllers, version skew
flux stats -A                 # counts of running/failing/suspended per kind
flux get all -A --status-selector ready=false
```

`flux get all -A` is the single best overview: one row per object with
`REVISION`, `SUSPENDED`, `READY`, and `MESSAGE`. Filter to failures with
`--status-selector ready=false`.

Confirm the API versions this install actually serves rather than assuming; Flux
has moved several kinds to GA and the manifests you find online may be older:

```bash
flux check 2>&1 | grep toolkit
```

On a current v2 install these are `kustomize.toolkit.fluxcd.io/v1`,
`source.toolkit.fluxcd.io/v1`, `helm.toolkit.fluxcd.io/v2`, and
`notification.toolkit.fluxcd.io/v1` (Alert and Provider were `v1beta3` on older
releases; v2.10 serves them as `v1`).

## Triage order

```bash
# 1. What is not ready, everywhere.
flux get all -A --status-selector ready=false

# 2. Read the condition on the failing object. The message is the diagnosis.
kubectl get kustomization <name> -n <ns> -o jsonpath='{.status.conditions}' | jq

# 3. What the controller was doing.
kubectl logs -n flux-system deploy/kustomize-controller --tail=100
kubectl logs -n flux-system deploy/helm-controller --tail=100
kubectl logs -n flux-system deploy/source-controller --tail=100

# 4. Who owns this object and where did it come from.
flux trace <kind>/<name> -n <ns>
flux tree kustomization <name> -n <ns>
```

`flux tree` prints the full inventory a Kustomization owns, including resources
created indirectly through a HelmRelease. It is the fastest way to answer "is
Flux actually managing this object".

> `flux trace` and `flux events` print a preview/under-development banner. They
> work, but treat their output format as unstable.

## Is it suspended

```bash
# Every suspended Flux object, any kind.
kubectl get kustomizations,helmreleases,gitrepositories,ocirepositories,helmrepositories -A \
  -o jsonpath='{range .items[?(@.spec.suspend==true)]}{.kind}{"/"}{.metadata.namespace}{"/"}{.metadata.name}{"\n"}{end}'

flux resume kustomization <name> -n <ns>
```

Do **not** filter the `SUSPENDED` column of `flux get all` positionally with
`awk`. Kinds print different column counts — Alert and Receiver rows have no
revision — so the column index shifts and you silently read `READY` instead.

Note that `spec.suspend` is *absent* rather than `false` when unset, so a
`?(@.spec.suspend==false)` filter matches nothing at all. Only test for `true`.

A suspended object reports its **last** state and never changes. This looks
exactly like a stuck reconciliation and wastes more debugging time than any
other single cause. Check it first. Suspension survives restarts and is
frequently left behind after someone debugged an incident weeks ago.

## Force a reconcile

```bash
flux reconcile source git flux-system                     # re-fetch the source
flux reconcile kustomization <name> -n <ns> --with-source  # source, then apply
flux reconcile helmrelease <name> -n <ns> --with-source
```

Without `--with-source`, reconciling a Kustomization re-applies the **artifact
already stored** by source-controller. If your Git commit is not showing up, the
source has not been re-fetched, and reconciling the Kustomization alone will
faithfully re-apply the old revision.

## Gotchas

- **`READY=True` means "applied successfully", not "working".** A Kustomization
  is Ready when the server accepted the manifests. Use `wait: true` plus
  `healthChecks` to make readiness depend on the workload actually becoming
  available.
- **The revision in `flux get` is the *source* revision, not what is running.**
  Compare against the workload's own state before concluding a deploy landed.
- **`dependsOn` waits for Ready, so it inherits the above.** Without `wait: true`
  on the dependency, dependents start as soon as manifests apply, not when the
  thing works.
- **`dependsOn` is namespaced.** Referencing a Kustomization in another namespace
  requires the `namespace` field; without it Flux looks in its own namespace and
  reports the dependency as not found.
- **Interval is a floor, not a schedule.** A long interval delays drift
  correction; `retryInterval` governs the failure path and defaults to `interval`.
- **Flux only corrects drift for fields it manages.** Fields added by another
  controller or a `kubectl edit` on a field Git does not set are left alone.
- **`force: true` is for immutable-field conflicts, not drift.** It deletes and
  recreates the object instead of patching it, so it can be destructive.
- **Deleting a Kustomization deletes everything it owns** when `prune: true`.
  Suspend before deleting if you want the workloads to survive.
- **A HelmRelease failure often has nothing to do with Helm.** If the chart's
  source is stale or unauthenticated, the HelmRelease reports a chart error.

## References

| File | Covers |
|---|---|
| [references/sources.md](references/sources.md) | GitRepository, OCIRepository, HelmRepository, auth, verification |
| [references/kustomization.md](references/kustomization.md) | Build failures, health checks, dependsOn, postBuild, pruning, drift |
| [references/helmrelease.md](references/helmrelease.md) | Install/upgrade failures, remediation, values, chart refs, stuck releases |
| [references/repo-structure.md](references/repo-structure.md) | Repo layout, environments, tenancy, bootstrap, secrets |
