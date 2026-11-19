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
  build the per-instance application image (`ekwi_prod`), push it to GHCR (and optionally a
  self-hosted registry — dual-push), then cut the GitHub Release. The PR CI gate calls the **same** workflow with
  `dry-run: true` (build only), so the build logic lives once.

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

**This flow does not scan the image.** A dated decision moves OS and base scanning upstream to the repository
that owns the base — for `ekwi_prod` that is `ekwi/php`, and the gate lives on its `5` line only — and moves
application dependencies to Dependabot on the committed, fully resolved `app/composer.lock` and
`app/pnpm-lock.yaml`, which is what lets Dependabot see the tree actually shipped, transitive included.

⚠ **This is about the instance image only.** The Maven reusable keeps its own scan: nothing downstream of the
fat jar gates its dependencies, and the client repositories have no scanner of their own.

**Know what this costs, in three parts.**

1. **JS has no synchronous gate at ship time** any more, only Dependabot's asynchronous alert.
   `roave/security-advisories` (require-dev) still blocks a known Composer advisory at resolution time, so
   the PHP side keeps a gate; the JS side does not.
2. **The OS surface stops being re-examined on the fleet's cadence.** The pass removed here ran on every
   instance PR against a fresh vulnerability database. Upstream scanning runs *at base publication* and
   never again — `docker-php` has no scheduled scan. A CVE disclosed after a base is published is seen by
   nobody until the next base release.
3. ⚠ **An instance on the `4` line has no OS gate at all**, upstream or here.

⚠ **And check the precondition rather than assume it.** The upstream gate runs *at base publication*, so a
base published before the gate landed has never been through it — and a pre-release run does not move a major
tag. Compare the tag's `last_updated` with the date the gate merged before treating the hand-over as done:

```bash
curl -s https://hub.docker.com/v2/repositories/ekwi/php/tags/5 \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['last_updated'])"
```

⚠ **Never hard-code the base major in `build-args`.** It is not uniform across the fleet — an instance in
production runs an older one — so a value copied from this README builds against the wrong base image, then tags
it, pushes it and lets the production overlay pull it, all on a green run. Both snippets read it in a job the
calling repository owns. Where that job reads from is the caller's choice; the Ekwi fleet uses each instance's own
`.env`. A value no instance decides, such as `TEMP_DIRECTORY`, is a literal — it is the `constant-args` default of
the action below, so it reaches the build without appearing in the snippet.

The reading itself lives in a composite action of this repository, so a fix to the mechanism travels by a repin
rather than by editing every consumer:

| input | default | |
|---|---|---|
| `keys` | **none** | the allow-list. No default on purpose: a `.env` may hold secrets, so what leaves the step is each consumer's decision, stated where they can see it |
| `required-keys` | `PHP_IMAGE_VERSION` | must end up in the output, or the step fails — no safe default for the base major |
| `env-file` | `.env` | |
| `constant-args` | `TEMP_DIRECTORY=/app/var/tmp` | properties of the image layout, not of the instance |

⚠ Every value named in `keys` ends up in `--build-arg`, so it stays readable in the image's metadata and in
`docker history`.

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
      - uses: ekwi-tech/release-workflow/.github/actions/read-build-inputs@<sha>   # pin by SHA
        id: read
        with:
          keys: PHP_IMAGE_VERSION DEFAULT_LOCALE LOCALE_LIST
          # DEFAULT_LOCALE and LOCALE_LIST are in no .env today; the Dockerfile's own ARG defaults then apply.

  release:
    needs: build-inputs
    permissions:
      contents: write         # create the tag + the GitHub Release
      packages: write         # push the image to GHCR
      actions: read           # the green gate reads CI conclusions
      pull-requests: read     # release notes resolve each commit's PR
    uses: ekwi-tech/release-workflow/.github/workflows/docker-image-release-reusable.yml@<sha>   # pin by SHA
    with:
      channel: ${{ inputs.channel }}
      initial-version: '0.1.0'
      ci-workflows: 'ci.yml'
      build-args: ${{ needs.build-inputs.outputs.build-args }}
      # base-image-pins: |        # the SAME list in both callers, and EVERY base — see "Pinning the
      #   …                        # base images by digest" below; a partial list fails the build
      # self-hosted-registry: registry.ekwi.tech   # uncomment once it is up (dual-push; deploys pull from it)
      # self-hosted-push-registry: registry:5000   # only if writes go to a different name than reads
    secrets:
      composer-auth: ${{ secrets.COMPOSER_AUTH }}
      # dockerhub-user / dockerhub-token: optional, to lift the Docker Hub base-image rate limit
      # registry-user / registry-token: required only when self-hosted-registry is set
