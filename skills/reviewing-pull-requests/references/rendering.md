# Rendering a change

When a chart version or a release's values change, render it with this
repository's values and diff the result. Only a failure the change caused
is a finding on the changed line: a value the chart's schema rejects, a
template that errors on this repository's values. A failed input the
change does not touch, or a source that failed to fetch, says nothing
about the change.

## helm template, old against new

Works for any Flux HelmRelease or Argo CD Helm Application. Write the
release's values to a file (from `spec.values` or `spec.source.helm.valuesObject`
at the base and head revisions), then:

```
helm template <release> oci://<registry>/<path>/<chart> --version <old> -n <ns> -f values-old.yaml 2>/dev/null > old.yaml
helm template <release> oci://<registry>/<path>/<chart> --version <new> -n <ns> -f values-new.yaml 2>/dev/null > new.yaml
diff old.yaml new.yaml
```

Expect the `helm.sh/chart` and `app.kubernetes.io/version` labels to move
on every object; the findings are in everything else.

## With a cluster

`flux diff kustomization <name> -n <ns> --path <dir>` builds a local
directory and diffs it against what the cluster runs (see the `flux`
skill's Kustomization reference). For Argo CD, `argocd app diff <app>
--local <dir>` does the same. Both need cluster access and show what this
merge would change on that cluster, which is a stronger check than an
offline render.

## flate

[flate](https://github.com/home-operations/flate) renders Flux HelmReleases
offline from the repository's own Flux objects. These invocations come
from the upstream skill and were not run here:

```
flate diff hr <name> --base origin/main --path <manifests>/<namespace> --no-progress -o diff
flate build hr <name> --path <manifests>/<namespace> --no-progress
flate build hr <name> -n <namespace> --path <manifests> --no-progress
```

The last form renders from the whole tree, for a HelmRelease whose
Kustomization depends on one in another namespace. Narrow a long build
with `--only-kinds <kind>`.

## What a render does not show

A render is an offline approximation of what the cluster applies: CRDs and
Secrets are left out, patches applied from a parent Kustomization or
ApplicationSet usually are not, templates that branch on
`.Capabilities` see the renderer's bundled Kubernetes version rather than
the cluster's, and nothing runs admission (Kyverno, Gatekeeper, OpenShift
SCCs). Within that, take the label values, resource names and ports the
search for dependents needs from the render rather than from a reading of
the template.
