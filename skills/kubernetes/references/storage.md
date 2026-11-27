# PVCs, StorageClasses, and mount failures

## PVC stuck Pending

```bash
kubectl describe pvc <pvc> -n <ns>          # events carry the real reason
kubectl get storageclass
kubectl get pvc -A -o custom-columns='NS:.metadata.namespace,PVC:.metadata.name,PHASE:.status.phase,SC:.spec.storageClassName'
```

| Event / symptom | Cause |
|---|---|
| `waiting for first consumer to be created` | **Not a bug.** `volumeBindingMode: WaitForFirstConsumer` — the PVC binds only once a pod using it is scheduled. |
| `no persistent volumes available for this claim` | No matching PV and no dynamic provisioner for that class. |
| `storageclass.storage.k8s.io "x" not found` | Typo, or the class does not exist in this cluster. |
| `failed to provision volume` | Provisioner error — read the CSI controller logs. |
| Pending with no events at all | `storageClassName: ""` explicitly disables dynamic provisioning. |

`WaitForFirstConsumer` and a Pending pod deadlock-looking pair is normal and
self-resolving: the pod is Pending because the PVC is unbound, the PVC is unbound
because it waits for the pod. The scheduler breaks the tie. If it persists, the
*pod* cannot schedule for an unrelated reason — debug the pod, not the PVC.

An omitted `storageClassName` gets the default class; an empty string gets none:

```bash
kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}'
```

Two default classes is a real and confusing failure — provisioning becomes
nondeterministic.

## Pod stuck ContainerCreating on a volume

```bash
kubectl describe pod <pod> -n <ns> | grep -A10 Events
kubectl logs -n <csi-ns> <csi-node-pod> -c csi-driver --tail=100
```

| Message | Meaning |
|---|---|
| `Multi-Attach error for volume` | An `RWO` volume is still attached to another node. Usually a pod that did not fully terminate; common after a node failure. |
| `timed out waiting for the condition` | Attach or mount hung — check the CSI node plugin on that node. |
| `FailedMount ... permission denied` | `fsGroup`/`runAsUser` mismatch with volume ownership. |
| `volume node affinity conflict` | The PV is pinned to a node/zone the pod cannot be scheduled to. |

**`ReadWriteOnce` means one *node*, not one pod.** Multiple pods on the same node
can share an RWO volume; a pod on a second node cannot. `ReadWriteOncePod`
enforces the stricter single-pod semantics.

## Expansion

```bash
kubectl get storageclass <sc> -o jsonpath='{.allowVolumeExpansion}'
kubectl patch pvc <pvc> -n <ns> -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
kubectl get pvc <pvc> -n <ns> -o jsonpath='{.status.conditions}' | jq
```

Expansion is **one-way** — PVCs cannot shrink. Some drivers require a pod restart
to finish the filesystem resize; watch for the `FileSystemResizePending`
condition.

## Reclaim policy

```bash
kubectl get pv -o custom-columns='PV:.metadata.name,CLAIM:.spec.claimRef.name,RECLAIM:.spec.persistentVolumeReclaimPolicy,STATUS:.status.phase'
```

`Delete` destroys the backing volume when the PVC goes away. `Retain` keeps it
and leaves the PV `Released` — which will **not** rebind to a new PVC until you
clear `spec.claimRef`. For anything holding real data, `Retain` plus a documented
recovery step is the safer default.

Deleting a namespace deletes its PVCs, and with `Delete` reclaim that destroys the
data with no confirmation.

## StatefulSet volumes

`volumeClaimTemplates` create one PVC per replica, named
`<template>-<statefulset>-<ordinal>`. They are deliberately **not** garbage
collected when the StatefulSet is deleted — scale-down leaves the PVCs behind so
scaling back up reattaches the same data. Clean up explicitly when you mean it.
