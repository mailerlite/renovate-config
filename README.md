# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) configurations for MailerLite repositories.

## Configs

| Config | Purpose |
|--------|---------|
| `base` | Common defaults inherited by all configs |
| `app`  | Application/microservice repositories. Never auto-merges |
| `service` | SRE-run services and containers that build on our base images: auto-merges base image bumps |
| `fleet` | Every Kubernetes Flux cluster repository: auto-merges internal images in dev/staging and on `clusters-hub` |
| `no-automerge` | Add after `app`, `service` or `fleet` to opt a repo out of all auto-merge |
| `flux`  | Deprecated, no repo uses it. Don't use it in new repos |

---

## Usage

Add a `.github/renovate.json` to your repository and extend the appropriate config:

**Application repo:**
```json
{
  "extends": ["github>mailerlite/renovate-config:app"]
}
```

**Service repo** (builds on `mailerlitehub/base-images`, released weekly):
```json
{
  "extends": ["github>mailerlite/renovate-config:service"]
}
```

**Flux cluster repo:**
```json
{
  "extends": ["github>mailerlite/renovate-config:fleet"]
}
```

**Other repo (not an app or Flux cluster):**
```json
{
  "extends": ["github>mailerlite/renovate-config:base"]
}
```

**Opt out of auto-merge** (main applications, or any repo where a person must merge every update):
```json
{
  "extends": ["github>mailerlite/renovate-config:app", "github>mailerlite/renovate-config:no-automerge"]
}
```

---

## How Renovate runs

