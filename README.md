# linkchar-frontend-workflows

Shared, versioned GitHub Actions workflows for Linkchar's **frontend** projects (the Next template today).

**Frontend only, on purpose.** Tags version the whole repository, so a release here becomes a Dependabot bump PR — review, auto-merge, production redeploy — in every project that calls it. Backend workflows belong in their own repository following the same pattern, so neither team's releases reach the other's projects.

Each project calls these workflows from a ~25-line `ci.yml` pinned to a release tag. Dependabot bumps that tag like any other action, so improving a workflow here means one release, and every project receives it as a reviewed PR.

| Workflow | What it does |
| --- | --- |
| [`next-ci.yml`](.github/workflows/next-ci.yml) | CI for projects built on `linkchar-next-template`: **Lint** (ESLint) and **Build** (type-check + `next build`) on a bare checkout, with dummy env values that satisfy the template's boot checks. |
| [`dependabot-review.yml`](.github/workflows/dependabot-review.yml) | On Dependabot PRs only: Claude reads the changelog of every bumped package, checks whether the project uses what changed — and, when CI fails, reads the failed jobs' logs — and returns a verdict; the workflow posts it as a comment and squash-merges patch/minor bumps that passed review **and** CI. Fix commits pushed to the PR by a [trusted actor](#fix-commits-from-a-trusted-actor) are reviewed as a code change in the same pass. |

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
    uses: Linkcharsoft/linkchar-frontend-workflows/.github/workflows/next-ci.yml@v2.2.0

  dependabot:
    needs: ci
    # Dependabot's PRs, whoever pushed last: the called workflow decides which pushers it trusts.
    if: ${{ !cancelled() && github.event.pull_request.user.login == 'dependabot[bot]' }}
    uses: Linkcharsoft/linkchar-frontend-workflows/.github/workflows/dependabot-review.yml@v2.2.0
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

A caller still gating the job on `github.actor == 'dependabot[bot]'` (the v2.0.0 example) keeps working, but never runs the review for a trusted actor's push: switch the `if` to the PR author as above to get fix commits reviewed and merged.

### Inputs

`next-ci.yml`

| Input | Default | Purpose |
| --- | --- | --- |
| `node-version-file` | `.nvmrc` | File that pins Node. |
| `build-env` | `""` | Extra `KEY=VALUE` lines for the Build job, one per line — for a project whose build requires more variables than the template's. |

| Secret | Purpose |
| --- | --- |
| `PACKAGES_TOKEN` | Optional. A token with `read:packages` on Linkcharsoft, written to the user's `.npmrc` before installing in both jobs, for projects that depend on private `@linkcharsoft/*` packages. |

#### Projects with the Linkchar CMS

`@linkcharsoft/cms` installs from GitHub Packages, and its build reads the published content from Supabase (the sitemap and every CMS page prerender, and the redirects past publishes left). The project passes the token and its Supabase project's public values:

```yaml
  ci:
    uses: Linkcharsoft/linkchar-frontend-workflows/.github/workflows/next-ci.yml@v2.2.0
    secrets:
      PACKAGES_TOKEN: ${{ secrets.PACKAGES_TOKEN }}
    with:
      build-env: |
        NEXT_PUBLIC_SUPABASE_URL=${{ vars.SUPABASE_URL }}
        NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=${{ vars.SUPABASE_PUBLISHABLE_KEY }}
```

`PACKAGES_TOKEN` goes under the project's *Secrets and variables* both in **Actions** and in **Dependabot** (Dependabot runs read only their own secrets); `SUPABASE_URL` and `SUPABASE_PUBLISHABLE_KEY` are repository **variables** (public values, the same the browser gets). Until the client's Supabase project exists, the Build job fails at `/sitemap.xml` with "Missing the Supabase URL".

`dependabot-review.yml`

