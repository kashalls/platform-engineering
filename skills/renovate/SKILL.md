---
name: renovate
description: Configure and debug Renovate dependency updates. Use when setting up renovate.json, grouping or scheduling update PRs, enabling automerge safely, reducing PR noise, pinning vs ranges, writing custom managers or regex managers for files Renovate does not understand natively, or when Renovate is not raising a PR you expected, raising too many, or updating the wrong thing.
---

# Renovate

Automated dependency updates. The whole game is **signal versus noise**: an
unconfigured Renovate produces so many PRs that the team ignores all of them,
including the security ones.

Config is validated against the published JSON schema. Reference it, and your
editor will complete and check options:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json"
}
```

Examples here are validated against **renovate-schema v44.39.1**.

## A starting config that is not noisy

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", ":dependencyDashboard"],
  "timezone": "Etc/UTC",
  "schedule": ["* 0-6 * * 1"],
  "prConcurrentLimit": 5,
  "minimumReleaseAge": "3 days",
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch", "digest"],
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "matchPackageNames": ["kubernetes/**"],
      "groupName": "kubernetes"
    },
    {
      "matchDatasources": ["docker"],
      "rangeStrategy": "replace"
    }
  ],
  "osvVulnerabilityAlerts": true
}
```

Each line earns its place:

- **`schedule`** batches PRs into a window instead of trickling them all week.
  It is cron-like but only the *time* fields are honored — Renovate runs when its
  scheduler runs, and `schedule` limits when it may **create** branches.
- **`prConcurrentLimit`** caps open PRs. Without it, onboarding a large repo opens
  dozens at once.
- **`minimumReleaseAge`** waits before proposing a release, which dodges the
  version that gets unpublished or hot-fixed hours later.
- **`dependencyDashboard`** puts everything Renovate *would* do in one issue, so
  suppressed updates stay visible instead of invisible.
- **`osvVulnerabilityAlerts`** uses osv.dev so security updates bypass your
  scheduling and grouping.

## Grouping

Grouping is the main noise lever. Update things together that must move together:

```json
{
  "packageRules": [
    {
      "matchPackageNames": ["@types/**"],
      "groupName": "type definitions",
      "automerge": true
    },
    {
      "matchDatasources": ["docker"],
      "matchUpdateTypes": ["minor", "patch"],
      "groupName": "container images"
    },
    {
      "matchManagers": ["github-actions"],
      "groupName": "github actions",
      "automerge": true
    }
  ]
}
```

Group when packages are released in lockstep (a framework and its plugins) or
when they are individually uninteresting (`@types/*`, linters, GitHub Actions).
Do **not** group unrelated majors — one bad update blocks the whole PR and you
lose the ability to merge the rest.

`matchUpdateTypes` accepts: `major`, `minor`, `patch`, `pin`, `pinDigest`,
`digest`, `lockFileMaintenance`, `rollback`, `bump`, `replacement`.

## Automerge

Automerge is safe in proportion to your test suite. Be honest about that.

```json
{
  "packageRules": [
    {
      "matchUpdateTypes": ["patch", "pin", "digest"],
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "matchUpdateTypes": ["major"],
      "dependencyDashboardApproval": true
    }
  ]
}
```

`automergeType` is one of `branch`, `pr` (the default), or `pr-comment`.

- `pr` opens a PR and merges it when checks pass — an audit trail, and branch
  protection still applies.
- `branch` pushes straight to the base branch when checks pass. No PR, no record.

**Automerge requires passing status checks to mean something.** A repo with no
CI automerges everything unconditionally. Start with `digest` and `patch` on
dev-only dependencies, and widen once you trust the suite.

`dependencyDashboardApproval: true` is the right treatment for majors: Renovate
lists them on the dashboard and only opens a PR after you tick the box.

## Pinning vs ranges

```json
{ "packageRules": [{ "matchDatasources": ["docker"], "rangeStrategy": "replace" }] }
```

`rangeStrategy` is one of `auto`, `pin`, `bump`, `replace`, `widen`,
`update-lockfile`, `in-range-only`.

- **Applications: pin exactly.** Reproducible builds, and Renovate becomes the
  single mechanism by which versions change.
- **Libraries: keep ranges.** Pinning a library's dependencies forces conflicts
  on your consumers.
- **Container images: pin by digest.** A tag is mutable; `digest` updates then
  become real, reviewable changes.

## When Renovate is not doing what you expect

1. **Read the Dependency Dashboard issue.** It lists detected updates, what is
   rate-limited, what is awaiting approval, and what errored. Most questions end
   here.
2. **Enable debug logging.** For the hosted app, add the repo to
   `logLevel: debug` via the dashboard; self-hosted, set `LOG_LEVEL=debug`.
3. **Check the config is actually valid**:

```bash
npx --yes --package renovate -- renovate-config-validator
```

4. **Check the file is being detected.** Renovate only manages files its managers
   recognize. A version string in a shell script or a Terraform local is invisible
   until you write a custom manager.

## Custom managers

For versions Renovate cannot find natively:

```json
{
  "customManagers": [
    {
      "customType": "regex",
      "managerFilePatterns": ["/^\\.github/workflows/.+\\.ya?ml$/"],
      "matchStrings": [
        "# renovate: datasource=(?<datasource>.*?) depName=(?<depName>.*?)\\s+.*?_VERSION: \"(?<currentValue>.*)\""
      ],
      "versioningTemplate": "semver"
    }
  ]
}
```

Then annotate the source:

```yaml
# renovate: datasource=github-releases depName=kubernetes/kubectl
KUBECTL_VERSION: "1.36.4"
```

The named capture groups `datasource`, `depName`, and `currentValue` are
required; `versioningTemplate` and `registryUrlTemplate` fill gaps the regex
cannot. Test the regex against the real file before committing — a
near-miss silently matches nothing rather than erroring.

## Gotchas

- **Removed options validate cleanly and do nothing.** `matchPackagePatterns` is
  no longer in the schema (superseded by `matchPackageNames` with glob support),
  but `packageRules` entries do not forbid unknown keys — so an outdated rule
  passes validation and silently never matches. Verified against v44.39.1.
  When a rule "does nothing", check the option still exists before debugging the
  pattern.
- **`config:recommended` is the current default preset.** Older material widely
  references `config:base`; if you are copying a config from a blog post, check
  the preset name against the current docs rather than assuming it still resolves.
- **`schedule` limits branch creation, not merging.** An already-open PR can
  automerge outside the window.
- **`extends` presets are merged, and order matters.** A later preset overrides
  an earlier one; your own settings override all of them only if they are outside
  `extends`.
- **`packageRules` are applied in order and all matching rules merge**, later
  winning. Put broad rules first and specific overrides last, or a general rule
  will clobber a specific one.
- **`enabled: false` on a rule stops updates silently.** It will not appear as a
  pending update anywhere except the dashboard.
- **Renovate cannot fix a lockfile it cannot generate.** If it lacks the right
  runtime or private registry credentials, it opens PRs that fail to build.
- **Vulnerability alerts need the platform's advisory data enabled** — on GitHub
  that means Dependency Graph and Dependabot alerts switched on for the repo.
