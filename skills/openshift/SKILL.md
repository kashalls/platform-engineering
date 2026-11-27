---
name: openshift
description: Troubleshoot and build on OpenShift (OCP/OKD). Use when a pod is rejected by a SecurityContextConstraint, when a Route does not serve traffic or TLS is wrong, when a ClusterOperator is Degraded or an upgrade is stuck, when Operator Lifecycle Manager subscriptions fail to install, when BuildConfigs or ImageStreams misbehave, or when porting vanilla Kubernetes manifests to OpenShift and they will not start.
---

# OpenShift

OpenShift is Kubernetes plus opinionated defaults. Almost everything that works
on vanilla Kubernetes works here, and the exceptions cluster in four places:

1. **SCCs** reject pods that vanilla Kubernetes would run.
2. **Routes** exist alongside (and predate) Ingress.
3. **Cluster Operators** own the control plane; you do not edit their output.
4. **ImageStreams / BuildConfigs** add an image indirection layer.

`oc` is a superset of `kubectl` — every `kubectl` command works with `oc`. Use
`oc` for the OpenShift-specific verbs below. Commands here are verified against
`oc` 4.22.

## Ground the cluster

```bash
oc version
oc whoami
oc whoami --show-console
oc get clusterversion                      # the cluster's own version + upgrade state
oc get clusteroperators                    # anything not Available=True is your problem
```

`oc get co` (ClusterOperators) is the OpenShift equivalent of "is the control
plane healthy". Any operator with `AVAILABLE=False`, `DEGRADED=True`, or stuck
`PROGRESSING=True` explains a wide range of otherwise-baffling symptoms.

```bash
oc get co -o json | jq -r '.items[] | select(.status.conditions[] |
  select(.type=="Degraded" and .status=="True")) | .metadata.name'
```

## Symptom → route

```
Pod will not start, works on vanilla k8s?  → SCC       → references/scc.md
Route returns nothing / TLS wrong?          → Routes    → references/routes-networking.md
ClusterOperator Degraded / upgrade stuck?   → CVO/OLM   → references/operators-and-upgrades.md
Operator install stuck in Pending?          → OLM       → references/operators-and-upgrades.md
Build fails / image not updating?           → Builds    → references/builds-and-images.md
Porting manifests from vanilla k8s?         → SCC first → references/scc.md
```

## The single most common OpenShift problem

A Deployment that runs fine on vanilla Kubernetes fails here with:

```
pods "web-xxxx" is forbidden: unable to validate against any security context constraint
```

By default OpenShift assigns the `restricted-v2` SCC, which **drops all
capabilities (only `NET_BIND_SERVICE` may be added back), forbids running as
root, and forces a random UID** from the
namespace's allocated range. Images with a hardcoded `USER 0`, or that write to
a fixed path owned by a specific UID, fail.

Diagnose without redeploying:

```bash
oc adm policy scc-subject-review -z <serviceaccount> -f deployment.yaml
oc get pod <pod> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

The first says which SCCs *would* admit the pod; the second says which one
actually admitted a running pod. Full treatment in
[references/scc.md](references/scc.md) — including why granting `anyuid` is
usually the wrong fix.

## Projects, not just namespaces

A `Project` is a Namespace with extra metadata and its own creation workflow.

```bash
oc new-project <name>
oc project <name>              # switch, like kubens
oc get projects                # only those you can see
```

`oc get projects` filters by what you may access, while `kubectl get namespaces`
requires cluster-level list permission and fails for ordinary users. That
difference confuses people migrating in.

Deleting a Project deletes the Namespace and everything in it.

## Useful oc verbs with no kubectl equivalent

```bash
oc status                                   # topology of the current project
oc adm policy who-can <verb> <resource>     # reverse RBAC lookup
oc adm top pods                             # like kubectl top
oc rsh <pod>                                # exec a shell, picks a sane default
oc debug deployment/<name> --as-root        # debug pod from a workload's template
oc debug node/<node>                        # host shell, chroot /host
oc adm must-gather                          # full diagnostic bundle for support
oc extract secret/<name> --to=- --keys=tls.crt
oc logs -f bc/<buildconfig>                 # follow a build
```

`oc debug` builds a *copy* of the workload's pod template with probes removed and
the command replaced by a shell. It is the right way to inspect a pod that is
crashlooping too fast to `exec` into.

`oc adm must-gather` writes a large directory of cluster state. It is what
support asks for, and it is also a good local artifact for a postmortem.

## Gotchas

- **`restricted-v2` replaced `restricted`** as the default SCC in OCP 4.11+.
  Manifests and blog posts referencing `restricted` may not describe your cluster.
- **SCC selection is by priority, then by most-restrictive-that-fits** — not
  "the most permissive one you have". Granting an extra SCC does not guarantee
  it will be used. Check the `openshift.io/scc` annotation to see the winner.
- **Never edit a resource owned by a ClusterOperator.** The CVO reverts it,
  usually within minutes, and your change vanishes with no event explaining why.
  Use the operator's own config CR instead.
- **A Route without `host` gets a generated one.** The name is derived from the
  Route and namespace, so recreating the Route elsewhere, or renaming it,
  changes the host and breaks DNS pointed at the old one. Set `host` explicitly
  for anything external.
- **ImageStream tags can be resolved at deploy time**, so a Deployment can change
  image without any manifest change. Surprising when you are chasing "where did
  this version come from".
- **The internal registry may not be exposed by default.** Pushing to it from
  outside requires `oc registry login` and an exposed route.
- **Bare pods created as cluster-admin can use any SCC.** A pod you create
  directly is admitted against *your* permissions, so it may run where the
  workload will not. Pods created by a Deployment or other controller are
  admitted against the pod's ServiceAccount; test with `scc-subject-review -z`.

## References

| File | Covers |
|---|---|
| [references/scc.md](references/scc.md) | SecurityContextConstraints, UID ranges, porting manifests |
| [references/routes-networking.md](references/routes-networking.md) | Routes vs Ingress, TLS termination, the router |
| [references/operators-and-upgrades.md](references/operators-and-upgrades.md) | ClusterOperators, CVO, OLM, upgrade troubleshooting |
| [references/builds-and-images.md](references/builds-and-images.md) | BuildConfigs, ImageStreams, the internal registry |