```

`ci.yml` (PR gate — same reusable, `dry-run`, no push/release). It repeats `release.yml`'s `build-inputs` job,
because the gate must build on the arguments the release will use. **The same `keys:` — not necessarily the same
job:** the gate may add steps of its own, and `instance-skeleton` adds a `composer.lock` check there that has no
place in a release. Shown here without that job:

```yaml
name: CI
on:
  pull_request: { branches: [main] }
  # Also on push to main: the release green gate needs a green ci.yml run on the EXACT release commit (a squash-merge
  # mints a new SHA the PR run never covered). This dry-run build is that signal.
  push: { branches: [main] }

permissions: {}

jobs:
  # build-inputs: copy the job from release.yml above — the `needs` below names it, so this snippet does not
  # validate on its own. Keep its `keys:` identical to the release's, or the gate builds on other arguments.

  build:
    needs: build-inputs
    permissions:
      contents: read
    uses: ekwi-tech/release-workflow/.github/workflows/docker-image-release-reusable.yml@<sha>   # pin by SHA
    with:
      channel: pre-release      # ignored under dry-run, but the input is required
      dry-run: true
      build-args: ${{ needs.build-inputs.outputs.build-args }}
      # base-image-pins: |        # the SAME list in both callers, and EVERY base — see "Pinning the
      #   …                        # base images by digest" below; a partial list fails the build
    secrets:
      composer-auth: ${{ secrets.COMPOSER_AUTH }}
```

Make **`build / <reusable job name>`** a required check in the ruleset — **two** segments, and see the naming rule
under Semantic Checks below. Select it from the detected-checks list rather than typing it.

Docker-specific inputs beyond the shared table: `image-name` (defaults to `github.repository`), `dockerfile`,
`build-target`, `build-context`, `build-args`, `base-image-pins`, `platform`, `ghcr-registry`,
`self-hosted-registry`, `self-hosted-push-registry`.

### Writing and reading a registry under two names

A self-hosted registry can answer to two names and accept writes on only one — the public name is served by
an ingress that refuses `PUT POST PATCH DELETE` whatever the source, while the registry itself listens on its
own network. `self-hosted-push-registry` names the host to **write** to:

```yaml
      self-hosted-registry: registry.ekwi.tech   # what deployments pull
      self-hosted-push-registry: registry:5000   # what the runner writes to
```

The login, the tag and the push all use the write host. **A registry stores `<repository>:<tag>` and no
host**, so the image written as `registry:5000/<name>:<version>` is the one pulled as
`registry.ekwi.tech/<name>:<version>` — the two names serve the same registry. The run log prints the
reference it pushed and the one it is pullable as; the job never contacts the second.

Leaving the input empty writes to `self-hosted-registry`, which is what every caller does today.

⚠ **The write host must look like a host to Docker.** A bare lowercase `registry` resolves to Docker Hub
while the login goes to the host you named — measured. The run refuses it, refuses anything that is not
`name` or `name:port` (no scheme, no path), and refuses a write host with no `self-hosted-registry` beside
it.

⚠ **Two things this input does not give you.** It is the **daemon** that pushes, so the daemon has to
resolve the write host and accept it on plain HTTP where that applies — that side belongs to
`ekwi-tech/tooling-infra#36`. And the runner is chosen **here**: this workflow is `runs-on: ubuntu-latest`,
hard-coded, so until that changes you get a GitHub-hosted runner that resolves no internal name. Adding an
input for it is not obviously enough — the same workflow serves the pull-request gate in `dry-run`, which
has to stay away from the write path. *Reasoning, not measurement.*

⚠ `registry-user` and `registry-token` are the credentials of the **write** host.

### Pinning the base images by digest

`ekwi/php:<major>` is a **major-line tag, re-published in place**: two builds of the same commit, days apart,
can sit on different bytes. `base-image-pins` closes that — one `<ref>=sha256:<hex>` per line, each becoming a
named build context that substitutes the matching reference. For `ekwi_prod` of the next-major skeleton that
is **three** lines, not two:

