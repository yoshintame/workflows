# workflows

Shared GitHub Actions reusable workflows for `yoshintame/*` repositories.

| Workflow | Purpose | Project requirements |
|---|---|---|
| `ci.yml` | `hk check --all --slow --no-fail-fast` | `mise.toml` with `hk` and the step tools, `hk.pkl`; `bun.lock` / `bun.lockb` or `uv.lock` is installed when present; dependencies from GitHub Packages are not supported |
| `ci-nix.yml` | `nix flake check -L` | `flake.nix` exposing `checks` (treefmt-nix, git-hooks.nix, own checks) |
| `release-semantic.yml` | semantic-release for a single-package repo | `mise.toml` with `bun`, `semantic-release` in `devDependencies` |
| `release-changesets.yml` | Changesets v3 publish without a Version PR | `mise.toml` with `bun`, `@changesets/cli` v3 in `devDependencies`; versions are bumped locally with `changeset version` |

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

Reference workflows by full commit SHA with the version comment; Renovate updates both.

## Registries and permissions

The package's `publishConfig.registry` selects the registry; the workflows take no registry input. Release jobs set `GITHUB_TOKEN` for the whole job and write `//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}` to `~/.npmrc`, the same placeholder a project `.npmrc` uses, so every npm and bun call in the job authenticates to GitHub Packages and the token reaches no other registry. npm tries OIDC trusted publishing first, so the same workflow serves both registries.

Permissions are granted by the caller. Release jobs inherit them; check jobs narrow them.

| Workflow | npmjs.com | GitHub Packages |
|---|---|---|
| `release-semantic.yml` | `contents`, `issues`, `pull-requests`: write; `id-token: write` | `contents`, `issues`, `pull-requests`, `packages`: write |
| `release-changesets.yml` | `contents: write`, `id-token: write`, `packages: read` | `contents: write`, `packages: write` |

`release-changesets.yml` always needs `packages`: its `prepare` job requests `packages: read`.

npmjs.com trusted publishing:

- publish the first version manually: a trusted publisher can only be added to an existing package;
- register the trusted publisher with the caller file `ci.yml` of the package repository, not this repository;
- enable **Allow npm publish** on the trusted publisher: configurations created after 2026-09-03 allow only `npm stage publish` by default;
- `repository.url` in `package.json` must match the GitHub repository.

GitHub Packages: scoped package name and `publishConfig.registry: https://npm.pkg.github.com`.

## Changesets mode

`release-changesets.yml` publishes versions that are bumped and committed locally. Any changeset file in `.changeset` fails the run, empty changesets included: Changesets checks for unpublished versions only when `.changeset` holds no changesets, so a pending one would silently hold back an already bumped release.

## Concurrency

Release workflows use the concurrency group `release`. Do not reuse that group name in the caller: GitHub detects a deadlock between the caller and the called workflow.

## Isolation

`release-changesets.yml` builds and packs in `prepare` without `id-token` and publishes in a separate job. `release-semantic.yml` installs, builds (`prepack`) and publishes in one job with `id-token`, because semantic-release runs `npm publish` on the package directory.

## Nix inputs

`ci-nix.yml` accepts an optional `ssh_private_key` secret for private `git+ssh` flake inputs; the key needs read access to every such repository.

## Versioning

Releases are `vX.Y.Z` tags. A breaking change to inputs, required permissions or project requirements is a major version.

## Checks

`mise install && hk check --all` runs actionlint (with shellcheck for `run:` scripts) and zizmor on this repository.
