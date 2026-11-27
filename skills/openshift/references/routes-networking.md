# Routes, Ingress, and the router

## Route vs Ingress

OpenShift has both. `Route` (`route.openshift.io/v1`) predates Ingress and is
richer; an `Ingress` on OpenShift is **converted into a Route** by an ingress
controller shim. Debug the Route, even when you wrote an Ingress.

```bash
oc get routes -A
oc get route <name> -o yaml
oc describe route <name>
oc get ingress <name> -o jsonpath='{.metadata.annotations}'   # shim annotations
```

```bash
# Did my Ingress produce a Route?
oc get routes -n <ns> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.ownerReferences[*].kind}{"\n"}{end}'
```

An Ingress that produced no Route means the shim rejected it — usually an
unsupported field or a missing `spec.rules[].host`.

## Status is on the Route

```bash
oc get route <name> -o jsonpath='{.status.ingress[*].conditions}' | jq
```

`Admitted: False` with a reason is the whole diagnosis. Common reasons:

| Reason | Cause |
|---|---|
| `HostAlreadyClaimed` | Another Route in another namespace already owns that host. First claim wins, permanently. |
| `RouteNotAdmitted` | The router's namespace/route selector excludes this Route. |
| `ExtendedValidationFailed` | Certificate does not match the host, or key/cert mismatch. |

`HostAlreadyClaimed` across namespaces is a genuine multi-tenancy trap: a
misspelled host in one team's namespace can permanently block another team's real
host until the offending Route is deleted.

## TLS termination

```yaml
spec:
  host: app.apps.example.com
  to: {kind: Service, name: web}
  port: {targetPort: http}
  tls:
    termination: edge          # edge | passthrough | reencrypt
    insecureEdgeTerminationPolicy: Redirect   # Allow | Redirect | None
```

| Termination | Router does | Backend receives |
|---|---|---|
| `edge` | terminates TLS | **plain HTTP** |
| `passthrough` | forwards encrypted bytes | TLS, must serve its own cert |
| `reencrypt` | terminates, then re-encrypts to backend | TLS |

The classic mistake is `edge` against a backend that only speaks HTTPS, which
gives a 502. `passthrough` cannot do path-based routing or inject headers,
because the router never sees the plaintext.

For `reencrypt` you must supply `destinationCACertificate` unless the backend's
cert is signed by a CA the router already trusts. For internal services signed by
the cluster's service CA, use the service CA bundle.

Omit `tls` entirely and the route is plain HTTP.

## Host assignment

If `spec.host` is unset, the router assigns
`<route-name>-<namespace>.<ingress-domain>`:

```bash
oc get ingresses.config/cluster -o jsonpath='{.spec.domain}'
```

Note `ingresses.config/cluster` (the cluster config) is a different resource from
a namespaced `ingress`. Getting these confused is common.

**Re-applying a manifest without `host` can trigger reassignment**, breaking DNS
that pointed at the old name. Either always set `host` explicitly, or never
include the field and let it stay assigned.

## The router itself

```bash
oc get pods -n openshift-ingress
oc logs -n openshift-ingress deploy/router-default --tail=100
oc get ingresscontroller -n openshift-ingress-operator default -o yaml
```

The router is HAProxy. Per-route tuning is done with annotations:

```yaml
metadata:
  annotations:
    haproxy.router.openshift.io/timeout: 60s
    haproxy.router.openshift.io/balance: roundrobin
    haproxy.router.openshift.io/disable_cookies: "true"
    haproxy.router.openshift.io/rate-limit-connections: "true"
```

A default router timeout that is shorter than your application's long-poll or
upload duration produces intermittent 504s that look like application faults.
`haproxy.router.openshift.io/timeout` is the fix.

## Troubleshooting order

```bash
# 1. Service has endpoints? (same as vanilla Kubernetes)
oc get endpointslices -n <ns> -l kubernetes.io/service-name=<svc>

# 2. Route admitted?
oc get route <name> -o jsonpath='{.status.ingress[*].conditions}' | jq

# 3. Does it work from inside the cluster?
oc run nettest --rm -it --restart=Never --image=nicolaka/netshoot -- \
  curl -sv http://<svc>.<ns>.svc.cluster.local:8080/

# 4. Does the router see it?
oc logs -n openshift-ingress deploy/router-default --tail=50

# 5. DNS resolves the host to the router's LB?
dig +short <host>
```

Steps 1 and 3 rule out everything OpenShift-specific. If the Service has no
endpoints, the Route is irrelevant.

## NetworkPolicy and multitenant isolation

OpenShift SDN's multitenant mode and OVN-Kubernetes network policies both apply.
Additionally, OpenShift ships default policies in some namespaces.

```bash
oc get networkpolicies -A
oc get network.config/cluster -o jsonpath='{.spec.networkType}'
```

Egress from a namespace to the router, and to cluster DNS, must be allowed once
any egress policy exists in that namespace — the same trap as vanilla Kubernetes.