Renovate runs in `mailerlite/clusters-hub` on [renovate-operator](https://github.com/mogenius/renovate-operator), one `RenovateJob` per org under `cluster/releases/renovate/`. A run is started in two ways:

- **Schedule:** weekdays only. mailerlite 03:00, mailersend 04:00, mailercheck 05:00 UTC.
- **Webhook:** a service's `build.yml` calls `mailerlite/workflows/.github/workflows/renovate.yaml` after it pushes a new image. That starts a run on the cluster repos straight away, so they get the image bump the same day instead of the next weekday morning. Repos without this job wait for the schedule.

A run still follows each rule's `schedule`, so a weekend tag gets its PR on Monday.

### Auto-merge

The `automerge` label means the PR merges itself. A rule that sets the label without `automerge: true` is a bug.

How a PR with `automerge: true` gets merged:

1. Renovate turns on GitHub auto-merge for the PR. GitHub merges it as soon as the required checks pass. This needs **Settings → General → Allow auto-merge** on the repo.
2. If that setting is off, Renovate merges the PR itself on its next run, once every check is green. So the setting decides how fast a PR merges, not whether it merges.

The required check is `Kritika / Review`, from the org ruleset **AutoMerge Kritika** (one per org, default branch only). A repo that auto-merges must be in that ruleset, or a PR can merge before Kritika reviews it. People with the Write, Maintain or Admin role bypass the ruleset, so it does not change how they push or merge. Renovate and other GitHub Apps can't bypass it.

To turn auto-merge on for a repo: extend `service` or `fleet`, add the repo to the org's AutoMerge Kritika ruleset, and turn on Allow auto-merge.

### Weekly flow

```
Mon 01:00  base-docker-images rebuild-weekly     patch release of every base image
Mon 03:00  Renovate (schedule), service repos    base image PR → Kritika → auto-merge   [service]
Tue        service rebuild-weekly                patch release tag, now on the new base
           service build.yml                     image pushed to mailerlitehub/containers, webhook
           Renovate (webhook), cluster repos     dev PR → Kritika → auto-merge          [fleet]
                                                 production PR → a person merges it
```

Production is manual everywhere except `clusters-hub`, and except images that a cluster repo opts in with its own rule (e.g. `mailerlite-mcp` in `mailerlite/clusters-fleet`).

---

## Package policy

Packages are classified as **internal** or **external** based on their origin.

### Internal packages

| Type | Pattern |
|------|---------|
| Docker | `europe-docker.pkg.dev/mailerlitehub/` |
| Docker | `europe-docker.pkg.dev/mailerlite-gcp/` |
| Docker | `europe-docker.pkg.dev/mailercheck-usa/` |
| Docker | `europe-docker.pkg.dev/mailersend-214215/` |
| Docker | `europe-docker.pkg.dev/mailerlite-v2/` |
| GitHub Actions / Releases | `mailerlite/*`, `mailersend/*`, `mailercheck/*` |

Internal packages:
- No `minimumReleaseAge` — PRs are created immediately on release
- No digest pinning (except internal Docker images in Flux cluster repos — see below)
- Patch, minor and digest updates auto-merge only where a config says so (`service` base images, `fleet` dev/staging and `clusters-hub`)
- Major updates create a PR with no auto-merge

### External packages

All packages not matching the internal patterns above.

- `minimumReleaseAge: 3 days` — a PR is only created after the version has been available for 3 days (supply chain safety buffer)
- No auto-merge — all updates require manual review
- Docker images and GitHub Actions are digest-pinned

### Vulnerability alerts

Applies to all packages regardless of internal/external classification:

- `minimumReleaseAge` is bypassed — security PRs are opened immediately
- PRs receive a `security` label and are prioritised above all other updates
- Powered by OSV (`osvVulnerabilityAlerts`)

### Ignored packages

| Package | Reason |
|---------|--------|
| `mailerlite/workflows` | Reusable workflow repo, pinned at `main`, no versioned releases |
| `mailerlite/base-docker-images` (as GH Action) | Reusable workflow repo, pinned at `main`, no versioned releases |
| `elasticsearch` majors | Intentional hold — major Elasticsearch upgrades require manual intervention |

---

## What each config does

### base

Applied to all configs. Sets organisation-wide defaults:

- Semantic commit messages
- Dependency dashboard with auto-close
- Digest pinning enabled globally for external packages
- `minimumReleaseAge: 3 days` for external packages; `0 days` for internal
- Vulnerability alerts bypass the age gate and are prioritised
- Digest-only pin updates for external images grouped into a single PR

**Managers enabled by default:** `dockerfile`, `docker-compose`, `github-actions`, `custom.regex`

Repos extending `base` directly get these managers. Repos extending `app`, `service` or `fleet` override this list with their own.

### app

For application and microservice repositories. Managers:

| Manager | What it tracks |
|---------|---------------|
| `dockerfile` | `FROM` and `ARG`/`ENV` versions in `Dockerfile*` and `Makefile` |
| `docker-compose` | `image:` fields in compose files |
| `helmv3` | Helm chart dependencies in `Chart.yaml` |
| `github-actions` | `uses:` in `.github/workflows/` |
| `custom.regex` | Annotated versions (see Annotations section) |

Internal Docker images in Dockerfiles and compose files are **not** digest-pinned. External images are.

Nothing auto-merges. Every update creates a PR.

### service

Extends `app`. For SRE-run services and containers that build `FROM` our base images, including SRE-run repos that also have a skaffold dev flow (e.g. `robusta`, `tracking-ingress`, the mailersend `smtp-*` services). Not for the main applications.

- Enables language managers as well as `app`'s: `gomod`, `npm`, `composer`, `pep621`, `pip_requirements`, `poetry`, `bundler`, `cargo`. Test and vendor directories are ignored (`:ignoreModulesAndTests`).
- **Auto-merge:** only base image bumps (patch, minor, digest), on weekdays once checks pass: `europe-docker.pkg.dev/mailerlitehub/base-images/*` and the `mailerlitehub/containers/trixie-node-*` images built in `mailerlite/containers`. Every other update gets a PR.

This is the first half of the [weekly flow](#weekly-flow).

### fleet

For every Kubernetes Flux cluster repository. Supports both layouts: `dev/` + `production/` (the `clusters-fleet` repos) and `clusters/staging/` + `clusters/production/` + `base/` (`clusters-classic`). `clusters-hub` (`cluster/`) is a single production cluster with no environment folders. Managers:

| Manager | What it tracks |
|---------|---------------|
| `flux` | HelmRelease, HelmRepository, GitRepository, etc. |
| `kubernetes` | Raw Kubernetes manifests |
| `github-actions` | `uses:` in `.github/workflows/` |
| `custom.regex` | Annotated versions in cluster YAML |
| `custom.jsonata` | app-template `image` fields (repository/tag/digest) |

**Environment awareness:** PRs are labelled and branch-prefixed by environment based on file path: `dev`, `staging`, `production` and `base`. Minor and patch updates are always separate PRs.

**Digest pinning:** Internal Docker images in Flux cluster files **are** digest-pinned (unlike in app repos). This ensures cluster deployments are fully deterministic. The `flux` manager itself has digest pinning disabled because the inline `tag@digest` format is not valid in HelmRelease values — digest tracking is handled by the custom managers instead.

**Auto-merge:** dev/staging and `clusters-hub` auto-merge internal images; every other production cluster gets a PR for a person to merge. The app-template chart is excluded and `pgbouncer` never auto-merges.

| Update type | dev / staging | `clusters-hub` | other production |
|-------------|---------------|----------------|------------------|
| Internal Docker patch / minor / digest | Auto-merge on weekdays once checks pass | Auto-merge on weekdays once checks pass | PR only |
| Internal Docker major | PR only | PR only | PR only |
| External | PR only | PR only | PR only |

A cluster repo can opt one image in to production auto-merge with a rule in its own config. `mailerlite/clusters-fleet` does this for `mailerlite-mcp`, whose team controls production by cutting release tags.

---

## Annotations

Renovate can only update values it can detect. For cases where the version lives in a non-standard location, add a `# renovate:` comment on the line above.

### Dockerfile — ENV variable

Use when a tool version is stored in an `ENV` or `ARG` and installed manually (e.g. via `curl`).

```dockerfile
# renovate: datasource=docker depName=gcr.io/google.com/cloudsdktool/cloud-sdk
ENV CLOUD_SDK_VERSION="555.0.0"
```

Supported fields in the comment:

| Field | Required | Description |
|-------|----------|-------------|
| `datasource` | Yes | e.g. `docker`, `github-releases`, `pypi` |
| `depName` | Yes | Full package/image name |
| `versioning` | No | Override versioning scheme, e.g. `loose` |
| `extractVersion` | No | Regex to extract the version from the upstream tag |
| `currentDigest` | No | Written back by Renovate when pinning digests |

### Dockerfile — internal images

Internal images from all GCP artifact registries are tracked automatically — no annotation needed. `mailerlitehub` images (including all `base-images/*`) are handled by the `app` preset's custom regex managers; images from the other registries by the standard Dockerfile manager.

```dockerfile
FROM europe-docker.pkg.dev/mailerlitehub/octopus/octopus:1.2.3

COPY --from=europe-docker.pkg.dev/mailerlite-gcp/myimage/myimage:1.0.0 /bin/tool /bin/tool
```

For `mailerlitehub` images the image reference is detected in `FROM`, `COPY --from=` and `ARG <NAME>=` lines:

```dockerfile
# Literal tag - the whole tag is tracked, the language suffix (-1.26) is kept as is
ARG BASE_IMAGE=europe-docker.pkg.dev/mailerlitehub/base-images/go:v1.0.34-1.26
FROM ${BASE_IMAGE} AS build

# Tag ending in a build arg - only the version before the variable is tracked,
# v1.1.4 is bumped and -${PHP_VERSION} is left untouched
ARG PHP_VERSION=8.4
FROM europe-docker.pkg.dev/mailerlitehub/base-images/php-apps:v1.1.4-${PHP_VERSION} AS base
```

Base images are tagged `v<semver>-<language version>`. Legacy tags such as `go-v1.0.8-1.26` or `base-v1.0.14` are not valid versions and will not receive updates — switch them to the `v1.0.x-…` form once by hand.

Internal images in Dockerfiles are **not** digest-pinned.

### Helm values — image field

Use in `helmchart/values.(dev|prod|staging|production).yaml` files when an image tag is defined inline.

```yaml
# renovate: datasource=docker depName=europe-docker.pkg.dev/mailerlite-v2/landings/landings
image: europe-docker.pkg.dev/mailerlite-v2/landings/landings:1.5.0@sha256:abc123...
```

### Flux cluster — OCI Helm chart

Use for Helm charts served from an OCI container registry (datasource is `docker`, not `helm`).

```yaml
# renovate: datasource=docker registryUrl=https://europe-docker.pkg.dev/mailerlitehub/charts chart=pgbouncer
version: 1.2.3
```

### Flux cluster — app-template `repository` + `tag` (automatic)

HelmReleases using [bjw-s/app-template](https://github.com/bjw-s-labs/app-template) are detected automatically — no annotation needed. Works for internal images, Docker Hub, and ghcr.io.

Renovate handles digest pinning in two ways depending on whether a `digest:` field is present:

**With `digest:` field** — digest is written to the dedicated field, tag stays clean:

```yaml
image:
  repository: europe-docker.pkg.dev/mailerlitehub/screenshoter/screenshoter
  tag: v3.2.3
  digest: sha256:abc123...
```

**Without `digest:` field** — digest is appended inline to the tag:

```yaml
image:
  repository: europe-docker.pkg.dev/mailerlitehub/screenshoter/screenshoter
  tag: v3.2.3@sha256:abc123...
```

To switch a resource from inline to separate field, manually add `digest: ""` once — Renovate will populate and maintain it from that point on.

> Detection uses JSONata queries, so no `# renovate:` annotation is needed.

### Flux cluster — image field

Use when the full image reference (including tag) appears on a single line.

```yaml
# renovate: datasource=docker depName=europe-docker.pkg.dev/mailerlitehub/myapp/myapp
image: europe-docker.pkg.dev/mailerlitehub/myapp/myapp:2.0.1@sha256:abc123...
```

### Pre-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/some-org/some-hook
    # renovate: datasource=github-releases depName=some-org/some-hook
    rev: v1.2.3
```
