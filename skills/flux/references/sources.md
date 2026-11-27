# Sources: GitRepository, OCIRepository, HelmRepository, Bucket

Sources fetch and store an artifact. Everything downstream consumes that
artifact, so a stale or failed source poisons every Kustomization under it while
producing errors that look like Kustomization problems.

```bash
flux get sources all -A
kubectl get gitrepository <name> -n <ns> -o jsonpath='{.status.conditions}' | jq
kubectl logs -n flux-system deploy/source-controller --tail=100
```

`status.artifact.revision` is the ground truth for "what Flux actually has":

```bash
kubectl get gitrepository <name> -n <ns> -o jsonpath='{.status.artifact.revision}'
```

Compare it to `git rev-parse HEAD` on the branch. If they differ, the problem is
here, not downstream.

## Failure messages

| Message | Cause |
|---|---|
| `authentication required` / `couldn't find remote ref` | Bad or missing credentials, or the ref does not exist. |
| `unknown revision` | The branch/tag in `spec.ref` is gone or misspelled. |
| `host key verification failed` | `known_hosts` missing from the SSH secret. |
| `x509: certificate signed by unknown authority` | Self-hosted Git with a private CA; needs `caFile` in the secret. |
| `no such host` | Cluster DNS or egress, not Git. |
| `artifact revision ... does not match` | Verification (cosign/GPG) failed. |
| `context deadline exceeded` | `spec.timeout` too low, or a very large repo. |

## Git authentication

SSH secrets need exactly three keys, and `known_hosts` is the one people forget:

```bash
flux create secret git <name> \
  --url=ssh://git@github.com/org/repo \
  --ssh-key-algorithm=ecdsa \
  --namespace=flux-system
```

That command generates the keypair and populates `identity`, `identity.pub`, and
`known_hosts`. Hand-built secrets that omit `known_hosts` fail with `host key
verification failed`, which reads like a network error but is not.

For HTTPS, the keys are `username` and `password` (a PAT, not a password, for
most providers).

The secret must live in **the same namespace as the source object**. This is the
most common cross-namespace mistake in Flux.

## OCIRepository

OCI artifacts are increasingly the default for both manifests and charts. The
`REVISION` is `<tag>@<digest>`, and a digest that never changes while the tag
moves means the registry is serving a cached manifest.

```bash
kubectl get ocirepository <name> -n <ns> -o jsonpath='{.status.artifact.revision}'
```

Pin by digest for immutability, or use `spec.ref.semver` to track a range:

```yaml
spec:
  ref:
    semver: ">=1.0.0 <2.0.0"   # or: tag / digest
```

Verify signatures rather than trusting the tag:

```yaml
spec:
  verify:
    provider: cosign
    secretRef:
      name: cosign-pub
```

With keyless cosign, omit `secretRef` and set `matchOIDCIdentity`. A verification
failure surfaces as the source being not-Ready, which correctly blocks the
rollout — that is the point.

## Intervals and rate limits

`spec.interval` on a source is a polling floor. Short intervals across many
sources will hit provider rate limits (notably GitHub and Docker Hub), which
appear as intermittent `toomanyrequests` or 429s in source-controller logs.

For push-driven updates, use a `Receiver` with a webhook instead of a short
interval:

```bash
kubectl get receivers -A
kubectl get receiver <name> -n <ns> -o jsonpath='{.status.webhookPath}'
```

That gives near-instant reconciliation while leaving the interval as a slow
safety net.
