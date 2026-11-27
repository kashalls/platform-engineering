# BuildConfigs, ImageStreams, and the internal registry

Optional machinery. Many clusters build elsewhere and only consume external
images — in which case skip this. It matters when an image "updates itself"
without a manifest change, or a deploy uses an unexpected version.

## ImageStreams

An `ImageStream` is a named set of tags that resolve to image digests. It is an
indirection layer between your manifests and a registry.

```bash
oc get imagestreams -n <ns>
oc describe is <name> -n <ns>
oc get istag <name>:<tag> -n <ns> -o jsonpath='{.image.dockerImageReference}'
oc import-image <name>:<tag> --from=<registry>/<repo>:<tag> --confirm
```

`oc describe is` shows each tag, what it currently points to, and the history of
what it pointed to before. That history answers "which image was actually running
last Tuesday" when nothing in Git changed.

Two behaviors that surprise people:

- **`scheduled: true` on a tag re-imports periodically**, so the digest behind a
  tag changes with no action from you. Combined with a triggered Deployment, the
  cluster deploys new images on its own.
- **`--reference-policy=local`** makes pods pull through the internal registry
  rather than the upstream one, which changes the failure mode when the internal
  registry is down.

## Image triggers

```yaml
metadata:
  annotations:
    image.openshift.io/triggers: |
      [{"from":{"kind":"ImageStreamTag","name":"web:latest"},
        "fieldPath":"spec.template.spec.containers[?(@.name==\"web\")].image"}]
```

This rewrites the container image field **in the live object** when the tag
moves. So the running Deployment's image differs from what is in Git, and a
GitOps tool will see permanent drift.

If you use Argo CD or Flux, either do not use image triggers, or add the image
field to `ignoreDifferences`. Running both is a fight neither wins.

```bash
# What is actually deployed vs what Git says.
oc get deploy <name> -o jsonpath='{.spec.template.spec.containers[*].image}'
```

## BuildConfigs

```bash
oc get bc -n <ns>
oc start-build <bc> --follow
oc start-build <bc> --from-dir=. --follow      # build local source
oc logs -f bc/<bc>
oc get builds -n <ns>
oc describe build <build> -n <ns>
```

Build strategies: `Docker` (a Containerfile), `Source` (S2I), `Custom`, and
`Pipeline` (deprecated in favor of OpenShift Pipelines/Tekton).

| Failure | Cause |
|---|---|
| `Failed (FetchSourceFailed)` | Git credentials or an unreachable repo. The build's source secret is namespaced. |
| `Failed (PushImageToRegistryFailed)` | The builder ServiceAccount lacks push rights, or the registry is unavailable. |
| `Failed (OutOfMemoryKilled)` | Build pod hit its limit; set `resources` on the BuildConfig. |
| `Failed (DockerBuildFailed)` | The build itself — read `oc logs`. |
| Build never starts | No builder ServiceAccount, or quota exhausted. |

Builds run as pods, so all normal pod debugging applies:

```bash
oc get pods -n <ns> -l openshift.io/build.name=<build>
```

Builds also obey SCCs. A Containerfile needing root at build time can fail on
clusters with tightened build defaults.

## The internal registry

```bash
oc get configs.imageregistry.operator.openshift.io/cluster -o yaml
oc get pods -n openshift-image-registry
```

Notable defaults:

- **On some install profiles the registry starts `Removed`** with no storage
  configured. Builds that push to it then fail until storage is set. This is
  intentional — bare-metal and some cloud installs will not guess a backend.
- **It is not exposed externally by default.** Pushing from outside needs a route
  and `oc registry login`.

```bash
oc registry login
oc registry info
```

Internal pulls use `image-registry.openshift-image-registry.svc:5000/<ns>/<is>:<tag>`.
That hostname only resolves inside the cluster.

## Deciding whether to use any of this

Reasonable default for a new platform: build images in CI, push to an external
registry, reference them by **digest** in Git, and let a GitOps controller
deploy. That gives reproducibility and one source of truth.

Reach for ImageStreams and BuildConfigs when you specifically want in-cluster
builds, S2I, or the tag-history audit trail — and then accept that image triggers
and GitOps drift detection do not mix.
