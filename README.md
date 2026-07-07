# renovate-config

Shared [Renovate](https://docs.renovatebot.com/) configurations for MailerLite repositories.

## Configs

| Config | Purpose |
|--------|---------|
| `base` | Common defaults inherited by all configs |
| `app` | Application/microservice repositories |
| `flux` | Kubernetes Flux cluster repositories |

---

## Usage

Add a `.github/renovate.json` to your repository and extend the appropriate config:

**Application repo:**
```json
{
  "extends": ["github>mailerlite/renovate-config:app"]
}
```

**Flux cluster repo:**
```json
{
  "extends": ["github>mailerlite/renovate-config:flux"]
}
```

**Other repo (not an app or Flux cluster):**
```json
{
  "extends": ["github>mailerlite/renovate-config:base"]
}
```

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
- Patch and minor updates auto-merge where applicable (see per-config sections below)
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

Repos extending `base` directly get these managers. Repos extending `app` or `flux` override this list with their own.

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

Internal GHA patch and minor updates auto-merge. All other updates create a PR.

### flux

For Kubernetes Flux cluster repositories. Managers:

| Manager | What it tracks |
|---------|---------------|
| `flux` | HelmRelease, HelmRepository, GitRepository, etc. |
| `kubernetes` | Raw Kubernetes manifests |
| `github-actions` | `uses:` in `.github/workflows/` |
| `custom.regex` | Annotated versions in cluster YAML |
| `custom.jsonata` | app-template `image` fields (repository/tag/digest) |

**Environment awareness:** PRs are labelled and branch-prefixed by environment based on file path (`clusters/dev/`, `clusters/staging/`, `clusters/prod/`, `clusters/production/`). Minor and patch updates are split into separate PRs by default, except for internal Docker images in dev/staging where both auto-merge so the split adds no value.

**Digest pinning:** Internal Docker images in Flux cluster files **are** digest-pinned (unlike in app repos). This ensures cluster deployments are fully deterministic. The `flux` manager itself has digest pinning disabled because the inline `tag@digest` format is not valid in HelmRelease values — digest tracking is handled by the custom managers instead.

**Auto-merge:** PRs that auto-merge receive an `automerge` label.

| Update type | dev / staging | prod |
|-------------|--------------|------|
| Internal Docker patch | Auto-merge (CI required) | Auto-merge (CI required) |
| Internal Docker minor | Auto-merge (CI required) | Auto-merge, 1 day buffer (CI required) |
| Internal Docker pinDigest | Auto-merge (CI required) | Auto-merge (CI required) |
| Internal Docker major | PR only | PR only |
| External | PR only | PR only |

The 1-day buffer on prod minor means by the time the PR is created, the change has already been running in dev/staging for at least a day via webhook — it acts as a natural soak gate.

Internal GHA patch and minor updates auto-merge across all repos (including Flux).

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

Internal images from all GCP artifact registries are tracked automatically via the standard Dockerfile manager — no annotation needed.

```dockerfile
FROM europe-docker.pkg.dev/mailerlitehub/octopus/octopus:1.2.3

COPY --from=europe-docker.pkg.dev/mailerlite-gcp/myimage/myimage:1.0.0 /bin/tool /bin/tool
```

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
