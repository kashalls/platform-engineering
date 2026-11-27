# Applications, app-of-apps, and ApplicationSets

## Application spec

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web
  namespace: argocd                 # must be where Argo watches
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # cascade delete
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo
    targetRevision: main
    path: apps/web
  destination:
    server: https://kubernetes.default.svc    # or: name: in-cluster
    namespace: web
  syncPolicy:
    automated: {prune: true, selfHeal: true}
    syncOptions: [CreateNamespace=true]
```

Points that bite:

- **The Application must live in the namespace Argo watches** (`argocd` by
  default). An Application elsewhere is simply ignored, with no error anywhere.
  Enabling "apps in any namespace" requires explicit configuration.
- **`destination.server` and `destination.name` are alternatives.** Setting both
  is an error; the server URL must match a registered cluster **exactly**,
  including trailing slash.
- **The `resources-finalizer` is what makes deletion cascade.** With it, deleting
  the Application deletes everything it manages. Without it, resources are
  orphaned. Neither is wrong — but know which you have before deleting.
- **`targetRevision: HEAD`** tracks the default branch. Pin to a tag or commit
  for anything you want reproducible.

## Sources: Helm and Kustomize

```yaml
  source:
    repoURL: https://charts.example.com
    chart: web
    targetRevision: 1.2.3
    helm:
      releaseName: web
      valueFiles: [values-prod.yaml]
      values: |
        replicaCount: 3
      parameters:
        - {name: image.tag, value: v1.2.3}
      ignoreMissingValueFiles: false
```

Argo renders Helm with `helm template` — it does **not** create a Helm release.
`helm list` shows nothing, hooks behave as Argo hooks rather than Helm hooks, and
chart logic depending on `.Release.IsUpgrade` behaves as a fresh install every
time.

`valueFiles` paths are relative to the chart root. A values file living elsewhere
in the repo needs multi-source (below) or `$values` referencing.

```yaml
  source:
    kustomize:
      images: ["web=registry/web:v1.2.3"]
      namePrefix: prod-
      commonLabels: {env: prod}
```

## Multi-source applications

```yaml
spec:
  sources:
    - repoURL: https://charts.example.com
      chart: web
      targetRevision: 1.2.3
      helm:
        valueFiles: ["$values/envs/prod/values.yaml"]
    - repoURL: https://github.com/org/config
      targetRevision: main
      ref: values                 # names this source for $values
```

This is the standard way to keep an upstream chart and your values in separate
repos. The `ref` source contributes no manifests of its own — it exists only to
be referenced.

## App-of-apps

A parent Application whose manifests are other Applications.

```yaml
spec:
  source:
    path: apps           # a directory of Application YAMLs
    directory:
      recurse: true
```

Behavior worth knowing before adopting it:

- Deleting the parent **cascades to every child** and everything they manage.
- Child sync status does not roll up into the parent's health by default; the
  parent can be `Synced`/`Healthy` while children are broken.
- Ordering between children needs sync waves on the child Applications, **and**
  a health check for the `Application` kind. Since v1.8 Argo CD does not assess
  Application health by default, so without the custom health check in
  `argocd-cm` the parent moves to the next wave without waiting for the child.

## ApplicationSets

Generate Applications from a template. The controller is a separate Deployment
(`argocd-applicationset-controller`).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git:
        repoURL: https://github.com/org/repo
        revision: main
        directories:
          - path: apps/*
  template:
    metadata:
      name: '{{.path.basename}}'
    spec:
      project: default
      source: {repoURL: ..., path: '{{.path.path}}', targetRevision: main}
      destination: {server: https://kubernetes.default.svc, namespace: '{{.path.basename}}'}
```

Generators: `list`, `cluster`, `git` (directories or files), `matrix`, `merge`,
`scmProvider`, `pullRequest`, `clusterDecisionResource`.

```yaml
  syncPolicy:
    applicationsSync: create-only    # or: create-update, create-delete, sync
    preserveResourcesOnDeletion: true
```

**A generator that suddenly returns fewer items deletes the corresponding
Applications**, and with cascade finalizers that deletes their workloads. An
empty or unreachable generator source is the dangerous case. `create-only` or
`preserveResourcesOnDeletion: true` blunt this; test generator changes with
`--dry-run` first.

Template variable syntax depends on `goTemplate`: without it, the legacy
syntax is `{{path.basename}}`; with `goTemplate: true`, it is Go-template
`{{.path.basename}}`. Examples online mix the two; check `goTemplate` before
copying one. `missingkey=error` makes a reference to a field the generator did
not return fail instead of rendering as `<no value>`.

## Diagnosing a generated Application

```bash
kubectl get applicationset <name> -n argocd -o jsonpath='{.status.conditions}' | jq
kubectl logs -n argocd deploy/argocd-applicationset-controller --tail=100
```

If a generated Application is missing, the problem is the generator, not the
template — check what the generator actually returned before editing the template.
