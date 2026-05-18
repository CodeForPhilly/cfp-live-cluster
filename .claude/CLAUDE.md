# cfp-live-cluster — agent instructions

GitOps repo for the CodeForPhilly **live** (production) Kubernetes cluster on Linode LKE. Source-projected via hologit. Don't edit deployed branches directly — change the workspace, project, push the result.

The Envoy Gateway migration tracked in [#144](https://github.com/CodeForPhilly/cfp-live-cluster/issues/144) is **in flight** — sandbox completed its equivalent migration in May 2026 and this repo is following its lead. Several patterns documented below describe the target post-migration state; current `main` may still be on the ingress-nginx path until that work lands.

## Source pipeline

```
JarvusInnovations/cluster-template
  └─ civic-cloud/cluster-template
      └─ this repo (cfp-live-cluster)
          └─ projected branches → live cluster
```

This repo pulls civic-cloud via `.holo/sources/civic-cloud.toml`.

To refresh a holosource: `git holo source fetch <name>`. **Never** `git fetch <upstream-url> <refspec>` — that auto-pulls upstream tags into local `refs/tags/` and pollutes the tag namespace.

## How projection works

- **Workspace files** are what humans edit. `.holo/branches/<branch>/` configs map workspace paths to source content.
- `git holo project <branch>` runs the pipeline and prints a tree SHA on the last stdout line. Inspect with `git ls-tree -r <SHA>` or `git diff <ref> <SHA>`.
- Two branches matter:
  - `k8s-manifests` — manifests only
  - `k8s-manifests-github` — manifests + GitHub Actions workflows (overlays on top of `k8s-manifests`)
- Deploy lifecycle: push to `main` → `Build k8s-manifests` workflow → `releases/k8s-manifests` → Deploy PR auto-opens → merge → `deploys/k8s-manifests` → `K8s: Deploy k8s-manifests` workflow → `kubectl apply` to cluster.
- The deploy workflow's "Apply manifests: deleted resources" step removes anything that disappears from the projection. Drop a file from the workspace → resource deleted on next deploy.

### Lenses

`.holo/lenses/<name>.toml` describes per-source transformations:

- **helm3** — renders a chart against the app's `release-values.yaml`
- **kustomize** — builds a kustomization
- **k8s-normalize** — routes flat manifests into the `<namespace>/<Kind>/<name>.yaml` layout

Cluster-scoped resources land in `_/<Kind>/<name>.yaml`.

## Directory map

| Path | Purpose |
|---|---|
| `_infra/` | Cluster-level infra (cert-manager issuers, envoy-gateway config) |
| `_gateways/` | Per-app Gateway + HTTPRoute pairs, one file per app |
| `_admins/`, `_docs/` | Admin RBAC + docs source (workload dirs stay bare) |
| `<app>/` | App workspace — `release-values.yaml` for helm, `kustomization.yaml` for kustomize |
| `<app>.secrets/` | SealedSecrets for that namespace |
| `cert-manager.issuers.yaml` | Legacy nginx-solver ClusterIssuers at repo root — kept parallel to `_infra/cert-manager/issuers-gateway.yaml` until per-app cutover finishes (see [#144](https://github.com/CodeForPhilly/cfp-live-cluster/issues/144)) |
| `echo-http.yaml` | Raw Namespace+Deployment+Service+Ingress for the echo-http probe |
| `.holo/sources/` | Holosource pins (URL + ref) |
| `.holo/branches/<branch>/` | Holomappings — source content → workspace path |
| `.holo/lenses/` | Lens configs |

`_` prefix means "not a workload namespace." Workspace convention; projected tree drops it. `_infra/` and `_gateways/` were added in the in-flight migration; `_admins/` and `_docs/` followed the convention in a separate refactor.

Workload roots currently in tree: `balancer/` (kustomize), `browserless-chrome/` (raw yaml), `chime/`, `choose-native-plants/`, `code-for-philly/`, `grafana/`, `sealed-secrets/`, `third-places/`, `vaultwarden/` (all helm).

## Standing patterns

Target post-Envoy-migration state. Mimic these for new work.

### Per-app routing

Each public-facing app gets `_gateways/<app>.yaml` containing:

- `Gateway` in the app's namespace with one HTTPS listener per hostname, `cert-manager.io/cluster-issuer: letsencrypt-prod-gateway` annotation, certificateRef to a `<descriptive>-gw-tls` Secret per listener
- `HTTPRoute` with `parentRefs` attached **only** to the per-app Gateway (no `main-gateway`)

HTTP (port 80) is handled globally by `_infra/envoy-gateway/http-redirect.yaml` — a single `HTTPRoute` on `main-gateway` that 301s everything to HTTPS. ACME challenge paths bypass it via Gateway API conflict resolution (cert-manager creates an `Exact`-path HTTPRoute per challenge).

### Cert Secret naming

`<app>-gw-tls` (or `<descriptive>-gw-tls` for multi-hostname apps where each listener gets its own cert) — distinct from legacy `<app>-tls` so the two paths coexist during migration.

### ClusterIssuers — parallel during migration

Two pairs exist side by side until the migration finishes:

- `letsencrypt-{prod,staging}` (nginx-solver) in `cert-manager.issuers.yaml` at the repo root — keeps existing Ingress-managed Certs renewing
- `letsencrypt-{prod,staging}-gateway` (gatewayHTTPRoute solver) in `_infra/cert-manager/issuers-gateway.yaml` — for new Gateway-issued Certs

New Gateway resources should annotate `letsencrypt-prod-gateway`. The nginx-solver pair gets removed in phase 6 of #144, after all Ingresses are gone.

### Envoy Gateway

- `EnvoyProxy` resource has `mergeGateways: true` — every Gateway shares one Envoy data plane and one LoadBalancer. **Do not disable.**
- `GatewayClass` is named `eg`
- The shared HTTP `main-gateway` lives in `envoy-gateway-system`; per-app Gateways attach implicitly via the merged data plane.

## Before pushing a PR — required QA

Run a local projection and diff it against the deployed tree. **No PR ships without this.**

```bash
# 1. Commit everything first
git status   # must be clean

# 2. Fetch and project against the deploy branch's layout
git fetch origin
SHA=$(git holo project k8s-manifests-github 2>&1 | tail -1)

# 3. Diff
git diff --name-status origin/deploys/k8s-manifests "$SHA"
git diff --stat origin/deploys/k8s-manifests "$SHA"

# 4. Spot-check content for changed files
git show "$SHA":<path>
```

If using `git holo project --working` to test uncommitted changes, project `k8s-manifests` (not `-github`) and expect the deploy workflow files to show as deletions — they live in `k8s-manifests-github`, not `k8s-manifests`. Harmless noise. Committing first is usually simpler.

The diff is the definitive preview. Read it carefully — admission webhooks add defaults that show up here (HTTPRoutes get default `PathPrefix: /` matches, etc.) and side effects of changed helm values can surface as unrelated-looking ConfigMap or Deployment edits (e.g. `ingress.enabled: false` clearing `server.domain` on grafana, `MB_SITE_URL` on metabase if it ever lands here).

## Common operations

### Add a new app

1. Create workspace dir + resources (chart values or kustomize)
2. Add holomapping at `.holo/branches/k8s-manifests/<app>/`
3. Add lens config at `.holo/lenses/<app>.toml` if applicable
4. Add `_gateways/<app>.yaml` if it needs external HTTPS (post-migration); pre-migration it still uses an Ingress on the chart
5. Run the projection + diff (above)

### Bump an upstream chart version

Edit the version pin in `.holo/sources/<name>.toml` → `git holo source fetch <name>` → project → diff.

For chart versions owned by the upstream chain: bump in cluster-template or civic-cloud → wait for release → bump civic-cloud pin here.

### Disable an Ingress on a helm-managed app

`<app>/release-values.yaml`: `ingress.enabled: false`. If the chart inferred its public hostname from `ingress.hosts[0]` (historical examples: grafana → `grafana.ini.server.domain`), set it directly via the chart's other values. Verify via render diff. This is phase 5 of #144 — only do it after the app's DNS has cut over to Envoy.

## Cluster context

Things not in any single grep-able file:

- **Wildcard DNS**: `*.live.k8s.phl.io` resolves to the cluster's ingress LB. During the Envoy migration this stays pointed at ingress-nginx; per-hostname A records override the wildcard as each app cuts over to Envoy.
- **Apex domains in tree**: `balancerproject.org`, `choosenativeplants.com` (+ `www.`), `codeforphilly.org` (+ `www.`), `penn-chime.phl.io`, `vaultwarden.phl.io`, `bitwarden.phl.io`. Apex ACME challenges only work once DNS points at Envoy — plan cutover and cert issuance together for these.
- **Linode LKE LoadBalancer hairpin**: now native (in-cluster pods can reach the cluster's LB external IP). hairpin-proxy was the historical workaround and is scheduled for removal as part of #144 (the civic-cloud v1.9.2 bump drops it from the projection).
- **ingress-nginx + hairpin-proxy still present** on this cluster as of writing — both go away in #144 (phases 1 and 5.5 respectively). Don't reintroduce after they're gone.
- **No cnpg / shared-cluster** on this cluster yet. If a database is needed, it ships per-app (e.g. vaultwarden runs its own PostgreSQL StatefulSet via the gissilabs chart; chime + third-places similar).

## Guardrails

Take these only with explicit user authorization:

- `kubectl apply/delete/patch` against shared-infra namespaces: `kube-system`, `cert-manager`, `ingress-nginx` (until decommissioned), `envoy-gateway-system` (once present), `sealed-secrets`
- Force-pushes to `releases/k8s-manifests` or `deploys/k8s-manifests`
- Merging upstream release PRs (cluster-template, civic-cloud) — user handles these
- Restarting deployments in shared namespaces
- Bumping `.holo/sources/civic-cloud.toml` (carries cert-manager + Gateway API + Envoy Gateway + hairpin-proxy removal — phase-1 of #144 lives in that bump)

Editing workspace files in this repo and opening PRs are fine without per-action approval.

## Known external issues

- **hologit shallow-clone race** ([JarvusInnovations/hologit#450](https://github.com/JarvusInnovations/hologit/issues/450)) — `Build k8s-manifests` intermittently fails with `fatal: shallow file has changed since we read it`. Rerun the workflow.

For repo-local issues, check the open issue list directly — anything I'd list here will rot.

## References

- Migration umbrella: [#144](https://github.com/CodeForPhilly/cfp-live-cluster/issues/144)
- Sandbox-equivalent (already complete, source for patterns): [cfp-sandbox-cluster#130](https://github.com/CodeForPhilly/cfp-sandbox-cluster/issues/130)
- Sandbox repo (for prior-art on patterns): <https://github.com/CodeForPhilly/cfp-sandbox-cluster>
- Upstream cluster-template: <https://github.com/JarvusInnovations/cluster-template>
- civic-cloud cluster-template: <https://github.com/civic-cloud/cluster-template>
- Hologit: <https://github.com/JarvusInnovations/hologit>
