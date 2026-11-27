# ClusterOperators, the CVO, OLM, and upgrades

## Two different "operator" concepts

- **ClusterOperators** (`oc get co`) are the platform's own components, owned by
  the Cluster Version Operator. You configure them via their config CRs; you do
  not edit what they produce.
- **OLM operators** (`oc get csv`) are add-ons installed from a catalog via
  Subscriptions.

Different objects, different troubleshooting. Confusing them wastes time.

## ClusterOperators

```bash
oc get clusteroperators
oc get co <name> -o jsonpath='{.status.conditions}' | jq
```

Four conditions matter, and the combination is the diagnosis:

| Condition | Meaning when true |
|---|---|
| `Available` | The operator's function works. **False is an outage.** |
| `Degraded` | Working, but not to spec — the most common real alert. |
| `Progressing` | Actively changing state. Normal during an upgrade, a problem if permanent. |
| `Upgradeable` | False blocks cluster upgrades until resolved. |

```bash
# Every operator that is Degraded, with its message.
oc get co -o json | jq -r '.items[] | . as $co |
  .status.conditions[] | select(.type=="Degraded" and .status=="True") |
  "\($co.metadata.name): \(.message)"'
```

The `message` field is written for humans and usually names the exact failing
resource. Read it before touching anything.

**Never edit resources a ClusterOperator manages.** The CVO reconciles them back,
typically within minutes, with no event explaining the reversion. Configure
through the operator's config CR (`oc get <operator>.operator.openshift.io` or
the relevant `config.openshift.io` resource) or, for genuine exceptions, an
unmanaged override — which is unsupported and should be temporary.

## Upgrades

```bash
oc get clusterversion
oc adm upgrade                      # what is available
oc adm upgrade --to=<version>       # a version listed by the command above
oc get clusterversion -o jsonpath='{.status.conditions}' | jq
```

A stuck upgrade is almost always one ClusterOperator that will not go
`Progressing: False`:

```bash
oc get co | grep -v 'True.*False.*False'
```

`Upgradeable: False` on any operator blocks the *next* minor upgrade and states
why in its message — commonly a deprecated API still in use, or an add-on
operator that does not support the target version.

```bash
# Deprecated APIs still being called; blocks minor upgrades.
oc get apirequestcounts -o json | jq -r '.items[] |
  select(.status.removedInRelease != null) |
  "\(.metadata.name) removed in \(.status.removedInRelease)"'
```

`apirequestcounts` is OpenShift-specific and genuinely useful: it names the API
and, drilled into, the users and workloads still calling it.

## OLM

```bash
oc get subscriptions -A
oc get csv -A                       # ClusterServiceVersions: installed operators
oc get installplan -A
oc get catalogsource -n openshift-marketplace
```

The chain is `CatalogSource → PackageManifest → Subscription → InstallPlan → CSV`.
A break anywhere upstream stalls everything downstream.

| Symptom | Check |
|---|---|
| Subscription stuck, no InstallPlan | CatalogSource not Ready — often an unreachable registry. |
| InstallPlan `RequiresApproval` | `installPlanApproval: Manual`; approve it explicitly. |
| CSV `Pending` | Unmet dependency, or the operator's required namespace/OperatorGroup is missing. |
| CSV `Failed` | Read `.status.message`. Frequently RBAC or a webhook. |
| Operator installs, does nothing | OperatorGroup targets the wrong namespaces. |

```bash
oc get csv <name> -n <ns> -o jsonpath='{.status.phase}{"\t"}{.status.message}'
oc get catalogsource -n openshift-marketplace -o custom-columns='NAME:.metadata.name,STATE:.status.connectionState.lastObservedState'
oc get operatorgroup -n <ns> -o yaml
```

**An OperatorGroup mismatch is the quiet failure.** An operator scoped to watch
one namespace, installed with an OperatorGroup targeting another, installs
cleanly and then ignores your CRs. There is no error — just nothing happening.

Approving a manual InstallPlan:

```bash
oc patch installplan <name> -n <ns> --type=merge -p '{"spec":{"approved":true}}'
```

## MachineConfig

Node-level configuration is managed by the Machine Config Operator, which
**reboots nodes** to apply changes, one at a time per pool.

```bash
oc get machineconfigpool
oc get mcp <pool> -o jsonpath='{.status.conditions}' | jq
oc get machineconfig
```

`UPDATING=True` on a pool means nodes are rebooting right now. A pool stuck
`Degraded` names the node and reason in its conditions; the usual causes are a
node that cannot drain (see PodDisruptionBudgets) or a bad MachineConfig that
fails to render.

Do not configure nodes by SSH-ing in. The MCO reverts it on the next reconcile,
and the node may be marked degraded for having unexpected state.

## Collecting evidence

```bash
oc adm must-gather
oc adm must-gather -- /usr/bin/gather_network_logs      # targeted variants exist
oc adm inspect clusteroperator/<name>
```

`oc adm inspect` is the lighter-weight option when you already know which
operator is unhappy and do not need a full cluster dump.