| Input | Default | Purpose |
| --- | --- | --- |
| `ci-result` | — (required) | `needs.<ci job>.result`. Only `success` can merge. |
| `automerge` | `minor` | `minor` merges patch/minor bumps; any other value (use `off`) only reviews. Set the `DEPENDABOT_AUTOMERGE` repo variable to change it per project without touching the YAML. |
| `model` | `claude-sonnet-5-5` | Model used for the review. |
| `max-turns` | `40` | Caps the agent's turns on large grouped PRs. |
| `trusted-actors` | `inferencia-runner[bot]` | Comma-separated logins, besides Dependabot, allowed to push fix commits to a Dependabot PR and still get it reviewed and merged. See [below](#fix-commits-from-a-trusted-actor). |

### Merge policy

A Dependabot PR is merged only when **all** of these hold:

1. CI (`Lint` + `Build`) succeeded.
2. Claude's verdict is `ok`.
3. The highest update in the PR is a patch or minor — the highest of `dependabot/fetch-metadata` and of the `update-type` lines in every Dependabot commit message, never the agent's word. Majors are reviewed and commented, never merged; an update type that cannot be determined never merges either.
4. Every commit in the PR is Dependabot's (with a verified signature) or a trusted actor's, the run was triggered by one of them, and the head is still the reviewed commit (`--match-head-commit`).
5. `automerge` is `minor`.

The squash commit is titled `[ CHORE ] Bump … (#N)`, following the commit convention.

> **Merging to `main` deploys to production.** Amplify builds `main` on push, so an auto-merged bump ships without a human in the loop. Lint, types and the build are the gate; the Claude verdict is a changelog review, not a security control — the 7-day `cooldown` in `dependabot.yml` is what keeps a freshly published malicious release out.

### Fix commits from a trusted actor

A PR the review does not clear stays open for a human — or for a bot that fixes it. When an actor listed in `trusted-actors` pushes a commit to the Dependabot branch, CI and the review run again on the new head:

- The run must be triggered by Dependabot or a trusted actor, and the PR must be Dependabot's. A push by anyone else skips the review, so nothing merges.
- Every commit is classified by its GitHub author: Dependabot's (signature verified), a trusted actor's, or anyone else's. The trusted actor must author its commits as its own account (for a GitHub App, `<id>+<app>[bot]@users.noreply.github.com`); a commit whose email maps to no account counts as someone else's.
- The trusted actor's commits are listed for the review, which reviews them as a code change: the verdict is `ok` only if the fix is correct and minimal for the bump, and CI passed.
- Any commit by someone else blocks the merge; the verdict comment says which.
- The update type comes from Dependabot's own commit messages, so extra commits do not let a major through.

## Setup

1. **Secret, in every calling project** — create `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`) **twice**, with the same value: under the project's *Settings → Secrets and variables → **Dependabot*** and under *Settings → Secrets and variables → **Actions***. Runs triggered by Dependabot only read Dependabot secrets; runs triggered by a trusted actor's push (e.g. a GitHub App) only read Actions secrets. The caller passes it in, so a secret stored in **this** repository never reaches anyone. On GitHub Free organization secrets do not reach private repositories — on a paid plan one organization secret of each kind, shared with the frontend repositories, replaces the per-repo ones. Without it the review is skipped with a warning and nothing merges.
2. **Access** — keep this repository **public** (it holds no secrets): callers and Dependabot then reach it on any plan. If it must be private, allow access from the organization's repositories under *Settings → Actions → General → Access*, and add it under *Organization settings → Advanced Security → Grant Dependabot access to private repositories*.

## Releasing

1. Change the workflow on a branch. To try it before releasing, point a project's `ci.yml` at the branch (`@my-branch`) in a throwaway PR — the template is the natural test bed.
2. Add an entry to [`CHANGELOG.md`](CHANGELOG.md): the review of each project's bump PR reads it to explain what changed.
3. Merge, then tag `vX.Y.Z` and publish a GitHub release with the same notes.
   - **patch / minor**: compatible — projects auto-merge the bump.
   - **major**: anything a caller must adapt to (a renamed or required input, a renamed job, a new required secret or permission). Projects get it as a PR that waits for a human.

Never move or reuse a tag: projects pin exact tags so that every change reaches them as a reviewable PR.
