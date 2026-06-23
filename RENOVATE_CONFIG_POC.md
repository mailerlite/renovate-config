# Renovate Config Redesign — POC

## Motivation

The current config was built incrementally and lacks a coherent policy for **internal vs external packages**. Auto-merge is absent entirely, meaning every update — including internal patch bumps we fully own and CI already validates — requires manual PR approval. This creates unnecessary toil and delays.

This redesign introduces a tiered policy: internal packages get fast-tracked with auto-merge in safe environments; external packages stay conservative with age gates and always require a PR.

---

## Scope

Renovate will manage **three package types**:

| Manager | Examples |
|---|---|
| Docker images | `europe-docker.pkg.dev/mailerlitehub/*`, external base images |
| GitHub Actions | `mailerlite/*`, `actions/checkout`, `docker/build-push-action` |
| Helm charts | `app-template`, OCI charts, HelmReleases |

**Explicitly excluded:** language package managers (Go modules, Composer, pip, npm, etc.). These are out of scope.

---

## Package Classification

A package is **internal** if it matches any of these patterns:

| Type | GCP project / org |
|---|---|
| Docker | `mailerlitehub` |
| Docker | `mailerlite-gcp` |
| Docker | `mailercheck-usa` |
| Docker | `mailersend-214215` |
| GitHub Actions / Releases | `mailerlite/*` |
| GitHub Actions / Releases | `mailersend/*` |
| GitHub Actions / Releases | `mailercheck/*` |

All of these are hosted under `*.pkg.dev` (Docker) or the MailerLite GitHub org (Actions). Everything else is **external**.

---

## Update Policy

### Internal packages

| Update type | dev + staging | prod |
|---|---|---|
| `patch` | Auto-merge (CI required) | Auto-merge (CI required, `minimumReleaseAge: 1 day`) |
| `minor` | Auto-merge (CI required) | PR only |
| `major` | PR only | PR only |
| `pinDigest` (digest-only rebuild, no version change) | Auto-merge (CI required) | PR only |

- `minimumReleaseAge` for prod internal patches is set to `1 day`. Since dev/staging is webhook-triggered immediately on release and auto-merges, by the time prod's PR is created the change has already been running in dev/staging for ~1 day — it acts as a natural soak gate.
- dev + staging have no `minimumReleaseAge` — they receive updates immediately via webhook.
- All internal packages are expected to follow strict semver. The auto-merge policy depends on this.

### External packages

| Update type | All environments |
|---|---|
| `patch` | PR only |
| `minor` | PR only |
| `major` | PR only |
| `pinDigest` | Grouped PR (see below) |

- `minimumReleaseAge: 3 days` — packages must be available for 3 days before a PR is created. This is a supply chain safety buffer.
- No auto-merge, ever.

### Vulnerability alerts (all packages)

- `minimumReleaseAge: 0 days` — security fixes are surfaced immediately, bypassing the 3-day gate.
- `labels: ["security"]`
- `prPriority: 99` — these PRs are created before any other update.
- Powered by OSV (`osvVulnerabilityAlerts: true`).

---

## Auto-merge: Flux repos only

Auto-merge applies **only to Flux GitOps repos** (i.e., repos using `flux.json`). Application repos (`app.json`) do not auto-merge anything — they always produce PRs.

In Flux repos, the environment is detected by file path:

| File path contains | Environment | Role |
|---|---|---|
| `clusters/dev/` | dev | Non-production (used by some repos) |
| `clusters/staging/` | staging | Non-production (used by other repos) |
| `clusters/prod/` | prod | Production |

**Dev and staging are mutually exclusive** — a given Flux repo uses one or the other, never both. They serve the same purpose (non-production validation) and are treated identically by this policy.

Auto-merge fires when **all** of the following are true:
1. Package is internal
2. Update type is `patch`, `minor`, or `pinDigest` (dev/staging) — or `patch` only (prod, after Phase 2)
3. File is in the correct environment path
4. All required CI checks pass

### Rollout phases

**Phase 1 — label only (initial deploy)**
Auto-merge is **disabled**. Instead, every PR that *would* be auto-merged in Phase 2 receives an `automerge_test` label. The team can monitor these PRs to validate the policy is selecting the right updates before committing to merging them automatically.

**Phase 2 — auto-merge enabled**
Once the team is satisfied with the Phase 1 observations, auto-merge is switched on by changing the package rules from `addLabels: ["automerge_test"]` to `automerge: true`. The `automerge_test` label is replaced by no label (or kept for visibility — TBD).

| Phase | dev + staging patch/minor/pinDigest | prod patch |
|---|---|---|
| Phase 1 | `automerge_test` label, no merge | `automerge_test` label, no merge |
| Phase 2 | Auto-merge (CI required) | Auto-merge (CI required, 1-day age gate) |

---

