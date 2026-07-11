# Reliable GitHub Snapshot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the unavailable public stats endpoints with two repository-hosted SVG cards that GitHub Actions refreshes weekly.

**Architecture:** A single GitHub Actions workflow invokes `cicirello/user-statistician@v1.22.0` twice to render separate profile and language cards under `assets/github-snapshot/`. The profile README references the committed SVG files, so the most recent successful render remains visible even if a future run fails.

**Tech Stack:** GitHub Actions YAML, `actions/checkout@v6`, `cicirello/user-statistician@v1.22.0`, Markdown/HTML

## Global Constraints

- Use only the built-in `GITHUB_TOKEN`; no personal access token is required.
- Restrict workflow permissions to `contents: write`.
- Generate `assets/github-snapshot/profile.svg` and `assets/github-snapshot/languages.svg`.
- Keep the rest of `README.md` unchanged.
- Run weekly and support manual dispatch.
- Preserve the last successfully committed SVG if a later run fails.
- Avoid empty commits when generated output is unchanged.

---

### Task 1: Add the snapshot generation workflow

**Files:**
- Create: `.github/workflows/update-github-snapshot.yml`

**Interfaces:**
- Consumes: GitHub's built-in `secrets.GITHUB_TOKEN` and public profile data for the checked-out repository owner.
- Produces: `assets/github-snapshot/profile.svg` and `assets/github-snapshot/languages.svg`, committed by the action only when content changes.

- [ ] **Step 1: Write the static workflow checks before the workflow exists**

Run:

```bash
test -f .github/workflows/update-github-snapshot.yml
```

Expected: FAIL with a non-zero exit status because the workflow file does not exist.

- [ ] **Step 2: Create the workflow**

Create `.github/workflows/update-github-snapshot.yml` with exactly:

```yaml
name: Update GitHub Snapshot

on:
  schedule:
    - cron: "17 3 * * 1"
  workflow_dispatch:
  push:
    branches:
      - main
    paths:
      - ".github/workflows/update-github-snapshot.yml"

concurrency:
  group: github-snapshot
  cancel-in-progress: false

permissions:
  contents: write

jobs:
  update:
    runs-on: ubuntu-latest

    steps:
      - name: Check out profile repository
        uses: actions/checkout@v6

      - name: Generate profile statistics
        uses: cicirello/user-statistician@v1.22.0
        with:
          image-file: assets/github-snapshot/profile.svg
          include-title: false
          hide-keys: languages
          colors: light
          show-border: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Refresh checkout after generated commit
        run: git pull --ff-only

      - name: Generate language distribution
        uses: cicirello/user-statistician@v1.22.0
        with:
          image-file: assets/github-snapshot/languages.svg
          include-title: false
          hide-keys: general, contributions, repositories
          max-languages: 8
          colors: light
          show-border: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The `git pull --ff-only` step ensures the second action invocation starts from the commit created by the first invocation.

- [ ] **Step 3: Validate the workflow structure**

Run:

```bash
ruby -e 'require "yaml"; YAML.load_file(".github/workflows/update-github-snapshot.yml", aliases: true)'
grep -F 'permissions:' .github/workflows/update-github-snapshot.yml
grep -F 'contents: write' .github/workflows/update-github-snapshot.yml
grep -F 'cicirello/user-statistician@v1.22.0' .github/workflows/update-github-snapshot.yml
grep -F 'assets/github-snapshot/profile.svg' .github/workflows/update-github-snapshot.yml
grep -F 'assets/github-snapshot/languages.svg' .github/workflows/update-github-snapshot.yml
```

Expected: all commands exit 0; the YAML parser prints nothing and each `grep` prints its matching line.

- [ ] **Step 4: Commit the workflow**

```bash
git add .github/workflows/update-github-snapshot.yml
git commit -m "ci: generate reliable GitHub snapshot"
```

Expected: one commit containing only the new workflow.

### Task 2: Generate and verify repository-hosted SVG assets

**Files:**
- Generate: `assets/github-snapshot/profile.svg`
- Generate: `assets/github-snapshot/languages.svg`

**Interfaces:**
- Consumes: `.github/workflows/update-github-snapshot.yml`.
- Produces: valid, non-empty SVG files for the README.

- [ ] **Step 1: Trigger the workflow manually**

Run:

```bash
gh workflow run update-github-snapshot.yml --repo Starry-49/Starry-49
```

Expected: exit 0 and confirmation that the workflow dispatch event was created.

- [ ] **Step 2: Watch the dispatched run**

Run:

```bash
run_id="$(gh run list --repo Starry-49/Starry-49 --workflow update-github-snapshot.yml --limit 1 --json databaseId --jq '.[0].databaseId')"
gh run watch "$run_id" --repo Starry-49/Starry-49 --exit-status
```

Expected: the run completes with conclusion `success`.

- [ ] **Step 3: Pull and validate both generated files**

Run:

```bash
git pull --ff-only
test -s assets/github-snapshot/profile.svg
test -s assets/github-snapshot/languages.svg
grep -F '<svg' assets/github-snapshot/profile.svg
grep -F '<svg' assets/github-snapshot/languages.svg
```

Expected: all commands exit 0 and both `grep` commands print an SVG opening element.

- [ ] **Step 4: Validate raw GitHub delivery**

Run:

```bash
curl -fsSI https://raw.githubusercontent.com/Starry-49/Starry-49/main/assets/github-snapshot/profile.svg
curl -fsSI https://raw.githubusercontent.com/Starry-49/Starry-49/main/assets/github-snapshot/languages.svg
```

Expected: both responses include `HTTP/2 200` (or `HTTP/1.1 200`) and an SVG-compatible content type.

### Task 3: Point the profile README at the generated assets

**Files:**
- Modify: `README.md` in the `## GitHub Snapshot` section.

