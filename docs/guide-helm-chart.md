# Guide du chart Helm `charts/hr-app`

## 🎯 Objectif

Expliquer comment est structuré le chart Helm `charts/hr-app`, la différence entre `values.yaml` et les trois overlays `values-<env>.yaml`, et comment le tester en local avec `helm lint`/`helm template` avant de committer. Document autonome.

---

## ⚙️ Structure du chart

```
charts/hr-app/
├── Chart.yaml                  # name: hr-app, version: 0.1.0, appVersion: "1.0.0"
├── values.yaml                  # valeurs par défaut du chart — délibérément non déployables
├── values-dev.yaml               # overrides dev (patché par le CI de repo-app sur push develop)
├── values-staging.yaml           # overrides staging (patché par le CI de repo-app sur push/merge main)
├── values-prod.yaml               # overrides prod (patché par promote-prod.yml sur tag v*.*.*)
└── templates/
    ├── deployment-backend.yaml    # Deployment hr-backend
    ├── deployment-frontend.yaml   # Deployment hr-frontend
    ├── hpa-backend.yaml           # HorizontalPodAutoscaler hr-backend conditionnel
    ├── service-backend.yaml       # Service ClusterIP hr-backend (port 8081) + annotation NEG
    ├── service-frontend.yaml      # Service ClusterIP hr-frontend (port 80) + annotation NEG
    ├── ingress.yaml                # Ingress GCE natif conditionnel
    ├── networkpolicy-default-deny.yaml  # Deny-all ingress+egress conditionnel
    ├── networkpolicy-backend.yaml        # Ingress hr-backend limité à hr-frontend + probes GCE + egress DNS/Postgres
    └── networkpolicy-frontend.yaml       # Egress hr-frontend limité à hr-backend + DNS
```

Il n'y a ni `_helpers.tpl`, ni `NOTES.txt`, ni template `StatefulSet` aujourd'hui — Postgres tourne comme CR CloudNativePG géré par des Applications ArgoCD séparées, pas par ce chart.

## 🚀 `values.yaml` vs `values-<env>.yaml`

**`values.yaml`** — les valeurs par défaut du chart, volontairement non déployables telles quelles (`registry.host`/`registry.repository` vides, tags `"UNSET"`) :

```yaml
registry:
  host: ""
  repository: ""

backend:
  image: { name: hr-backend, tag: "UNSET" }
  replicas: 1
  port: 8081
  database:
    enabled: false
    secretName: ""
    cnpgClusterName: ""
  resources:
    requests: { cpu: "100m", memory: "256Mi" }
    limits:   { cpu: "500m", memory: "512Mi" }
  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 3
    targetCPUUtilizationPercentage: 70

frontend:
  image: { name: hr-frontend, tag: "UNSET" }
  replicas: 1
  port: 80
  resources:
    requests: { cpu: "50m", memory: "64Mi" }
    limits:   { cpu: "200m", memory: "128Mi" }

ingress:
  enabled: false
  host: ""
  staticIpName: ""

networkPolicy:
  enabled: true
```

**`values-staging.yaml`** — un des trois overlays réels (même schéma pour `values-dev.yaml`/`values-prod.yaml`, valeurs différentes) :

```yaml
registry:
  host: europe-west1-docker.pkg.dev
  repository: pfe-2026-495220/registry-staging-pfe
backend:
  replicas: 1
  image: { name: hr-backend, tag: "2e5169e" }
  autoscaling: { enabled: true }
  database: { enabled: true, secretName: "pg-staging-app", cnpgClusterName: "pg-staging" }
frontend:
  replicas: 1
  image: { name: hr-frontend, tag: "2e5169e" }
ingress:
  enabled: true
  host: hr-staging.local
  staticIpName: ip-hr-staging
networkPolicy:
  enabled: true
```

