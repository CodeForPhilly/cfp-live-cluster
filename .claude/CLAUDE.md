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
| --- | --- |
| `_infra/` | Cluster-level infra (cert-manager issuers, envoy-gateway config) |
| `_gateways/` | Per-app Gateway + HTTPRoute pairs, one file per app |
| `_admins/`, `_docs/` | Admin RBAC + docs source (workload dirs stay bare) |
| `<app>/` | App workspace — `release-values.yaml` for helm, `kustomization.yaml` for kustomize |
| `<app>.secrets/` | SealedSecrets for that namespace |
| `echo-http.yaml` | Raw Namespace+Deployment+Service for the echo-http probe (routing via `_gateways/echo-http.yaml`) |
| `.holo/sources/` | Holosource pins (URL + ref) |
| `.holo/branches/<branch>/` | Holomappings — source content → workspace path |
| `.holo/lenses/` | Lens configs |

`_` prefix means "not a workload namespace." Workspace convention; projected tree drops it. `_infra/` and `_gateways/` were added in the in-flight migration; `_admins/` and `_docs/` followed the convention in a separate refactor.

Workload roots currently in tree: `balancer/` (kustomize), `chime/`, `choose-native-plants/`, `code-for-philly/`, `grafana/`, `sealed-secrets/`, `third-places/`, `vaultwarden/` (all helm).

## Standing patterns

Target post-Envoy-migration state. Mimic these for new work.

### Per-app routing

Each public-facing app gets `_gateways/<app>.yaml` containing:

- `Gateway` in the app's namespace with one HTTPS listener per hostname, `cert-manager.io/cluster-issuer: letsencrypt-prod-gateway` annotation, certificateRef to a `<descriptive>-gw-tls` Secret per listener
- `HTTPRoute` with `parentRefs` attached **only** to the per-app Gateway (no `main-gateway`)

HTTP (port 80) is handled globally by `_infra/envoy-gateway/http-redirect.yaml` — a single `HTTPRoute` on `main-gateway` that 301s everything to HTTPS. ACME challenge paths bypass it via Gateway API conflict resolution (cert-manager creates an `Exact`-path HTTPRoute per challenge).

### Cert Secret naming

`<app>-gw-tls` (or `<descriptive>-gw-tls` for multi-hostname apps where each listener gets its own cert) — distinct from legacy `<app>-tls` so the two paths coexist during migration.

### ClusterIssuers

Only one pair exists: `letsencrypt-{prod,staging}-gateway` (gatewayHTTPRoute solver) in `_infra/cert-manager/issuers-gateway.yaml`. Annotate Gateways with `letsencrypt-prod-gateway`.

The nginx-solver pair (`letsencrypt-{prod,staging}`, formerly `cert-manager.issuers.yaml` at the repo root) was **deleted** in phase 6 of #144, along with ingress-nginx itself. Don't recreate them. They were a footgun long before they were removed: with ingress-nginx unreachable in-cluster (proxy-protocol — see Cluster context), they could not issue a certificate, yet they still looked perfectly usable. During the July 2026 incident an attempt to "fix" stuck certs by repointing Gateways at `letsencrypt-prod` cost hours — every challenge sat unschedulable, because cert-manager's self-check runs *in-cluster*, where nginx could never answer.

The only way to fix a stuck cert is to make the hostname reachable on the path its solver actually uses.

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

