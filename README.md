# reusable-workflows

Shared GitHub Actions workflows.

## `dependabot-ci.yml`: Dependency CI

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

Secrets are passed by name, only by repos that need them: `ci_read_app_key`,
`npm_token`, `github_packages_token`. Callers in dryan-org may use
`secrets: inherit` instead (it does not cross orgs).

The status check appears as **`ci / build`**. Pin the ruleset's required check
to that.

Inputs (all optional): `node-version` (default `lts/*`), `python-version`
(default `3.13`).

### Private dryan-org git dependencies

Give the repo the secret `CI_READ_APP_KEY` (private key of the read-only
`dryan-ci-read` App, installed on dryan-org) and pass it:

```yaml
secrets:
  ci_read_app_key: ${{ secrets.CI_READ_APP_KEY }}
```

The workflow mints a read-only token and routes `git@github.com:dryan-org/`,
`ssh://git@github.com/dryan-org/` and `https://github.com/dryan-org/` through
it, so private git dependencies (npm, uv, cargo) and submodules in dryan-org
work from a repo in any org. The key reaches only the token step, never build
or test code. Checkout runs with `persist-credentials: false` so its
`GITHUB_TOKEN` header does not override the App token.

The `ssh_key` secret (ssh-agent with a deploy key) is **deprecated** and will
be removed.

### Repo setup hook

If the repo has `.github/actions/ci-setup/action.yml`, it runs after checkout
and the App token step, before submodule init. Composite actions cannot read
secrets, so the workflow passes the secrets it received as the JSON input
`secrets`; read one with `fromJSON(inputs.secrets).NAME`. A secret only the
setup action uses needs `secrets: inherit` (dryan-org callers).

```yaml
name: CI setup
inputs:
  secrets:
    description: The secrets the workflow received, as JSON
    required: true
runs:
  using: composite
  steps:
    - shell: bash
      env:
        FONTAWESOME_TOKEN: ${{ fromJSON(inputs.secrets).FONTAWESOME_TOKEN }}
      run: |
        npm config set "@fortawesome:registry" https://npm.fontawesome.com/
        npm config set "//npm.fontawesome.com/:_authToken" "$FONTAWESOME_TOKEN"
```

### Repos with infra-dependent tests

If a repo's test suite needs a database, secrets, or services the generic job
can't provide, don't use this stub; give the repo a real `ci.yml` of its own
and point the ruleset at that check instead.

## Versioning

Callers pin `@v1`. The `v1` tag moves forward for backward-compatible changes;
breaking changes get `@v2`.
