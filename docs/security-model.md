# Security model

What this pipeline defends against, how, and where the limits are.

CI is an attractive target. It holds credentials, it runs on every push, and it executes a great deal of third-party code: actions, dependencies, and install hooks. A build system that publishes packages is a way to reach everyone who installs them.

## Threats

### A compromised third-party action

An action is code from someone else's repository running inside our job, with our token. Referencing it by tag means the version we reviewed and the version that runs are only related by the goodwill of whoever can move the tag.

This is the most exercised path in practice. The `tj-actions/changed-files` compromise in 2025 worked exactly this way: tags across the action's history were repointed at a malicious commit, and every workflow referencing a tag began dumping runner memory into its logs on the next push.

**Mitigation.** Every third-party action is pinned to a full commit SHA. A SHA names one immutable object, so a repointed tag changes nothing for us. Dependabot proposes pin updates weekly, which turns the upgrade into a reviewable diff instead of a silent substitution.

**Limit.** Pinning fixes what code runs, not whether that code was safe when it was pinned. A malicious commit that we pin to is a malicious commit we run. Pin updates need the same review as any dependency bump, and the diff between two SHAs of an action is the thing to read.

### A compromised dependency

`npm ci` runs install hooks from the whole transitive tree, and tests import all of it. Any of that code runs with whatever the job's token can do.

**Mitigation.** Test jobs get `contents: read` and nothing more. A token that can only read a public repository is close to worthless. `ignore-scripts` is available and should be on wherever the dependency tree allows it, which removes install hooks from the picture entirely.

Publishing is separated from testing. The publish job installs and builds, but nothing that runs during a normal pull request has any path to a registry credential, because there is no registry credential to reach.

**Limit.** A dependency compromised in a way that survives to the published artifact is not caught here. That is what verification of the artifact itself is for, which is a different product.

### Credential theft from the runner

The default `GITHUB_TOKEN` is broad, and `actions/checkout` writes it into `.git/config` unless told otherwise. Every later step in the job can read it, including third-party actions and test code.

**Mitigation.** `permissions: {}` at the top of every workflow, granted back per job. `persist-credentials: false` on every checkout, so the token does not outlive the step that used it.

### Publishing without authorization

The classic failure is a stored registry token. It works from anywhere, it does not expire, and every workflow that can read it can publish.

**Mitigation.** There is no stored token. Publishing authenticates over OIDC, with a token minted per run and bound to the repository, workflow filename and environment. The registry validates those claims before accepting an upload, so a token leaked from a run cannot be replayed elsewhere.

The `release` environment adds a required human approval on the one step that cannot be undone. A published version number cannot be reused even after a yank.

**Limit.** OIDC binds to a workflow file in a repository. Anyone who can merge a change to that workflow can change what it publishes. Branch protection on the default branch is part of this control, not separate from it.

### Injection through workflow expressions

Interpolating `${{ github.event.pull_request.title }}` or a branch name directly into a `run:` block splices attacker-controlled text into a shell. A pull request titled with a shell metacharacter then executes.

**Mitigation.** Untrusted values are passed through `env:` and referenced as shell variables, which are never re-parsed as workflow syntax. `zizmor` runs over these workflows on every change and fails on template injection, over-broad permissions and unpinned actions.

### Untrusted code with elevated privileges

`pull_request_target` runs with write permissions and access to secrets, in the context of the base repository. Checking out the fork head under it hands both to a stranger.

**Mitigation.** Nothing here uses `pull_request_target`. Fork pull requests run under `pull_request`, without secrets, which is the correct default. Where a customer-facing workflow genuinely needs it, the pattern is documented in `phylax-workflows` and checks out the base commit only.

### A stale committed build output

A JavaScript Action runs the bundle committed to the repository, not the source. A bundle that has fallen behind its source means the reviewed code and the running code are different, which is a supply chain problem wearing a maintenance costume.

**Mitigation.** `committed-build-script` rebuilds the output in CI and fails if it differs from what is checked in.

## Deliberate omissions

**No runner egress filtering.** Tools that restrict runner network access are effective, and they work by inserting an agent into the trusted path of every job, usually reporting to a third-party service. That is a real dependency to add to a pipeline whose entire argument is about minimizing them. The tradeoff is worth revisiting for self-hosted runners, where the same control can be enforced at the network rather than inside the job.

**No self-hosted runners.** They accumulate state between jobs, which is a persistence foothold that ephemeral hosted runners do not offer.

**Audit does not gate pull requests.** Blocking a pull request on advisories in dependencies it did not touch produces failures the author cannot fix, and checks that cannot be acted on get disabled. Advisories in unchanged dependencies are swept on a schedule instead, where a tracked upgrade is the response.

## What this does not cover

This protects the pipeline that builds and publishes. It does not verify the artifacts themselves, it does not attest to what happens after installation, and it does not replace review of the code being built.

## Reporting

Report a vulnerability in these workflows privately through the security advisory form on this repository. Do not open a public issue.