**Interfaces:**
- Consumes: `assets/github-snapshot/profile.svg` and `assets/github-snapshot/languages.svg`.
- Produces: a responsive Snapshot section with no dependency on `github-readme-stats.vercel.app`.

- [ ] **Step 1: Confirm the old dependency is present**

Run:

```bash
grep -F 'github-readme-stats.vercel.app' README.md
```

Expected: PASS and output containing the two existing remote image URLs.

- [ ] **Step 2: Replace only the Snapshot image block**

Replace the two existing `<img>` lines inside the centered `GitHub Snapshot` block with:

```html
<a href="https://github.com/Starry-49">
  <img width="49%" src="./assets/github-snapshot/profile.svg" alt="Starry-49 GitHub activity statistics" />
</a>
<a href="https://github.com/Starry-49?tab=repositories">
  <img width="49%" src="./assets/github-snapshot/languages.svg" alt="Starry-49 language distribution" />
</a>
```

Do not modify any other README section.

- [ ] **Step 3: Verify the README references and removed dependency**

Run:

```bash
grep -F './assets/github-snapshot/profile.svg' README.md
grep -F './assets/github-snapshot/languages.svg' README.md
if grep -F 'github-readme-stats.vercel.app' README.md; then exit 1; fi
```

Expected: the first two commands print their matching lines; the final command exits 0 without output.

- [ ] **Step 4: Review the focused diff**

Run:

```bash
git diff -- README.md
```

Expected: only the two old remote image lines are replaced by the two linked repository-hosted images.

- [ ] **Step 5: Commit the README update**

```bash
git add README.md
git commit -m "fix: load GitHub snapshot from repository"
```

Expected: one commit containing only the Snapshot block update.

### Task 4: End-to-end verification and idempotency

**Files:**
- Verify: `.github/workflows/update-github-snapshot.yml`
- Verify: `assets/github-snapshot/profile.svg`
- Verify: `assets/github-snapshot/languages.svg`
- Verify: `README.md`

**Interfaces:**
- Consumes: all previous task outputs.
- Produces: evidence that the profile is stable, visible, and does not create unnecessary update commits.

- [ ] **Step 1: Push the README commit**

Run:

```bash
git push origin main
```

Expected: the local `main` branch is pushed successfully.

- [ ] **Step 2: Verify the public profile HTML references both committed assets**

Run:

```bash
curl -fsSL https://github.com/Starry-49 | grep -F 'assets/github-snapshot/profile.svg'
curl -fsSL https://github.com/Starry-49 | grep -F 'assets/github-snapshot/languages.svg'
```

Expected: both commands exit 0 and print matching profile HTML.

- [ ] **Step 3: Record the current repository head and run the workflow again**

Run:

```bash
before="$(gh api repos/Starry-49/Starry-49/commits/main --jq .sha)"
gh workflow run update-github-snapshot.yml --repo Starry-49/Starry-49
run_id="$(gh run list --repo Starry-49/Starry-49 --workflow update-github-snapshot.yml --limit 1 --json databaseId --jq '.[0].databaseId')"
gh run watch "$run_id" --repo Starry-49/Starry-49 --exit-status
after="$(gh api repos/Starry-49/Starry-49/commits/main --jq .sha)"
test "$before" = "$after"
```

Expected: the run succeeds and `test` exits 0, proving unchanged data caused no commit.

- [ ] **Step 4: Final dependency and asset checks**

Run:

```bash
if curl -fsSL https://raw.githubusercontent.com/Starry-49/Starry-49/main/README.md | grep -F 'github-readme-stats.vercel.app'; then exit 1; fi
curl -fsSL https://raw.githubusercontent.com/Starry-49/Starry-49/main/assets/github-snapshot/profile.svg | grep -F '<svg'
curl -fsSL https://raw.githubusercontent.com/Starry-49/Starry-49/main/assets/github-snapshot/languages.svg | grep -F '<svg'
```

Expected: the old endpoint check exits cleanly without output and both asset checks print an SVG opening element.
