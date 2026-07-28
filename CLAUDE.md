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
│       ├── cnpg-cluster-dev.yaml       # sync-wave 0 — CNPG `Cluster` CR `pg-dev` (Postgres 16, db/owner `hrapp`, 1 instance), ns dev, GCS backups via Workload Identity
│       ├── cnpg-cluster-staging.yaml   # sync-wave 0 — CNPG `Cluster` CR `pg-staging` (Postgres 16, db/owner `hrapp`, 1 instance — scaled down from 3), ns staging, GCS backups via Workload Identity
│       ├── cnpg-cluster-prod.yaml      # sync-wave 0, no automated syncPolicy — CNPG `Cluster` CR `pg-prod` (Postgres 16, db/owner `hrapp`, 3 instances/HA), ns prod, GCS backups via Workload Identity
│       ├── cnpg-network-policy-dev.yaml    # sync-wave -1, automated — NetworkPolicy allowing port 5432 into pg-dev from dev + cnpg-system
│       ├── cnpg-network-policy.yaml    # sync-wave -1, automated — NetworkPolicy allowing port 5432 into pg-staging from staging + cnpg-system
│       └── cnpg-network-policy-prod.yaml   # sync-wave -1, automated (unlike cnpg-cluster-prod.yaml) — NetworkPolicy allowing port 5432 into pg-prod from prod + cnpg-system
├── manifests/
│   ├── cnpg-network-policy/networkpolicy-postgres.yaml       # raw manifest cnpg-network-policy.yaml (staging) points at
│   ├── cnpg-network-policy-dev/networkpolicy-postgres.yaml   # raw manifest cnpg-network-policy-dev.yaml points at
│   └── cnpg-network-policy-prod/networkpolicy-postgres.yaml  # raw manifest cnpg-network-policy-prod.yaml points at
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