## Digest Pinning Strategy

Digest pinning makes image references immutable — `nginx:1.25.3` becomes `nginx:1.25.3@sha256:abc...`. This protects against tag mutation (a tag being silently overwritten).

| Package type | Strategy | Reason |
|---|---|---|
| External Docker | Tag + digest | Tags are mutable on external registries; digest ensures reproducibility |
| Internal Docker | Tag + digest | Kept for deterministic deployments; digest-only rebuilds auto-merge in dev/staging |
| External GH Actions | Commit SHA (full pin) | Supply chain hardening — prevents a force-pushed tag from running malicious code |
| Internal GH Actions | Commit SHA (full pin) | Consistent hardening across all actions; Renovate maintains these automatically so operational cost is low |

### On `pinDigest` updates

A `pinDigest` update is **not a version change** — it means the same tag was rebuilt (e.g. OS security patch baked into a base layer). These have no semver classification. For external images they are grouped into a single PR. For internal images in Flux dev/staging they auto-merge.

---

## Schedule & Triggering

Schedule is **not configured in the Renovate config** — it is managed externally:

| Trigger | Mechanism |
|---|---|
| Scheduled run | Cron job triggers the Renovate runner (GH Actions workflow or renovate-operator) |
| Internal package webhook | New internal release triggers Renovate immediately via `workflow_dispatch` or renovate-operator native webhook |

Keeping schedule out of the Renovate config keeps the two concerns separate: *what* Renovate does is in the config, *when* it runs is in the infrastructure.

---

## Config File Structure

The existing three-file structure is preserved.

```
base.json      ← shared defaults (all repos)
app.json       ← extends base, for microservice repos
flux.json      ← extends base, for Flux GitOps repos (adds env-aware auto-merge)
```

### base.json — key changes from current

| Setting | Current | New |
|---|---|---|
| `minimumReleaseAge` | `"3 days"` (global) | `"3 days"` (external only, via packageRule) |
| Major update gate | `dependencyDashboardApproval: true` | Removed — creates PR directly |
| Schedule | hourly (in GH Actions workflow) | Removed from Renovate config — managed externally by cron/webhook |
| Internal image release age | `null` (via packageRule) | `"0 days"` (explicit, cleaner) |

Preserved as-is: `semanticCommits`, `dependencyDashboard`, `dependencyDashboardAutoclose`, `rebaseWhen`, `labels`, `prConcurrentLimit: 0`, `prHourlyLimit: 0`, `osvVulnerabilityAlerts`, `vulnerabilityAlerts`.

### app.json — key changes from current

- No auto-merge added (Flux only).
- All GH Actions (internal and external) are now SHA-pinned. The previous `pinDigests: false` exception for `mailerlite/base-docker-images` is removed.

### flux.json — key changes from current

- New `packageRule` for Phase 1: matches internal packages + dev/staging/prod file paths + relevant update types → adds `automerge_test` label. No actual auto-merge yet.
- Phase 2 will replace the label rule with `automerge: true` for dev/staging and prod patch.
- `dependencyDashboardApproval` for majors removed (same as base).

---

## Preserved Exceptions

These carry over unchanged:

| Exception | Reason |
|---|---|
| `mailerlite/workflows` disabled | Reusable workflow repo with no versioned releases — Renovate cannot meaningfully track it |
| `elasticsearch` major updates disabled | Intentional hold on major Elasticsearch upgrades |
| External digest-only pins grouped into one PR | Reduces noise from frequent base image rebuilds |

---

## What This Replaces / Removes

| Removed behaviour | Why |
|---|---|
| `dependencyDashboardApproval` for all major updates | Too much friction for internal majors; a PR is sufficient |
| Schedule in Renovate config | Schedule is now an infrastructure concern (cron/webhook), not a config concern |
| Global `minimumReleaseAge: 3 days` | Now only applies to external packages |

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Internal team ships a breaking change as a patch | Strict semver policy must be enforced across all internal package teams before enabling auto-merge |
| CI is absent or flaky on a repo | Auto-merge requires `automergeType: "pr"` + CI checks — if no checks are configured, Renovate will not auto-merge |
| Digest-only auto-merges silently deploy a bad rebuild | Same CI gate applies; a bad rebuild will fail tests and block the merge |
| SHA pinning breaks external GH Actions readability | SHA + comment in the PR description shows the resolved tag — Renovate includes this automatically |

---

## Open Questions for the Team

1. **Semver discipline** — which internal package teams need to be notified before this goes live? Auto-merge is only safe once all internal packages follow strict semver.
2. **CI coverage** — are there any Flux repos where CI does not run on Renovate PRs? Those repos would need CI configured before auto-merge is safe.
3. **Renovate-operator vs workflow_dispatch** — has a decision been made on the webhook approach? This affects how the schedule trigger is configured.
