# platform-engineering

A handbook for platform engineering, packaged as [Claude Code](https://claude.com/claude-code)
skills. Each skill is a troubleshooting-first reference for one platform
component: symptom → route → verified commands → the traps nobody warns you about.

Commands in these skills were run before they were written down; the few that
were not say so where they appear. Where a claim comes from a specification, a
CRD schema, or upstream source, it is quoted rather than paraphrased.

## Skills

| Skill | Use it when |
|---|---|
| [`kubernetes`](skills/kubernetes/) | Pods Pending/CrashLooping/OOMKilled, Services with no endpoints, PVCs unbound, RBAC denials, node pressure |
| [`openshift`](skills/openshift/) | SCC rejections, Routes not serving, ClusterOperators degraded, OLM installs stuck, builds and ImageStreams |
| [`flux`](skills/flux/) | Kustomizations or HelmReleases not ready, sources failing to fetch, Git changes not landing, drift and pruning |
| [`argocd`](skills/argocd/) | Applications OutOfSync or Degraded, syncs that hang, diffs that never converge, ApplicationSets, projects and RBAC |
| [`kyverno`](skills/kyverno/) | Policies not blocking, blocking too much, mutations not firing, PolicyReports, exceptions, CI testing |
| [`linux-networking`](skills/linux-networking/) | Connections refused or timing out, DNS oddities, resource exhaustion, systemd units failing, TLS handshakes |
| [`observability`](skills/observability/) | Writing PromQL/LogQL, choosing what to measure, noisy or silent alerts, cardinality blowups, SLOs and error budgets |
| [`incident-triage`](skills/incident-triage/) | Something is broken now, or you are writing the postmortem afterwards |
| [`reviewing-pull-requests`](skills/reviewing-pull-requests/) | Judging whether a Renovate bump or manifest change is safe to merge: upstream breaking changes, what in the repo depends on it, what a render proves |
| [`renovate`](skills/renovate/) | Dependency update PRs are too noisy, too quiet, or updating the wrong thing |
| [`conventional-commits`](skills/conventional-commits/) | Writing commit messages, enforcing them, wiring automated releases |
| [`platform-mindset`](skills/platform-mindset/) | Deciding what to build, designing golden paths, escaping the ticket queue |

Skills with a `references/` directory load additional depth on demand, so the
top-level `SKILL.md` stays short.

## Install

As a Claude Code plugin:

```
/plugin marketplace add kashalls/platform-engineering
/plugin install platform-engineering
```

Or copy individual skill directories from `skills/` into a project's
`.claude/skills/` (or `~/.claude/skills/` for every project). Skills there are
discovered automatically; cloning this repository alone does not install them.

## Using them

Skills load automatically when a request matches their description. You can also
invoke one directly:

```
/kubernetes why is this pod stuck Pending
/flux the HelmRelease says retries exhausted
/incident-triage checkout is returning 500s
```

## Scope

These are written to be portable: no cluster names, hostnames, or topology from
any particular environment. Version-sensitive claims name the version they were
verified against, because Kubernetes and its ecosystem move faster than any
handbook.

Verified against: Kubernetes v1.36, Flux v2.10.0, Argo CD v3.5.1, `oc` v4.22.9,
Kyverno v1.19.0, Renovate schema v44.39.1.

## Contributing

The bar for a claim in this repo is that you ran it. If you are adding a command,
run it and paste the real output shape. If you are adding a gotcha, it should be
something that cost you time — generic advice is what the upstream docs are for.

## License

Apache-2.0. See [LICENSE](LICENSE).
