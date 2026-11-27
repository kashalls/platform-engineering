---
name: kyverno
description: Write, test, and debug Kyverno policies. Use when a policy is not blocking what it should, is blocking something it should not, when a mutation or generation rule does not fire, when PolicyReports show unexpected failures, when workloads are rejected by admission and you need to find which policy did it, or when designing policy rollout with Audit/Enforce, exceptions, and CI testing.
---

# Kyverno

Kyverno is a Kubernetes-native policy engine. Policies are YAML, matched against
admission requests and (in background scans) against existing resources.

Verified against Kyverno **v1.19.0**. Kyverno's API surface has grown: check what
your cluster serves before copying examples.

```bash
kubectl get crds | grep kyverno
kubectl get pods -n kyverno
```

## Two generations of policy API

| API group | Kinds | Notes |
|---|---|---|
| `kyverno.io/v1` | `ClusterPolicy`, `Policy` | The classic engine — `validate`/`mutate`/`generate` with patterns and JMESPath. Still the most widely used. |
| `policies.kyverno.io/v1beta1` | `ValidatingPolicy`, `MutatingPolicy`, `GeneratingPolicy`, `ImageValidatingPolicy`, `DeletingPolicy` (+ `Namespaced*` variants) | Newer CEL-based types, aligned with upstream ValidatingAdmissionPolicy. |
| `kyverno.io/v2` | `PolicyException`, `CleanupPolicy`, `ClusterCleanupPolicy` | Exceptions and TTL cleanup. |
| `wgpolicyk8s.io/v1alpha2` | `PolicyReport`, `ClusterPolicyReport` | Results, including background scans. |

`ClusterPolicy` is cluster-scoped; `Policy` is namespaced and only matches
resources in its own namespace. Examples online almost always use `ClusterPolicy`
— using `Policy` by mistake produces a policy that silently matches nothing
outside its namespace.

## Which policy blocked my deploy

The admission error names the policy and the rule:

```
Error from server: admission webhook "validate.kyverno.svc-fail" denied the
request: resource Pod/default/web was blocked due to the following policies

require-team-label:
  check-team-label: 'validation error: The label ''team'' is required.'
```

Then:

```bash
kubectl get clusterpolicy <name> -o yaml
kubectl get policyreport -A                       # namespaced results
kubectl get clusterpolicyreport                   # cluster-scoped results
kubectl logs -n kyverno deploy/kyverno-admission-controller --tail=100
```

```bash
# Every failing result, across all namespaces.
kubectl get policyreport -A -o json | jq -r '.items[].results[]?
  | select(.result=="fail")
  | "\(.policy)/\(.rule)\t\(.resources[]?.namespace)/\(.resources[]?.name)\t\(.message)"'
```

## Test policies locally — do this before applying anything

The CLI runs the real engine offline. This is the fastest loop and works in CI.

```bash
kyverno apply policy.yaml --resource pod.yaml
kyverno apply policy.yaml --resource pod.yaml --resource deployment.yaml
kyverno apply ./policies/ --cluster                 # against live cluster resources
```

Real output:

```
Applying 3 policy rule(s) to 2 resource(s)...
policy require-team-label -> resource default/Pod/bad failed:
1 - check-team-label validation error: The label 'team' is required.
    rule check-team-label failed at path /metadata/labels/

pass: 1, fail: 1, warn: 0, error: 0, skip: 0
```

The `failed at path` fragment is the most useful part — it points at the exact
field the pattern rejected.

For CI, use declarative test files that assert expected outcomes:

```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: require-team-label-test
policies: [policy.yaml]
resources: [good.yaml, bad.yaml]
results:
  - policy: require-team-label
    rule: check-team-label
    kind: Pod
    resources: [good]
    result: pass
  - policy: require-team-label
    rule: check-team-label
    kind: Pod
    resources: [bad]
    result: fail
```

```bash
kyverno test .              # walks the directory for kyverno-test.yaml files
kyverno test <git-url>
```

A test "passes" when the actual result matches the declared `result`, so a
`result: fail` expectation passing means the policy correctly rejected the
resource.

## Autogen: the biggest surprise

A rule matching only `Pod` **automatically generates additional rules** for
Deployment, DaemonSet, StatefulSet, Job, CronJob, and ReplicaSet. That is why one
authored rule reports as three or more.

Verified: applying a Pod-only rule to a Deployment produces

```
1 - autogen-check-team-label validation error: The label 'team' is required.
    rule autogen-check-team-label failed at path /spec/template/metadata/labels/team/
```

Consequences:

- **Do not also write your own Deployment rule** — you will get duplicate,
  possibly conflicting, enforcement.
- The generated rule checks `/spec/template/...`, so error paths differ from the
  Pod rule's. Do not be confused by a path you never wrote.
- Autogen is controlled by the `pod-policies.kyverno.io/autogen-controllers`
  annotation; set it to `none` to disable, or list specific controllers.

```yaml
metadata:
  annotations:
    pod-policies.kyverno.io/autogen-controllers: none
```

## Roll out in Audit first

```yaml
spec:
  rules:
    - name: check-team-label
      validate:
        failureAction: Audit        # Audit | Enforce — per rule
```

`Audit` records results in PolicyReports without blocking. Always start there,
read the reports, then flip to `Enforce`. Going straight to `Enforce` on an
existing cluster blocks deploys nobody expected to be blocked.

**The field moved, and the old one is deprecated.** The v1.19 CRD describes
`spec.validationFailureAction` as *"Deprecated, use validationFailureAction under
the validate rule instead."* The current location is per-rule:
`spec.rules[].validate.failureAction`. The deprecated spec-level field still
works, so a policy carrying both is ambiguous to readers — set only the new one.

```bash
kubectl get clusterpolicy -o custom-columns='NAME:.metadata.name,LEGACY_ACTION:.spec.validationFailureAction,BACKGROUND:.spec.background,READY:.status.ready'
kubectl get clusterpolicy -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.rules[*].validate.failureAction}{"\n"}{end}'
```

## Gotchas

- **`status.ready: false` means the policy is not active.** A policy with a bad
  JMESPath or an unresolvable context entry loads but never enforces.
- **Kyverno cannot block what it does not see.** Its webhooks are generated from
  the policies installed; a `resourceFilters` exclusion in the Kyverno config
  (which excludes `kube-system` and Kyverno's own namespace by default) means
  policies never apply there.
- **`failurePolicy: Fail` plus a down Kyverno blocks the cluster.** This is the
  standard self-inflicted outage. Know how to delete the webhook configurations
  as break-glass.
- **Background scans only run when `background: true`**, and background mode
  cannot use admission-only context like `request.userInfo` — a rule referencing
  it will error in background scans.
- **`generate` rules need Kyverno to have permission** on the target kind.
  Kyverno's ClusterRole must be extended for anything beyond the defaults, and
  the failure appears as a `UpdateRequest` error, not an admission error.
- **Mutations apply in admission order**, and a later mutating webhook from
  another controller can overwrite Kyverno's change.
- **Exceptions are objects, not annotations.** `PolicyException` must be enabled
  in the Kyverno config and is often restricted to specific namespaces.

## References

| File | Covers |
|---|---|
| [references/writing-policies.md](references/writing-policies.md) | validate, mutate, generate, matching, context, JMESPath |
| [references/operations.md](references/operations.md) | Rollout, exceptions, reports, performance, break-glass |
