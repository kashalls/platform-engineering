---
name: kubernetes
description: Troubleshoot and build on Kubernetes. Use when a pod is Pending, CrashLoopBackOff, OOMKilled, ImagePullBackOff, or Terminating; when a Service, DNS lookup, Ingress, or NetworkPolicy is not routing; when a PVC will not bind; when RBAC denies an action; when a node is NotReady or under pressure; or when writing/reviewing manifests, probes, resources, and scheduling constraints. Covers kubectl triage, jsonpath queries, ephemeral debug containers, and admission failures.
---

# Kubernetes

Triage-first reference. Start from the **symptom**, not from the resource type.

Your knowledge of API versions, field names, and defaults may be outdated, and clusters
vary widely by version and distro. **Verify against the live cluster before asserting.**

## Ground the cluster first

Never diagnose against assumptions. Two commands, always:

```bash
kubectl version -o json | jq -r '.serverVersion.gitVersion'   # server version, not client
kubectl auth whoami                                            # who you are, before blaming RBAC
```

Then confirm a field actually exists on this server rather than trusting memory:

```bash
kubectl explain deployment.spec.template.spec.topologySpreadConstraints --recursive
kubectl api-resources | grep -i <kind>
```

## Symptom → route

```
Pod not running?
├─ Pending              → no node fits: capacity, taints, affinity, PVC → references/scheduling.md
├─ ContainerCreating    → stuck mounting or CNI → references/storage.md, references/networking.md
├─ CrashLoopBackOff     → container starts then exits → references/pod-failures.md
├─ OOMKilled            → memory limit too low, or a leak → references/pod-failures.md
├─ ImagePullBackOff     → registry auth, tag, or rate limit → references/pod-failures.md
├─ Init:*               → an initContainer is failing → references/pod-failures.md
├─ Terminating (stuck)  → finalizer or preStop hook → references/pod-failures.md
└─ Running, not Ready   → readiness probe failing → references/pod-failures.md

Pod runs but traffic fails?
├─ Service has no endpoints  → selector/labels mismatch  → references/networking.md
├─ DNS does not resolve      → CoreDNS, ndots, search    → references/networking.md
├─ Ingress 404/502/503       → class, backend, TLS       → references/networking.md
└─ Works only sometimes      → NetworkPolicy, MTU, probe → references/networking.md

Cluster-level?
├─ Node NotReady / pressure  → references/nodes-and-capacity.md
├─ "forbidden" / "cannot"    → references/rbac-and-admission.md
└─ Webhook denied / timeout  → references/rbac-and-admission.md
```

## The first four commands

Run these before anything else. Most incidents resolve here.

```bash
# 1. What is actually not healthy, cluster-wide.
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# 2. What the control plane is complaining about, newest last.
kubectl events -A --types=Warning

# 3. The object's own story: events are at the bottom of describe.
kubectl describe pod <pod> -n <ns>

# 4. Why it died last time, which is not the same as why it is failing now.
kubectl logs <pod> -n <ns> --previous --tail=100
```

`--previous` is the one people forget. A CrashLoopBackOff pod's current logs are
usually empty because the container has not started yet; the evidence is in the
prior instance.

## High-signal queries

Verified against a v1.36 server. Each returns one line per object, greppable.

```bash
# Everything that has restarted, worst first.
kubectl get pods -A --sort-by=.status.containerStatuses[0].restartCount \
  -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,RESTARTS:.status.containerStatuses[*].restartCount'

# Why each container last terminated (OOMKilled vs Error vs Completed).
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}{end}' \
  | awk -F'\t' '$3!=""'

# Node health at a glance, without reading describe.
kubectl get nodes -o custom-columns='NODE:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,MEM:.status.conditions[?(@.type=="MemoryPressure")].status,DISK:.status.conditions[?(@.type=="DiskPressure")].status'

# How full a node really is (requests, not usage).
kubectl describe node <node> | grep -A8 'Allocated resources'

# Actual usage, needs metrics-server.
kubectl top nodes
kubectl top pods -A --sort-by=memory
```

## Debugging a running container

Prefer ephemeral containers over `exec` — they work on distroless images that
have no shell, and they do not require the target image to ship debug tools.

```bash
# Attach a debug container sharing the pod's namespaces.
kubectl debug -it <pod> -n <ns> --image=nicolaka/netshoot --target=<container>

# Copy a broken pod with a shell, without disturbing the original.
kubectl debug <pod> -n <ns> -it --image=nicolaka/netshoot --copy-to=<pod>-debug

# Debug a node itself (mounts host / at /host).
kubectl debug node/<node> -it --image=nicolaka/netshoot
```

`--target=<container>` is required to share the *process* namespace with the app
container. Without it you get a container in the same pod but you cannot see the
app's processes or `/proc`.

## Gotchas

- **`--field-selector=status.phase!=Running` is noisy.** It matches `Succeeded`
  pods too, so every completed Job shows up as if it were broken. Exclude both
  phases, as in the command above. Field selectors also only support a small set
  of fields per resource; unsupported fields fail rather than being ignored.
- **`kubectl get events` is sorted by creation, not by time.** Use the newer
  `kubectl events` subcommand, or `--sort-by=.lastTimestamp` on `get`. The
  default order is close to useless during an incident.
- **Events expire.** Default retention is one hour. An empty event list on an
  hours-old failure means nothing; fall back to logs and object status.
- **`describe` truncates events**, and only shows events for that object. A pod
  failing to schedule because of a PVC has the real event on the PVC.
- **`restartCount` in `--sort-by` reads container `[0]` only.** A multi-container
  pod sorts by whichever container the API happens to list first. Use `[*]` in
  `custom-columns` to display all of them (it renders as `206,0`).
- **`kubectl apply` silently keeps fields you deleted** if they were set by a
  different field manager. Check `metadata.managedFields`, or use
  `--server-side --force-conflicts` when fighting another controller.
- **`kubectl top` reads metrics-server, not the cluster.** If it errors, that is
  a metrics-server problem, not evidence about the workload.
- **Deleting a stuck pod does not force it gone.** `--force --grace-period=0`
  removes the API object while the container may still run on the node. It is a
  last resort, and on StatefulSets it risks two instances writing the same volume.

## References

| File | Covers |
|---|---|
| [references/pod-failures.md](references/pod-failures.md) | CrashLoop, OOM, image pull, probes, init, stuck Terminating |
| [references/scheduling.md](references/scheduling.md) | Pending pods, taints, affinity, topology spread, priority |
| [references/networking.md](references/networking.md) | Services, endpoints, DNS, Ingress, NetworkPolicy |
| [references/storage.md](references/storage.md) | PVC binding, StorageClass, mount failures, expansion |
| [references/nodes-and-capacity.md](references/nodes-and-capacity.md) | NotReady, pressure, evictions, requests vs limits |
| [references/rbac-and-admission.md](references/rbac-and-admission.md) | Forbidden errors, ServiceAccounts, webhooks |
| [references/manifests.md](references/manifests.md) | Writing probes, resources, PDBs, and rollout strategy |