Postgres now exists in all three environments, each as its own self-managed CloudNativePG `Cluster` CR (via the `apps/children/cnpg-cluster-<env>.yaml` Applications above), not a `StatefulSet` and not a Helm sub-chart dependency of `hr-app`. `charts/hr-app/templates/deployment-backend.yaml` conditionally (`backend.database.enabled`) injects `SPRING_DATASOURCE_URL`/`USERNAME`/`PASSWORD` from a CNPG-generated per-env secret named in `backend.database.secretName`; all three `values-<env>.yaml` now set `database.enabled: true` with `secretName: pg-dev-app` / `pg-staging-app` / `pg-prod-app` respectively. `staging` and `dev` run single-instance (`cluster.instances: 1`) CNPG clusters; only `prod` runs HA (`cluster.instances: 3`, with a `preferred` pod anti-affinity on `topologyKey: kubernetes.io/hostname`) — staging was deliberately scaled down from its earlier 3-instance HA setup as part of moving to one CNPG Cluster per environment. `cnpg-cluster-prod.yaml` has no `syncPolicy.automated` block (manual `argocd app sync cnpg-cluster-prod` only, matching `hr-prod`'s own no-auto-sync philosophy), while its companion `cnpg-network-policy-prod.yaml` stays on automated sync since a firewall rule landing early is harmless. Each env's CNPG `Cluster` binds to its own GCP service account (`sa-cnpg-<env>-backup@...`) via Workload Identity for GCS-backed barman-cloud backups. `repo-app`'s backend now has the JPA/JDBC dependency reconnected (see `repo-app/CLAUDE.md`) but still no business-domain entities beyond a connectivity-check one.

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
- **`networkPolicy.dnsClusterIP` no longer exists** (removed in `d9d5a19`) — DNS egress in `networkpolicy-backend.yaml`/`networkpolicy-frontend.yaml` now uses two `namespaceSelector`+`podSelector` peers in `kube-system` (`k8s-app: kube-dns` and `k8s-app: node-local-dns`), not an `ipBlock`. Root cause of the whole saga: GKE's NodeLocal DNSCache intercepts port-53 traffic and answers from a per-node `node-local-dns` pod instead of `kube-dns` — confirmed live via `cilium-dbg monitor --type drop`. This is cluster-recreate-safe (no hardcoded/env-specific IP to update by hand anymore). See `docs/issues-rencontrees.md` Issue 4 for the full history (it went `namespaceSelector` → `ipBlock 0.0.0.0/0` → `ipBlock` scoped to the ClusterIP → this final dual-selector form — four iterations, not two).
- **`backend.database.cnpgClusterName`** (added in `88aae5f`) parameterizes the `hr-backend` egress-to-Postgres `NetworkPolicy` rule's `podSelector` (`cnpg.io/cluster: <value>`) per environment (`pg-dev`/`pg-staging`/`pg-prod`). Before this it was hardcoded to `pg-staging` in the template, which silently blocked DB access via NetworkPolicy in `dev` and `prod` (only `staging` happened to match) — a real bug, not just a docs gap.
- **`backend.autoscaling.enabled`** activates `hpa-backend.yaml`, scaling `hr-backend` on CPU utilization against `backend.resources.requests.cpu`: `true` in staging (base 1 replica) and prod (`minReplicas: 2`/`maxReplicas: 5`), **`false`** in dev.
- Both `deployment-backend.yaml` and `deployment-frontend.yaml` set a soft `podAntiAffinity` (`preferredDuringSchedulingIgnoredDuringExecution`, weight 100, `topologyKey: kubernetes.io/hostname`) spreading same-app pods across nodes — best-effort, not a hard anti-affinity requirement, and identical across all three environments.
- **`backend.database.enabled` / `backend.database.secretName`**: `true` in all three overlays today, each pointing at that env's own CNPG-generated secret (`pg-dev-app`, `pg-staging-app`, `pg-prod-app`) — see the CNPG paragraph above. `values.yaml`'s defaults (`enabled: false`, `secretName: ""`) stay non-deployable on purpose.

## App-of-Apps (ArgoCD)

`apps/root-app.yaml`'s `source.path` **must be `apps/children`**, not `apps` — pointing at `apps` would make ArgoCD watch the folder containing `root-app.yaml` itself (non-recursive), silently never create any child Application, while still reporting `root-app` Healthy. `apps/children/` holds three files today: `dev.yaml` (`hr-dev` → ns `dev`), `staging.yaml` (`hr-staging` → ns `staging`), `prod.yaml` (`hr-prod` → ns `prod`) — adding another environment is just committing a new `apps/children/<env>.yaml`.

`hr-dev` and `hr-staging` run `syncPolicy.automated: {prune: true, selfHeal: true}` — **any manual `kubectl edit`/`kubectl scale` on the `dev`/`staging` namespaces gets reverted** within a few minutes. **`hr-prod` has no `automated` block at all** — a commit to `values-prod.yaml` only leaves it `OutOfSync`; deployment requires a manual `argocd app sync hr-prod` (or the ArgoCD UI), and hand-edits to the `prod` namespace are *not* auto-reverted since there is no `selfHeal` watching it. All changes must still go through a commit here regardless of environment. Rollback is a `git revert` (never `git push --force` — `selfHeal` compares state, not history) followed by a push to `main`.

All three children carry identical `ignoreDifferences`, load-bearing, not decorative:
- Deployment `/spec/replicas` — needed because `hpa-backend.yaml` patches this field at runtime; without the exception `selfHeal`/a sync would revert HPA scaling back to `backend.replicas` every reconcile loop (this exception applies to *all* Deployments in the chart, including `hr-frontend`, which has no HPA).
- Deployment `/status/terminatingReplicas` — fluctuates naturally during rolling updates.

There is no `StatefulSet` `ignoreDifferences` entry (and no `StatefulSet` in the chart at all — Postgres runs as a CNPG `Cluster` CR outside `charts/hr-app`, see below).

## Known discrepancy

`README.md`'s "Limitations actuelles" section (no resource limits, no probes, Azure/AKS references in `GITOPS_PFE.md`) is **stale** — the chart has since grown `resources` (requests/limits), readiness/liveness probes on every workload, `hpa-backend.yaml`, and three `NetworkPolicy` templates. Trust `docs/*.md` and the templates themselves over `README.md`/`GITOPS_PFE.md` for current state. Ingress being disabled (`ingress.enabled: false`, reachable only via `kubectl port-forward`) is the one limitation still accurate.

`README.md`, `docs/guide-argocd-gitops.md`, `docs/guide-helm-chart.md`, `docs/architecture-globale.md`, `docs/decisions-architecture.md`, and `docs/lifecycle-pipeline.md` are *also* stale on environment count — all of them predate `dev`/`prod` and still describe `apps/children/` as holding a single `staging.yaml`/`hr-staging` Application and the chart as having only `values.yaml`+`values-staging.yaml`. Three environments exist today (`hr-dev`, `hr-staging`, `hr-prod`); see App-of-Apps above. None of these docs mention CNPG/Postgres at all (not even the single-env staging version) — that entire subsystem, across all three environments, is undocumented outside this file and the inline comments in `apps/children/cnpg-*.yaml`/`manifests/cnpg-network-policy*/`. Trust this file and the actual manifests over any of those docs for current environment/CNPG state; they were left as-is rather than rewritten wholesale (see repo convention of flagging staleness here instead of chasing every doc).

A Postgres `StatefulSet` + HPA briefly existed in `charts/hr-app` (commits `fa12a0c`, `3c86ac4`) and was fully removed (`83792b4`) — don't go looking for a Postgres template or `Chart.yaml` dependency there, that stays gone. Postgres came back afterward (commits `2058586`, `6422edf`) as a separate CNPG `Cluster` CR managed by its own `apps/children/cnpg-*.yaml` Applications, wired to `hr-backend` only via the `backend.database.enabled` env vars in `deployment-backend.yaml` — architecturally unrelated to the removed `StatefulSet` approach. What started as staging-only CNPG later grew a `dev` cluster and an HA `prod` cluster (commits `3038cb1`, `e6d4321`), with staging itself scaled back down to single-instance (`340416d`) along the way.

## `docs/` index

Detailed, self-contained (French) guides — read the relevant one before making non-trivial changes:
- `docs/architecture-globale.md` — why three separate repos, what belongs to each, why a change in one never triggers another's pipeline
- `docs/lifecycle-pipeline.md` — full push-to-pod trace: exact CI jobs, files read/written, commands at each arrow
- `docs/guide-argocd-gitops.md` — App-of-Apps mechanics, `kubectl`/`argocd` observability commands, rollback procedure
- `docs/guide-helm-chart.md` — full template ↔ values reference table, local testing workflow
- `docs/guide-deploiement-infra.md` — Terraform provisioning (lives in `repo-infrastructure`, referenced from here for the ArgoCD bootstrap handoff)
- `docs/guide-securite.md` — every real security control and the concrete failure it prevents (WIF, Trivy, NetworkPolicy/Dataplane V2, Shielded Nodes, Binary Authorization, IAM scoping, Checkov) plus an explicit "not implemented yet" list
- `docs/decisions-architecture.md` — ADRs: Helm vs Terraform `kubernetes` provider, ArgoCD pull-based vs push-from-CI, why no Ansible
- `docs/issues-rencontrees.md` — real incidents with root cause and fix: ArgoCD CRD sync race, spot-node `kubectl wait` race, Trivy-blocked Tomcat CVEs, DNS-egress NetworkPolicy vs. NodeLocal DNSCache (Issue 4)
- `GITOPS_PFE.md` (repo root) — original note on the ArgoCD CRD race; superseded/expanded by `docs/issues-rencontrees.md` Issue 1

## Key Constraints

- Never hand-edit `backend.image.tag`/`frontend.image.tag` in `values-dev.yaml`/`values-staging.yaml` — `repo-app`'s CI overwrites them on every push to `develop`/`main`. Never hand-edit them in `values-prod.yaml` either — `promote-prod.yml` overwrites them on every `v*.*.*` tag promotion.
- Never hand-edit cluster resources in the `dev`/`staging` namespaces — `selfHeal` reverts it. The `prod` namespace has no `selfHeal` (`hr-prod` has no `automated` syncPolicy) — hand-edits there simply persist until the next manual `argocd app sync hr-prod`, which is a smaller footgun, not a protection.
- `apps/root-app.yaml` `source.path` must stay `apps/children`.
- No `resources` override in any of `values-dev.yaml`/`values-staging.yaml`/`values-prod.yaml` is intentional — all three inherit `values.yaml`'s requests/limits.
- DNS egress in the app NetworkPolicies now targets `kube-dns` **and** `node-local-dns` by selector (no `ipBlock`, no `networkPolicy.dnsClusterIP` — removed in `d9d5a19`); it survives a GKE cluster recreate with nothing to update by hand. Don't reintroduce an `ipBlock`-on-ClusterIP approach without re-reading `docs/issues-rencontrees.md` Issue 4 first — it took four iterations to land here.
- `backend.database.cnpgClusterName` must be set per environment (`pg-dev`/`pg-staging`/`pg-prod`) — a hardcoded `pg-staging` here previously broke the backend's NetworkPolicy egress to Postgres in `dev`/`prod` silently (`88aae5f`).
- `cnpg-cluster-prod.yaml` has no `syncPolicy.automated` — a change to prod's CNPG `Cluster` spec (e.g. scaling `instances`) only leaves `cnpg-cluster-prod` `OutOfSync`; it needs its own manual `argocd app sync cnpg-cluster-prod`, separate from `hr-prod`'s sync. `cnpg-network-policy-prod` stays automated, so the firewall rule for a new prod Postgres topology can land ahead of the manual Cluster sync.
- CNPG pods' egress NetworkPolicy (`manifests/cnpg-network-policy*/`) is intentionally fully open in every environment (not scoped like `hr-backend`/`hr-frontend`) — don't "tighten" it without re-testing against the GKE Workload Identity metadata-server rewrite issue documented in `docs/guide-securite.md`.