- **ingress-nginx is GONE** (removed 2026-07-14, #144 phase 5.5) — namespace, IngressClass, admission webhook, and its NodeBalancer at `104.237.148.175`. There are zero `Ingress` objects in the cluster. All routing is Envoy Gateway. Don't reintroduce either.
- **Why it had to go, and the lesson worth keeping.** ingress-nginx ran with `use-proxy-protocol: true` plus the Linode `linode-loadbalancer-proxy-protocol: v2` annotation, so it required a PROXY header on every connection — and only the Linode NodeBalancer added one. In-cluster traffic (to the LB external IP *or* the ClusterIP) is short-circuited by kube-proxy straight to the pods, never traverses the NodeBalancer, and arrives with no header. **hairpin-proxy** was the in-cluster haproxy that re-added it. Phase 1 removed hairpin-proxy on the belief that Linode LKE now does LoadBalancer hairpin natively — true, and **irrelevant**: the packets reach the backend fine, they just arrive without the header. Since cert-manager must pass an **in-cluster** HTTP-01 self-check before it will even ask Let's Encrypt to validate, that silently killed every nginx cert renewal on 2026-05-18. Nine certs expired together on 2026-07-12 before anyone noticed. The generalisable rule: **a component in the request path that requires a header only the load balancer supplies is unreachable from inside the cluster — and cert issuance depends on in-cluster reachability.**
- **Never enable proxy protocol on Envoy.** Envoy Gateway supports it via `ClientTrafficPolicy.enableProxyProtocol`. Turning it on to recover real client IPs would recreate the exact trap above — cert-manager's self-checks would start failing against Envoy too. The cost of leaving it off is that apps see the NodeBalancer's IP rather than the client's in `x-forwarded-for`.
- **Wildcard DNS**: `*.live.k8s.phl.io` → the Envoy LB `45.79.246.168`. DNS is managed in OpenTofu at [CodeForPhilly/ops](https://github.com/CodeForPhilly/ops) → `tofu/dns`; a host with no specific record simply follows the wildcard, so new apps need no DNS change at all.
- **Apex domains in tree**: `balancerproject.org`, `choosenativeplants.com` (+ `www.`), `codeforphilly.org` (+ `www.`), `penn-chime.phl.io`, `vaultwarden.phl.io`, `bitwarden.phl.io`. Apex ACME challenges only work once DNS points at Envoy — plan cutover and cert issuance together for these. `choosenativeplants.com` is at **Namecheap**, not Cloud DNS, so it can't be moved from the ops repo.
- **A new hostname is briefly down between DNS and cert.** The cert can't issue until the hostname resolves to Envoy (Let's Encrypt has to reach the solver), and Envoy's HTTPS listener doesn't program until the cert Secret exists — meanwhile HTTP 301s into a listener that isn't there. Roughly 60–90s. Keep TTLs at 60s.
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
- **cert-manager Challenges outlive their Order and deadlock renewals.** A Challenge carries owner references to *both* its Order and the ClusterIssuer. Kubernetes GC only reaps a dependent once **all** owners are gone, so when the Order is deleted the Challenge survives as a half-orphan — `processing: true`, no controller left to drive it, no GC path to remove it. It then holds its hostname's slot forever, because **cert-manager's scheduler won't run two HTTP-01 challenges for the same dnsName concurrently**. Every renewal for that hostname is silently starved: the new Challenge is created but never scheduled, and sits with a completely empty `.status` while cert-manager logs nothing about it at all.

  Symptom to watch for: `kubectl get challenges -A` shows a challenge `pending` for many days, and a second challenge for the same `dnsName` with a blank STATE. Fix is to delete the stale one by name. This is what turned a broken solver into nine simultaneously-expired certs.

For repo-local issues, check the open issue list directly — anything I'd list here will rot.

## References

- Migration umbrella: [#144](https://github.com/CodeForPhilly/cfp-live-cluster/issues/144)
- Sandbox-equivalent (already complete, source for patterns): [cfp-sandbox-cluster#130](https://github.com/CodeForPhilly/cfp-sandbox-cluster/issues/130)
- Sandbox repo (for prior-art on patterns): <https://github.com/CodeForPhilly/cfp-sandbox-cluster>
- Upstream cluster-template: <https://github.com/JarvusInnovations/cluster-template>
- civic-cloud cluster-template: <https://github.com/civic-cloud/cluster-template>
- Hologit: <https://github.com/JarvusInnovations/hologit>
