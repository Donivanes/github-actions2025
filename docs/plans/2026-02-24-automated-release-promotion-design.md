# Automated Release Promotion: staging → main

Release-please only on main, with automated promotion and backmerge workflows.

## Problem

Release-please running on both `staging` and `main` causes version drift. The manifest, changelog, and package.json diverge because each branch manages versions independently. Manual merges between branches are error-prone.

## Requirements

- No pre-releases on staging — staging accumulates conventional commits only
- Release-please runs exclusively on `main` for production releases
- One-click promotion from staging → main via GitHub Actions (creates a PR)
- Auto-backmerge main → staging after a release is published
- No version conflicts between branches

## Architecture

Three GitHub Actions workflows:

| Workflow | Trigger | Purpose |
|---|---|---|
| `release.yml` | Push to `main` | Release-please creates/updates the Release PR |
| `promote.yml` | Manual (workflow_dispatch) | Creates a PR from staging → main |
| `backmerge.yml` | GitHub Release published | Merges main → staging automatically |

### Flow

```
staging ──(commits)──> [Click Promote] ──> PR staging→main
                                                │
                                          merge PR (merge commit, not squash)
                                                │
                                        main ──(push)──> release-please creates Release PR
                                                                    │
                                                              merge Release PR
                                                                    │
                                                          GitHub Release published
                                                          version bump + changelog
                                                                    │
                                                          [Auto-backmerge] main→staging
```

## Workflow Details

### release.yml (simplified)

Runs only on `main`. No conditional config, no staging logic.

```yaml
name: Release

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        id: release
        with:
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json
```

### promote.yml (new)

Manual trigger that creates a PR from staging → main. Checks for existing PR and verifies there are commits to promote.

```yaml
name: Promote to Production

on:
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  promote:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Create promotion PR
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          EXISTING_PR=$(gh pr list --base main --head staging --state open --json number --jq '.[0].number')
          if [ -n "$EXISTING_PR" ]; then
            echo "Promotion PR #$EXISTING_PR already exists"
            exit 0
          fi

          COMMITS=$(git log main..origin/staging --oneline)
          if [ -z "$COMMITS" ]; then
            echo "No new commits to promote"
            exit 0
          fi

          gh pr create \
            --base main \
            --head staging \
            --title "chore: promote staging to production" \
            --body "$(cat <<EOF
          ## Promotion: staging → main

          ### Commits included:
          $(git log main..origin/staging --oneline --no-merges)
          EOF
          )"
```

### backmerge.yml (new)

Triggers when a GitHub Release is published. Attempts direct merge; falls back to creating a PR if conflicts arise.

```yaml
name: Backmerge to Staging

on:
  release:
    types: [published]

permissions:
  contents: write
  pull-requests: write

jobs:
  backmerge:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: staging
          fetch-depth: 0

      - name: Merge main into staging
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          git fetch origin main
          if git merge origin/main --no-edit; then
            git push origin staging
            echo "Backmerge successful"
          else
            git merge --abort
            gh pr create \
              --base staging \
              --head main \
              --title "chore: backmerge main into staging" \
              --body "Auto-backmerge failed due to conflicts. Please resolve manually."
            echo "::warning::Backmerge had conflicts. Created PR for manual resolution."
          fi
```

## Merge Strategy Constraint

The promotion PR (staging → main) **must** be merged with "Create a merge commit", not squash. Squashing would collapse all commits into a single `chore: promote staging to production` commit, which release-please would ignore since `chore:` is not a releasable commit type. The individual `feat:` and `fix:` commits need to appear in main's history.

## Migration Steps

1. Delete `release-please-config-staging.json`
2. Clear `.gitattributes` (remove `merge=ours` rules)
3. Update `release.yml` to only target `main`
4. Create `promote.yml` and `backmerge.yml`
5. Sync staging with main: `git checkout staging && git merge main && git push`
6. Close any open release-please PRs targeting staging
7. Delete the remote `release-please--branches--staging--components--github-actions` branch

## Why This Works Without Conflicts

- Staging never modifies version files (`package.json` version, manifest, changelog) — only application code changes via conventional commits
- Main's version files are only modified by release-please (via Release PRs)
- Backmerge keeps staging synchronized with main's version after each release
- Promotion merges are clean because staging's version matches main's (from previous backmerge)