Différences notables entre les trois overlays :
- **`registry.repository`** : `dev` et `staging` pointent tous deux sur `registry-staging-pfe` ; `prod` pointe sur `registry-prod-pfe` (registre isolé, alimenté uniquement par `crane copy`).
- **`backend.image.tag`/`frontend.image.tag`** : SHA courts (7 car.) en `dev`/`staging`, patchés automatiquement par le CI de `repo-app` (`yq`, voir [lifecycle-pipeline.md](lifecycle-pipeline.md)) ; version sémantique `vX.Y.Z` en `prod`, patchée par `promote-prod.yml`. **Ne jamais éditer ces champs à la main** — le prochain push/promotion les écrase.
- **`backend.autoscaling.enabled`** : `true` en `staging` (base 1 replica) et `prod` (`minReplicas: 2`/`maxReplicas: 5`), `false` en `dev`.
- **`backend.database.*`** : `true`/renseigné dans les trois overlays aujourd'hui, chacun pointant sur le secret et le nom de cluster CNPG de son propre environnement (`pg-dev-app`/`pg-dev`, `pg-staging-app`/`pg-staging`, `pg-prod-app`/`pg-prod`). `values.yaml` garde ses défauts non déployables (`enabled: false`) exprès.
- **`ingress.host`/`ingress.staticIpName`** : `hr-<env>.local` / `ip-hr-<env>` dans les trois — voir la section Ingress plus bas.
- Aucun des trois overlays ne redéfinit `resources` — les requests/limits de `values.yaml` s'appliquent telles quelles partout.

L'Application ArgoCD `hr-<env>` référence explicitement `helm.valueFiles: [values-<env>.yaml]` — c'est ArgoCD qui applique la fusion `values.yaml` (défauts du chart) + `values-<env>.yaml` (overrides), exactement comme le ferait `helm template -f values.yaml -f values-<env>.yaml`.

## 🚀 Les templates et leurs valeurs injectées

| Template | Ressource | `.Values.*` utilisés |
|---|---|---|
| `deployment-backend.yaml` | Deployment `hr-backend` | `backend.replicas`, `registry.*`, `backend.image.*`, `backend.port`, `backend.resources`, `backend.database.*` (injecte `SPRING_DATASOURCE_URL`/`USERNAME`/`PASSWORD` depuis le secret CNPG si `enabled`) |
| `deployment-frontend.yaml` | Deployment `hr-frontend` | `frontend.replicas`, `registry.*`, `frontend.image.*`, `frontend.port`, `frontend.resources` |
| `hpa-backend.yaml` | HorizontalPodAutoscaler `hr-backend` (conditionnel) | `backend.autoscaling.enabled` (garde), `minReplicas`/`maxReplicas`/`targetCPUUtilizationPercentage` |
| `service-backend.yaml` | Service ClusterIP `hr-backend` + annotation `cloud.google.com/neg` | `backend.port` (`targetPort` ; le `port` exposé, `8081`, est en dur) |
| `service-frontend.yaml` | Service ClusterIP `hr-frontend` + annotation `cloud.google.com/neg` | `frontend.port` (`targetPort` ; le `port` exposé, `80`, est en dur) |
| `ingress.yaml` | Ingress `hr-ingress` (conditionnel, GCE natif) | `ingress.enabled`, `ingress.host`, `ingress.staticIpName` |
| `networkpolicy-default-deny.yaml` | NetworkPolicy `default-deny-all` (conditionnel) | `networkPolicy.enabled` — `podSelector: {}` sur tout le namespace |
| `networkpolicy-backend.yaml` | NetworkPolicy `hr-backend` (conditionnel) | `networkPolicy.enabled`, `backend.port`, `backend.database.cnpgClusterName` — ingress depuis `app: hr-frontend` **et** depuis les plages de health-check GCE (`130.211.0.0/22`, `35.191.0.0/16`), egress DNS (`kube-dns` + `node-local-dns`) et Postgres |
| `networkpolicy-frontend.yaml` | NetworkPolicy `hr-frontend` (conditionnel) | `networkPolicy.enabled`, `frontend.port`, `backend.port` — ingress ouvert, egress vers `hr-backend` + DNS |

