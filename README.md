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
- [`docker-image-release-reusable.yml`](./.github/workflows/docker-image-release-reusable.yml) — **Ekwi instances**:
  build the per-instance application image (`ekwi_prod`), scan it (Trivy), push it to GHCR (and optionally a
  self-hosted registry — dual-push), then cut the GitHub Release. The PR CI gate calls the **same** workflow with
  `dry-run: true` (build + scan only), so the build logic lives once.

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

## Usage — Ekwi instances (Docker image)

Two thin callers pointing at the **same** reusable: `release.yml` (real) and `ci.yml` (the PR gate, `dry-run`).

Scanning is split: **dependencies (Composer/npm under `/app`) block** the build (fixable here), **base OS is
report-only** (SARIF to the Security tab — needs `security-events: write`, best-effort). The dependency gate is
scoped with `trivy-lib-skip-dirs` (default `usr/local`) so base-shipped language runtime (global npm, FrankenPHP's
Go, PHP) is excluded — it is covered by docker-php's own scan, not the instance. This split is transitional: a
dated decision removes **both** passes from the instance image once the base-image gate covers the line that
instance builds on, app dependencies moving to Dependabot on the committed lockfiles. That gate exists on the `5`
line only, not on `4`, where most of the fleet still builds. Until it does, both passes stay.

⚠ **Never hard-code the base major in `build-args`.** It is not uniform across the fleet — an instance in
production runs an older one — so a value copied from this README builds against the wrong base image, then tags
it, pushes it and lets the production overlay pull it, all on a green run. Both snippets read it in a job the
calling repository owns. Where that job reads from is the caller's choice; the Ekwi fleet uses each instance's own
`.env`. A value no instance decides, such as `TEMP_DIRECTORY`, stays a literal — the rule is about what differs.

⚠ **These snippets assume the next-major Dockerfile of `instance-skeleton`**: the private-package token as a
BuildKit secret, and no `VITE_PORT` build arg. On the `0.5` line, where most instances still are, that Dockerfile
takes the token as an `ARG` — the build fails on the first private package whatever `build-args` carries, and
`VITE_PORT` is still a build input there. Do not adopt this flow before an instance moves to that line.

`release.yml`:

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
  build-inputs:
    name: Read the build inputs
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      build-args: ${{ steps.read.outputs.build-args }}
    steps:
      - uses: actions/checkout@<sha>   # pin by SHA
        with:
          persist-credentials: false
      - name: Read the non-secret build inputs
        id: read
        run: |
          set -euo pipefail
          [ -f .env ] || { echo "::error::.env not found at the repository root"; exit 1; }
          # An allow-list, not an exclusion: name what may leave this step, so a secret in the source cannot.
          # DEFAULT_LOCALE and LOCALE_LIST are in no .env today; the Dockerfile's own ARG defaults then apply.
          : > args
          for k in PHP_IMAGE_VERSION DEFAULT_LOCALE LOCALE_LIST; do
            # \r matters: a .env edited on Windows would otherwise make `FROM ekwi/php:4\r-builder`.
            v="$(grep -m1 "^${k}=" .env | cut -d= -f2- | tr -d "\"'\r" || true)"
            [ -n "${v}" ] && printf '%s=%s\n' "${k}" "${v}" >> args
          done
          printf 'TEMP_DIRECTORY=/app/var/tmp\n' >> args
          # No safe default for the base major — refuse rather than build on the wrong line.
          grep -q '^PHP_IMAGE_VERSION=' args || { echo "::error::PHP_IMAGE_VERSION is not set in .env"; exit 1; }
          { echo 'build-args<<ARGS'; cat args; echo 'ARGS'; } >> "$GITHUB_OUTPUT"

  release:
    needs: build-inputs
    permissions:
      contents: write         # create the tag + the GitHub Release
      packages: write         # push the image to GHCR
      actions: read           # the green gate reads CI conclusions
      pull-requests: read     # release notes resolve each commit's PR
      security-events: write  # upload the base OS Trivy report (SARIF) to the Security tab
    uses: ekwi-tech/release-workflow/.github/workflows/docker-image-release-reusable.yml@<sha>   # pin by SHA
    with:
      channel: ${{ inputs.channel }}
      initial-version: '0.1.0'
      ci-workflows: 'ci.yml'
      build-args: ${{ needs.build-inputs.outputs.build-args }}
      # self-hosted-registry: registry.ekwi.tech   # uncomment once it is up (dual-push; deploys pull from it)
    secrets:
      composer-auth: ${{ secrets.COMPOSER_AUTH }}
      # dockerhub-user / dockerhub-token: optional, to lift the Docker Hub base-image rate limit
      # registry-user / registry-token: required only when self-hosted-registry is set
