# RBAC and admission

## Reading a Forbidden error

```
Error from server (Forbidden): pods is forbidden:
User "system:serviceaccount:app:worker" cannot list resource "pods"
in API group "" in the namespace "prod"
```

Every element matters: the **subject**, the **verb**, the **resource**, the **API
group** (`""` is core), and the **namespace**. Mismatches in API group and in
subresource are the two that hide in plain sight.

## Test permissions instead of reading YAML

```bash
kubectl auth whoami
kubectl auth can-i list pods -n prod
kubectl auth can-i --list -n prod                       # everything you may do here
kubectl auth can-i list pods -n prod \
  --as=system:serviceaccount:app:worker                 # impersonate the SA
kubectl auth can-i '*' '*' --as=system:serviceaccount:app:worker
```

Impersonation with `--as` is the fastest way to answer "can this workload do X"
without deploying anything. It needs impersonate privileges yourself.

## The four objects

| Object | Scope | Grants |
|---|---|---|
| `Role` | namespace | permissions inside one namespace |
| `ClusterRole` | cluster | permissions cluster-wide, or a reusable template |
| `RoleBinding` | namespace | binds a Role **or a ClusterRole** into one namespace |
| `ClusterRoleBinding` | cluster | binds a ClusterRole everywhere |

A `RoleBinding` referencing a `ClusterRole` grants that ClusterRole's permissions
**only within the RoleBinding's namespace**. That is the idiomatic way to reuse
built-in roles like `view` or `edit` per namespace.

There is no deny. RBAC is purely additive, so permissions are the union of every
binding for a subject. You cannot subtract with a rule — you remove the binding.

```bash
kubectl get rolebindings,clusterrolebindings -A -o json | jq -r '
  .items[] | select(.subjects[]? | .name=="<sa-name>") |
  "\(.kind)/\(.metadata.name) -> \(.roleRef.kind)/\(.roleRef.name)"'
```

## Traps

- **Subresources are separate resources.** `pods/log`, `pods/exec`,
  `pods/portforward` must be granted by name. `get pods` does not imply logs.
- **`roleRef` is immutable.** Changing which Role a binding points to requires
  delete and recreate.
- **The API group is part of the match.** A rule for `deployments` in `""` (core)
  grants nothing; Deployments live in `apps`. Get the group from
  `kubectl api-resources`.
- **Wildcards are evaluated at request time**, so they quietly cover CRDs
  installed after the Role was written. `*` on `*` is a cluster-admin equivalent.
- **`escalate` and `bind` guard privilege escalation.** Without them you cannot
  create a Role granting more than you already hold. This is a feature.
- **Every pod gets a ServiceAccount token by default.** Set
  `automountServiceAccountToken: false` on workloads that never call the API.

## Admission webhooks

Admission runs after authn/authz and before persistence: mutating webhooks first,
then validating.

```bash
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations
kubectl get validatingwebhookconfiguration <name> -o jsonpath='{.webhooks[*].failurePolicy}'
```

| Symptom | Cause |
|---|---|
| `failed calling webhook ... connection refused` | The webhook's backing pods are down. With `failurePolicy: Fail` this blocks all matching writes. |
| `context deadline exceeded` | Webhook slower than `timeoutSeconds` (max 30). |
| `x509: certificate signed by unknown authority` | `caBundle` is stale — usually a cert-manager `ca-injector` annotation that did not fire. |
| Object silently changed after apply | A mutating webhook rewrote it. Diff against `kubectl get -o yaml`. |

**A webhook with `failurePolicy: Fail` and no running backend can lock the
cluster out of creating the very pods that would fix it.** This is the classic
self-inflicted outage: a policy controller crashes, its webhook rejects every
write, and its own Deployment cannot roll out. Recovery is deleting the webhook
configuration, then restoring it once the controller is healthy.

```bash
kubectl delete validatingwebhookconfiguration <name>   # break glass
```

Check `namespaceSelector` and `objectSelector` before assuming a webhook applies —
most well-built policies exclude `kube-system` for exactly this reason.

## Health of the control plane

```bash
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez?verbose'
kubectl get apiservices | grep -v True     # broken aggregated APIs
```

A failing aggregated APIService (metrics-server, a custom API) makes
`kubectl get all` and some controllers fail in confusing, unrelated ways.
