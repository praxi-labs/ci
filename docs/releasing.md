# Releasing

Both publish workflows authenticate over OIDC. No registry token is stored anywhere, in either repository secrets or organization secrets.

## Why no token

A long-lived `NPM_TOKEN` is a credential that works from any machine, for anyone who can read it, until someone remembers to rotate it. Every workflow that can read it can publish, and a compromised dependency running in a job that holds it can publish too.

OIDC replaces that with a token minted per run, valid for minutes, and bound to this repository, this workflow and this environment. It cannot be exfiltrated and reused elsewhere, because the registry checks the claims before accepting the upload.

This is also the property we ask customers to look for in what they install, so our own releases carry it.

## One-time setup on npm

Configure a trusted publisher on the package, at `npmjs.com/package/<name>/access`:

| Field | Value |
| --- | --- |
| Provider | GitHub Actions |
| Organization | `praxi-labs` |
| Repository | the publishing repository |
| Workflow | `release.yml` |
| Environment | `release` |

The workflow filename must match exactly. That claim is what stops a different workflow in the same repository from publishing.

Once configured, remove any `NPM_TOKEN` from repository and organization secrets. Leaving it in place keeps the weaker path open, which defeats the change.

## One-time setup on PyPI

Add a pending publisher at `pypi.org/manage/account/publishing/`, with the same four values. PyPI calls it a pending publisher until the first release, after which it binds to the project.

Then remove any `PYPI_API_TOKEN` from secrets.

## The release environment

Both workflows run in a `release` environment. Create it under repository settings and add a required reviewer.

Without a reviewer the environment is only a label. With one, pushing a tag opens an approval rather than shipping, so a mistaken or malicious tag does not reach a registry unattended. It is the one manual gate in the pipeline, and it sits at the only step that is irreversible: a published version number cannot be reused, even after a yank.

## Cutting a release

Version, tag, push.

```sh
npm version minor
git push --follow-tags
```

For Python, edit `phylax/version.py`, then:

```sh
git commit -am "Release 0.2.0"
git tag -a v0.2.0 -m "v0.2.0"
git push --follow-tags
```

The tag triggers the release workflow, which reruns the tests, builds, verifies the tag against the package version, and waits for approval.

## The version check

Both workflows refuse to publish when the tag and the package version disagree.

The failure it prevents is mundane and expensive: tagging `v0.2.0` against a `package.json` still reading `0.1.9` publishes `0.1.9` again, or fails halfway. Because a version number can never be reused on either registry, recovering means burning a version and explaining the gap.

For Python, pass `version-path` so the check has a file to read:

```yaml
jobs:
  publish:
    uses: praxi-labs/ci/.github/workflows/pypi-publish.yml@v1
    with:
      version-path: phylax/version.py
    permissions:
      contents: read
      id-token: write
```

## Prereleases

Publish a prerelease under a dist-tag so it does not become what `npm install` resolves to:

```yaml
with:
  dist-tag: next
```

For PyPI, point at TestPyPI first:

```yaml
with:
  repository-url: https://test.pypi.org/legacy/
```

TestPyPI needs its own trusted publisher, configured the same way.

## Verifying a release

Check the provenance on what actually shipped, rather than trusting that the pipeline did its job:

```sh
npm audit signatures
```

For PyPI, attestations appear on the release page and can be verified against the workflow identity that produced them.

If provenance is missing from a published version, something bypassed this pipeline. Treat that as an incident rather than as a formatting problem.
