# Nodes, pressure, and eviction

## Node not Ready

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,MEM:.status.conditions[?(@.type=="MemoryPressure")].status,DISK:.status.conditions[?(@.type=="DiskPressure")].status,PID:.status.conditions[?(@.type=="PIDPressure")].status'
kubectl describe node <node> | grep -A15 Conditions
```

`Ready: Unknown` means the kubelet stopped reporting — a node/network problem.
`Ready: False` means the kubelet is reporting and saying it is unhealthy, usually
with a `message` naming the runtime or the CNI. The distinction decides whether
you debug the network or the node.

On the node itself:

```bash
systemctl status kubelet
journalctl -u kubelet -n 200 --no-pager
crictl ps -a          # containers, bypassing the kubelet
crictl info           # runtime + CNI status
```

Common causes: disk full on `/var/lib/kubelet` or `/var/lib/containerd`, expired
kubelet client certificate, a CNI plugin that failed to install its config, or
memory exhaustion killing the kubelet itself.

## Pressure and eviction

The kubelet evicts pods when it crosses an eviction threshold. Defaults are
roughly `memory.available<100Mi`, `nodefs.available<10%`,
`imagefs.available<15%`, but they are kubelet flags — check the actual node
rather than assuming:

```bash
kubectl get --raw "/api/v1/nodes/<node>/proxy/configz" | jq '.kubeletconfig.evictionHard'
```

The kubelet ranks eviction candidates by:

1. Whether the pod's usage of the starved resource **exceeds its requests**.
2. **Pod priority** (`priorityClassName`).
3. How far usage exceeds requests.

QoS class is not an input to that ranking, but it predicts the outcome: a
`BestEffort` pod (no requests) always exceeds its requests, and a `Guaranteed`
pod (requests == limits for every container and resource) cannot exceed them,
so it is evicted only when system daemons need the memory.

```bash
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,QOS:.status.qosClass'
```

To protect a critical workload from eviction, set requests that cover its real
usage (requests == limits gives `Guaranteed`) and give it a higher priority.

**Evicted pods are not deleted.** They accumulate as `Failed` objects and clutter
triage output:

```bash
kubectl get pods -A --field-selector=status.phase==Failed
kubectl delete pods -A --field-selector=status.phase==Failed
```

## Capacity, requests, and usage

Three different numbers, constantly confused:

```bash
kubectl describe node <node> | grep -A8 'Allocated resources'   # requests (what schedules)
kubectl top node <node>                                          # usage (what is happening)
kubectl get node <node> -o jsonpath='{.status.allocatable}' | jq # allocatable (the ceiling)
```

`capacity` is the hardware. `allocatable` is capacity minus kubelet/system
reserved. Scheduling is against **allocatable vs requests** — usage is irrelevant
to it. A node at 9% CPU usage can legitimately refuse pods because it is 100%
requested.

## Draining

```bash
kubectl cordon <node>                      # stop new scheduling, disturb nothing
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

`--ignore-daemonsets` is nearly always required; DaemonSet pods are not evictable
by design. `--delete-emptydir-data` is a data-loss acknowledgement — those volumes
are gone.

Drain respects PodDisruptionBudgets and will **hang indefinitely** rather than
violate one. A PDB with `minAvailable` equal to the replica count blocks drains
forever:

```bash
kubectl get pdb -A
kubectl get pdb <pdb> -n <ns> -o jsonpath='{.status}' | jq
```

`status.disruptionsAllowed: 0` is the reason your drain is stuck. Fix the PDB or
scale up first — do not reach for `--force`, which deletes unmanaged pods outright.
