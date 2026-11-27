# Reading a registry

Read a container image or an OCI chart from its registry rather than
guessing from a tag. `crane` reads registries anonymously and prints JSON;
`helm` covers charts when `crane` is not installed.

## crane

The `crane` commands below come from the upstream skill this one was
adapted from and were not run here.

- `crane ls <registry>/<repo>` lists tags. `sha256-…` tags are signatures
  and attestations, not versions.
- `crane digest <image>:<tag>` resolves a tag to its digest.
- `crane manifest <image>:<tag>` shows a multi-platform image's platforms.
  A new tag missing an architecture the cluster's nodes run is a finding.
- `crane config --platform linux/amd64 <image>@<digest>` shows when the
  image was built (`created`) and its labels:
  `org.opencontainers.image.source` names the upstream project, and
  `org.opencontainers.image.revision` the commit it was built from. For an
  image rebuilt by a third-party packaging repository, that revision is a
  commit in the packaging repository, not the app's: its diff says why a
  digest-only update was rebuilt.
- `crane config <chart>:<version>` on an OCI chart prints its `Chart.yaml`
  as JSON: `version`, `appVersion` and `sources`, at the old and the new
  tag.

## helm

Works for OCI charts and, with `--repo <url>`, for classic Helm
repositories. Helm prints `Pulled:` and `Digest:` lines on stderr; drop
them when diffing.

```
helm show chart  oci://<registry>/<path>/<chart> --version <version> 2>/dev/null
helm show values oci://<registry>/<path>/<chart> --version <version> 2>/dev/null
```

`show chart` gives `version`, `appVersion`, `kubeVersion`, `sources` and
`home`. Diff `show values` at the old and new version to find renamed or
removed values and changed default images:

```
diff <(helm show values oci://<registry>/<path>/<chart> --version <old> 2>/dev/null) \
     <(helm show values oci://<registry>/<path>/<chart> --version <new> 2>/dev/null)
```
