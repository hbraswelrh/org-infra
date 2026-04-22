## Why

After each org-infra release, a maintainer must manually update 9 SHA references across
8 workflow files, then run the sync workflow to propagate the pinned consumer workflows
to all downstream repos. This manual step was an accepted trade-off when SHA pinning was
introduced (see `pin-workflows-to-release`), but it is error-prone and creates friction
in the release cycle. Automating it eliminates a mechanical toil step and ensures SHA
pins are always consistent with the latest release.

## What Changes

- Add a post-release automation step (GitHub Actions workflow or job) that, after a
  release tag is created, updates all `@<old-sha> # vX.Y.Z` references in consumer
  workflow files to `@<new-release-sha> # <new-tag>`, commits the result, and opens
  a PR (or triggers the sync workflow).
- The automation performs a deterministic find-and-replace: locate all
  `complytime/org-infra/.github/workflows/reusable_*.yml@<sha> # v...` patterns and
  replace the SHA and version comment with the new release values.

## Non-goals

- Changing the SHA-pinning convention itself (full 40-char SHA + version comment stays).
- Modifying the sync script (`sync-org-repositories.py`) or `sync-config.yml`.
- Automating the sync-to-downstream-repos step (that remains `workflow_dispatch`).
- Introducing mutable tag references (`@v1`, `@main`) as an alternative to SHA pins.

## Capabilities

### New Capabilities

- `sha-pin-updater`: Automatically update org-infra reusable workflow SHA pins in
  consumer workflows after a release is published, producing a PR with the updated
  references ready for review and merge.

### Modified Capabilities

(none)

## Impact

- **Workflows**: The `release.yml` workflow or a new companion workflow gains a
  post-release step/job that updates SHA pins and creates a PR.
- **Files modified by automation**: The same 8 files with 9 references that are
  currently updated manually:
  - `ci_checks.yml` (1 reference)
  - `ci_compliance.yml` (1 reference)
  - `ci_crapload.yml` (1 reference)
  - `ci_dependencies.yml` (2 references)
  - `ci_scheduled.yml` (1 reference)
  - `ci_security.yml` (2 references)
  - `reusable_scheduled.yml` (1 reference)
- **Downstream repos**: No direct impact. The sync workflow remains a separate manual
  trigger. The automation only ensures that when sync runs, the consumer workflows
  contain the correct SHA pins.
- **Repos requiring the synced update** (all non-excluded repos that receive `ci_*`
  workflows):
  - `complytime/complytime`
  - `complytime/complyctl`
  - `complytime/complyscribe`
  - `complytime/complytime-providers`
  - `complytime/complytime-collector-components`
  - `complytime/complytime-policies`
  - `complytime/complytime-demos`
  - `complytime/gemara-content-service`
  - `complytime/.github`
  - `complytime/community`
