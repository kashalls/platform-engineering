# Pending pods and scheduling

A Pending pod has **not been assigned a node**. The scheduler records why on the
pod itself, and that message is the whole diagnosis:

```bash
kubectl describe pod <pod> -n <ns> | grep -A10 Events
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")]}' | jq
```

The message reads like `0/5 nodes are available: 3 Insufficient cpu, 2 node(s)
had untolerated taint {node-role.kubernetes.io/control-plane: }`. It accounts for
**every** node, and the counts sum to the cluster size. Read it literally.

## Decoding the scheduler message

| Fragment | Cause | Check |
|---|---|---|
| `Insufficient cpu` / `memory` | Requests exceed remaining allocatable | `kubectl describe node <n> \| grep -A8 'Allocated resources'` |
| `untolerated taint` | Node is tainted, pod has no toleration | `kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints[*].key'` |
| `node(s) didn't match Pod's node affinity/selector` | `nodeSelector`/`nodeAffinity` matches nothing | `kubectl get nodes --show-labels` |
| `pod has unbound immediate PersistentVolumeClaims` | PVC is not Bound | see storage.md |
| `didn't match pod topology spread constraints` | Spread rules cannot be satisfied | check `topologySpreadConstraints` |
| `node(s) had volume node affinity conflict` | PV is pinned to a zone/node the pod cannot use | check PV `nodeAffinity` |
| `Too many pods` | Node hit its max-pods cap | `kubectl get node <n> -o jsonpath='{.status.allocatable.pods}'` |

## Requests are what schedule; limits are not

The scheduler only reads `resources.requests`. Limits are enforced at runtime by
the kubelet and cgroups, and are invisible to scheduling. Consequences:

- A pod with no requests schedules **anywhere**, then competes for CPU and gets
  evicted first under memory pressure (QoS class `BestEffort`).
- A node can be 100% *requested* and nearly idle, and still reject pods. The fix
  is right-sizing requests, not adding nodes.
- `requests == limits` on both CPU and memory gives QoS `Guaranteed`, the last
  thing evicted.

```bash
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.qosClass}'
```

**CPU limits throttle rather than kill.** A container hitting its CPU limit is
throttled by CFS, showing up as latency, not as restarts. Many teams set CPU
requests and deliberately omit CPU limits for latency-sensitive workloads.

## Taints and tolerations

Taints repel; tolerations permit. A toleration does not *attract* a pod — it only
removes an objection. To attract, you also need `nodeAffinity`.

```bash
kubectl taint nodes <node> key=value:NoSchedule        # add
kubectl taint nodes <node> key=value:NoSchedule-       # remove (trailing dash)
```

Effects: `NoSchedule` (no new pods), `PreferNoSchedule` (soft), `NoExecute`
(also evicts pods already running that do not tolerate it).

`NoExecute` tolerations may carry `tolerationSeconds`, which is how the
node-lifecycle controller drains pods off a node that goes NotReady. The default
injected toleration is 300 seconds, which is why pods take ~5 minutes to
reschedule after a node dies.

## Affinity

- `nodeAffinity` — attract to nodes by label.
- `podAffinity` — co-locate with other pods.
- `podAntiAffinity` — spread away from other pods.

`requiredDuringSchedulingIgnoredDuringExecution` is hard: unsatisfiable means
Pending forever. `preferredDuringScheduling…` is a soft weight.

`IgnoredDuringExecution` in both names is literal: once a pod is running, changing
node labels never evicts it. Affinity is a scheduling-time filter only.

**Anti-affinity is expensive at scale.** `topologyKey: kubernetes.io/hostname`
with a large pod count makes the scheduler compare every candidate node against
every existing pod. Prefer `topologySpreadConstraints` for even distribution.

## Topology spread

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule   # or ScheduleAnyway
    labelSelector:
      matchLabels: {app: web}
```

`maxSkew` is the allowed difference between the most and least populated
topology domain. `DoNotSchedule` turns a spread violation into a Pending pod —
during a zone outage that can block scaling entirely. `ScheduleAnyway` degrades
to a preference, which is usually what you want for availability.

The `labelSelector` must match the pods being spread, which is normally the
pod's own labels. A mismatched selector silently spreads against nothing.

## Priority and preemption

```bash
kubectl get priorityclass
```

Higher `value` wins. When a high-priority pod cannot schedule, the scheduler
evicts lower-priority pods to make room, and the victims show a `Preempted`
event. `preemptionPolicy: Never` opts out of evicting others while still giving
the pod queue priority.

Two ways this surprises people: a `globalDefault: true` PriorityClass silently
re-prioritizes every pod that does not name one, and a system-critical class on
a workload that does not deserve it will happily evict real traffic.
