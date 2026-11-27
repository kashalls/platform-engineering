# AppProjects, RBAC, and tenancy

## AppProject

An `AppProject` constrains what Applications in it may do. `default` is
unrestricted, which makes it fine for a single-team install and wrong for a
shared one.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/org/team-a-*
  destinations:
    - server: https://kubernetes.default.svc
      namespace: team-a-*
  clusterResourceWhitelist: []            # empty = no cluster-scoped resources
  namespaceResourceBlacklist:
    - {group: "", kind: ResourceQuota}
    - {group: "", kind: LimitRange}
  roles:
    - name: deployer
      policies:
        - p, proj:team-a:deployer, applications, sync, team-a/*, allow
```

The three restrictions that do the real work:

- **`sourceRepos`** — which repos may be deployed from. Without this, a tenant can
  point an Application at any repo, including one they control.
- **`destinations`** — which cluster/namespace pairs. Wildcards are allowed.
- **`clusterResourceWhitelist`** — cluster-scoped kinds. An **empty list denies
  all**, which is the safe default; omitting the field entirely is not the same
  thing.

A denied resource surfaces during sync as a message naming the project, e.g.
`resource :Namespace is not permitted in project team-a`. That reads like an RBAC
error but is a project restriction — different fix.

```bash
kubectl get appprojects -n argocd
kubectl get appproject <name> -n argocd -o yaml
```

## RBAC

Policy lives in the `argocd-rbac-cm` ConfigMap, in Casbin format:

```yaml
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:dev, applications, get, */*, allow
    p, role:dev, applications, sync, team-a/*, allow
    p, role:dev, applications, delete, */*, deny
    g, my-org:team-a, role:dev
```

- `p` is a permission: `p, <subject>, <resource>, <action>, <object>, allow|deny`
- `g` is a grouping: `g, <user-or-group>, <role>`
- Object is `<project>/<application>`, so `team-a/*` scopes to a project.
- **Deny always wins** over allow, regardless of order.

Built-in roles are `role:readonly` and `role:admin`. `policy.default` applies to
any authenticated user with no matching rule — set it to `role:readonly` or
`""`, never `role:admin`.

Verify rather than reason about it:

```bash
argocd account can-i sync applications 'team-a/web'
argocd account get-user-info
```

Changes to `argocd-rbac-cm` are picked up without a restart, but a stale session
keeps its old claims until re-login.

## Repository credentials

```bash
argocd repo add https://github.com/org/repo --username x --password <token>
argocd repo add git@github.com:org/repo.git --ssh-private-key-path ~/.ssh/id_ed25519
argocd repo list
```

Credential **templates** match by URL prefix, so one entry covers every repo under
an org:

```bash
argocd repocreds add https://github.com/org --username x --password <token>
```

Repository secrets live in the `argocd` namespace labelled
`argocd.argoproj.io/secret-type: repository`. A repo that "cannot be found"
almost always means the credential's URL prefix does not match the Application's
`repoURL` — including `https://` vs `git@`, and a trailing `.git`.

```bash
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
```

## Clusters

```bash
argocd cluster list
argocd cluster add <kube-context> --name prod
```

`argocd cluster add` creates a ServiceAccount in the target cluster and stores
its credentials as a Secret labelled
`argocd.argoproj.io/secret-type: cluster`. By default it grants cluster-admin;
`--namespace` restricts it to specific namespaces instead, which is what you want
for anything but a lab.

The `destination.server` in an Application must match the registered cluster URL
**character for character**. `https://kubernetes.default.svc` and the in-cluster
name `in-cluster` refer to the same target but are not interchangeable strings.
