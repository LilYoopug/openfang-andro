# Upstream ARM Release Sync Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Automate this fork so new upstream releases sync into `main`, preserve fork-owned files, and publish only the ARM native Linux artifact under `vX.Y.Z-andro` tags.

**Architecture:** Add one orchestration workflow that polls upstream release metadata, merges `upstream/main`, restores protected files, pushes `main`, and creates the fork tag/release. Reuse the existing ARM build workflow for tag-driven artifact publishing, and remove the multi-platform release workflow from the automated tag path.

**Tech Stack:** GitHub Actions, `actions/checkout`, `actions/github-script`, shell scripting, existing ARM build workflow, POSIX shell installer

---

## Chunk 1: Spec And Release Flow

### Task 1: Add the upstream sync orchestrator workflow

**Files:**
- Create: `.github/workflows/sync-upstream-release.yml`
- Modify: `.github/workflows/build-openfang-full-arm.yml`
- Modify: `.github/workflows/release.yml`

- [ ] **Step 1: Write the failing configuration expectation**

Define the desired behavior in the workflow itself:
- hourly `schedule`
- `workflow_dispatch`
- detect upstream release
- skip if `vX.Y.Z-andro` already exists
- merge and restore protected files
- create or update the fork release

- [ ] **Step 2: Verify the old configuration is insufficient**

Run: `rg -n 'tags:|workflow_dispatch|schedule|v\\*-andro|v\\*' .github/workflows`
Expected: no orchestrator workflow exists, ARM build triggers on all `v*`, and release workflow still triggers on all `v*`

- [ ] **Step 3: Implement the orchestrator workflow**

Create `.github/workflows/sync-upstream-release.yml` with:
- `permissions: actions: write, contents: write`
- `schedule: '0 * * * *'`
- `workflow_dispatch`
- steps to fetch upstream, query latest upstream release via GitHub API, merge `upstream/main`, restore `.github/workflows`, `install.sh`, and `README.md`, push `main`, create `vX.Y.Z-andro`, create/update the fork release, and dispatch the ARM build workflow explicitly

- [ ] **Step 4: Narrow the automated release path**

Update:
- `.github/workflows/build-openfang-full-arm.yml` to support `workflow_dispatch` with a `release_tag` input, trigger on `v*-andro`, and package only the `openfang` binary
- `.github/workflows/release.yml` to manual-only so it no longer runs on `v*-andro`

- [ ] **Step 5: Verify the new workflow wiring**

Run: `rg -n 'schedule:|workflow_dispatch|v\\*-andro|push:' .github/workflows/sync-upstream-release.yml .github/workflows/build-openfang-full-arm.yml .github/workflows/release.yml`
Expected: orchestrator has hourly schedule, ARM build watches `v*-andro`, and `release.yml` no longer watches tag pushes

## Chunk 2: Fork-Owned Files

### Task 2: Align the installer with the ARM-only release artifact

**Files:**
- Modify: `install.sh`

- [ ] **Step 1: Define the failing expectation**

Current installer still supports non-ARM targets and points at another fork. The new behavior must only allow Linux ARM64 and download `openfang-aarch64-unknown-linux-gnu-native.tar.gz` from this repository.

- [ ] **Step 2: Verify the old installer behavior**

Run: `rg -n 'REPO=|x86_64|Darwin|openfang-\\$\\{_target\\}\\.tar\\.gz' install.sh`
Expected: current script references another repo and non-ARM targets

- [ ] **Step 3: Implement the installer simplification**

Update `install.sh` so it:
- points to `LilYoopug/openfang-andro`
- accepts only Linux `aarch64|arm64`
- downloads `openfang-aarch64-unknown-linux-gnu-native.tar.gz`
- emits clear errors for unsupported environments

- [ ] **Step 4: Verify syntax and intent**

Run: `sh -n install.sh && rg -n 'LilYoopug/openfang-andro|aarch64-unknown-linux-gnu-native|Unsupported OS|Unsupported architecture' install.sh`
Expected: shell parses cleanly and only ARM-native path remains

### Task 3: Rewrite the README positioning for the fork

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Define the failing expectation**

The README currently reads like a general-purpose fork. It needs to explicitly say this repo is for Linux AArch64 users and explain the OpenSSL-related compatibility problem motivating the fork.

- [ ] **Step 2: Verify the current README language**

Run: `rg -n 'Self experimenting|curl -fsSL|Windows|Darwin|Linux AArch64|OpenSSL' README.md`
Expected: current copy does not clearly position the fork as ARM-only

- [ ] **Step 3: Implement the README changes**

Update the top sections so they clearly state:
- this fork targets Linux AArch64 only
- it publishes only the ARM native artifact
- it exists for systems affected by the OpenSSL compatibility issue the user described
- install instructions use this fork's installer/release path

- [ ] **Step 4: Verify the README messaging**

Run: `rg -n 'Linux AArch64|ARM|OpenSSL|native artifact|openfang-andro' README.md`
Expected: the README now clearly describes the fork purpose and scope

## Chunk 3: Final Verification

### Task 4: Run repository-level verification

**Files:**
- Modify: `.github/workflows/sync-upstream-release.yml`
- Modify: `.github/workflows/build-openfang-full-arm.yml`
- Modify: `.github/workflows/release.yml`
- Modify: `install.sh`
- Modify: `README.md`

- [ ] **Step 1: Run whitespace and patch sanity checks**

Run: `git diff --check`
Expected: no output

- [ ] **Step 2: Run shell syntax verification**

Run: `sh -n install.sh`
Expected: exit 0

- [ ] **Step 3: Run workflow syntax validation**

Run: `python3 - <<'PY'\nimport pathlib, sys\nimport yaml\nfor path in map(pathlib.Path, [\n    '.github/workflows/sync-upstream-release.yml',\n    '.github/workflows/build-openfang-full-arm.yml',\n    '.github/workflows/release.yml',\n]):\n    with path.open() as fh:\n        yaml.safe_load(fh)\nprint('ok')\nPY`
Expected: prints `ok`

- [ ] **Step 4: Inspect final diff scope**

Run: `git diff --stat`
Expected: only the intended workflow, installer, README, and spec/plan docs changed
