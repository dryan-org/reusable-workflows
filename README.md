# reusable-workflows

Shared GitHub Actions workflows.

## `dependabot-ci.yml` — Dependency CI

A generic `workflow_call` CI job for repos that don't have their own. Detects
npm / uv-Python / cargo and runs install → build → test → audit. Install and the
project's own build/test scripts **gate** the check; audits are advisory (a
dependency bump that only partially clears alerts still needs to be mergeable).

Its purpose is to give the [Dependabot autofix
pipeline](https://github.com/dryan/infrastructure/blob/main/dependabot-autofix-setup.md)
a real required check so its PRs can use GitHub native auto-merge instead of
being merged directly by hofn.

### Use it

Add `.github/workflows/ci.yml` to the target repo:

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main, master]
jobs:
  ci:
    uses: dryan-org/reusable-workflows/.github/workflows/dependabot-ci.yml@v1
```

The status check appears as **`ci / build`**. Pin the ruleset's required check
to that.

Inputs (all optional): `node-version` (default `lts/*`), `python-version`
(default `3.13`).

### Repos with infra-dependent tests

If a repo's test suite needs a database, secrets, or services the generic job
can't provide, don't use this stub — give the repo a real `ci.yml` of its own
and point the ruleset at that check instead.

## Versioning

Callers pin `@v1`. The `v1` tag moves forward for backward-compatible changes;
breaking changes get `@v2`.
