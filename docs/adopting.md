# Adopting

How to wire a repository into the shared pipeline, and every input each workflow accepts.

## The caller

Callers live in the consuming repository at `.github/workflows/ci.yml`. They should stay small. Anything that looks like build logic belongs here instead, so every repository gets the fix at once.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions: {}

jobs:
  node:
    uses: praxi-labs/ci/.github/workflows/node.yml@v1
    permissions:
      contents: read
```

Two lines are doing real work.

`concurrency` cancels a superseded run when someone pushes again to the same pull request, so a branch under active development does not queue four runs of the same suite. It is scoped to `cancel-in-progress` on pull requests only, because cancelling a run on `main` would leave the default branch without a result.

`permissions: {}` at the top of the caller matters because a called workflow can never hold more than the caller grants. Declaring the empty set and granting per job means a new job cannot silently inherit write access.

## node.yml

| Input | Type | Default | Notes |
| --- | --- | --- | --- |
| `node-versions` | string | `'["20", "22", "24"]'` | JSON array. Should cover the floor in your `engines` field. |
| `runners` | string | `'["ubuntu-latest"]'` | JSON array. Add `windows-latest` where paths matter. |
| `working-directory` | string | `.` | Directory holding `package.json`. |
| `primary-node-version` | string | `22` | Used for the single-run packaging job. |
| `ignore-scripts` | boolean | `false` | Install without running dependency lifecycle hooks. |
| `upload-package` | boolean | `true` | Upload the packed tarball for inspection. |
| `committed-build-script` | string | none | npm script that regenerates a build output committed to the repository. |
| `timeout-minutes` | number | `15` | |

Lint, typecheck and build run through `npm run <script> --if-present`, so a repository without a `lint` script is not forced to invent one. `npm test` is not optional.

The packaging job runs `npm pack` and prints the tarball contents. Reading that list is how you notice a source map, a fixture directory or a `.env` that was about to ship, which is far cheaper before publication than after.

### Committed build output

A JavaScript Action runs the bundle straight from the repository, so `dist/` is committed and can silently fall behind its source. Set `committed-build-script` and CI rebuilds it and fails if the result differs:

```yaml
jobs:
  node:
    uses: praxi-labs/ci/.github/workflows/node.yml@v1
    with:
      committed-build-script: package
    permissions:
      contents: read
```

Leave it unset for a normal library, where nothing built is checked in.

### Install scripts

`ignore-scripts: true` blocks dependency lifecycle hooks at install. It is the stronger default, because a postinstall hook is arbitrary code from a transitive dependency running with the job's token.

It is off by default only because packages that compile a native addon at install time break without it. Turn it on for any repository whose dependencies are pure JavaScript.

## python.yml

| Input | Type | Default | Notes |
| --- | --- | --- | --- |
| `python-versions` | string | `'["3.10", "3.11", "3.12", "3.13"]'` | JSON array. |
| `runners` | string | `'["ubuntu-latest"]'` | JSON array. |
| `working-directory` | string | `.` | Directory holding `pyproject.toml`. |
| `primary-python-version` | string | `3.12` | Used for the lint and build jobs. |
| `dev-extra` | string | `dev` | Optional dependency group installed for tests. |
| `upload-distributions` | boolean | `true` | Upload the sdist and wheel. |
| `timeout-minutes` | number | `15` | |

The build job runs `twine check --strict`, which catches the metadata problems PyPI rejects at upload. Finding them here rather than at release time means a failed publish does not leave a version number burned.

It also lists the contents of both the sdist and the wheel, for the same reason the Node job lists the tarball.

## codeql.yml

| Input | Type | Default | Notes |
| --- | --- | --- | --- |
| `languages` | string | required | JSON array, for example `'["javascript-typescript"]'`. |
| `queries` | string | `security-extended` | Query suite. |
| `build-mode` | string | `none` | `none` for interpreted languages. |
| `timeout-minutes` | number | `30` | |

Run it on a schedule as well as on pull requests. The query suites are updated independently of your code, so a weekly run finds issues in code that has not changed.

## dependency-review.yml

| Input | Type | Default | Notes |
| --- | --- | --- | --- |
| `fail-on-severity` | string | `high` | Lowest severity that fails. |
| `allow-licenses` | string | none | Comma separated SPDX identifiers. Empty disables the gate. |
| `comment-summary-in-pr` | string | `on-failure` | |
| `timeout-minutes` | number | `10` | |

Only runs on `pull_request`, since it compares a base against a head and has nothing to compare on a push.

The license gate is off by default because a wrong list fails builds for reasons the author cannot fix. Turn it on deliberately, with a list your legal position actually supports.

## workflow-lint.yml

| Input | Type | Default | Notes |
| --- | --- | --- | --- |
| `zizmor-persona` | string | `regular` | `regular`, `pedantic` or `auditor`. |
| `timeout-minutes` | number | `10` | |

Worth running in any repository with workflows of its own, not only this one.

## scorecard.yml

| Input | Type | Default | Notes |
| --- | --- | --- | --- |
| `publish-results` | boolean | `true` | Required for the badge. |
| `timeout-minutes` | number | `15` | |

Schedule it weekly. It is slow, it does not depend on your commits, and running it per push adds nothing.

Publishing exposes the score and the checks behind it, nothing about the code.

## audit.yml

| Input | Type | Default | Notes |
| --- | --- | --- | --- |
| `ecosystem` | string | required | `npm` or `pypi`. |
| `working-directory` | string | `.` | |
| `audit-level` | string | `high` | npm severity floor. |
| `node-version` | string | `22` | |
| `python-version` | string | `3.12` | |
| `timeout-minutes` | number | `10` | |

Schedule it daily and keep it off pull requests. Its whole purpose is finding advisories in dependencies nobody touched, which is not something a pull request author can act on.

## Release workflows

Covered in [releasing](releasing.md), including the one-time trusted publishing setup on npm and PyPI.

## Recommended set

For a published library:

| Workflow | Trigger |
| --- | --- |
| `node.yml` or `python.yml` | push to `main`, pull request |
| `dependency-review.yml` | pull request |
| `codeql.yml` | push to `main`, pull request, weekly |
| `workflow-lint.yml` | push to `main`, pull request |
| `audit.yml` | daily |
| `scorecard.yml` | weekly |
| `npm-publish.yml` or `pypi-publish.yml` | tag matching `v*` |

Working callers for both languages are in [examples](../examples/).

## Matrix and support floors

The matrix should cover what your package claims to support. A package declaring `engines: { "node": ">=18" }` while CI tests only Node 22 is making a promise nothing checks.

Where the declared floor is an end-of-life runtime, raise the floor rather than adding an end-of-life version to the matrix. Testing against a runtime that no longer receives security fixes gives an assurance nobody should rely on.
