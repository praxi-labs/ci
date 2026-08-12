# CI

Shared GitHub Actions workflows for the repositories in this organization.

Every repository here builds something that other people install: SDKs, a GitHub Action, an MCP server, an editor extension. What runs in these pipelines is part of the supply chain we ask customers to trust, so the pipelines are treated as production code rather than as glue.

This repository holds that pipeline once. Repositories call it instead of copying YAML, which means a fix to how we build reaches every project at the same time and none of them quietly drift.

## Using it

A repository adds a small caller workflow. Everything else lives here.

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

See [adopting](docs/adopting.md) for the full set, and [examples](examples/) for a working caller per language.

## What runs

| Workflow | Runs on | Purpose |
| --- | --- | --- |
| `node.yml` | push, pull request | Install, lint, typecheck, test across a Node matrix, then pack and inspect the tarball. |
| `python.yml` | push, pull request | Test across a Python matrix, lint with ruff, build and check the distributions. |
| `codeql.yml` | push, pull request, weekly | Static analysis into GitHub code scanning. |
| `dependency-review.yml` | pull request | Blocks a pull request that introduces a vulnerable or wrongly licensed dependency. |
| `workflow-lint.yml` | push, pull request | actionlint for correctness, zizmor for workflow security. |
| `scorecard.yml` | weekly | OSSF Scorecard, published as a badge. |
| `audit.yml` | daily | Sweeps installed dependencies for newly published advisories. |
| `npm-publish.yml` | tag | Publishes to npm with provenance, over OIDC. |
| `pypi-publish.yml` | tag | Publishes to PyPI with attestations, over OIDC. |

## The decisions worth knowing

### Actions are pinned to a commit, not a tag

A tag is a movable pointer. Whoever controls an action's repository can repoint `v4` at any commit, and every workflow that references `v4` runs it on the next push, with whatever secrets that job holds. This is not hypothetical: it is how the `tj-actions/changed-files` compromise reached tens of thousands of repositories in 2025.

Every third-party action here is referenced by full commit SHA, with the human readable version in a trailing comment:

```yaml
uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

A SHA cannot be repointed. Dependabot raises the pin on a schedule, so the cost is a reviewable pull request rather than a manual audit. `self-check.yml` fails if a pin ever loses its version comment, since a bare SHA nobody can read is a pin nobody will review.

### Nothing gets a token it does not need

Every workflow opens with `permissions: {}` and grants back only what a given job uses. Most jobs get `contents: read` and nothing else. Code scanning uploads get `security-events: write`. Publishing gets `id-token: write` and no more.

The reason is blast radius. A test job runs your dependency tree's install scripts, and a compromised dependency inherits whatever that job's token can do. A token that can only read the repository is worth far less to an attacker than the default write token.

Checkouts also pass `persist-credentials: false`. Without it, the token is written into `.git/config` and stays readable to every later step in the job, including ones that run third-party code.

### Publishing uses OIDC, not a stored token

Neither publish workflow holds a registry credential. npm and PyPI both authenticate through short-lived OIDC tokens minted per run and scoped to this repository and workflow.

A long-lived `NPM_TOKEN` in organization secrets is a credential that works from anywhere, forever, for anyone who can trigger a workflow that reads it. An OIDC token is minted for one run, expires in minutes, and cannot be replayed from somewhere else.

Both publishes emit provenance, so anyone installing a package can verify which commit and which workflow built it. We ask customers to check provenance on what they depend on, so our own releases carry it.

Both are also gated on a `release` environment, which is where a required reviewer belongs. That makes cutting a release a deliberate act rather than a side effect of pushing a tag.

### Pull requests block on what the change introduces

New advisories land daily against dependencies nobody touched. A pipeline that fails on every known finding fails on days when nothing about the code moved, and a check that fails for reasons the author cannot act on is a check that gets switched off.

So the two are separated. `dependency-review.yml` runs on pull requests and blocks only on what that diff introduces, which is always actionable by the person who opened it. `audit.yml` runs on a schedule against the whole tree and catches advisories in dependencies that did not change, where the right response is a tracked upgrade rather than a red mark on an unrelated pull request.

### The workflows are themselves linted

`actionlint` catches the errors YAML will happily accept, such as a misspelled context or an expression that silently evaluates to empty. `zizmor` catches the security ones, including template injection, over-broad permissions and unpinned actions.

actionlint is installed from a pinned release and verified against a recorded SHA256 before it runs. Piping a download straight into a shell would undermine the point of the job doing the checking.

## Versioning

Call these workflows by the major tag, `@v1`. It moves forward across compatible changes and never across a breaking one.

Pinning to `@main` means an edit here changes what runs in every repository with no review anywhere. Pinning to a full SHA is the strictest option and is reasonable for a repository that wants to move deliberately, at the cost of a pull request per update.

## Contributing

Changes are validated by `self-check.yml`, which lints these workflows with the same tooling they apply to everyone else and verifies that every pin is documented.

Test a change against a real repository before tagging it. Point one caller at the branch, confirm it behaves, then move the major tag:

```sh
git tag -fa v1 -m "v1"
git push --force origin v1
```

## License

MIT
