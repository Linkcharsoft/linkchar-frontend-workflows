# Changelog

Releases follow semver from the caller's point of view: a **major** is anything a project's `ci.yml` must adapt to.

## 1.0.0

- `next-ci.yml`: **Lint** and **Build** (type-check + `next build`) for projects on `linkchar-next-template`, with dummy env values and an optional `build-env` input.
- `dependabot-review.yml`: Claude review of Dependabot PRs with a structured verdict, posted by the workflow; squash-merge of patch/minor bumps when the verdict is `ok` and CI succeeded, gated by `automerge`.
