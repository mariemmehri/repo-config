# CLAUDE.md — repo-config

This file provides guidance to Claude Code when working inside `repo-config/`.
The parent `../CLAUDE.md` covers the full platform (app, infra, CI/CD overview).

## What this repo is

The GitOps **source of truth**. No application code, no Terraform — only a Helm chart and ArgoCD `Application` manifests. ArgoCD (running inside the GKE cluster) polls this repo continuously and reconciles cluster state to match it; nothing here is applied by hand via `kubectl`/`helm` in normal operation. There is no `.github/workflows/` in this repo — a commit here triggers nothing on GitHub, only ArgoCD's poll loop reacts to it.

Only one environment exists: `staging`. There is no `values-dev.yaml`/`values-prod.yaml`, no `_helpers.tpl`, no `NOTES.txt`.

## Repository Structure

```
repo-config/
├── apps/
│   ├── root-app.yaml           # App-of-Apps root; applied once by hand (bootstrap-argocd job), watches apps/children/
│   └── children/
│       └── staging.yaml        # ArgoCD Application "hr-staging" → renders charts/hr-app with values-staging.yaml
├── charts/hr-app/
│   ├── Chart.yaml
│   ├── values.yaml              # chart defaults — deliberately non-deployable (empty registry, tag "UNSET")
│   ├── values-staging.yaml      # staging overrides — image tags patched automatically by repo-app's CI (yq)
│   └── templates/
│       ├── deployment-backend.yaml / deployment-frontend.yaml
│       ├── hpa-backend.yaml            # conditional: backend.autoscaling.enabled
│       ├── service-backend.yaml / service-frontend.yaml
│       ├── ingress.yaml                # conditional: ingress.enabled (false today)
│       └── networkpolicy-{default-deny,backend,frontend}.yaml  # conditional: networkPolicy.enabled
└── docs/                         # deep-dive guides (French) — see index below
```

There is no database in this stack — the chart has no persistence layer, no StatefulSet, and no Postgres dependency.

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

### Live-cluster verification
```bash
gcloud container clusters get-credentials gke-staging-pfe --region europe-west1-b --project pfe-2026-495220
kubectl get applications -n argocd
kubectl get application hr-staging -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'   # expect: Synced Healthy
kubectl get deployment hr-backend -n staging -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'         # tag must match latest "ci: update image tags to <SHA7>" commit
kubectl get pods -n staging
```

## Architecture: values.yaml vs values-staging.yaml

Helm merges both files (`values.yaml` provides defaults, `values-staging.yaml` overrides); ArgoCD's `hr-staging` Application declares `helm.valueFiles: [values-staging.yaml]`, which does the same merge internally. `values-staging.yaml` does **not** redefine `resources` — the requests/limits in `values.yaml` apply as-is in staging.

- **`backend.image.tag` / `frontend.image.tag`** in `values-staging.yaml` are patched automatically by `repo-app`'s CI via `yq` on every push to `main` (`ci: update image tags to <SHA7>` commits) — never edit by hand, the next push overwrites it.
- **`networkPolicy.dnsClusterIP`** is the ClusterIP of the cluster's `kube-dns` Service (`kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.clusterIP}'`) — cluster-specific, must be updated by hand if the cluster is ever recreated (`destroy-staging` then `apply`), otherwise DNS egress breaks. See `docs/issues-rencontrees.md` Issue 4 for why this is an `ipBlock` and not a `podSelector`/`namespaceSelector`.
- **`backend.autoscaling.enabled: true`** in staging activates `hpa-backend.yaml`, scaling `hr-backend` on CPU utilization against `backend.resources.requests.cpu`.

## App-of-Apps (ArgoCD)

`apps/root-app.yaml`'s `source.path` **must be `apps/children`**, not `apps` — pointing at `apps` would make ArgoCD watch the folder containing `root-app.yaml` itself (non-recursive), silently never create any child Application, while still reporting `root-app` Healthy. `apps/children/staging.yaml` (Application `hr-staging`) is the only child today; adding an environment is just committing a new `apps/children/<env>.yaml`.

Both Applications run `syncPolicy.automated: {prune: true, selfHeal: true}` — **any manual `kubectl edit`/`kubectl scale` on the `staging` namespace gets reverted** within a few minutes. All changes must go through a commit here. Rollback is a `git revert` (never `git push --force` — `selfHeal` compares state, not history) followed by a push to `main`.

`hr-staging`'s `ignoreDifferences` are load-bearing, not decorative:
- Deployment `/spec/replicas` — needed because `hpa-backend.yaml` patches this field at runtime; without the exception `selfHeal` would revert HPA scaling back to `backend.replicas` every reconcile loop (this exception applies to *all* Deployments in the chart, including `hr-frontend`, which has no HPA).
- Deployment `/status/terminatingReplicas` — fluctuates naturally during rolling updates.
- StatefulSet `/spec/volumeClaimTemplates/0/status` — the API server injects a `status` sub-object into the live PVC template once it exists, a field that can never appear in Helm's rendered manifest.

## Known discrepancy

`README.md`'s "Limitations actuelles" section (no resource limits, no probes, Azure/AKS references in `GITOPS_PFE.md`) is **stale** — the chart has since grown `resources` (requests/limits), readiness/liveness probes on every workload, `hpa-backend.yaml`, and three `NetworkPolicy` templates. Trust `docs/*.md` and the templates themselves over `README.md`/`GITOPS_PFE.md` for current state. Ingress being disabled (`ingress.enabled: false`, reachable only via `kubectl port-forward`) is the one limitation still accurate.

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

- Never hand-edit `backend.image.tag`/`frontend.image.tag` in `values-staging.yaml` — CI overwrites them on every push to `repo-app`.
- Never hand-edit cluster resources in the `staging` namespace — `selfHeal` reverts it.
- `apps/root-app.yaml` `source.path` must stay `apps/children`.
- No `resources` override in `values-staging.yaml` is intentional — it inherits `values.yaml`'s requests/limits.
- Recreating the GKE cluster invalidates `networkPolicy.dnsClusterIP` — update it before the next deploy or DNS egress breaks.
