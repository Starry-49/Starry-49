# Reliable GitHub Snapshot Design

## Goal

Replace the profile README's failing public `github-readme-stats.vercel.app` images with repository-hosted SVG assets that load from GitHub and refresh automatically.

## Scope

- Add a GitHub Actions workflow that generates two SVG cards:
  - overall GitHub activity statistics
  - most-used languages
- Store generated cards under `assets/github-snapshot/`.
- Update the README's `GitHub Snapshot` section to reference those repository files.
- Preserve a centered, responsive two-card layout.
- Do not change the rest of the profile content.

## Architecture

The workflow runs on a weekly schedule, on manual dispatch, and when its own configuration changes. It uses the repository's built-in `GITHUB_TOKEN` with read access to collect public profile data, renders SVG cards, and commits changed assets back to `main`.

The README references the committed SVG files using repository-relative URLs. GitHub therefore renders the last successfully generated snapshot even if a later scheduled workflow fails.

## Components

### Snapshot workflow

A workflow at `.github/workflows/update-github-snapshot.yml` will:

1. Check out the repository.
2. Generate the profile and language SVG cards into `assets/github-snapshot/`.
3. Commit and push only when the generated assets changed.

Permissions will be limited to `contents: write`, which is required to update the generated assets. Concurrent scheduled runs will be grouped to avoid overlapping commits.

### Generated assets

The workflow owns:

- `assets/github-snapshot/profile.svg`
- `assets/github-snapshot/languages.svg`

These are build artifacts and should not be edited manually.

### README presentation

The existing two remote image URLs will be replaced with links to the committed SVG files. Each image will use a percentage width with a sensible maximum so the cards sit side by side on wide screens and remain usable on narrow screens. Alt text will identify each card.

## Data Flow

1. GitHub triggers the workflow.
2. The generator reads public data for `Starry-49`.
3. The workflow writes deterministic SVG output.
4. Changed SVG files are committed to the profile repository.
5. The profile README loads the SVGs directly from the repository.

## Failure Handling

- If generation fails, the workflow exits without replacing or deleting the existing cards.
- The README continues to show the last committed successful snapshot.
- The workflow commits only changed files to avoid empty commits.
- A manual dispatch trigger allows recovery after configuration or upstream changes.

## Security

- Use only the built-in `GITHUB_TOKEN`; no personal access token is required.
- Restrict workflow permissions to repository contents.
- Do not expose secrets or user-supplied content in generated SVG markup.

## Verification

Implementation is complete when:

1. A manual workflow run succeeds.
2. Both SVG files exist in `assets/github-snapshot/`.
3. Their raw GitHub URLs return SVG content successfully.
4. The profile page displays both cards without requests to `github-readme-stats.vercel.app`.
5. A second workflow run with unchanged data does not create an unnecessary commit.
