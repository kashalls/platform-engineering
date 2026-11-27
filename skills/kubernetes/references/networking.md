# Services, DNS, Ingress, NetworkPolicy

Work outward from the pod. Do not start at the Ingress.

```
pod listening? → endpoints populated? → DNS resolves? → policy allows? → ingress routes?
```

## Step 1: is the Service selecting anything

The single most common Service bug is a selector that matches no pods.

```bash
kubectl get endpointslices -n <ns> -l kubernetes.io/service-name=<svc>
kubectl get svc <svc> -n <ns> -o jsonpath='{.spec.selector}'; echo
kubectl get pods -n <ns> --show-labels
```

Empty `ENDPOINTS` means one of:

- **Selector does not match pod labels.** Compare the two outputs above literally;
  a Deployment's `spec.selector` and the Service's `spec.selector` are unrelated
  and drift independently.
- **Pods are not Ready.** Only Ready pods land in endpoints by default. This is
  the intended behavior of a failing readiness probe, not a Service bug.
- **`targetPort` names a port the container does not declare.** A named
  `targetPort` must match a `containerPort.name` in the pod spec.

> `kubectl get endpoints` still works but the `v1 Endpoints` API is deprecated
> from v1.33 onward in favor of `discovery.k8s.io/v1 EndpointSlice`, and it
> prints a deprecation warning. Prefer `endpointslices`.

`port` is what clients dial, `targetPort` is the container's port. Getting these
backwards produces a Service with healthy endpoints that refuses connections.

## Step 2: is the pod actually listening

A process bound to `127.0.0.1` is unreachable from outside the pod, no matter how
correct the Service is.

```bash
kubectl debug -it <pod> -n <ns> --image=nicolaka/netshoot --target=<container> \
  -- ss -lntp
```

Look for `0.0.0.0:<port>` or `[::]:<port>`. `127.0.0.1:<port>` is the bug.

## Step 3: DNS

Cluster DNS gives every Service `<svc>.<ns>.svc.cluster.local`.

```bash
kubectl run dnscheck --rm -it --restart=Never --image=nicolaka/netshoot -- \
  nslookup <svc>.<ns>.svc.cluster.local

kubectl get svc -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

A pod's `/etc/resolv.conf` looks like this (verified on a v1.36 cluster):

```
search <ns>.svc.cluster.local svc.cluster.local cluster.local
nameserver <cluster-dns-ip>
options ndots:5
```

**`ndots:5` is the classic latency trap.** Any name with fewer than 5 dots is
tried against every entry in `search` *first*. Resolving `api.example.com`
(2 dots) issues three failing cluster lookups before the public one succeeds.
Fix by fully qualifying external names with a trailing dot (`api.example.com.`)
or lowering `ndots` per pod via `dnsConfig`.

Other DNS failure shapes:

- **Resolves inside `kube-system` but not elsewhere** — a NetworkPolicy is
  blocking egress to port 53. DNS is the most commonly forgotten egress rule.
- **Intermittent failures / 5s hangs** — classic conntrack race on UDP DNS.
  Mitigate with NodeLocal DNSCache or `use-vc` in `dnsConfig` options.
- **`dnsPolicy: Default`** means "inherit the node's resolv.conf", which is *not*
  cluster DNS. `ClusterFirst` is the default for pods, despite the naming.
- **Headless Services** (`clusterIP: None`) resolve to pod IPs directly, one A
  record per Ready pod. There is no load balancing and no stable VIP.

## Step 4: NetworkPolicy

```bash
kubectl get networkpolicies -A
kubectl get networkpolicy <np> -n <ns> -o yaml
```

The rules that trip people:

- **Policies are additive and default-deny-once-selected.** A namespace with zero
  policies allows everything. The moment *one* policy selects a pod, that pod
  denies all traffic of that direction except what some policy permits.
- **Ingress and egress are separate.** Adding any egress policy blocks DNS unless
  you explicitly allow UDP/TCP 53 to kube-dns.
- **`namespaceSelector` and `podSelector` in the same list element are ANDed**;
  as two separate list elements they are ORed. This is the most frequent silent
  mistake in policy authoring.
- **NetworkPolicy needs a CNI that implements it.** With a CNI that does not,
  the objects apply cleanly and enforce nothing. Verify before trusting them.

## Step 5: Ingress

```bash
kubectl get ingress -A
kubectl describe ingress <ing> -n <ns>
kubectl logs -n <ingress-ns> -l app.kubernetes.io/name=<controller> --tail=100
```

| Symptom | Usual cause |
|---|---|
| 404 from the controller | No matching host/path rule, or `ingressClassName` unset so no controller claimed it |
| 503 | Backend Service has no Ready endpoints (go back to step 1) |
| 502 | Backend is up but speaking the wrong protocol (TLS vs plaintext), or timing out |
| TLS warning / default cert | Secret missing, in the wrong namespace, or not referenced in `tls[]` |
| Works by IP, not by host | DNS record missing, or host rule does not match the `Host` header |

Ingress TLS secrets must live **in the same namespace as the Ingress**. A
cert-manager Certificate in another namespace is invisible to it.

`ingressClassName` replaced the old `kubernetes.io/ingress.class` annotation. An
Ingress with neither is claimed only if a controller is marked default; otherwise
it sits there doing nothing, with no error.

## Connectivity probe from inside the cluster

```bash
kubectl run nettest --rm -it --restart=Never --image=nicolaka/netshoot -- bash
# then, inside:
curl -sv http://<svc>.<ns>.svc.cluster.local:<port>/
dig +short <svc>.<ns>.svc.cluster.local
nc -zv <svc>.<ns>.svc.cluster.local <port>
```

Run it **in the same namespace as the client**, not a convenient one — namespace
determines which NetworkPolicies and DNS search path apply.
