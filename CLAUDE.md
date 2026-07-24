# CLAUDE.md — repo-config

This file provides guidance to Claude Code when working inside `repo-config/`.
The parent `../CLAUDE.md` covers the full platform (app, infra, CI/CD overview).

## What this repo is

The GitOps **source of truth**. No application code, no Terraform — only a Helm chart and ArgoCD `Application` manifests. ArgoCD (running inside the GKE cluster) polls this repo continuously and reconciles cluster state to match it; nothing here is applied by hand via `kubectl`/`helm` in normal operation. There is no `.github/workflows/` in this repo — a commit here triggers nothing on GitHub, only ArgoCD's poll loop reacts to it.

Three environments exist today: `dev`, `staging`, `prod` (each with its own `values-<env>.yaml` and its own ArgoCD child Application) — see root `../CLAUDE.md` for the build-once/promote-to-prod pattern that drives them. There is still no `_helpers.tpl`, no `NOTES.txt`.

## Repository Structure

```
repo-config/
├── apps/
│   ├── root-app.yaml           # App-of-Apps root; applied once by hand (bootstrap-argocd job), watches apps/children/
│   └── children/
│       ├── dev.yaml            # ArgoCD Application "hr-dev" → charts/hr-app + values-dev.yaml → ns dev
│       ├── staging.yaml        # ArgoCD Application "hr-staging" → charts/hr-app + values-staging.yaml → ns staging
│       ├── prod.yaml           # ArgoCD Application "hr-prod" → charts/hr-app + values-prod.yaml → ns prod (no automated syncPolicy)
│       ├── cert-manager.yaml           # sync-wave -2 — TLS prereq for the CNPG barman-cloud backup plugin
│       ├── cnpg-operator.yaml          # sync-wave -1 — CloudNativePG operator (chart `cloudnative-pg` 0.29.0), ns cnpg-system
│       ├── cnpg-plugin-barman-cloud.yaml  # sync-wave -1 — barman-cloud backup plugin for CNPG
│       ├── cnpg-cluster-staging.yaml   # sync-wave 0 — CNPG `Cluster` CR `pg-staging` (Postgres 16, db/owner `hrapp`), ns staging, GCS backups via Workload Identity
│       └── cnpg-network-policy.yaml    # NetworkPolicy allowing port 5432 into pg-staging from staging + cnpg-system
├── manifests/cnpg-network-policy/networkpolicy-postgres.yaml  # the raw manifest cnpg-network-policy.yaml's Application points at
├── charts/hr-app/
│   ├── Chart.yaml               # no `dependencies:` — CNPG is deployed as independent ArgoCD Applications above, not a Helm sub-chart dependency of hr-app
│   ├── values.yaml               # chart defaults — deliberately non-deployable (empty registry, tag "UNSET")
│   ├── values-dev.yaml           # dev overrides — registry-staging-pfe, backend.autoscaling.enabled: false
│   ├── values-staging.yaml       # staging overrides — image tags patched automatically by repo-app's CI (yq)
│   ├── values-prod.yaml          # prod overrides — isolated registry-prod-pfe, replicas 2, autoscaling 2-5; tags patched by promote-prod.yml, not repo-app's push CI
│   └── templates/
│       ├── deployment-backend.yaml / deployment-frontend.yaml  # both set a soft podAntiAffinity (topologyKey: kubernetes.io/hostname)
│       ├── hpa-backend.yaml            # conditional: backend.autoscaling.enabled
│       ├── service-backend.yaml / service-frontend.yaml
│       ├── ingress.yaml                # conditional: ingress.enabled (false today)
│       └── networkpolicy-{default-deny,backend,frontend}.yaml  # conditional: networkPolicy.enabled
└── docs/                         # deep-dive guides (French) — see index below
```