Les `resources` (requests/limits CPU et mémoire) sont injectées via :
```yaml
resources:
  {{- .Values.backend.resources | toYaml | nindent 12 }}
```
(et l'équivalent `.Values.frontend.resources` côté frontend) — jamais codées en dur dans le template.

Chaque Deployment expose une `readinessProbe`/`livenessProbe` HTTP GET sur son propre port (backend : `/api/health-check` ; frontend : `/`), plus un `podAntiAffinity` soft (`preferredDuringSchedulingIgnoredDuringExecution`, `topologyKey: kubernetes.io/hostname`) répartissant les pods d'une même app sur des nodes différents.

**Autoscaling (`hpa-backend.yaml`)** : basé sur le % d'utilisation CPU par rapport à `backend.resources.requests.cpu` (le `metrics-server` requis est un composant GKE standard déjà présent). `deployment-backend.yaml` a un `spec.replicas` statique piloté par `backend.replicas`, mais une fois le HPA actif c'est lui qui décide du nombre de replicas réel — voir [guide-argocd-gitops.md](guide-argocd-gitops.md) pour l'entrée `ignoreDifferences` nécessaire côté ArgoCD.

**Base de données (`backend.database.*`)** : quand `enabled: true`, `deployment-backend.yaml` injecte `SPRING_DATASOURCE_URL`/`USERNAME`/`PASSWORD` depuis le secret Kubernetes nommé par `secretName`, généré par le CR CloudNativePG `Cluster` de l'environnement (`pg-<env>-app`). `cnpgClusterName` paramètre en plus le `podSelector` de la règle d'egress Postgres dans `networkpolicy-backend.yaml` (`cnpg.io/cluster: <valeur>`) — doit correspondre au nom réel du cluster CNPG de l'environnement, sinon l'egress réseau vers Postgres est silencieusement bloqué même si le secret est correct.

## 🚀 Ingress (GCE natif, activé dans les 3 environnements)

`ingress.yaml` utilise le contrôleur Ingress **natif de GKE** (`kubernetes.io/ingress.class: "gce"`) — pas de pod contrôleur à installer, contrairement à ingress-nginx. Chaque `values-<env>.yaml` a `ingress.enabled: true`, un `ingress.host` (`hr-dev.local`/`hr-staging.local`/`hr-prod.local` — domaines qui ne résolvent qu'en local, via le fichier hosts, faute de domaine public possédé) et un `ingress.staticIpName` (`ip-hr-dev`/`ip-hr-staging`/`ip-hr-prod`, réservé côté Terraform dans `repo-infrastructure/environments/staging/main.tf`). Pas de TLS pour l'instant — HTTP simple, un `ManagedCertificate` Google ne pouvant pas se valider sur un domaine non résolvable publiquement.

Les Services `hr-backend`/`hr-frontend` portent l'annotation `cloud.google.com/neg: '{"ingress": true}'` pour le container-native load balancing (NEG) qu'exploite l'Ingress GCE — requiert un cluster VPC-native (c'est le cas de `gke-staging-pfe`). Le template route `/api` vers `hr-backend:8081` et `/` vers `hr-frontend:80`.

Le health-check du load balancer GCE atteint les pods directement via le NEG, depuis les plages IP `130.211.0.0/22`/`35.191.0.0/16` de Google — c'est pour cette raison que `networkpolicy-backend.yaml` a une deuxième règle d'ingress autorisant ces plages en plus de `hr-frontend` (sans elle, le health-check échoue et chaque requête `/api` reçoit un 502 côté LB, même si le pod est `Ready` côté Kubernetes). `hr-frontend` n'a pas eu besoin de cet ajout — sa règle d'ingress n'a jamais eu de restriction `from:`.

## ✅ Tester le chart en local avant de committer

```bash
cd repo-config

# 1. Lint statique — attrape les erreurs de syntaxe/structure du chart
helm lint charts/hr-app

# 2. Rendu complet avec les valeurs de staging — vérifie ce qui serait réellement appliqué
helm template hr-staging charts/hr-app \
  -f charts/hr-app/values.yaml \
  -f charts/hr-app/values-staging.yaml

# 3. Vérifier une valeur précise sans tout relire (ex: les resources du backend)
helm template hr-staging charts/hr-app \
  -f charts/hr-app/values.yaml \
  -f charts/hr-app/values-staging.yaml \
  --show-only templates/deployment-backend.yaml
```
Même commandes pour `dev`/`prod` — remplacer le nom de release et `-f charts/hr-app/values-<env>.yaml`.

`helm template` ne contacte aucun cluster — c'est un rendu purement local des manifests Kubernetes finaux, exactement ce qu'ArgoCD calcule en interne avant de comparer au cluster réel.

Si Helm n'est pas installé localement, le même rendu peut être obtenu via l'image officielle :
```bash
docker run --rm -v "$(pwd):/work" -w /work alpine/helm:3.14.0 \
  template hr-staging charts/hr-app -f charts/hr-app/values.yaml -f charts/hr-app/values-staging.yaml
```

## 🔗 Pour la suite

- Comment ArgoCD applique ce chart automatiquement, dans les 3 environnements : [guide-argocd-gitops.md](guide-argocd-gitops.md)
- D'où viennent les tags d'image patchés dans `values-<env>.yaml`, y compris le flux de promotion prod : [lifecycle-pipeline.md](lifecycle-pipeline.md)
- Pourquoi Helm plutôt que le provider Terraform `kubernetes` pour déployer l'appli : [decisions-architecture.md](decisions-architecture.md)
- Détail des mesures de sécurité (NetworkPolicy, DNS, CNPG) : [guide-securite.md](guide-securite.md)
