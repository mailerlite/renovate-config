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

---

## What each config does

### base

Applied automatically to all configs that extend `app` or `flux`. Sets organisation-wide defaults:

- Semantic commit messages
- Dependency dashboard with auto-close
- Digest pinning for all Docker images and GitHub Actions
- 1-day minimum release age before a PR is opened
- Security vulnerability PRs bypass the age gate and are opened immediately
- Major updates require manual approval via the dependency dashboard
- Digest-only pin updates are grouped into a single PR (no version change)

### app

For application and microservice repositories. Enables:

- `dockerfile` — external images in `Dockerfile*` and `Makefile`
- `docker-compose` — images in compose files
- `helmv3` — Helm chart dependencies
- `github-actions` — actions in `.github/workflows/`
- `regex` — custom annotations (see below)

### flux

For Kubernetes Flux cluster repositories. Enables:

- `flux` — HelmRelease, GitRepository, etc. in cluster YAML files
- `kubernetes` — raw Kubernetes manifests
- `github-actions` — actions in `.github/workflows/`
- `regex` — custom annotations (see below)

Updates run weekly on Monday at 5 AM UTC, except Groot which automerges immediately at any time.

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

Internal images from `europe-docker.pkg.dev/mailerlitehub/` are tracked automatically — no annotation needed.

```dockerfile
FROM europe-docker.pkg.dev/mailerlitehub/octopus/octopus:1.2.3@sha256:abc123...

COPY --from=europe-docker.pkg.dev/mailerlitehub/swiss-army-knife/swiss-army-knife:1.0.0 /bin/tool /bin/tool
```

Both `FROM` and `COPY --from` are supported, with or without a digest.

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

HelmReleases using [bjw-s/app-template](https://github.com/bjw-s-labs/app-template) are detected automatically via the native `helm-values` manager — no annotation needed.

```yaml
image:
  repository: europe-docker.pkg.dev/mailerlitehub/screenshoter/screenshoter
  tag: v3.2.3
  digest: sha256:abc123...  # optional - if present, Renovate keeps it updated here
```

Works for internal images, Docker Hub, and ghcr.io. If a `digest:` field is present, Renovate writes the digest there. If absent, it appends inline as `tag: v3.2.3@sha256:...`.

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
