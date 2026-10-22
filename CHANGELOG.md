# Changelog

Releases follow semver from the caller's point of view: a **major** is anything a project's `ci.yml` must adapt to.

## 2.3.0

- `dependabot-review.yml`: when the review clears a PR whose update type is not auto-merged (a major, or an unknown type), the Merge job now comments "Not auto-merged: …" instead of only logging it, so whoever follows the PR (a person or a panel) knows it is waiting to be tested.

## 2.2.0

- `next-ci.yml`: optional `PACKAGES_TOKEN` secret. When given, both jobs write it to the user's `.npmrc` before installing, so projects that depend on private `@linkcharsoft/*` packages (the Linkchar CMS) install from GitHub Packages. Projects that don't pass it are unaffected.

## 2.1.0

- `dependabot-review.yml`: new `trusted-actors` input (default `inferencia-runner[bot]`). A push by a trusted actor to a Dependabot PR re-runs the review, which reviews its commits as a code change and returns `ok` only if the fix is correct and minimal for the bump; such a PR can then merge. Commits by anyone else still block the merge. To use it, gate the calling job on the PR author (`github.event.pull_request.user.login == 'dependabot[bot]'`) instead of `github.actor`, and add `CLAUDE_CODE_OAUTH_TOKEN` as an Actions secret too — see the README. Callers that change nothing behave as before.
- `dependabot-review.yml`: the update type is the highest of `dependabot/fetch-metadata` and of the `updated-dependencies:` metadata in every Dependabot commit message, so the patch/minor-only rule holds when the PR carries extra commits. Dependabot commits must have a verified signature.
- `dependabot-review.yml`: steps renamed to "Check the review token" and "Review", and the posted summary is kept free of signatures and attribution lines.

## 2.0.0

- **Breaking:** the job calling `dependabot-review.yml` must grant `actions: read` (a called workflow cannot raise its caller's permissions, so a v1 caller fails to start).
- `dependabot-review.yml`: when CI fails, the tail of each failed job's log is handed to the review, which now names the failing job and step, quotes the errors with their files and proposes the fix.

## 1.0.0

- `next-ci.yml`: **Lint** and **Build** (type-check + `next build`) for projects on `linkchar-next-template`, with dummy env values and an optional `build-env` input.
- `dependabot-review.yml`: Claude review of Dependabot PRs with a structured verdict, posted by the workflow; squash-merge of patch/minor bumps when the verdict is `ok` and CI succeeded, gated by `automerge`.
