# repo-config — Source de vérité GitOps

Ce dépôt est la **source de vérité GitOps** de la plateforme. ArgoCD surveille ce dépôt en continu et réconcilie l'état du cluster Kubernetes vers l'état déclaré ici. Aucune modification directe via `kubectl` ou `helm` ne doit être faite en régime normal — tout passe par Git (exception : `dev`/`staging` ont `selfHeal` qui annule de toute façon tout hand-edit ; `prod` n'a pas de `selfHeal`, donc un hand-edit y persiste jusqu'au prochain sync manuel, ce qui n'en fait pas une pratique recommandée pour autant).

---

## Contenu

```
repo-config/
├── apps/
│   ├── root-app.yaml               # App-of-Apps racine — applique une seule fois par le job bootstrap-argocd
│   └── children/
│       ├── dev.yaml / staging.yaml / prod.yaml     # Application ArgoCD hr-<env> → namespace <env>
│       ├── cert-manager.yaml                       # sync-wave -2 — prérequis TLS pour le plugin de backup CNPG
│       ├── cnpg-operator.yaml                      # sync-wave -1 — opérateur CloudNativePG
│       ├── cnpg-plugin-barman-cloud.yaml           # sync-wave -1 — plugin de backup barman-cloud
│       ├── cnpg-cluster-{dev,staging,prod}.yaml    # sync-wave 0 — CR Cluster CNPG par environnement
│       └── cnpg-network-policy-{dev,staging,prod}.yaml  # sync-wave -1 — NetworkPolicy port 5432 par environnement
│
├── manifests/
│   └── cnpg-network-policy*/networkpolicy-postgres.yaml   # manifests bruts référencés par les Applications ci-dessus
│
└── charts/hr-app/                  # Helm chart unique de l'application RH
    ├── Chart.yaml
    ├── values.yaml                 # défauts délibérément non déployables (registry vide, tag "UNSET")
    ├── values-{dev,staging,prod}.yaml
    └── templates/
        ├── deployment-{backend,frontend}.yaml    # probes readiness/liveness, resources, podAntiAffinity
        ├── hpa-backend.yaml                       # conditionnel — backend.autoscaling.enabled
        ├── service-{backend,frontend}.yaml        # annotation cloud.google.com/neg (container-native LB)
        ├── ingress.yaml                           # conditionnel — GCE natif, activé dans les 3 environnements
        └── networkpolicy-{default-deny,backend,frontend}.yaml   # conditionnel — networkPolicy.enabled
```

**Trois environnements existent aujourd'hui : `dev`, `staging`, `prod`** — chacun avec son propre `values-<env>.yaml` et sa propre Application ArgoCD. Ajouter un nouvel environnement se limite à committer `apps/children/<env>.yaml` + `charts/hr-app/values-<env>.yaml`.

---

## Comment fonctionne le GitOps

### Flux de mise à jour — dev / staging (automatisé)

```
CI Pipeline (repo-app, ci.yml)
       │  push develop → patch values-dev.yaml   |   push/merge main → patch values-staging.yaml
       │  yq .backend.image.tag = .frontend.image.tag = <SHA7>
       │  commit "ci: update <env> image tags to <SHA7>", push vers ce dépôt (via token GitHub App, pas un PAT)
       ▼
ArgoCD détecte le changement (polling)
       │
       ▼
helm upgrade hr-<env> (prune: true, selfHeal: true)
       │
       ▼
Nouveaux pods déployés avec la nouvelle image
```

### Flux de promotion — prod (manuel, en deux temps)

Un tag `v*.*.*` sur un commit `main` déclenche `promote-prod.yml` (dans `repo-app`, gardé par l'environnement GitHub `production`) : l'image **déjà construite** en staging est copiée telle quelle (digest préservé, `crane copy`, jamais rebuild) vers le registre prod isolé, puis `values-prod.yaml` est patché ici (commit `ci: promote <version> to prod`). **Ce commit ne déploie rien tout seul** — `hr-prod` n'a pas de `syncPolicy.automated`, il reste `OutOfSync` jusqu'à un `argocd app sync hr-prod` manuel (ou l'UI). Promotion et déploiement sont délibérément découplés.

### App-of-Apps

`root-app` (`apps/root-app.yaml`, appliqué une seule fois par le job `bootstrap-argocd`) surveille le dossier `apps/children/` avec `source.path: apps/children` — **ce chemin doit rester `apps/children`**, pas `apps` (sinon ArgoCD surveille le dossier contenant `root-app.yaml` lui-même, ne crée jamais aucune Application enfant, et rapporte quand même `root-app` Healthy). Chaque fichier YAML dans ce dossier devient une `Application` ArgoCD gérée automatiquement.

---

## Manifests ArgoCD

### `apps/children/staging.yaml` (même forme pour `dev.yaml`/`prod.yaml`, namespace et values file différents)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hr-staging
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/mariemmehri/repo-config
    targetRevision: main
    path: charts/hr-app
    helm:
      valueFiles:
        - values-staging.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers: [/status/terminatingReplicas, /spec/replicas]
    - group: networking.k8s.io
      kind: Ingress
      jsonPointers: [/metadata/annotations, /metadata/finalizers]
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
      - RespectIgnoreDifferences=true
```

**Points importants :**
- `prune: true` — si une ressource est supprimée de Git, elle est supprimée du cluster
- `selfHeal: true` — reverte tout changement manuel sur le cluster. **`hr-prod` n'a pas de bloc `automated` du tout** — ni `prune`, ni `selfHeal` — le déploiement prod nécessite un `argocd app sync hr-prod` explicite à chaque fois.
- L'exception `ignoreDifferences` sur `Deployment /spec/replicas` est nécessaire depuis `hpa-backend.yaml` — sans elle, `selfHeal` reverterait le scaling du HPA à `backend.replicas` à chaque cycle de réconciliation. Elle s'applique à tous les Deployments du chart, y compris `hr-frontend` qui n'a pas de HPA.
- L'exception sur `Ingress /metadata/annotations` et `/metadata/finalizers` est nécessaire depuis l'ajout de l'Ingress GCE natif — le contrôleur GKE écrit ses propres annotations (bookkeeping NEG/backend) et un finalizer sur l'objet après création ; sans cette exception, chaque cycle de réconciliation verrait l'Ingress `OutOfSync` et `selfHeal` se battrait avec le contrôleur.

### `apps/root-app.yaml`

Appliqué une seule fois par le job `bootstrap-argocd` de `repo-infrastructure` (`kubectl apply -f apps/root-app.yaml`), jamais par ArgoCD lui-même ni par Terraform.

---

## Helm Chart — `charts/hr-app/`

### Structure des values

**`values.yaml`** — défauts délibérément non déployables (registry vide, tag `"UNSET"`) :

```yaml
registry: { host: "", repository: "" }

backend:
  image: { name: hr-backend, tag: "UNSET" }
  replicas: 1
  port: 8081
  database: { enabled: false, secretName: "", cnpgClusterName: "" }
  resources:
    requests: { cpu: "100m", memory: "256Mi" }
    limits:   { cpu: "500m", memory: "512Mi" }
  autoscaling: { enabled: false, minReplicas: 1, maxReplicas: 3, targetCPUUtilizationPercentage: 70 }

frontend:
  image: { name: hr-frontend, tag: "UNSET" }
  replicas: 1
  port: 80
  resources:
    requests: { cpu: "50m", memory: "64Mi" }
    limits:   { cpu: "200m", memory: "128Mi" }

ingress: { enabled: false, host: "", staticIpName: "" }
networkPolicy: { enabled: true }
```

**`values-staging.yaml`** — overrides staging (tags patchés automatiquement par la CI de `repo-app`) :

```yaml
registry:
  host: europe-west1-docker.pkg.dev
  repository: pfe-2026-495220/registry-staging-pfe
backend:
  autoscaling: { enabled: true }
  database: { enabled: true, secretName: "pg-staging-app", cnpgClusterName: "pg-staging" }
ingress:
  enabled: true
  host: hr-staging.local
  staticIpName: ip-hr-staging
```

`values-dev.yaml` suit le même schéma (`registry-staging-pfe`, `autoscaling.enabled: false`, `pg-dev`). `values-prod.yaml` pointe vers `registry-prod-pfe` (registre isolé), a `replicas: 2` et `autoscaling: {enabled: true, minReplicas: 2, maxReplicas: 5}`, et son tag est un `vX.Y.Z` patché par `promote-prod.yml`, pas un SHA.

### Templates Kubernetes

| Template | Ressource créée | Détail |
|----------|-----------------|--------|
| `deployment-backend.yaml` | Deployment `hr-backend` | probes readiness/liveness sur `/api/health-check`, `resources`, injecte les identifiants Postgres depuis le secret CNPG si `backend.database.enabled` |
| `deployment-frontend.yaml` | Deployment `hr-frontend` | Image Nginx, mêmes probes/`resources`, `podAntiAffinity` soft partagée avec le backend |
| `hpa-backend.yaml` | HorizontalPodAutoscaler | conditionnel `backend.autoscaling.enabled` |
| `service-backend.yaml` / `service-frontend.yaml` | Service ClusterIP | annotation `cloud.google.com/neg: '{"ingress": true}'` — container-native load balancing pour l'Ingress GCE |
| `ingress.yaml` | Ingress (conditionnel) | `kubernetes.io/ingress.class: "gce"` — contrôleur natif GKE, pas ingress-nginx. Routes `/api` → backend, `/` → frontend |
| `networkpolicy-default-deny.yaml` | NetworkPolicy | deny-all ingress+egress, base zero-trust |
| `networkpolicy-backend.yaml` | NetworkPolicy | ingress depuis `hr-frontend` + depuis les plages de probe GCE (`130.211.0.0/22`, `35.191.0.0/16`), egress DNS + Postgres |
| `networkpolicy-frontend.yaml` | NetworkPolicy | ingress ouvert, egress vers `hr-backend` + DNS |

Aucun template `StatefulSet` — Postgres tourne comme CR CloudNativePG `Cluster`, géré par des Applications ArgoCD séparées (`apps/children/cnpg-cluster-<env>.yaml`), pas par ce chart.

### Ingress — natif GKE/GCE, activé dans les 3 environnements

`ingress.yaml` utilise le contrôleur Ingress **natif de GKE** (`kubernetes.io/ingress.class: "gce"`) — pas de pod contrôleur à installer. Chaque `values-<env>.yaml` a `ingress.enabled: true`, un `ingress.host` (`hr-<env>.local` — ne résout qu'en local via le fichier hosts, faute de domaine public possédé) et un `ingress.staticIpName` (réservé côté Terraform dans `repo-infrastructure`). **Pas de TLS pour l'instant** — HTTP simple ; un `ManagedCertificate` Google ne peut pas se valider sur un domaine non résolvable publiquement.

---

## Utilisation locale

```bash
# Lint
helm lint charts/hr-app

# Rendu des manifests (dry-run) — staging
helm template hr-staging charts/hr-app \
  -f charts/hr-app/values.yaml \
  -f charts/hr-app/values-staging.yaml

# Même chose pour dev/prod — remplacer le nom de release et le -f values-<env>.yaml

# Installation locale (namespace de test, jamais staging directement)
kubectl create namespace staging
helm upgrade --install hr-staging charts/hr-app -f charts/hr-app/values-staging.yaml --namespace staging
helm uninstall hr-staging --namespace staging
```

Si Helm n'est pas installé localement :
```bash
docker run --rm -v "$(pwd):/work" -w /work alpine/helm:3.14.0 \
  template hr-staging charts/hr-app -f charts/hr-app/values.yaml -f charts/hr-app/values-staging.yaml
```

### Vérification sur le cluster réel

```bash
gcloud container clusters get-credentials gke-staging-pfe --region europe-west1-b --project pfe-2026-495220
kubectl get application hr-staging -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'   # attendu : Synced Healthy
kubectl get pods -n staging
```
Même schéma pour `hr-dev`/namespace `dev` et `hr-prod`/namespace `prod` — `hr-prod` `OutOfSync` entre une promotion et le prochain sync manuel est normal, pas un bug.

---

## Mise à jour des tags d'image

Gérée automatiquement par `yq` dans la CI de `repo-app` (`ci.yml` pour dev/staging, `promote-prod.yml` pour prod) :

```bash
yq eval '.backend.image.tag = strenv(IMAGE_TAG) | .frontend.image.tag = strenv(IMAGE_TAG)' \
  -i charts/hr-app/values-<env>.yaml
```

**Ne jamais modifier les tags manuellement** dans `values-dev.yaml`/`values-staging.yaml`/`values-prod.yaml` — le prochain push/promotion les écrase.

---

## Ce qui existe réellement aujourd'hui (pour éviter toute confusion avec une doc plus ancienne)

- **Ingress activé et fonctionnel** dans les 3 environnements (GCE natif) — ce n'est plus une limitation.
- **`resources` (requests/limits)** définis dans `values.yaml`, hérités par les 3 environnements.
- **`readinessProbe`/`livenessProbe`** sur tous les workloads (backend et frontend).
- **`HorizontalPodAutoscaler`** sur `hr-backend`, activé en staging et prod.
- **Trois `NetworkPolicy`** (deny-all + isolation backend/frontend), activées par défaut.
- **PostgreSQL réel** via CloudNativePG, un cluster par environnement — pas de base en mémoire.
- **Pas encore implémenté** : TLS/HTTPS sur l'Ingress, `BackendConfig`/`FrontendConfig` GCE, domaine public/DNS, `securityContext` Kubernetes (`runAsNonRoot` etc.) sur les Deployments, observabilité (Prometheus/Grafana), External Secrets Operator.

---

## Pour aller plus loin

- [CLAUDE.md](CLAUDE.md) — inventaire complet des templates, référence template↔values, détail App-of-Apps
- `docs/` — guides détaillés (français) : architecture globale, cycle de vie complet, ArgoCD/GitOps, chart Helm, déploiement infra, sécurité, décisions d'architecture, problèmes rencontrés. Voir l'index dans `CLAUDE.md`.
- [GITOPS_PFE.md](GITOPS_PFE.md) — note historique sur la course de synchronisation CRD ArgoCD ; contexte étendu dans `docs/issues-rencontrees.md` (Issue 1)
