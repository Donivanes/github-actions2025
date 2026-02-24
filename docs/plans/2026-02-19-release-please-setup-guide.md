# Release-Please Setup Guide for Node.js Projects

Automated releases on `main` with a staging → main promotion workflow, using `googleapis/release-please-action@v4`.

## Architecture

- **Release-please** runs only on `main` — it opens Release PRs with version bumps and changelogs.
- **Promote workflow** (`promote.yml`) — manually triggered to create a PR from `staging` → `main`.
- **Backmerge workflow** (`backmerge.yml`) — automatically merges `main` → `staging` after each release to keep branches in sync.
- **Staging** accumulates conventional commits without any version management.

## Prerequisites

- Node.js project with `package.json`
- Conventional commits enforced (commitlint + husky already configured)
- Two branches: `main` (production) + `staging` (integration)
- GitHub repository

## Step 1: Enable GitHub Actions PR permissions

Go to your repository on GitHub:

**Settings → Actions → General → Workflow permissions**

- Select "Read and write permissions"
- Check "Allow GitHub Actions to create and approve pull requests"
- Save

## Step 2: Create files on `main`

### `.github/workflows/release.yml`

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

### `.github/workflows/promote.yml`

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

          COMMITS=$(git log main..origin/staging --oneline --no-merges)
          if [ -z "$COMMITS" ]; then
            echo "No new commits to promote"
            exit 0
          fi

          gh pr create \
            --base main \
            --head staging \
            --title "chore: promote staging to production" \
            --body "## Promotion: staging → main

          ### Commits included:
          ${COMMITS}"
```

### `.github/workflows/backmerge.yml`

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

### `release-please-config.json`

```json
{
  "packages": {
    ".": {
      "release-type": "node",
      "changelog-path": "CHANGELOG.md"
    }
  }
}
```

### `.release-please-manifest.json`

```json
{
  ".": "0.0.0"
}
```

Set `"0.0.0"` if this is a new project, or the current version from your `package.json` if the project already has releases.

### Commit and push

```bash
git add .github/workflows/ release-please-config.json .release-please-manifest.json
git commit -m "feat: add release-please automated releases"
git push origin main
```

## How it works

### Development on staging

1. Push conventional commits (`feat:`, `fix:`, etc.) to `staging` via feature branches.
2. Staging accumulates changes — no version bumps or release PRs happen here.

### Promoting to production

1. Go to **Actions → Promote to Production → Run workflow**.
2. The workflow creates a PR from `staging` → `main` listing all included commits.
3. **Merge the promotion PR using "Create a merge commit"** (not squash). This preserves the individual conventional commits so release-please can read them.
4. Release-please detects the new commits on `main` and opens a **Release PR** with the version bump and changelog.
5. Merge the Release PR → GitHub Release is created.

### Automatic backmerge

After a release is published, the backmerge workflow automatically merges `main` → `staging` to keep version files in sync. If there are conflicts, it creates a PR for manual resolution.

### Manual trigger

You can also trigger a release-please run manually: **Actions → Release → Run workflow**.

## Important: Merge strategy for promotion PRs

When merging promotion PRs (staging → main), **always use "Create a merge commit"**. Do not use squash merge. Release-please needs to see the individual conventional commits to determine the correct version bump and generate the changelog.

## Troubleshooting

### "GitHub Actions is not permitted to create or approve pull requests"

You missed Step 1. Go to Settings → Actions → General → enable "Allow GitHub Actions to create and approve pull requests".

### Release PR not appearing after promotion

- Ensure commits follow conventional commit format (`feat:`, `fix:`, etc.). Non-conventional commits are ignored by release-please.
- Verify the promotion PR was merged with "Create a merge commit", not squash.
- Check that the Release workflow triggered: Actions tab → Release workflow.

### Backmerge conflicts

If the automatic backmerge fails, a PR is created from `main` → `staging`. Resolve conflicts manually and merge.

### "Validation Failed: tag_name already_exists"

A tag from a previous release conflicts. Fix:

```bash
gh release delete <tag-name> --yes --cleanup-tag
gh workflow run release.yml --ref main
```