```

`ci.yml` (PR gate — same reusable, `dry-run`, no push/release). It repeats `release.yml`'s `build-inputs` job
verbatim — the gate must build on the arguments the release will use — and is shown here without it:

```yaml
name: CI
on:
  pull_request: { branches: [main] }
  # Also on push to main: the release green gate needs a green ci.yml run on the EXACT release commit (a squash-merge
  # mints a new SHA the PR run never covered). This dry-run build is that signal.
  push: { branches: [main] }

permissions: {}

jobs:
  # build-inputs: copy the job from release.yml above, verbatim — the `needs` below names it, so this
  # snippet does not validate on its own.

  build:
    needs: build-inputs
    permissions:
      contents: read
      security-events: write  # upload the base OS Trivy report (SARIF); best-effort
    uses: ekwi-tech/release-workflow/.github/workflows/docker-image-release-reusable.yml@<sha>   # pin by SHA
    with:
      channel: pre-release      # ignored under dry-run, but the input is required
      dry-run: true
      build-args: ${{ needs.build-inputs.outputs.build-args }}
    secrets:
      composer-auth: ${{ secrets.COMPOSER_AUTH }}
```

Make **`build / <reusable job name>`** a required check in the ruleset — **two** segments, and see the naming rule
under Semantic Checks below. Select it from the detected-checks list rather than typing it.

Docker-specific inputs beyond the shared table: `image-name` (defaults to `github.repository`), `dockerfile`,
`build-target`, `build-context`, `build-args`, `platform`, `ghcr-registry`, `self-hosted-registry`,
`trivy-severity`, `trivy-lib-skip-dirs`.

## Usage — Conventional-Commit PR title (every repo)

Replace the repo's whole `commitlint.yml` job with a thin caller. No inputs — the vocabulary is fixed on purpose,
one everywhere:

```yaml
name: Semantic Checks
on:
  pull_request:
    types: [opened, edited, synchronize, reopened]

permissions: {}

jobs:
  title:
    name: PR title
    permissions:
      pull-requests: read     # the action reads the PR title through the automatic GITHUB_TOKEN
    uses: ekwi-tech/release-workflow/.github/workflows/commitlint-reusable.yml@<sha>   # pin by SHA
```

**Make the check required** in the repo's ruleset — but mind the name. A reusable-workflow check is named
`<caller job name> / <reusable job name>`, and no workflow `name:` appears in it — measured on this repository's
own check runs. With the caller above, that is `PR title / Conventional Commits on PR Title`. **Select it from the ruleset's detected-checks list** rather than
typing it. For a fleet-uniform required check, keep both halves stable across consumers — the job
`name: PR title` and (never reworded) the reusable's job name; changing either silently stops the gate blocking.

## Inputs (shared)

| Input | Required | Default | Notes |
|---|---|---|---|
| `channel` | yes | — | `pre-release` (X.Y.Z-beta.N) or `latest` (stable X.Y.Z). Forward the caller's dispatch input. |
| `initial-version` | no | `0.1.0` | Version for the very first release, when the repo has no tag yet. |
| `ci-workflows` | yes, except on a Docker dry-run, which skips the gate | — | Space-separated workflow filenames that must have a successful run on the release commit (the green gate). |
| `dry-run` | no | `false` | Skip the publish — no tag, no Release — and the main-branch and green-gate checks with it. In the Docker flow it also skips the version and **the push**: build and scan only, which is the PR gate. |

## Requirements

- A **releasing** caller must grant `contents: write`, `actions: read`, `pull-requests: read` (see the snippets);
  the **Docker image** flow additionally needs `packages: write` (push to GHCR) and, for the base OS report,
  `security-events: write` (SARIF upload; best-effort — a missing grant only skips the report). A **dry-run gate**
  grants `contents: read` and that same `security-events: write`, nothing more. ⚠ A Dependabot pull request is
  not given the repository's secrets, and the Docker flow requires `composer-auth`: set that value as a
  Dependabot secret too, or its dry-run cannot build. The reusable declares no permissions
  of its own and inherits the caller's grant — a reusable that asked for more than the caller granted would fail at
  startup, so the grant lives with the caller, once. The tag-only workflow needs no secrets — it uses only the
  automatic `GITHUB_TOKEN`; the Maven client passes the `packages-token` named secret; the Docker image flow passes
  `composer-auth` (required) and, optionally, `dockerhub-*` / `registry-*` (see that section).
- The actions and this workflow are **public**, so any consumer — public or private — resolves them with no
  access setting (see *Why this is public* below).

## Why this is public

A workflow body carries no secrets — it uses only the caller's automatically-provided `GITHUB_TOKEN` — and a
**public** consumer such as a fork cannot reference a **private** reusable workflow. Public is what lets every
consumer, public or private, use it with no access setting to forget.

Design rationale (why a reusable workflow rather than another composite action, why the one-way dependency, why
tag-only) is recorded in the fleet ADR in the docs vault.
