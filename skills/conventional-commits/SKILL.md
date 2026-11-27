---
name: conventional-commits
description: Write and enforce Conventional Commits. Use when writing a commit message, choosing a type or scope, expressing a breaking change, setting up commitlint or a commit-msg hook, wiring semantic-release or release-please for automated versioning and changelogs, or deciding a team's commit conventions and PR title rules.
---

# Conventional Commits

A commit message convention that machines can parse, so version bumps and
changelogs stop being manual. Based on the [v1.0.0
spec](https://www.conventionalcommits.org/en/v1.0.0/).

## Structure

```
<type>[optional scope][!]: <description>

[optional body]

[optional footer(s)]
```

Concretely:

```
feat(api): add pagination to the accounts endpoint

Cursor-based, because offset pagination was scanning the whole table
on deep pages.

Refs: #482
```

## Types

The spec **requires only two**, and ties them to SemVer:

| Type | Meaning | SemVer |
|---|---|---|
| `feat` | A new feature | MINOR |
| `fix` | A bug fix | PATCH |

Everything else is convention, not spec. The widely used set:

| Type | Use for |
|---|---|
| `docs` | Documentation only |
| `refactor` | Neither fixes a bug nor adds a feature |
| `perf` | Improves performance |
| `test` | Adding or correcting tests |
| `build` | Build system or dependencies |
| `ci` | CI configuration and pipelines |
| `chore` | Maintenance with no src or test change |
| `revert` | Reverts a previous commit |
| `style` | Formatting only, no code meaning changed |

Agree the list once and enforce it in `commitlint`. A type nobody can define
(`chore` is the usual offender) becomes a dumping ground and the changelog stops
being useful.

## Breaking changes

Two ways, per the spec, and you may use both:

```
feat(api)!: remove the v1 accounts endpoint
```

```
feat(api): remove the v1 accounts endpoint

BREAKING CHANGE: /v1/accounts is gone. Use /v2/accounts.
```

Rules straight from the spec:

- **`BREAKING CHANGE` MUST be uppercase.** Everything else in a Conventional
  Commit is case-insensitive to implementors; this is the sole exception.
  `Breaking change:` is not recognized and your major bump will not happen.
- *"BREAKING-CHANGE MUST be synonymous with BREAKING CHANGE, when used as a token
  in a footer."* The hyphenated form is equally valid.
- *"If `!` is used, `BREAKING CHANGE:` MAY be omitted from the footer section, and
  the commit description SHALL be used to describe the breaking change."*
- A breaking change is MAJOR **regardless of type** — `fix!:` and `refactor!:`
  both trigger a major bump.

Prefer using both `!` and the footer: `!` is visible in `git log --oneline`, and
the footer has room to say what to do about it.

## Footers

Per the spec, a footer is *"a word token, followed by either a `:<space>` or
`<space>#` separator, followed by a string value"*. Tokens use hyphens in place
of spaces — `Reviewed-by`, not `Reviewed by` — with `BREAKING CHANGE` as the
documented exception.

```
Refs: #123
Reviewed-by: A. Developer
Co-authored-by: B. Developer <b@example.com>
BREAKING CHANGE: config key `timeout` renamed to `timeout_seconds`
```

## Scopes

Optional, and a noun for the affected area: `feat(auth):`, `fix(billing):`.

- Keep a **short, closed list** and enforce it (`scope-enum` in commitlint).
  Free-form scopes drift into synonyms and stop being groupable.
- In a monorepo, scope by package or service. That is what makes per-package
  changelogs possible.
- Skip the scope rather than inventing one for a change that spans everything.

## Writing the description

- **Imperative mood**: "add", not "added" or "adds". The test is that it completes
  *"if applied, this commit will ___"*.
- Lowercase start, no trailing period.
- Under ~72 characters.
- Say **what changed**, and use the body for **why**. The diff already shows how.

```
fix(auth): reject tokens issued before a password change      # good
fix(auth): Fixed the bug with tokens.                          # bad
```

The body is where the value is. Six months later, "why" is the only thing nobody
can reconstruct from the code.

## Enforcement

`commitlint` with the conventional config, run from a `commit-msg` hook:

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
echo "export default {extends: ['@commitlint/config-conventional']}" > commitlint.config.mjs
```

```yaml
# .github/workflows/commitlint.yaml — validate PR commits in CI too
- uses: actions/checkout@v4
  with:
    fetch-depth: 0        # --from needs the base commit in history
- run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to HEAD --verbose
```

Local hooks are bypassable with `--no-verify`, so enforce in CI as well. If you
squash-merge, the **PR title** becomes the commit message — lint that instead,
or the convention silently does nothing.

## Automated releases

Two mainstream tools; pick one.

- **semantic-release** — releases on every push to the release branch. Highest
  automation, least control over timing.
- **release-please** — opens a release PR that accumulates changes; merging it
  cuts the release. Better when you want a human to choose the moment.

Both derive the bump from types (`feat` → minor, `fix` → patch, breaking →
major) and build the changelog from descriptions. That is the payoff: the
convention only earns its cost if something consumes it.

Before adopting either, decide what happens to `chore` and `refactor` commits —
by default they produce no release at all, which surprises teams whose work is
mostly maintenance.

## Gotchas

- **`BREAKING CHANGE` in lowercase does nothing.** Silent, and you find out when
  the major release did not happen.
- **Squash merges discard individual commit messages.** Whatever ends up as the
  squash message is what tooling sees.
- **`revert` handling is tool-specific**; the spec does not standardize how a
  revert affects the version.
- **A `feat` that is really a fix inflates minor versions**, and vice versa. The
  type is a semantic claim, not a formality.
- **Retrofitting on an existing repo** means the first generated changelog covers
  only conventional commits. Set the tooling's starting tag deliberately.
