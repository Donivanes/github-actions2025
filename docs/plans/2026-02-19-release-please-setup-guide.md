# Release-Please GitFlow Setup Guide for Node.js Projects

Automated releases with pre-releases on staging and full releases on main, using `googleapis/release-please-action@v4`.

## Prerequisites

- Node.js project with `package.json`
- Conventional commits enforced (commitlint + husky already configured)
- GitFlow branching: `main` (production) + `staging` (integration)
- GitHub repository

## Step 1: Enable GitHub Actions PR permissions

Go to your repository on GitHub:

**Settings → Actions → General → Workflow permissions**

- Select "Read and write permissions"
- Check "Allow GitHub Actions to create and approve pull requests"
- Save

## Step 2: Register the merge=ours git driver

Run this once in your local clone:

```bash
git config merge.ours.driver true
```

This enables the `merge=ours` strategy used in `.gitattributes` to prevent merge conflicts on release-managed files.

## Step 3: Create files on `main`

### `.github/workflows/release.yml`

```yaml
name: Release

on:
  push:
    branches: [main, staging]
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
          target-branch: ${{ github.ref_name }}
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json
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

### `.gitattributes`

```
release-please-config.json merge=ours
.release-please-manifest.json merge=ours
CHANGELOG.md merge=ours
```

This ensures these three files always keep the target branch's version during merges, preventing conflicts between staging and main.

### Commit and push

```bash
git add .github/workflows/release.yml release-please-config.json .release-please-manifest.json .gitattributes
git commit -m "feat: add release-please automated releases"
git push origin main
```

## Step 4: Set up `staging` branch

```bash
git checkout -b staging
```

Modify `release-please-config.json` to enable pre-releases:

```json
{
  "packages": {
    ".": {
      "release-type": "node",
      "changelog-path": "CHANGELOG.md",
      "prerelease": true,
      "prerelease-type": "rc",
      "versioning": "prerelease"
    }
  }
}
```

Commit and push:

```bash
git add release-please-config.json
git commit -m "feat: enable pre-release versioning on staging"
git push -u origin staging
```

## How it works

### On staging (pre-releases)

1. Push conventional commits (`feat:`, `fix:`, etc.) to staging.
2. Release-please opens/updates a **Release PR** against staging with a pre-release version (e.g., `v1.2.0-rc.0`).
3. Merge that Release PR → GitHub **Pre-Release** is created, `CHANGELOG.md` updated, `package.json` version bumped.

### On main (full releases)

1. Merge staging into main.
2. Release-please opens/updates a **Release PR** against main with the full version (e.g., `v1.2.0`).
3. Merge that Release PR → GitHub **Release** is created, `CHANGELOG.md` updated, `package.json` version bumped.

### Manual trigger

You can also trigger a release manually via the GitHub Actions UI: **Actions → Release → Run workflow**.

## Handling merge conflicts (staging → main)

### Auto-resolved files (via `.gitattributes` merge=ours)

These files keep main's version automatically — no action needed:

- `release-please-config.json`
- `.release-please-manifest.json`
- `CHANGELOG.md`

### `package.json` (manual resolution)

The `version` field will conflict because release-please bumps it independently on each branch. When this happens, **always keep main's version**. Release-please on main will bump it correctly in the next release PR.

## Troubleshooting

### "GitHub Actions is not permitted to create or approve pull requests"

You missed Step 1. Go to Settings → Actions → General → enable "Allow GitHub Actions to create and approve pull requests".

### "Validation Failed: tag_name already_exists"

A tag from a previous release system or from staging conflicts with what main is trying to create. Fix:

```bash
gh release delete <tag-name> --yes --cleanup-tag
gh workflow run release.yml --ref main
```

### Release PR not appearing

- Ensure commits follow conventional commit format (`feat:`, `fix:`, etc.). Non-conventional commits like `update stuff` are ignored by release-please.
- Check that the workflow triggered: Actions tab → Release workflow.

### merge=ours not working

You need to register the driver locally. Run `git config merge.ours.driver true` in your clone. Each developer who merges staging → main needs to run this once.
