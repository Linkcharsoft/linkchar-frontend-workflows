# Changelog

Releases follow semver from the caller's point of view: a **major** is anything a project's `ci.yml` must adapt to.

## 2.0.0

- **Breaking:** the job calling `dependabot-review.yml` must grant `actions: read` (a called workflow cannot raise its caller's permissions, so a v1 caller fails to start).
- `dependabot-review.yml`: when CI fails, the tail of each failed job's log is handed to the review, which now names the failing job and step, quotes the errors with their files and proposes the fix.

## 1.0.0

- `next-ci.yml`: **Lint** and **Build** (type-check + `next build`) for projects on `linkchar-next-template`, with dummy env values and an optional `build-env` input.
- `dependabot-review.yml`: Claude review of Dependabot PRs with a structured verdict, posted by the workflow; squash-merge of patch/minor bumps when the verdict is `ok` and CI succeeded, gated by `automerge`.
