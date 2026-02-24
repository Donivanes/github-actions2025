# Automated Release Promotion Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace the dual-branch release-please setup with a single-branch (main) release-please + promotion and backmerge workflows.

**Architecture:** Release-please runs only on `main`. A manual `promote.yml` workflow creates PRs from staging → main. A `backmerge.yml` workflow auto-merges main → staging after each release. Staging accumulates conventional commits without version management.

**Tech Stack:** GitHub Actions, release-please-action@v4, gh CLI

---

### Task 1: Simplify release.yml

**Files:**
- Modify: `.github/workflows/release.yml`

**Step 1: Update the workflow**

Replace the current contents of `.github/workflows/release.yml` with:

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

Changes from current:
- Remove `staging` from `branches`
- Remove `target-branch: ${{ github.ref_name }}`
- Remove conditional config-file selection

**Step 2: Verify the file**

Run: `cat .github/workflows/release.yml`
Expected: The simplified workflow above, no references to staging.

**Step 3: Commit**

```bash
git add .github/workflows/release.yml
git commit -m "refactor: simplify release workflow to only target main"
```

---

### Task 2: Create promote.yml

**Files:**
- Create: `.github/workflows/promote.yml`

**Step 1: Create the workflow file**

Create `.github/workflows/promote.yml` with:

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
            --body "$(cat <<EOF
          ## Promotion: staging → main

          ### Commits included:
          ${COMMITS}
          EOF
          )"
```

**Step 2: Verify the file**

Run: `cat .github/workflows/promote.yml`
Expected: The workflow above with proper YAML syntax.

**Step 3: Commit**

```bash
git add .github/workflows/promote.yml
git commit -m "feat: add promote-to-production workflow"
```

---

### Task 3: Create backmerge.yml

**Files:**
- Create: `.github/workflows/backmerge.yml`

**Step 1: Create the workflow file**

Create `.github/workflows/backmerge.yml` with:

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

**Step 2: Verify the file**

Run: `cat .github/workflows/backmerge.yml`
Expected: The workflow above with proper YAML syntax.

**Step 3: Commit**

```bash
git add .github/workflows/backmerge.yml
git commit -m "feat: add auto-backmerge workflow after releases"
```

---

### Task 4: Clean up staging-specific config

**Files:**
- Delete: `release-please-config-staging.json`
- Modify: `.gitattributes`

**Step 1: Delete the staging config**

```bash
rm release-please-config-staging.json
```

**Step 2: Clear .gitattributes**

Replace contents of `.gitattributes` with an empty file (or remove merge=ours rules). The `merge=ours` strategy is no longer needed because staging won't have independent version management.

If there are non-release-related entries in `.gitattributes`, keep those. Only remove the `merge=ours` lines.

Current contents to remove:
```
package.json merge=ours
.release-please-manifest.json merge=ours
CHANGELOG.md merge=ours
```

**Step 3: Verify**

Run: `ls release-please-config-staging.json` — should not exist
Run: `cat .gitattributes` — should be empty or have no merge=ours lines

**Step 4: Commit**

```bash
git add -A
git commit -m "chore: remove staging release-please config and merge=ours rules"
```

---

### Task 5: Sync staging with main and clean up remote branches

**Step 1: Sync staging version with main**

```bash
git checkout staging
git merge main --no-edit
git push origin staging
```

This ensures staging has the same version as main (1.4.0 instead of 1.4.0-rc).

**Step 2: Close open release-please PRs for staging**

Check for open release-please PRs targeting staging:

```bash
gh pr list --base staging --state open --json number,title
```

Close any that match `release-please--branches--staging`:

```bash
gh pr close <PR_NUMBER> --comment "Closing: release-please removed from staging"
```

**Step 3: Delete the remote staging release-please branch**

```bash
git push origin --delete release-please--branches--staging--components--github-actions 2>/dev/null || true
```

**Step 4: Push all changes**

```bash
git push origin staging
```

---

### Task 6: Update documentation

**Files:**
- Modify: `docs/plans/2026-02-19-release-please-setup-guide.md`

**Step 1: Update the setup guide**

Replace the existing setup guide content to reflect the new simplified workflow. Document:
- Release-please only runs on main
- How to use the Promote workflow
- How backmerge works automatically
- The merge strategy constraint (use "Create a merge commit" for promotion PRs, not squash)
- Remove all pre-release/staging config instructions

**Step 2: Commit**

```bash
git add docs/
git commit -m "docs: update release setup guide for new promotion workflow"
```

---

### Task 7: End-to-end verification

**Step 1: Verify workflow files exist and are valid YAML**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/release.yml'))"
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/promote.yml'))"
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/backmerge.yml'))"
```

All three should complete without errors.

**Step 2: Verify staging config is gone**

```bash
test ! -f release-please-config-staging.json && echo "OK: staging config removed"
```

**Step 3: Verify .gitattributes has no merge=ours**

```bash
! grep -q "merge=ours" .gitattributes 2>/dev/null && echo "OK: no merge=ours rules"
```

**Step 4: Verify versions are aligned**

```bash
echo "Main version:" && git show main:package.json | grep '"version"'
echo "Staging version:" && git show staging:package.json | grep '"version"'
```

Both should show the same version (1.4.0 or whatever main is at).

**Step 5: Push to trigger workflows**

```bash
git push origin staging
```

Then verify in GitHub Actions that:
- The Release workflow does NOT trigger on staging push
- The Promote workflow appears in the Actions tab with a "Run workflow" button
