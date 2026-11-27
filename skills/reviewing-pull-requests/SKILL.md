---
name: reviewing-pull-requests
description: Review a pull request or Renovate update in a GitOps repository (Flux or Argo CD, on Kubernetes or OpenShift). Use when judging whether a dependency bump is safe to merge, finding a container image's or Helm chart's upstream and its breaking changes, checking what in the repo depends on a changed label, resource name, value or CRD field, or deciding what a passing render or check actually proves. references/registry.md reads images and OCI charts with crane or helm; references/rendering.md renders and diffs HelmReleases and Applications.
---

# Reviewing pull requests in a GitOps repository

In a GitOps repository a merge to the tracked branch is a rollout: Flux or
Argo CD applies it within minutes, so the review is the last gate. Secrets
usually arrive as references (ExternalSecret, SealedSecret, SOPS): judge
key names and mappings from the manifests, never ask for the values.

Use the sections that fit: [a Renovate update](#a-renovate-update);
[a change to the manifests](#a-change-to-the-manifests), which covers a
Renovate update's files too; and [what a check proves](#what-a-check-proves),
for every review.

## A Renovate update

Judge whether the update is safe to merge: what changed upstream between
the old and new versions, and whether anything in this repository depends
on it. Read GitHub with `gh`, a project's own changelog or docs site with
`curl`, and an image or OCI chart from its registry as
`references/registry.md` describes.

### The update

Take from the title, body, labels and diff: the package and its datasource
(container image, Helm chart from a `HelmRepository` or `OCIRepository`,
GitHub Action, tool version, Renovate preset), the old and new version or
digest, the update type (major, minor, patch, digest), and the changed
files, which are the call sites that consume the dependency.

Renovate's body carries a collapsed **Release Notes** section with upstream
excerpts. Read it first. Go upstream when it is missing, truncated or
ambiguous, and always for a major update or a jump over several versions:
read every intermediate release, because breaking changes land in the
middle.

### Finding the upstream

- An image at `<registry>/<owner>/<repo>` usually comes from `<owner>/<repo>`
  on GitHub; otherwise the PR body or the image's
  `org.opencontainers.image.source` label names the source.
- Check an image update against the image itself: read the pinned digest's
  config as `references/registry.md` describes. Its version and revision
  labels show what it was built from, to compare with the release reviewed.
- A chart's `Chart.yaml` `sources` and `home` name its repository; read
  them from the registry without a checkout. Chart version and
  `appVersion` differ: read the chart's changelog, and the application's
  too when `appVersion` moved.
- A chart pulled from a mirror or a vendor's repackaging is still the
  upstream project's chart: find that project from `sources` and read its
  changelog there, not the mirror's commit log.
- A repository that publishes many charts prefixes release tags with the
  chart name (`<chart>-1.2.3`, not `v1.2.3`). When a release is not found,
  `gh release list -R <owner>/<repo>` and look for the chart name.
- A digest-only update moved no version: say whether the tag was rebuilt
  (a security rebuild, or a floating tag such as `latest`), and from what:
  compare the old and new digests' `created` and `revision`.
- A release note that says "see #2785" is read with `gh issue view` or
  `gh pr view`. Take an issue to the merged PR that closed it
  (`gh pr list -R <owner>/<repo> --search "#<n>" --state merged`): the PR
  says what changed and in what scope, where the issue and the note may not.

### What breaks

Flag in each release: `BREAKING CHANGE`, `⚠` or `!:` markers; removed or
renamed Helm values, CRD fields, environment variables and flags; a
required Kubernetes, OpenShift, Flux, Argo CD, OS or kernel version;
one-way schema or data migrations; changed defaults (authentication,
storage class, ports, probes, security context); deprecations that became
errors; new required keys with no default; and label values that changed
while the key stayed, or resource names that dropped or gained a prefix,
which break anything selecting on the old value while a search for the key
still matches.

Minor and patch updates carry breaking changes too. "Chart name prefix
removed" is ambiguous between label values and resource names: settle it
from the PR or the template, not the sentence.

On OpenShift, a chart that starts setting a fixed `runAsUser`, `fsGroup`,
`hostPath`, or added capabilities will be rejected under `restricted-v2`
even though it renders cleanly; the render does not run admission. See the
`openshift` skill's SCC reference.

An update also moves images that appear nowhere in the diff: a chart or
operator that changes its default images, or stops setting them and leaves
them to another operator's built-in defaults. Name each such image with its
old and new version, read from the chart's or operator's source at both
versions. Nothing in the repository pins them, so Renovate never raises
them and the review is the only place they show. The image a chart sets by
default, such as one for a workload its operator manages, is the default in
the chart's `values.yaml` at the new tag unless the release's values pin it.

For an update to the manifests, render it and search what depends on it as
[a change to the manifests](#a-change-to-the-manifests) describes. A
breaking change that touches nothing here is not a finding: one sentence,
with the search that came up empty, is enough.

### Renovate's rules

The repository's Renovate config (`renovate.json`, `.renovaterc.json5`,
or the `renovate` key in `package.json`) says how this update is treated:
its `automerge` rules say whether the PR merges on its own, in which case
the review is the last look before the merge; its `groupName` rules say
which PRs move several packages at once, each of which the review covers;
its labels and any `!` a preset adds to a major update should agree with
the title, and drift between them is worth a note. The `renovate` skill
covers the config itself.

## A change to the manifests

When a chart version or a release's values change, render it as
`references/rendering.md` describes.

### Exposure here

The question is not whether this repository sets the old thing but whether
anything here depends on it. Search the manifests tree for:

- **a label key or value**: whatever selects on it, such as `matchLabels`
  and `selector` fields of Services, ServiceMonitors and PodMonitors,
  NetworkPolicies and PodDisruptionBudgets, and label matchers in PromQL
  rules and dashboards;
- **a resource name**: whatever refers to it, such as Ingress, Route and
  HTTPRoute backends, cross-namespace DNS
  (`<name>.<namespace>.svc.cluster.local`), Flux `dependsOn` and
  `healthChecks`, and Argo CD `ignoreDifferences` and sync-wave ordering;
- **a value, flag or CRD field**: the release's `values`, `valuesFrom`,
  `postRenderers` (Flux) or `helm.valuesObject` and `parameters` (Argo CD),
  kustomize patches, including patches a parent applies to every release
  (a Flux Kustomization's `patches`, a FluxInstance's kustomize patches,
  an ApplicationSet template), and the custom resources other apps create
  from the dependency's CRDs (`apiVersion: <group>/...`).

An app configured through its own UI keeps that configuration on its
volume or in its database, not in git, so a search of this repository
proves nothing about it: an affected integration or setting there is one
the review could not verify.

Before prescribing a rename or a new value, confirm it from the upstream
PR's diff or from the chart's template at the new tag, read raw with:

```
gh api -H "Accept: application/vnd.github.raw+json" "repos/<owner>/<repo>/contents/<path>?ref=<tag>"
```

Without that, make it a check to do after the merge rather than an edit; a
wrong edit that gets applied is worse than none.

Anchor a finding to the changed line in the diff; name the dependent file
and line in its explanation.

## What a check proves

- Write an inference out, and only one the evidence carries. A successful
  render of the new chart proves the tag exists and the values fit its
  schema. A default the release changes to a value this repository already
  sets explicitly changes nothing here; a prerequisite that comes with it,
  such as a kernel or Kubernetes version, is checked on its own.
- Check a version requirement against the version the repository targets,
  read from where it pins it. A target in git is not proof that a rollout
  reached every node: say that the line rests on the target.
- Put the evidence in the claim: the value found and the file it came
  from, or the source that says so. "`auth.enabled: true` already set in
  the HelmRelease values" carries its proof; "auth checked" does not.
- What the repository cannot show is not a check: say what could not be
  verified and what to look at after the merge.

---

Adapted from `.agents/skills/reviewing-pull-requests` in
[onedr0p/home-ops](https://github.com/onedr0p/home-ops) (WTFPL), with the
repository-specific paths and tooling generalized.