```yaml
      base-image-pins: |
        docker/dockerfile:1.10=sha256:<64 lowercase hex>   # the `# syntax=` frontend
        ekwi/php:5=sha256:<64 lowercase hex>               # the runtime stage
        ekwi/php:5-builder=sha256:<64 lowercase hex>       # the builder stage
```

⚠ **Everything buildx pulls counts, not just `FROM` lines.** The `# syntax=` frontend on line 1 of the
Dockerfile is a material like any other, and so is an image named by `COPY --from=<image>` or
`RUN --mount=from=<image>`. Only a `COPY --from=<stage>` — an internal stage — is exempt. Miss one and the
build is refused; the error names the image and prints the line to add. `ekwi_dev` is not in `ekwi_prod`'s
graph, so its base is not pinned here.

⚠ **The ref is the reference AFTER the `ARG` expansion** — `ekwi/php:5-builder`, never
`ekwi/php:${PHP_IMAGE_VERSION}-builder`, which buildx rejects outright.

⚠ **Pass the same list to the PR gate as to the release** — same rule as `keys:` above, and for the same
reason: a gate that builds on other bytes than the release does not gate that release.

Read a digest with `docker buildx imagetools inspect ekwi/php:5 | awk '/^Digest:/{print $2; exit}'`, or take
it from the **Images by digest** table every run writes to its summary. ⚠ Not
`--format '{{.Manifest.Digest}}'`: a template whose body is only a `.Manifest…` reference silently falls back
to the full 10 KB human output.

**What the check does, and what it cannot do.** When the input is set, the build fails unless every base was
resolved *through a pin*. That is not belt-and-braces: buildx **silently ignores** a `--build-context` key
that matches no reference, and on the day you read the digests the tag resolves to those very digests — so a
typo yields a base that is unpinned *and* carries a listed digest. Comparing digest values alone passes it.
The workflow therefore reads the build provenance, where a pinned base and a tag-resolved one are written
differently, and refuses the second whatever its digest.

⚠ **It proves the build followed your list. It cannot prove the list is right.** Two digests swapped between
`ekwi/php:5` and `ekwi/php:5-builder` are obeyed to the letter and pass — measured. A digest belonging to a
different image does not get that far: buildx fails to pull it.

⚠ **Do not pin in the consumer's `Dockerfile`.** Template sync merges `-X theirs` without saying so and
excludes neither the Dockerfile nor the compose file, so a digest written there is force-overwritten in every
instance. (That sync is being replaced by `copier`; until it is, this holds.) The job pins, not the file.

⚠ **There is no path from `.env` to this input.** The composite action above emits `--build-arg` lines only;
a `PHP_IMAGE_DIGEST` added to `keys:` would travel as a build arg and pin nothing. Keeping the digests in
`.env` is reasonable — it *is* excluded from template sync — but the caller has to read them into this input
itself.

Leaving the input empty keeps the previous behaviour: bases resolve through their tag, nothing is verified,
and the run summary says so.

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
typing it. Both halves must then stay stable: if either moves, the check you made required **stops arriving**
and your pull requests block. Our half is pinned in `.github/published-check-names.txt` and cannot move without
failing our CI first. Yours — the caller's job `name:` — is pinned by nothing but you.

## Inputs (shared)

| Input | Required | Default | Notes |
|---|---|---|---|
| `channel` | yes | — | `pre-release` (X.Y.Z-beta.N) or `latest` (stable X.Y.Z). Forward the caller's dispatch input. |
| `initial-version` | no | `0.1.0` | Version for the very first release, when the repo has no tag yet. |
| `ci-workflows` | yes, except on a Docker dry-run, which skips the gate | — | Space-separated workflow filenames that must have a successful run on the release commit (the green gate). |
| `dry-run` | no | `false` | Skip the publish — no tag, no Release — and the main-branch and green-gate checks with it. In the Docker flow it also skips the version and **the push**: build only, which is the PR gate. |

## Requirements

- A **releasing** caller must grant `contents: write`, `actions: read`, `pull-requests: read` (see the snippets);
  the **Docker image** flow additionally needs `packages: write` (push to GHCR). A **dry-run gate** grants
  `contents: read`, nothing more. ⚠ `security-events: write` is no longer used by this flow: a caller still
  granting it is giving a scope nothing consumes. ⚠ A Dependabot pull request is
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
