## ADDED Requirements

### Requirement: Automatic SHA pin update on release

The system SHALL automatically update all reusable workflow SHA pin references in
consumer workflow files when a new org-infra release is published.

#### Scenario: Release triggers SHA pin update

- **WHEN** a new GitHub Release is published for org-infra
- **THEN** the system creates a pull request that replaces all reusable workflow SHA
  references with the new release commit SHA and version tag

#### Scenario: No release, no update

- **WHEN** a non-release event occurs (e.g., push to main, PR merge)
- **THEN** the system does not modify any SHA pin references

### Requirement: All references updated atomically

The system SHALL update all reusable workflow references in a single pull request,
covering every file that contains a pinned reference to an org-infra reusable workflow.

#### Scenario: All 9 references across 8 files updated

- **WHEN** the SHA pin update runs
- **THEN** every `uses:` line referencing a `complytime/org-infra` reusable workflow
  is updated to the new release SHA and version tag in one commit

#### Scenario: Partial update produces no PR

- **WHEN** the replacement operation fails to update one or more references
- **THEN** the system does not create the pull request and reports the failure

### Requirement: Pin format preserved

The system SHALL maintain the established pin format: full 40-character commit SHA
followed by a space, hash, space, and semver tag (e.g., `@<sha> # v1.2.3`).

#### Scenario: Format matches convention

- **WHEN** the system writes updated SHA references
- **THEN** each reference follows the pattern `@<40-hex-chars> # v<major>.<minor>.<patch>`

### Requirement: Human review before merge

The system SHALL produce a pull request for the SHA update rather than committing
directly to the default branch. The PR SHALL include a descriptive title and body
indicating the release version.

#### Scenario: PR created for review

- **WHEN** the SHA pin update completes successfully
- **THEN** a pull request is opened against the default branch with the updated files

#### Scenario: Direct push blocked

- **WHEN** the system attempts to apply SHA updates
- **THEN** the updates are committed to a feature branch, not directly to the default
  branch

### Requirement: Concurrency safety

The system SHALL prevent concurrent SHA pin update runs from producing conflicting
pull requests.

#### Scenario: Concurrent releases

- **WHEN** two releases are published in rapid succession
- **THEN** only the most recent release's SHA pin update run completes; earlier
  in-progress runs are cancelled

### Requirement: Idempotent operation

The system SHALL be safe to re-run. If the SHA pins already match the target release,
the system SHALL not create a duplicate pull request.

#### Scenario: Re-run after successful update

- **WHEN** the SHA pin update runs for a release whose pins are already applied
- **THEN** the system detects no changes and does not create a pull request
