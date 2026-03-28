# Upstream ARM Release Sync Design

## Goal

Automate this fork so it tracks new upstream OpenFang releases, syncs `upstream/main`
into `main`, preserves fork-owned files, and publishes only the
`openfang-aarch64-unknown-linux-gnu-native` artifact under fork-specific tags such
as `v0.5.5-andro`.

## Repository Context

This fork intentionally diverges from upstream in a small set of files that must not
be overwritten by automation:

- `.github/workflows/**`
- `install.sh`
- `README.md`

Everything else should follow upstream.

## Requirements

### Functional

1. Check for new upstream GitHub releases every hour and support manual runs.
2. If the latest upstream release tag is already mirrored as `vX.Y.Z-andro`, exit
   without changing the repository.
3. When a new upstream release exists:
   - fetch `upstream/main`
   - merge it into `main`
   - restore `.github/workflows/**`, `install.sh`, and `README.md` from the fork's
     pre-merge `HEAD`
   - push the merge commit to `origin/main`
   - create and push `vX.Y.Z-andro`
4. Publish a GitHub Release on the fork for `vX.Y.Z-andro` that copies the upstream
   release title/body.
5. Build and upload only `openfang-aarch64-unknown-linux-gnu-native.tar.gz` and its
   checksum for `vX.Y.Z-andro`.

### Non-Functional

1. Do not trigger the existing multi-platform `release.yml` on automated ARM tags.
2. Keep the automation idempotent and safe to rerun.
3. Fail before tagging if merge conflicts remain outside protected fork-owned files.
4. Use GitHub-provided credentials only, with least-necessary `contents: write`
   permission.

## Chosen Approach

Use a two-stage GitHub Actions flow:

1. A new orchestrator workflow polls upstream releases, performs the sync, restores
   fork-owned files, pushes `main`, and creates the `-andro` tag plus copied release.
2. The existing ARM build workflow is narrowed to react only to `v*-andro`, or to an
   explicit `workflow_dispatch` carrying a release tag input, build the ARM-native
   artifact, and upload assets into the release created by the sync job.

This keeps release orchestration separate from artifact building while still using
GitHub's normal tag-driven workflow execution.

## Workflow Design

### 1. `sync-upstream-release.yml`

Triggers:

- `schedule` hourly
- `workflow_dispatch`

High-level steps:

1. Checkout `main` with full history.
2. Configure git identity for the workflow actor.
3. Add or refresh `upstream` pointing at `https://github.com/RightNow-AI/openfang.git`.
4. Fetch `origin` and `upstream`.
5. Use the GitHub API to load the latest upstream release metadata:
   - upstream tag, for example `v0.5.5`
   - release name/title
   - release body
6. Compute the fork tag, for example `v0.5.5-andro`.
7. Exit early if `refs/tags/v0.5.5-andro` already exists on `origin`.
8. Capture the current fork-owned files from `HEAD`.
9. Merge `upstream/main` into `main` with `--no-ff --no-commit`.
10. Restore `.github/workflows/**`, `install.sh`, and `README.md` from the saved fork
    `HEAD`.
11. Commit the merge.
12. Push `main`.
13. Create the fork tag and push it.
14. Create or update a GitHub Release for `vX.Y.Z-andro` using the copied upstream
    title/body plus a small fork note indicating that this fork publishes only the ARM
    native Linux artifact.
15. Dispatch the ARM build workflow explicitly using `workflow_dispatch`, because
    pushes created with `GITHUB_TOKEN` do not trigger other workflows except
    `workflow_dispatch` and `repository_dispatch`.

### 2. `build-openfang-full-arm.yml`

Changes:

- Keep `workflow_dispatch`.
- Add an optional `release_tag` input for `workflow_dispatch`.
- Change tag trigger from `v*` to `v*-andro`.
- Keep the existing ARM Ubuntu runner and release packaging.
- Package only the `openfang` binary inside the archive.
- Upload the tarball and checksum to the already-created GitHub Release for the tag.

### 3. `release.yml`

Changes:

- Stop reacting to `v*-andro`.
- Make it manual-only so the file still exists for the fork but will not run during
  the automated ARM sync/release path.

## Installer and README

### `install.sh`

The installer should explicitly target this fork's ARM-native release artifact only.
It should:

- point to this fork's release page
- accept only Linux on `aarch64`/`arm64`
- download `openfang-aarch64-unknown-linux-gnu-native.tar.gz`
- fail clearly on unsupported OS/architecture combinations

### `README.md`

The README should explicitly describe the fork as:

- a Linux AArch64-specific fork
- intended for users hitting compatibility problems with newer OpenSSL-linked builds
- distributing only the native ARM Linux artifact from this repository

## Failure Handling

1. If upstream API lookup fails, the workflow should fail without changing the repo.
2. If the merge conflicts outside the protected files, the workflow should fail before
   commit/tag creation.
3. If `main` push succeeds but tag push fails, rerunning the workflow should detect the
   state and continue cleanly.
4. If the release already exists, the workflow should update it instead of failing.

## Verification

Local verification before claiming completion:

- `git diff --check`
- `sh -n install.sh`
- syntax validation for workflow YAML files via a parser/linter available in the
  environment
- inspect the rendered workflow logic for tag patterns and protected-file restore steps
