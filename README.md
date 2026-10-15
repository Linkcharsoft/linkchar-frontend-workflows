# linkchar-frontend-workflows

Shared, versioned GitHub Actions workflows for Linkchar's **frontend** projects (the Next template today).

**Frontend only, on purpose.** Tags version the whole repository, so a release here becomes a Dependabot bump PR — review, auto-merge, production redeploy — in every project that calls it. Backend workflows belong in their own repository following the same pattern, so neither team's releases reach the other's projects.

Each project calls these workflows from a ~25-line `ci.yml` pinned to a release tag. Dependabot bumps that tag like any other action, so improving a workflow here means one release, and every project receives it as a reviewed PR.

| Workflow | What it does |
| --- | --- |
| [`next-ci.yml`](.github/workflows/next-ci.yml) | CI for projects built on `linkchar-next-template`: **Lint** (ESLint) and **Build** (type-check + `next build`) on a bare checkout, with dummy env values that satisfy the template's boot checks. |
| [`dependabot-review.yml`](.github/workflows/dependabot-review.yml) | On Dependabot PRs only: Claude reads the changelog of every bumped package, checks whether the project uses what changed — and, when CI fails, reads the failed jobs' logs — and returns a verdict; the workflow posts it as a comment and squash-merges patch/minor bumps that passed review **and** CI. |

## Usage

`.github/workflows/ci.yml` in the project (the Next template already ships it):

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read

jobs:
  ci:
    uses: Linkcharsoft/linkchar-frontend-workflows/.github/workflows/next-ci.yml@v2.0.0

  dependabot:
    needs: ci
    if: ${{ !cancelled() && github.actor == 'dependabot[bot]' }}
    uses: Linkcharsoft/linkchar-frontend-workflows/.github/workflows/dependabot-review.yml@v2.0.0
    permissions:
      actions: read
      contents: write
      pull-requests: write
      id-token: write
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    with:
      ci-result: ${{ needs.ci.result }}
      automerge: ${{ vars.DEPENDABOT_AUTOMERGE || 'minor' }}
```

Plus `.github/dependabot.yml`, which GitHub reads only from the project itself and cannot be shared — copy it from the Next template, whose version is the canonical one.

### Inputs

`next-ci.yml`

| Input | Default | Purpose |
| --- | --- | --- |
| `node-version-file` | `.nvmrc` | File that pins Node. |
| `build-env` | `""` | Extra `KEY=VALUE` lines for the Build job, one per line — for a project whose build requires more variables than the template's. |

`dependabot-review.yml`

| Input | Default | Purpose |
| --- | --- | --- |
| `ci-result` | — (required) | `needs.<ci job>.result`. Only `success` can merge. |
| `automerge` | `minor` | `minor` merges patch/minor bumps; any other value (use `off`) only reviews. Set the `DEPENDABOT_AUTOMERGE` repo variable to change it per project without touching the YAML. |
| `model` | `claude-sonnet-5-5` | Model used for the review. |
| `max-turns` | `40` | Caps the agent's turns on large grouped PRs. |

### Merge policy

A Dependabot PR is merged only when **all** of these hold:

1. CI (`Lint` + `Build`) succeeded.
2. Claude's verdict is `ok`.
3. The highest update in the PR is a patch or minor (from `dependabot/fetch-metadata`, not from the agent). Majors are reviewed and commented, never merged.
4. Every commit in the PR is Dependabot's, and the head is still the reviewed commit (`--match-head-commit`).
5. `automerge` is `minor`.

The squash commit is titled `[ CHORE ] Bump … (#N)`, following the commit convention.

> **Merging to `main` deploys to production.** Amplify builds `main` on push, so an auto-merged bump ships without a human in the loop. Lint, types and the build are the gate; the Claude verdict is a changelog review, not a security control — the 7-day `cooldown` in `dependabot.yml` is what keeps a freshly published malicious release out.

## Setup

1. **Secret, in every calling project** — create `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`) under the project's *Settings → Secrets and variables → **Dependabot***. The caller passes it in, so a secret stored in **this** repository never reaches anyone. Runs triggered by Dependabot only read Dependabot secrets (an Actions secret with the same name is invisible to them), and on GitHub Free organization secrets do not reach private repositories — on a paid plan a single organization Dependabot secret shared with the frontend repositories replaces the per-repo ones. Without it the review is skipped with a warning and nothing merges.
2. **Access** — keep this repository **public** (it holds no secrets): callers and Dependabot then reach it on any plan. If it must be private, allow access from the organization's repositories under *Settings → Actions → General → Access*, and add it under *Organization settings → Advanced Security → Grant Dependabot access to private repositories*.

## Releasing

1. Change the workflow on a branch. To try it before releasing, point a project's `ci.yml` at the branch (`@my-branch`) in a throwaway PR — the template is the natural test bed.
2. Add an entry to [`CHANGELOG.md`](CHANGELOG.md): the review of each project's bump PR reads it to explain what changed.
3. Merge, then tag `vX.Y.Z` and publish a GitHub release with the same notes.
   - **patch / minor**: compatible — projects auto-merge the bump.
   - **major**: anything a caller must adapt to (a renamed or required input, a renamed job, a new required secret or permission). Projects get it as a PR that waits for a human.

Never move or reuse a tag: projects pin exact tags so that every change reaches them as a reviewable PR.
