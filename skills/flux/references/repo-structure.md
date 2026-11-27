# Repo structure, environments, and secrets

## Bootstrap

```bash
flux bootstrap github \
  --owner=<org> --repository=<repo> \
  --branch=main --path=clusters/prod \
  --personal=false
```

Bootstrap commits Flux's own manifests to `--path` and installs a
`GitRepository` + `Kustomization` pointing back at it, so Flux manages itself.
Re-running it is the supported upgrade path and is idempotent.

`--path` is per-cluster. Everything under it is what that cluster reconciles.

## Layout that scales

```
clusters/
  prod/
    flux-system/           # bootstrap output, do not hand-edit
    infrastructure.yaml    # Kustomization -> infrastructure/prod
    apps.yaml              # Kustomization -> apps/prod
  staging/
    ...
infrastructure/
  base/
  prod/                    # overlay
  staging/
apps/
  base/
  prod/
  staging/
```

The cluster directory holds only pointers; the actual manifests live in shared
`base/` trees with per-environment overlays. Environments differ by overlay and
by `postBuild.substitute`, not by duplicated YAML.

Layer with `dependsOn`, and give each layer real health gating:

```yaml
# apps.yaml
spec:
  dependsOn:
    - name: infrastructure
  wait: true
```

## Ordering CRDs

The recurring bootstrap problem: a Kustomization applies a CR whose CRD is
installed by another Kustomization, and fails with `no matches for kind`.

Split operators into two Kustomizations — one for the CRDs and controller, one
for the custom resources — and make the second depend on the first with
`wait: true`. Retry-until-success also works because Flux retries forever, but it
produces alert noise on every bootstrap.

## Secrets

Flux does not solve secrets; pick one of these.

**SOPS** — encrypt in Git, decrypt in-cluster:

```yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

The secret must be in the same namespace as the Kustomization. The key name
**must end with `.agekey`** to be detected as an age key — the name itself is not
special, the suffix is:

```bash
cat age.agekey | kubectl create secret generic sops-age \
  --namespace=flux-system --from-file=age.agekey=/dev/stdin
```

Only encrypt `data`/`stringData` values; encrypting metadata breaks
`kustomize build`.

**External Secrets Operator** — keep secrets in a real secret manager and sync:
nothing sensitive in Git at all, at the cost of a runtime dependency on the
provider.

Never put plaintext secrets in `postBuild.substitute` — substitution values are
not encrypted and end up in the Kustomization spec in Git.

## Multi-tenancy

```yaml
spec:
  serviceAccountName: tenant-a      # apply as this SA, not cluster-admin
  targetNamespace: tenant-a
  sourceRef:
    kind: GitRepository
    name: tenant-a
```

`serviceAccountName` is the actual isolation mechanism — without it, the
Kustomization applies with kustomize-controller's own permissions, which are
cluster-admin in a default install. A tenant Kustomization without it can write
anywhere regardless of which repo it came from.

Combine with `--no-cross-namespace-refs=true` on the controllers so a tenant
cannot reference another tenant's sources.

## Notifications

```bash
kubectl get alerts,providers -A
```

An `Alert` selects event sources and routes them to a `Provider`. Alert on
`Kustomization`/`HelmRelease` events at `severity: error` first; `info` includes
every successful reconcile and is far too noisy for a chat channel.

A silent Flux is not a healthy Flux — verify notifications actually deliver, or
the first sign of a broken reconcile will be a user report.

## Image automation

`ImageRepository` scans a registry, `ImagePolicy` selects a version, and
`ImageUpdateAutomation` commits the new tag back to Git. It is off unless you
install the image-reflector and image-automation controllers, which are **not**
part of a default bootstrap:

```bash
flux check 2>&1 | grep image
```

Many teams use Renovate for this instead, keeping version bumps as reviewable
pull requests rather than direct commits.
