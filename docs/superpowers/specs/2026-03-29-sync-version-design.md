# Sync Version Design

Date: 2026-03-29

## Goal

Ensure future fork releases and ARM binaries report the same semantic version.

For a synced upstream release tag `vX.Y.Z`, the fork should:

- publish release tag `vX.Y.Z-andro`
- build binaries that report `openfang X.Y.Z`

## Current Problem

`Sync Upstream Release` uses the upstream release tag and body to create the fork tag and release, but it does not update the Rust workspace version in `Cargo.toml`.

As a result:

- release/tag can say `v0.5.5-andro`
- built CLI still reports `0.5.1`

## Design

The upstream release tag is the single source of truth for versioning.

When the workflow resolves the latest upstream release:

1. Read upstream tag `vX.Y.Z`
2. Derive Cargo version `X.Y.Z` by stripping the leading `v`
3. During the sync commit, update `workspace.package.version` in the root `Cargo.toml`
4. Verify the updated file contains the expected version
5. Commit that version bump together with the upstream merge before creating the fork tag

## Workflow Changes

The `Sync Upstream Release` workflow should add:

- an output for the plain Cargo version derived from the upstream tag
- a step in the sync section that rewrites the root `Cargo.toml` version
- a validation check that fails if the version did not update as expected

## Behavior Rules

- The workflow should only touch the root workspace version field
- The version must always match the upstream release tag without the leading `v`
- If the upstream tag is not in the expected `vX.Y.Z` form, the workflow should fail rather than guessing
- The version update must happen before the fork tag is created

## Testing

Minimum regression coverage:

- a workflow-level validation step proving the expected version string is present in `Cargo.toml`
- manual verification after future sync runs:
  - release tag is `vX.Y.Z-andro`
  - `openfang --version` reports `X.Y.Z`

## Risks

- If upstream changes release tag format, the sync workflow will fail until adapted
- If version text replacement is too broad, it could edit the wrong field; the update must target only `workspace.package.version`

## Recommendation

Implement the version rewrite in `Sync Upstream Release` and fail fast on malformed tags or mismatched version state. This keeps future releases correct without requiring manual version bumps.
