# Implementation Plan: Rollback of Upstream Automated CI/Release Infrastructure

## 1. Context and Objective

In the latest merge from upstream (`main`), Astral's automated release infrastructure was merged into this repository. Because this repository is a **fork** (`Psychosoc1al/ruff-vscode-python-3.7`) that maintains Python 3.7+ backport support and relies on **manual publishing**, the upstream automated release system does not work here:

- Upstream relies on Astral's internal **Secure Token Service (STS)** (`secrets.RELEASES_STS_API_URL` and `astral-sh/github-actions/simple-token-service`) which does not exist in this fork and is hardcoded to Astral repository IDs.
- Upstream introduced bot-specific automated PR and merge workflows (`astral-releases-bot[bot]`, `release/auto/*` branches).
- Upstream changed the release workflow to depend on publishing to the official VS Code Marketplace (`publish-code-marketplace`), which requires Astral publisher credentials.
- Upstream extracted GitHub release and tag creation into a separate STS-authenticated reusable workflow (`release-tag.yml`), breaking standard `${{ secrets.GITHUB_TOKEN }}` tagging.

**Goal**: Roll back the incompatible upstream automated release workflows and STS configurations to restore clean manual release preparation and dispatching in this fork, while retaining legitimate CI improvements (such as `UV_NO_BUILD: "1"`, Node.js updates, and package bumps).

---

## 2. Proposed Changes

### A. Files to Delete

| File | Reason |
| --- | --- |
| `.github/secure-token-service-release.json` | Astral internal STS configuration; invalid for this fork. |
| `.github/workflows/release-on-merge.yml` | Bot merge listener hardcoded to `astral-sh/ruff-vscode` and `astral-releases-bot[bot]`. |
| `.github/workflows/release-tag.yml` | Reusable workflow requiring `RELEASES_STS_API_URL` to tag releases. |

### B. Files to Modify

#### 1. `.github/workflows/release.yml`
- **Revert `publish-github` job**:
  - Remove invocation of reusable workflow `release-tag.yml`.
  - Restore direct inline GitHub release and tag creation using `gh release create` and `${{ secrets.GITHUB_TOKEN }}`.
  - Restore job permissions: `contents: write`.
  - Revert `needs` list to `["validate-release", "publish-openvsx"]` (omitting `publish-code-marketplace`).
- **Simplify `workflow_dispatch` inputs and target commit resolution**:
  - Restore `version` as a required input (`required: true`).
  - Remove `release-pr` and `expected-sha` inputs.
  - In `validate-release`, remove the `steps.source` block checking for `astral-releases-bot[bot]` and instead use `${{ github.sha }}` / `${{ inputs.version }}` directly.

#### 2. `.github/workflows/release-prepare.yml`
- **Revert credentials & permissions**:
  - Restore top-level job permissions: `contents: write`, `pull-requests: write`.
  - Remove `id-token: write`.
  - Delete step `Get release PR token` (`simple-token-service`).
- **Remove automated promotion and bot logic**:
  - Remove `auto-promote` boolean input.
  - Remove `Check for a conflicting release PR` step.
  - Remove `Verify protection for the automated release branch` step.
  - Remove `Open automated release PR and enable auto-merge` step.
- **Restore standard manual PR creation**:
  - Restore `create-release-pr` action step using default permissions (`inputs.version`).
  - Make `version` input required (or keep `--plan-version` fallback as a convenience, while keeping execution fully manual without bot hooks).

#### 3. `CONTRIBUTING.md`
- Revert release documentation section back to the 4-step manual release workflow suitable for this fork, removing mentions of `astral-releases-bot`, branch rulesets for `release/auto/*`, and automated promotion.

---

## 3. Files to Keep As-Is (Do Not Roll Back)

- `.github/workflows/ci.yml`: Keep `UV_NO_BUILD: "1"` and Node `26.12.1`.
- `.github/renovate.json5`: Keep the Renovate schema update (`dependencyDashboard: { enabled: true }`).
- `pyproject.toml`, `package.json`, `package-lock.json`, `uv.lock`: Keep the updated version (`2026.98.0`) and bundled Ruff version (`0.17.6`).

---

## 4. Verification and Acceptance Criteria

1. **Clean Workflow Definitions**:
   - No occurrences of `RELEASES_STS_API_URL`, `simple-token-service`, or `secure-token-service-release.json` remain in `.github/`.
   - No occurrences of `astral-releases-bot` or `release/auto/` remain in workflow files.
2. **Workflow Syntax & Lint Validation**:
   - Run linter / syntax checks on YAML workflow files.
3. **Manual Dispatch Readiness**:
   - Both `release-prepare.yml` and `release.yml` can be invoked manually via `workflow_dispatch` using only standard `GITHUB_TOKEN` and configured repository secrets (`OPENVSX_TOKEN` / OpenVSX trusted publishing).
4. **Git Tree Status**:
   - All unintended files removed and changes committed cleanly.
