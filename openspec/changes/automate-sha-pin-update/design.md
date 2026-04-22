## Context

After each org-infra release, 9 SHA-pinned references across 8 workflow files must be
updated from the old release SHA to the new release SHA. This is currently a manual
find-and-replace step between cutting the release and running the org sync workflow.

The pinning convention (`@<40-char-sha> # vX.Y.Z`) was established in the
`pin-workflows-to-release` change. The files affected are always the same set of consumer
workflows (`ci_*.yml`) plus `reusable_scheduled.yml`, all containing `uses:` lines that
reference `complytime/org-infra/.github/workflows/reusable_*.yml@<sha>`.

The release workflow (`release.yml`) currently creates a git tag and publishes a GitHub
Release via release-drafter, then stops. There is no post-release automation.

## Goals / Non-Goals

**Goals:**

- Automatically update all reusable workflow SHA pins after a release is published.
- Produce a PR with the updated references for human review before merge.
- Maintain the existing `@<full-sha> # vX.Y.Z` format convention.
- Keep the automation entirely within GitHub Actions (no external CI, no new tools).

**Non-Goals:**

- Automating the sync-to-downstream-repos step (remains `workflow_dispatch`).
- Changing the SHA-pinning format or convention.
- Modifying the sync script or `sync-config.yml`.
- Auto-merging the SHA update PR (human review required).

## Decisions

### 1. Trigger: `release: published` event on a new workflow

Add a new workflow (`ci_update_sha_pins.yml`) triggered by the `release: published`
event. This fires automatically after `release.yml` creates the GitHub Release.

**Alternatives considered:**
- **Add a job to `release.yml` itself**: Rejected because the release workflow creates
  the tag on the current commit. The SHA update must happen in a *new* commit after the
  tag, which means the tag SHA is known only after the release job completes. Using a
  separate workflow triggered by the release event provides clean separation and avoids
  a circular commit-then-tag-then-commit flow.
- **Manual script run**: Rejected because it merely shifts the manual step from
  find-and-replace to running a script -- the automation should be fully hands-off.
- **`workflow_run` trigger on `release.yml`**: Would work but `release: published` is
  more semantic and directly tied to the event we care about. `workflow_run` would also
  fire on failed release runs.

### 2. Mechanism: `sed` find-and-replace in a shell step

Use a shell step with `sed` to replace all occurrences of the old SHA pattern with the
new release tag SHA. The pattern is deterministic:

```
complytime/org-infra/.github/workflows/reusable_.*\.yml@[0-9a-f]{40} # v[0-9]+\.[0-9]+\.[0-9]+
```

Replace the `@<sha> # vX.Y.Z` suffix with `@<new-sha> # <new-tag>`.

**Alternatives considered:**
- **Python script**: More robust regex handling but adds unnecessary complexity for a
  simple line-level replacement. The `sed` approach is a single command with no
  dependencies beyond the runner's default shell.
- **Custom GitHub Action**: Over-engineered for a 5-line shell script. Rejected per the
  "No Reinventing the Wheel" and "Simplicity" principles.

### 3. Output: PR via `peter-evans/create-pull-request`

The workflow creates a PR rather than pushing directly to `main`. This preserves the
review gate and branch protection rules.

**Alternatives considered:**
- **Direct push to `main`**: Rejected because it bypasses branch protection and code
  review requirements. The SHA update is mechanical but should still be reviewed to
  catch unexpected changes.
- **Open an issue instead**: Rejected because it still requires manual work to create
  the PR. The goal is to eliminate all manual steps.

### 4. Derive the release SHA from the git tag

The new SHA is obtained by resolving the release tag (`${{ github.event.release.tag_name }}`)
to its commit SHA using `git rev-parse`. This is reliable because the release workflow
enforces that the tag points to a commit on `main`.

**Alternatives considered:**
- **Use `github.sha` from the release event**: This is the SHA of the commit that
  triggered the workflow, which should be the tagged commit. However, `git rev-parse`
  on the tag is more explicit and self-documenting.

### 5. Workflow naming: `ci_update_sha_pins.yml`

Follows the `ci_` prefix convention for consumer/internal workflows. This workflow is
not synced to downstream repos (it is org-infra-specific) and should be added to the
`exclude_repos` or simply not listed in `sync-config.yml`.

**Alternatives considered:**
- **`post_release_update_pins.yml`**: Does not follow the established naming convention.
  Rejected for consistency.

## Risks / Trade-offs

- **[Risk] Race condition if release is re-published or drafted quickly.**
  Two concurrent runs could produce conflicting PRs.
  -> Mitigation: Use `concurrency` group on the workflow to cancel in-progress runs.

- **[Risk] `sed` pattern could match unintended lines.**
  If a comment or non-`uses:` line contains the pattern, it would be modified.
  -> Mitigation: Scope the `sed` pattern to lines starting with whitespace + `uses:`.
  Validate the diff in the PR review.

- **[Risk] PR auto-creation requires a token with write access.**
  The default `GITHUB_TOKEN` may not trigger downstream workflows on the PR.
  -> Mitigation: Use the default `GITHUB_TOKEN` for PR creation. If downstream CI
  triggers are needed, a PAT or GitHub App token can be configured later.

- **[Trade-off] The SHA update commit is not itself tagged.**
  The release tag points to the commit *before* the SHA update. This is intentional:
  the tag represents the state of the reusable workflows, not the consumer workflow
  pins. The consumer workflows simply point *at* that tagged state.

## Open Questions

- Should the workflow also trigger the org sync automatically after the SHA update PR
  is merged, or should sync remain fully manual? (Recommendation: keep sync manual for
  now; automate in a future change if desired.)
