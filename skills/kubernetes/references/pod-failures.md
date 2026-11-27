# Pod failure modes

## CrashLoopBackOff

The container starts, exits, and the kubelet delays the next restart with
exponential backoff, capped at 5 minutes. The initial delay and the cap have both
changed across recent releases and are tunable on newer kubelets, so read the real
delay out of the event instead of assuming it:

```bash
kubectl describe pod <pod> -n <ns> | grep -i 'back-off restarting'
```

The status itself tells you *nothing* about why the container exited.

```bash
kubectl logs <pod> -n <ns> --previous --tail=200          # the crashed instance
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.containerStatuses[*].lastState.terminated}' | jq
```

The `terminated` object is the diagnosis:

| `reason` | `exitCode` | Means |
|---|---|---|
| `Error` | 1, 2, … | App exited nonzero. Read the logs; this is an app bug or bad config. |
| `OOMKilled` | 137 | Hit the memory limit. See below. |
| `Error` | 137 | SIGKILL from outside — usually a failed liveness probe, not OOM. |
| `Error` | 143 | SIGTERM — graceful shutdown that the app treated as failure. |
| `Completed` | 0 | Exited successfully. For a Deployment this is still a crashloop: the process is not meant to return. |
| `ContainerCannotRun` | 128 | Bad `command`/`entrypoint`, or the binary is not executable in the image. |

Exit code 137 is ambiguous by design. Distinguish with:

```bash
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

`OOMKilled` means the kernel cgroup killer. Plain `Error` with 137 means something
sent SIGKILL — check for liveness probe failures in `kubectl describe`.

## OOMKilled

```bash
kubectl describe pod <pod> -n <ns> | grep -iE 'oom|limit|memory'
kubectl top pod <pod> -n <ns> --containers
```

Points to check before raising the limit:

- **The limit applies per container, not per pod.** A sidecar can OOM while the
  app is fine. `--containers` on `kubectl top` splits it out.
- **JVM/Node/Go runtimes do not read cgroup limits by default in older versions.**
  A JVM without `-XX:MaxRAMPercentage` sizes its heap from *host* memory and will
  always overshoot a small limit.
- **Page cache counts toward the cgroup limit.** A process doing heavy file I/O
  can be OOMKilled with a small RSS.
- **Raising the limit without raising the request** makes the pod Burstable and a
  first eviction candidate under node pressure.

## ImagePullBackOff / ErrImagePull

```bash
kubectl describe pod <pod> -n <ns> | grep -A5 Events
```

The event message distinguishes the cases:

| Message contains | Cause |
|---|---|
| `not found` / `manifest unknown` | Tag does not exist, or wrong registry path. |
| `unauthorized` / `authentication required` | Missing or wrong `imagePullSecrets`. |
| `toomanyrequests` | Docker Hub anonymous rate limit. Authenticate or mirror. |
| `no such host` / `i/o timeout` | Node DNS or egress problem, not a registry problem. |
| `no match for platform` | Image lacks the node's architecture. |

Verify the secret is actually attached and well-formed:

```bash
kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.imagePullSecrets}'
kubectl get secret <secret> -n <ns> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq
```

Pull secrets are namespaced. A secret in `default` does nothing for a pod in
another namespace, and this is the single most common cause.

## Running but not Ready

Readiness failing keeps the pod out of Service endpoints while leaving it running.

```bash
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.conditions}' | jq
kubectl describe pod <pod> -n <ns> | grep -iE 'readiness|liveness|startup'
```

Probe design rules that prevent most incidents:

- **Liveness must not check dependencies.** If liveness probes the database, a
  database blip restarts every replica at once and turns a degraded service into
  an outage. Liveness answers "is this process wedged", nothing more.
- **Readiness may check dependencies.** That is the correct place for it.
- **Slow starters need `startupProbe`, not a long `initialDelaySeconds`.** The
  startup probe suspends the other two until it passes; a long liveness delay
  instead leaves the app unprotected for that whole window.
- `failureThreshold × periodSeconds` is the real timeout. Defaults (3 × 10s) give
  30 seconds; that is often too tight for JVM apps.

## Init containers

```bash
kubectl logs <pod> -n <ns> -c <init-container>
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.initContainerStatuses}' | jq
```

Init containers run **in order, to completion**. `Init:0/3` means the first is
still running or failing. A wait-for-dependency init container that never
succeeds is indistinguishable from a hung pod without reading its logs.

Sidecars declared as `initContainers` with `restartPolicy: Always` (native
sidecars — on by default since v1.29, stable since v1.33) start before app
containers and keep running. If one of those crashes, the app container never
starts.

## Stuck Terminating

```bash
kubectl get pod <pod> -n <ns> -o jsonpath='{.metadata.finalizers}'
kubectl get pod <pod> -n <ns> -o jsonpath='{.metadata.deletionTimestamp}'
```

A `deletionTimestamp` with remaining finalizers means a controller has not
released the object. Find and fix that controller. Removing the finalizer by
hand orphans whatever it was protecting:

```bash
# Last resort, understand what you are orphaning first.
kubectl patch pod <pod> -n <ns> -p '{"metadata":{"finalizers":null}}' --type=merge
```

Without finalizers, a stuck Terminating pod is usually a `preStop` hook or a
`terminationGracePeriodSeconds` still counting down, or a node that is gone
(the pod object cannot be confirmed deleted while its kubelet is unreachable).

`--force --grace-period=0` deletes the API object only. On a partitioned node the
container keeps running. For StatefulSets that risks two pods with the same
identity writing one volume.
