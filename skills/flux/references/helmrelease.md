# HelmRelease

`helm.toolkit.fluxcd.io/v2`. The helm-controller runs Helm actions server-side;
there is no `helm` CLI state to inspect on your machine, but the underlying Helm
storage secrets are real and inspectable.

```bash
flux get helmreleases -A --status-selector ready=false
kubectl get helmrelease <name> -n <ns> -o jsonpath='{.status.conditions}' | jq
kubectl logs -n flux-system deploy/helm-controller --tail=100
```

## The chart source

Two ways to reference a chart. `chartRef` points at an `OCIRepository` or
`HelmChart` object; `chart.spec` declares a chart from a `HelmRepository` and
makes Flux create a `HelmChart` for you.

```yaml
# Modern: OCI artifact
spec:
  chartRef:
    kind: OCIRepository
    name: podinfo
```

```yaml
# Classic: HelmRepository + chart spec
spec:
  chart:
    spec:
      chart: podinfo
      version: "6.x"
      sourceRef:
        kind: HelmRepository
        name: podinfo
```

With the classic form, a generated `HelmChart` object sits between the release
and the repository, and **that** is often where the failure actually is:

```bash
kubectl get helmcharts -A
```

Its name is `<namespace>-<helmrelease-name>` and it lives in the namespace of the
*source*, not the HelmRelease. Chart pull failures show up there while the
HelmRelease reports a vaguer error.

## Failure messages

| Message | Cause |
|---|---|
| `chart pull error` / `no chart version found` | Version constraint matches nothing, or repo index is stale. |
| `install retries exhausted` | Every install attempt failed; see `upgrade`/`install` remediation below. |
| `upgrade retries exhausted` | Same for upgrades. The release is left in whatever state remediation chose. |
| `another operation (install/upgrade/rollback) is in progress` | A previous action was interrupted; the Helm release is stuck `pending-*`. |
| `cannot re-use a name that is still in use` | A release exists that Flux does not own — often a manual `helm install`. |
| `timed out waiting for the condition` | Chart's own `--wait` exceeded `spec.timeout`. |
| `values don't meet the specifications of the schema` | The chart ships `values.schema.json` and your values violate it. |

## Stuck `pending-install` / `pending-upgrade`

Helm stores release state in a Secret per revision. An interrupted operation
leaves it `pending-*` and every subsequent attempt refuses to run.

```bash
kubectl get secrets -n <ns> -l owner=helm,name=<release-name> \
  -o custom-columns='NAME:.metadata.name,STATUS:.metadata.labels.status,VERSION:.metadata.labels.version'
```

The fix is to let Flux redo the release. Suspending and resuming forces a fresh
reconcile:

```bash
flux suspend helmrelease <name> -n <ns>
flux resume helmrelease <name> -n <ns>
```

If it stays stuck, delete the newest `pending-*` release secret so Helm falls
back to the last good revision, then resume. Deleting the *only* release secret
makes Helm consider the release nonexistent and reinstall from scratch — fine for
stateless workloads, destructive for anything with a PVC or a database.

## Remediation

```yaml
spec:
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
      strategy: rollback          # or: uninstall
```

Per the CRD, `retries` *"is the number of retries that should be attempted on
failures before bailing. Remediation, using an uninstall, is performed between
each attempt. Defaults to '0', a negative integer equals to unlimited retries."*

Two consequences that surprise people:

- **The default is 0 retries**, so a transient failure sticks until something
  triggers a new reconcile.
- **Install remediation uninstalls between attempts.** A failing install of a
  chart with PVCs can repeatedly create and delete resources. Unlimited retries
  (`-1`) on a chart that will never succeed is a resource-churn loop.

`remediateLastFailure` decides whether the final failure is also rolled back or
left in place for inspection. Leaving it false makes debugging much easier.

## Drift detection

```yaml
spec:
  driftDetection:
    mode: enabled        # enabled | warn | disabled
```

Defaults to disabled, so **a `kubectl edit` on a Helm-managed resource persists
indefinitely** and Flux reports Ready. `warn` logs the diff without acting;
`enabled` corrects it. Turning this on for the first time in an established
cluster will surface a lot of accumulated manual changes at once.

## Values

```yaml
spec:
  values:                        # inline, lowest precedence of the two
    replicaCount: 2
  valuesFrom:
    - kind: ConfigMap
      name: app-values
      valuesKey: values.yaml
    - kind: Secret
      name: app-secrets
      targetPath: db.password    # place a single value at a path
```

`valuesFrom` entries merge in order, later overriding earlier, and all of them
override inline `values`. `targetPath` injects one scalar rather than a whole
document, which is how you keep a password out of Git without templating the
whole values file.

Changing a referenced ConfigMap or Secret **does** trigger a HelmRelease upgrade,
unlike a plain pod mount.

## dependsOn

HelmRelease `dependsOn` waits for the other HelmRelease to be Ready. Cross-namespace
references need the `namespace` field. Use it for genuine ordering (an operator's
CRDs before a chart that uses them), not as a general sequencing tool — it
serializes rollouts and slows recovery.
