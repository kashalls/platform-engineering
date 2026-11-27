# SecurityContextConstraints

SCCs are OpenShift's pod admission gate. They predate Pod Security Admission and
run in addition to it. If a pod runs on vanilla Kubernetes but not here, start
here.

## The error

```
pods "web-xxxx" is forbidden: unable to validate against any security
context constraint: [provider "restricted-v2": Forbidden: not usable by user
or serviceaccount, spec.containers[0].securityContext.runAsUser: Invalid
value: 0: must be in the ranges: [1000670000, 1000679999]]
```

Read it as a **list of every SCC that was tried and why each rejected the pod**.
The interesting line is the one for the SCC you expected to work.

## Which SCC applies

Selection is not "the most permissive one you can use". From the admission
controller's sort (`ByPriority` in `apiserver-library-go`):

1. **Higher `priority` field first** — a higher number moves to the front.
2. **Then lower restriction score** — *"the greater the score, the more lax the
   SCC is"*, and more restrictive sorts first.
3. **Then name**, compared as a string.

The first SCC in that order that admits the pod wins. So granting an
`anyuid` SCC does not mean it gets used — a more restrictive SCC that still
admits the pod is chosen instead. That is why "I granted the SCC and nothing
changed" is a common and confusing report.

```bash
# Which SCC actually admitted a running pod.
oc get pod <pod> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# Which SCCs would admit this manifest as this service account.
oc adm policy scc-subject-review -z <serviceaccount> -f deployment.yaml

# Which service accounts can create this pod at all.
oc adm policy scc-review -f deployment.yaml

oc get scc
oc describe scc restricted-v2
```

`scc-subject-review` against the real ServiceAccount is the fastest loop — it
answers the question without deploying anything.

## restricted-v2

The default for authenticated users since OCP 4.11 (`restricted` was the earlier
default and still exists). It:

- forbids running as root, and **assigns a random UID** from the namespace range
- drops all capabilities, and allows only `NET_BIND_SERVICE` to be added back
- requires `allowPrivilegeEscalation: false`
- requires a seccomp profile of `runtime/default`
- forbids host namespaces, host paths, and privileged mode

The namespace's UID range is an annotation, and it differs per namespace:

```bash
oc get namespace <ns> -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'
```

## Making an image work under restricted-v2

This is almost always better than granting a permissive SCC.

The random-UID rule means your image cannot assume a specific user exists. The
UID will be arbitrary, but the **GID is always 0**. So:

```dockerfile
RUN mkdir -p /app/data \
 && chgrp -R 0 /app \
 && chmod -R g=u /app          # group gets whatever the owner has
USER 1001
```

`chmod g=u` is the canonical OpenShift pattern: since the arbitrary UID runs with
group 0, group-writable paths work for any assigned UID.

Other required fixes:

- **Do not bind ports below 1024** unless the container adds `NET_BIND_SERVICE`
  in `capabilities.add`. Listen on 8080, not 80.
- **Do not write outside declared volumes.** Add an `emptyDir` for scratch space.
- **Do not hardcode `runAsUser`** in the manifest — let OpenShift assign one.
  A hardcoded UID outside the namespace range is rejected outright.
- **Applications that read `/etc/passwd` for the current user** fail because the
  arbitrary UID has no entry. Use `nss_wrapper`, or make the app tolerate it.

## When you genuinely need more

Grant to a **ServiceAccount**, never to a user or a whole group:

```bash
oc create serviceaccount privileged-thing -n <ns>
oc adm policy add-scc-to-user anyuid -z privileged-thing -n <ns>
```

`-z` means "service account in the current (or -n) namespace" and is what you
almost always want. `add-scc-to-user <scc> <user>` without `-z` grants to a human
identity instead, which is rarely intended.

Built-in SCCs, least to most permissive: `restricted-v2`, `nonroot-v2`,
`hostmount-anyuid`, `anyuid`, `hostnetwork-v2`, `hostaccess`, `privileged`.

- `anyuid` — lets the image's own `USER` apply. The usual minimum for images
  that insist on a fixed UID.
- `privileged` — effectively root on the node. Reserve for CNI/CSI/monitoring
  agents; a workload needing it is a design problem.

Prefer a **custom SCC** that grants exactly one thing over a built-in that grants
everything:

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: app-nonroot-fixed-uid
runAsUser:
  type: MustRunAsRange
  uidRangeMin: 1001
  uidRangeMax: 1001
# ... copy the rest from restricted-v2
```

Set `priority` deliberately on custom SCCs. Leaving it nil means it sorts behind
anything with a priority, and its restriction score decides placement.

## Pod Security Admission interaction

OCP 4.11+ also runs upstream PSA. A namespace labelled
`pod-security.kubernetes.io/enforce: restricted` rejects pods **before** SCCs are
consulted, with a different error mentioning `violates PodSecurity`. Two separate
gates, two separate error shapes — do not fix the wrong one.

```bash
oc get namespace <ns> -o jsonpath='{.metadata.labels}' | jq
```

OpenShift auto-labels namespaces based on the SCCs their service accounts can
use, so granting a permissive SCC usually relaxes PSA to match. When it does not,
label the namespace explicitly.