Postgres exists again in `staging`, but as a self-managed CloudNativePG `Cluster` CR (via the `apps/children/cnpg-*.yaml` Applications above), not a `StatefulSet` and not a Helm sub-chart dependency of `hr-app`. `charts/hr-app/templates/deployment-backend.yaml` conditionally (`backend.database.enabled`) injects `SPRING_DATASOURCE_URL`/`USERNAME`/`PASSWORD` from the CNPG-generated secret `pg-staging-app`; only `values-staging.yaml` sets `database.enabled: true` today (`dev`/`prod` stay on the `false` default, and there's no dev/prod CNPG cluster either). `repo-app`'s backend now has the JPA/JDBC dependency reconnected (see `repo-app/CLAUDE.md`) but still no business-domain entities beyond a connectivity-check one.

## Common Commands

```bash
cd repo-config

# Lint the chart
helm lint charts/hr-app

# Render the full manifest set as ArgoCD would (no cluster contact)
helm template hr-staging charts/hr-app \
  -f charts/hr-app/values.yaml \
  -f charts/hr-app/values-staging.yaml

# Render just one template
helm template hr-staging charts/hr-app \
  -f charts/hr-app/values.yaml -f charts/hr-app/values-staging.yaml \
  --show-only templates/deployment-backend.yaml

# Local install/uninstall (test namespace, not staging)
kubectl create namespace staging
helm upgrade --install hr-staging charts/hr-app -f charts/hr-app/values-staging.yaml --namespace staging
helm uninstall hr-staging --namespace staging
```

If Helm isn't installed locally: `docker run --rm -v "$(pwd):/work" -w /work alpine/helm:3.14.0 template hr-staging charts/hr-app -f charts/hr-app/values.yaml -f charts/hr-app/values-staging.yaml`.

Same commands work for `dev`/`prod` — swap the release name and `-f charts/hr-app/values-<env>.yaml` (e.g. `helm template hr-dev charts/hr-app -f charts/hr-app/values.yaml -f charts/hr-app/values-dev.yaml`).

### Live-cluster verification
```bash
gcloud container clusters get-credentials gke-staging-pfe --region europe-west1-b --project pfe-2026-495220
kubectl get applications -n argocd
kubectl get application hr-staging -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'   # expect: Synced Healthy
kubectl get deployment hr-backend -n staging -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'         # tag must match latest "ci: update image tags to <SHA7>" commit
kubectl get pods -n staging
```
Same pattern for the other two: `kubectl get application hr-dev -n argocd ...` / namespace `dev`, `kubectl get application hr-prod -n argocd ...` / namespace `prod`. `hr-prod`'s `sync.status` legitimately sits `OutOfSync` between a promotion commit and the next manual `argocd app sync hr-prod` — that is not a bug.

## Architecture: values.yaml vs values-<env>.yaml

Helm merges `values.yaml` (defaults) with exactly one `values-<env>.yaml` overlay; each ArgoCD child Application declares `helm.valueFiles: [values-<env>.yaml]`, which does the same merge internally. None of the three overlays redefine `resources` — the requests/limits in `values.yaml` apply as-is in every environment.

- **`backend.image.tag` / `frontend.image.tag`**: in `values-dev.yaml`/`values-staging.yaml` these are 7-char SHAs patched automatically by `repo-app`'s CI via `yq` (push to `develop`/`main` respectively, `ci: update <env> image tags to <SHA7>` commits). In `values-prod.yaml` the tag is a semantic version (`vX.Y.Z`) patched by `promote-prod.yml` on a `v*.*.*` tag push — a `crane copy` of the already-built staging-SHA image into the isolated `registry-prod-pfe`, never a rebuild. Never edit any of the three by hand — the next CI run/promotion overwrites it.
- **`registry.repository`** differs by environment: `dev` and `staging` both point at `registry-staging-pfe`; `prod` points at `registry-prod-pfe`.
- **`networkPolicy.dnsClusterIP`** is the ClusterIP of the cluster's `kube-dns` Service (`kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.clusterIP}'`) — cluster-specific; currently the same value in all three overlays since `dev`/`staging`/`prod` are namespaces on one shared GKE cluster. Must be updated by hand in all three files if the cluster is ever recreated (`destroy-staging` then `apply`), otherwise DNS egress breaks in every namespace. See `docs/issues-rencontrees.md` Issue 4 for why this is an `ipBlock` and not a `podSelector`/`namespaceSelector`.
- **`backend.autoscaling.enabled`** activates `hpa-backend.yaml`, scaling `hr-backend` on CPU utilization against `backend.resources.requests.cpu`: `true` in staging (base 1 replica) and prod (`minReplicas: 2`/`maxReplicas: 5`), **`false`** in dev.
- Both `deployment-backend.yaml` and `deployment-frontend.yaml` set a soft `podAntiAffinity` (`preferredDuringSchedulingIgnoredDuringExecution`, weight 100, `topologyKey: kubernetes.io/hostname`) spreading same-app pods across nodes — best-effort, not a hard anti-affinity requirement, and identical across all three environments.

## App-of-Apps (ArgoCD)

`apps/root-app.yaml`'s `source.path` **must be `apps/children`**, not `apps` — pointing at `apps` would make ArgoCD watch the folder containing `root-app.yaml` itself (non-recursive), silently never create any child Application, while still reporting `root-app` Healthy. `apps/children/` holds three files today: `dev.yaml` (`hr-dev` → ns `dev`), `staging.yaml` (`hr-staging` → ns `staging`), `prod.yaml` (`hr-prod` → ns `prod`) — adding another environment is just committing a new `apps/children/<env>.yaml`.

`hr-dev` and `hr-staging` run `syncPolicy.automated: {prune: true, selfHeal: true}` — **any manual `kubectl edit`/`kubectl scale` on the `dev`/`staging` namespaces gets reverted** within a few minutes. **`hr-prod` has no `automated` block at all** — a commit to `values-prod.yaml` only leaves it `OutOfSync`; deployment requires a manual `argocd app sync hr-prod` (or the ArgoCD UI), and hand-edits to the `prod` namespace are *not* auto-reverted since there is no `selfHeal` watching it. All changes must still go through a commit here regardless of environment. Rollback is a `git revert` (never `git push --force` — `selfHeal` compares state, not history) followed by a push to `main`.

All three children carry identical `ignoreDifferences`, load-bearing, not decorative:
- Deployment `/spec/replicas` — needed because `hpa-backend.yaml` patches this field at runtime; without the exception `selfHeal`/a sync would revert HPA scaling back to `backend.replicas` every reconcile loop (this exception applies to *all* Deployments in the chart, including `hr-frontend`, which has no HPA).
- Deployment `/status/terminatingReplicas` — fluctuates naturally during rolling updates.

There is no `StatefulSet` `ignoreDifferences` entry (and no `StatefulSet` in the chart at all — Postgres runs as a CNPG `Cluster` CR outside `charts/hr-app`, see below).

## Known discrepancy

`README.md`'s "Limitations actuelles" section (no resource limits, no probes, Azure/AKS references in `GITOPS_PFE.md`) is **stale** — the chart has since grown `resources` (requests/limits), readiness/liveness probes on every workload, `hpa-backend.yaml`, and three `NetworkPolicy` templates. Trust `docs/*.md` and the templates themselves over `README.md`/`GITOPS_PFE.md` for current state. Ingress being disabled (`ingress.enabled: false`, reachable only via `kubectl port-forward`) is the one limitation still accurate.

`README.md` and `docs/guide-argocd-gitops.md` are *also* stale on environment count — both still describe `apps/children/` as holding a single `staging.yaml`/`hr-staging` Application. Three exist today (`hr-dev`, `hr-staging`, `hr-prod`); see App-of-Apps above.

A Postgres `StatefulSet` + HPA briefly existed in `charts/hr-app` (commits `fa12a0c`, `3c86ac4`) and was fully removed (`83792b4`) — don't go looking for a Postgres template or `Chart.yaml` dependency there, that stays gone. Postgres came back afterward (commits `2058586`, `6422edf`) as a separate CNPG `Cluster` CR managed by its own `apps/children/cnpg-*.yaml` Applications, wired to `hr-backend` only via the `backend.database.enabled` env vars in `deployment-backend.yaml` — architecturally unrelated to the removed `StatefulSet` approach.

## `docs/` index

Detailed, self-contained (French) guides — read the relevant one before making non-trivial changes:
- `docs/architecture-globale.md` — why three separate repos, what belongs to each, why a change in one never triggers another's pipeline
- `docs/lifecycle-pipeline.md` — full push-to-pod trace: exact CI jobs, files read/written, commands at each arrow
- `docs/guide-argocd-gitops.md` — App-of-Apps mechanics, `kubectl`/`argocd` observability commands, rollback procedure
- `docs/guide-helm-chart.md` — full template ↔ values reference table, local testing workflow
- `docs/guide-deploiement-infra.md` — Terraform provisioning (lives in `repo-infrastructure`, referenced from here for the ArgoCD bootstrap handoff)
- `docs/guide-securite.md` — every real security control and the concrete failure it prevents (WIF, Trivy, NetworkPolicy/Calico, Shielded Nodes, Binary Authorization, IAM scoping, Checkov) plus an explicit "not implemented yet" list
- `docs/decisions-architecture.md` — ADRs: Helm vs Terraform `kubernetes` provider, ArgoCD pull-based vs push-from-CI, why no Ansible
- `docs/issues-rencontrees.md` — real incidents with root cause and fix: ArgoCD CRD sync race, spot-node `kubectl wait` race, Trivy-blocked Tomcat CVEs, Calico/ClusterIP DNS NetworkPolicy failure
- `GITOPS_PFE.md` (repo root) — original note on the ArgoCD CRD race; superseded/expanded by `docs/issues-rencontrees.md` Issue 1

## Key Constraints

- Never hand-edit `backend.image.tag`/`frontend.image.tag` in `values-dev.yaml`/`values-staging.yaml` — `repo-app`'s CI overwrites them on every push to `develop`/`main`. Never hand-edit them in `values-prod.yaml` either — `promote-prod.yml` overwrites them on every `v*.*.*` tag promotion.
- Never hand-edit cluster resources in the `dev`/`staging` namespaces — `selfHeal` reverts it. The `prod` namespace has no `selfHeal` (`hr-prod` has no `automated` syncPolicy) — hand-edits there simply persist until the next manual `argocd app sync hr-prod`, which is a smaller footgun, not a protection.
- `apps/root-app.yaml` `source.path` must stay `apps/children`.
- No `resources` override in any of `values-dev.yaml`/`values-staging.yaml`/`values-prod.yaml` is intentional — all three inherit `values.yaml`'s requests/limits.
- Recreating the GKE cluster invalidates `networkPolicy.dnsClusterIP` — it must be updated by hand in all three `values-<env>.yaml` files before the next deploy or DNS egress breaks in every namespace.
