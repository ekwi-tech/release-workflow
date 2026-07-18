# release-workflow

The ekwi-tech fleet's shared **reusable workflows** — behaviour that was being copied into every repo, kept in
one place so a consumer carries a thin caller and the fleet adopts a change through its ref.

**Release orchestration** — one reusable per ecosystem, because the steps differ where they must and are shared
where they can:

- [`composer-release-reusable.yml`](./.github/workflows/composer-release-reusable.yml) — **tag-only** repos (a
  Composer package, a CLI, this repo itself): publish by creating a git tag + GitHub Release.
- [`maven-sync-app-release-reusable.yml`](./.github/workflows/maven-sync-app-release-reusable.yml) — the
  **ekwi-sync clients**: build a Spring Boot fat jar, tag, attach the jar + SBOM + checksums (deployment is
  manual). `ekwi-sync-common` is a multi-module library that *deploys* to the Maven registry — a different flow,
  kept bespoke rather than forced into a shared shape.

All derive the version, prove CI is green and render the notes the same way; only the publish differs. The steps
were being copied into every `release.yml`; here they live once, and a consumer carries a ~12-line caller.

**Quality gate** — shared across every repo regardless of ecosystem:

- [`commitlint-reusable.yml`](./.github/workflows/commitlint-reusable.yml) — enforce a Conventional-Commit PR
  title (the action pin and the ten-type vocabulary, once). This feeds the same commits that
  next-version-action and release-notes-action read, so an unparseable title is a correctness problem, not a
  style one.

Each reusable workflow sits one level above the step-level actions and **uses** them:

```
your repo ──▶ release-workflow ──▶ next-version-action   (version policy)
                               └─▶ release-notes-action   (notes rendering)
```

The dependency is **one-way**: the two actions do *not* use these workflows — they dogfood their own release —
so there is no cycle. This repo dogfoods itself the same way ([`release.yml`](./.github/workflows/release.yml)
calls the tag-only reusable workflow).

## Usage — tag-only (Composer) consumers

Replace your repo's entire release job with a thin caller:

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      channel:
        description: 'Release channel — pre-release cuts X.Y.Z-beta.N, latest cuts a stable X.Y.Z.'
        required: true
        default: pre-release
        type: choice
        options: [pre-release, latest]

permissions: {}

jobs:
  release:
    permissions:
      contents: write        # create the tag + the GitHub Release
      actions: read          # the green gate reads CI conclusions
      pull-requests: read     # release-notes-action resolves each commit's PR
    # No concurrency block needed — the "one release at a time" guard is centralised in the reusable workflow.
    uses: ekwi-tech/release-workflow/.github/workflows/composer-release-reusable.yml@<sha>   # pin by SHA
    with:
      channel: ${{ inputs.channel }}
      initial-version: '2.2.0'          # your first-release version
      ci-workflows: 'test.yml quality.yml'   # workflows that must be green on the release commit
```

## Usage — ekwi-sync clients (Maven)

Same shape, pointing at the Maven reusable, plus the cross-repo registry token as a named secret:

```yaml
jobs:
  release:
    permissions:
      contents: write
      actions: read
      pull-requests: read
    uses: ekwi-tech/release-workflow/.github/workflows/maven-sync-app-release-reusable.yml@<sha>   # pin by SHA
    with:
      channel: ${{ inputs.channel }}
      initial-version: '0.1.0'
      ci-workflows: 'ci.yml'
    secrets:
      packages-token: ${{ secrets.EKWI_PACKAGES_TOKEN }}   # reads ekwi-sync-common's packages (cross-repo)
```

It also accepts `java-version` (default `25`) and `java-distribution` (default `temurin`).

## Usage — Conventional-Commit PR title (every repo)

Replace the repo's whole `commitlint.yml` job with a thin caller. No inputs — the vocabulary is fixed on purpose,
one everywhere:

```yaml
name: Semantic PR Title
on:
  pull_request:
    types: [opened, edited, synchronize, reopened]

permissions: {}

jobs:
  title:
    permissions:
      pull-requests: read     # the action reads the PR title through the automatic GITHUB_TOKEN
    uses: ekwi-tech/release-workflow/.github/workflows/commitlint-reusable.yml@<sha>   # pin by SHA
```

**Make the check required** in the repo's ruleset — but mind the name. Routing through a reusable renames the
check to `<caller-job> / validate-pr-title`, i.e. `title / validate-pr-title` with the job named above. Point the
ruleset's required status check at that exact name, or the gate silently stops blocking.

## Inputs (shared)

| Input | Required | Default | Notes |
|---|---|---|---|
| `channel` | yes | — | `pre-release` (X.Y.Z-beta.N) or `latest` (stable X.Y.Z). Forward the caller's dispatch input. |
| `initial-version` | no | `0.1.0` | Version for the very first release, when the repo has no tag yet. |
| `ci-workflows` | yes | — | Space-separated workflow filenames that must have a successful run on the release commit (the green gate). |
| `dry-run` | no | `false` | Run everything except tag/release creation — for a self-test. Also skips the main-branch and green-gate checks. |

## Requirements

- The caller **must** grant `contents: write`, `actions: read`, `pull-requests: read` (see the snippets). The
  reusable declares no permissions of its own and inherits the caller's grant — a reusable that asked for more
  than the caller granted would fail at startup, so the grant lives with the caller, once. The tag-only workflow
  needs no secrets — it uses only the automatic `GITHUB_TOKEN`; the Maven client additionally passes the
  `packages-token` named secret shown above.
- The actions and this workflow are **public**, so any consumer — public or private — resolves them with no
  access setting (see *Why this is public* below).

## Why this is public

A workflow body carries no secrets — it uses only the caller's automatically-provided `GITHUB_TOKEN` — and a
**public** consumer such as a fork cannot reference a **private** reusable workflow. Public is what lets every
consumer, public or private, use it with no access setting to forget.

Design rationale (why a reusable workflow rather than another composite action, why the one-way dependency, why
tag-only) is recorded in the fleet ADR in the docs vault.
