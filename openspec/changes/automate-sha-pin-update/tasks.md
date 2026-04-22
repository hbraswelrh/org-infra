## 1. Scaffold Workflow

- [x] 1.1 Create `.github/workflows/ci_update_sha_pins.yml` with `release: published` trigger, `name`, and `concurrency` group (`sha-pin-update`, cancel-in-progress: true)
- [x] 1.2 Add workflow-level `permissions: {}` and a single job `update-pins` running on `ubuntu-latest`

## 2. Job Permissions and Checkout

- [x] 2.1 Set job-level permissions: `contents: write`, `pull-requests: write`
- [x] 2.2 Add `actions/checkout` step (SHA-pinned) with `fetch-depth: 0` to access tags

## 3. Resolve Release SHA

- [x] 3.1 Add step to extract the release tag from `${{ github.event.release.tag_name }}`
- [x] 3.2 Add step to resolve the tag to its full 40-character commit SHA via `git rev-parse`
- [x] 3.3 Export `RELEASE_TAG` and `RELEASE_SHA` as step outputs for downstream steps

## 4. Find-and-Replace SHA Pins

- [x] 4.1 Add `sed` step targeting all 8 files: `ci_checks.yml`, `ci_compliance.yml`, `ci_crapload.yml`, `ci_dependencies.yml`, `ci_scheduled.yml`, `ci_security.yml`, `reusable_scheduled.yml` in `.github/workflows/`
- [x] 4.2 Use regex pattern scoped to `uses:` lines matching `complytime/org-infra/.github/workflows/reusable_.*\.yml@[0-9a-f]\{40\} # v[0-9]` and replace `@<old-sha> # v<old>` with `@<new-sha> # <new-tag>`
- [x] 4.3 Add a verification step that counts updated references and fails if the count is not exactly 9

## 5. Create Pull Request

- [x] 5.1 Add `peter-evans/create-pull-request` step (SHA-pinned) to create a PR from the modified files
- [x] 5.2 Configure PR title as `chore: update reusable workflow SHA pins to <tag>`, branch name as `chore/update-sha-pins-<tag>`, and a body describing the release
- [x] 5.3 Add condition to skip PR creation if `git diff` shows no changes (idempotency)

## 6. Validation

- [x] 6.1 Run `yamllint` on `.github/workflows/ci_update_sha_pins.yml` and verify it passes
- [x] 6.2 Verify all `uses:` references in the new workflow are SHA-pinned (no mutable tags)
- [x] 6.3 Dry-run the `sed` pattern against the current workflow files to confirm it matches exactly 9 lines and produces valid replacements
