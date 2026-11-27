# Writing Kyverno policies

Examples use `kyverno.io/v1` `ClusterPolicy`, verified with CLI v1.19.0.

## Matching

```yaml
rules:
  - name: check-something
    match:
      any:                          # OR across entries
        - resources:
            kinds: [Pod, Deployment]
            namespaces: ["prod-*"]
            names: ["web-*"]
            selector:
              matchLabels: {tier: frontend}
    exclude:
      any:
        - resources:
            namespaces: [kube-system]
        - subjects:
            - kind: ServiceAccount
              name: cluster-admin-sa
```

- `any` is OR, `all` is AND. Conditions **inside one entry** are ANDed.
- `exclude` always wins over `match`.
- Kinds may be qualified (`apps/v1/Deployment`) when a bare kind is ambiguous.
- `namespaces` accepts wildcards; `namespaceSelector` matches by label, which is
  usually more maintainable than a name pattern.

Excluding by `subjects` is how you let a specific controller bypass a rule
without disabling it for everyone.

## validate

**Pattern** — declarative shape matching, the most readable option:

```yaml
validate:
  failureAction: Enforce
  message: "Resource limits are required."
  pattern:
    spec:
      containers:
        - resources:
            limits:
              memory: "?*"          # ?* = any non-empty value
```

Pattern operators: `*` (zero or more chars), `?` (one char), `?*` (non-empty),
`|` (OR), `>`/`<`/`>=`/`<=`/`!=` for numbers and quantities, and `+`/`-` prefixes
on list elements for "must exist" / "must not exist".

A list in a pattern means **every element must match**. To require only one
element to match, use `anyPattern`.

**Deny with conditions** — for logic patterns cannot express:

```yaml
validate:
  failureAction: Enforce
  message: "Only approved registries are allowed."
  deny:
    conditions:
      any:
        - key: "{{ request.object.spec.containers[].image }}"
          operator: AnyNotIn
          value: ["registry.example.com/*"]
```

Operators include `Equals`, `NotEquals`, `In`, `NotIn`, `AnyIn`, `AllIn`,
`AnyNotIn`, `AllNotIn`, `GreaterThan`, `DurationGreaterThan`. Note the inverted
logic: `deny` blocks when the conditions are **true**.

**CEL** — for complex expressions, and the direction the project is heading:

```yaml
validate:
  cel:
    expressions:
      - expression: "object.spec.replicas >= 2"
        message: "At least two replicas are required."
```

## mutate

```yaml
mutate:
  patchStrategicMerge:
    metadata:
      labels:
        managed-by: platform
    spec:
      containers:
        - (name): "*"                 # conditional anchor: for every container
          imagePullPolicy: IfNotPresent
```

Anchors are the part worth learning:

| Anchor | Meaning |
|---|---|
| `(key)` | **Conditional** — apply the rest only where this matches |
| `+(key)` | **Add** — set only if absent, never overwrite |
| `<(key)` | **Global** — condition evaluated against the whole resource |
| `^(key)` | **Add-if-not-present** for list elements |
| `X(key)` | **Negation** — apply only if the key is absent |

`+(key)` is what you want for defaults, so a user's explicit value survives.

RFC 6902 JSON patches are also supported and are better for list surgery:

```yaml
mutate:
  patchesJson6902: |-
    - path: "/spec/template/spec/tolerations/-"
      op: add
      value: {key: "dedicated", operator: "Exists", effect: "NoSchedule"}
```

Mutations apply at admission. Existing resources are unaffected unless you use
`mutateExistingOnPolicyUpdate: true` with a `targets` block.

## generate

Creates or syncs resources in response to another resource appearing — the
canonical use is default NetworkPolicies or pull secrets in every new namespace.

```yaml
generate:
  apiVersion: v1
  kind: Secret
  name: registry-credentials
  namespace: "{{request.object.metadata.name}}"
  synchronize: true
  clone:
    namespace: platform
    name: registry-credentials
```

- `synchronize: true` keeps the copy updated and **restores it if deleted**.
  Without it, generation is one-shot.
- `clone` copies an existing object; `data` defines one inline.
- **Kyverno's ServiceAccount needs RBAC on the generated kind.** This is the
  usual reason a generate rule silently does nothing — the failure appears on an
  `UpdateRequest`, not in the admission response.

```bash
kubectl get updaterequests -n kyverno
kubectl logs -n kyverno deploy/kyverno-background-controller --tail=100
```

## Context and external data

```yaml
context:
  - name: cm
    configMap: {name: policy-config, namespace: platform}
  - name: allowedRegistries
    apiCall:
      urlPath: "/api/v1/namespaces/{{request.namespace}}"
      jmesPath: "metadata.annotations.\"policy/registries\""
```

Referenced as `{{ cm.data.key }}`. Context resolution failures make the whole
rule error rather than fail open or closed predictably — check `status.ready` on
the policy.

Variables available: `{{request.object}}`, `{{request.oldObject}}`,
`{{request.operation}}` (`CREATE`/`UPDATE`/`DELETE`), `{{request.userInfo}}`,
`{{serviceAccountName}}`, `{{images}}`.

**`request.userInfo` is admission-only.** A rule using it cannot run in
background scans; set `background: false` on such policies or they will error.

Test JMESPath expressions without a cluster:

```bash
kyverno jp query -i resource.yaml 'spec.containers[].image'
```

## Preconditions

Cheaper than `match` for value-based gating, and evaluated first:

```yaml
preconditions:
  all:
    - key: "{{request.operation}}"
      operator: NotEquals
      value: DELETE
```

Guarding on `request.operation` is worth doing habitually — a validate rule that
also fires on DELETE can prevent cleanup of resources that no longer comply.
