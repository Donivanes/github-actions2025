---
description: Create a GitHub Pull Request from the current branch against staging, with a detailed structured body.
---

Create a GitHub Pull Request for the current branch. Follow these steps precisely:

## 1. Gather context

Run the following shell commands **in parallel** to understand the current state:

- `git status` — check for uncommitted changes
- `git branch --show-current` — get current branch name
- `git log staging..HEAD --oneline` — list all commits since diverging from staging
- `git diff staging...HEAD --stat` — summarise changed files

## 2. Analyse changes

- Read the full diff with `git diff staging...HEAD` to understand every change.
- If there are uncommitted changes, warn me and ask whether to commit them first before proceeding.

## 3. Generate the PR

Using the analysis above, compose the PR **title** and **body** following this template exactly. Every section is mandatory — fill them all in based on the diff and commit history.

### Title

Use conventional-commit style and keep it concise, e.g. `feat: add Spanish NIF/CIF verification against AEAT`.

### Body template

````markdown
# Pull Request: <short descriptive name>

## Summary

<1-2 paragraph high-level description of what the PR does, the problem it solves, and the approach taken. Mention key technical decisions (e.g. algorithms, patterns) if noteworthy.>

**Branch:** `<current-branch>` → `staging`
**Commits (<N>):**

<numbered list of every commit with its message and a brief clarification of what it contains>

---

## What's included

<Group changes by area/concern. Use nested headings (###) and bullet lists. Be specific — mention file paths, function names, types, endpoints, schemas. Cover:>

### New API endpoint(s)
- <method, path, auth requirements, body schema, response shape, error handling>

### Server
- <services, types, infrastructure adapters, parsers, schemas, utils — with file paths>

### Client
- <domain (DTOs, entities, services, value objects), application (use cases), infrastructure (repositories), presentation (hooks, mutations/queries, components) — with file paths>
- <Note any removed/replaced files>

### Integration points
- <Which existing forms/pages/flows consume the new functionality and how>

### Other changes
- <Config, scripts, constants, minor tweaks in unrelated modules>

---

## Technical notes

<Bullet list of important implementation details: algorithms used, architectural decisions, scope boundaries, things intentionally left unchanged, etc.>

---

## How to test

<Numbered list of manual testing scenarios covering each integration point. Be specific about user actions and expected results.>

---

## Files changed (by area)

| Area | Files |
|------|--------|
| **<Area>** | `<file>`, `<file>`, ... |
| ... | ... |
````

Rules for filling the template:

- **Summary**: Write for a reviewer who hasn't seen the code. Explain *why* and *what*, not just *how*.
- **Commits**: List every commit from `git log staging..HEAD --oneline`. Add a dash and brief description of what each one contains.
- **What's included**: Be thorough. Group logically (API, Server, Client, etc.). Mention concrete file paths relative to the project root. Call out new files, deleted files, and renamed files explicitly.
- **Technical notes**: Highlight non-obvious decisions — algorithms, thresholds, patterns, things intentionally scoped out.
- **How to test**: Provide actionable scenarios a reviewer can follow. Cover happy path and error cases.
- **Files changed**: Summarise by area with relative paths. This table gives reviewers a quick map of the PR scope.

## 4. Create the PR via `gh`

Once I approve the generated content (or if everything is clear), run:

```bash
git push -u origin HEAD
```

Then create the PR:

```bash
gh pr create --base staging --title "<title>" --body "$(cat <<'EOF'
<body>
EOF
)"
```

## 5. Return the PR URL

After creation, output the PR URL so I can review it in the browser.
