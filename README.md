# workflows

Shared GitHub Actions reusable workflows for `yoshintame/*` repositories.

| Workflow | Purpose | Project requirements |
|---|---|---|
| `ci.yml` | `hk check --all --slow` | `mise.toml` with `hk` and the step tools, `hk.pkl`; `bun.lock` or `uv.lock` is installed when present |
| `ci-nix.yml` | `nix flake check -L` | `flake.nix` exposing `checks` (treefmt-nix, git-hooks.nix, own checks) |
| `release-semantic.yml` | semantic-release for a single-package repo | `mise.toml` with `bun`, `semantic-release` in `devDependencies` |
| `release-changesets.yml` | Changesets v3 publish without a Version PR | `mise.toml` with `bun`, `@changesets/cli` v3 in `devDependencies`; versions are bumped locally with `changeset version` |

Both release workflows take `registry`: `npm` (npmjs.com, OIDC trusted publishing) or `github` (GitHub Packages via `GITHUB_TOKEN`).

## Usage

One caller file per repository, always named `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions: {}

jobs:
  verify:
    uses: yoshintame/workflows/.github/workflows/ci.yml@<sha> # vX.Y.Z
    permissions:
      contents: read

  release:
    needs: verify
    if: github.event_name == 'push'
    uses: yoshintame/workflows/.github/workflows/release-semantic.yml@<sha> # vX.Y.Z
    permissions:
      contents: write
      issues: write
      pull-requests: write
      id-token: write
```

- Permissions are granted by the caller; release jobs inherit them, check jobs narrow them to `contents: read`.
- `release-changesets.yml`: grant `contents: write` plus `id-token: write` (npm) or `packages: write` (`registry: github`).
- npm trusted publisher is registered on the caller file (`ci.yml`) of the package repository, not on this repository.
- Reference workflows by full commit SHA with the version comment; Renovate updates both.

## Versioning

Releases are `vX.Y.Z` tags. A breaking change to inputs, required permissions or project requirements is a major version.

## Checks

`mise install && hk check --all` runs actionlint and zizmor on this repository. zizmor's `self-repository` rule is disabled until actionlint accepts the `$/` self-repository syntax.
