# Guide du chart Helm `charts/hr-app`

## 🎯 Objectif

Expliquer comment est structuré le chart Helm `charts/hr-app`, la différence entre `values.yaml` et `values-staging.yaml`, et comment le tester en local avec `helm lint`/`helm template` avant de committer. Document autonome.

---

## ⚙️ Structure du chart

```
charts/hr-app/
├── Chart.yaml               # name: hr-app, version: 0.1.0, appVersion: "1.0.0"
├── values.yaml               # valeurs par défaut du chart (non spécifiques à un environnement)
├── values-staging.yaml       # overrides pour l'environnement staging (patché par le CI de repo-app)
└── templates/
    ├── deployment-backend.yaml    # Deployment hr-backend
    ├── deployment-frontend.yaml   # Deployment hr-frontend
    ├── service-backend.yaml       # Service ClusterIP hr-backend (port 8081)
    ├── service-frontend.yaml      # Service ClusterIP hr-frontend (port 80)
    ├── ingress.yaml                # Ingress conditionnel ({{- if .Values.ingress.enabled }})
    ├── networkpolicy-default-deny.yaml  # Deny-all ingress+egress conditionnel ({{- if .Values.networkPolicy.enabled }})
    ├── networkpolicy-backend.yaml        # Ingress hr-backend limité à hr-frontend + egress DNS
    └── networkpolicy-frontend.yaml       # Egress hr-frontend limité à hr-backend + DNS
```

Il n'y a ni `_helpers.tpl`, ni `NOTES.txt`, ni `values-dev.yaml` aujourd'hui — un seul environnement (`staging`) est couvert.

## 🚀 `values.yaml` vs `values-staging.yaml`

**`values.yaml`** — les valeurs par défaut du chart, volontairement non déployables telles quelles (`registry.host`/`registry.repository` vides, tags `"UNSET"`) :

```yaml
registry:
  host: ""
  repository: ""

backend:
  image: { name: hr-backend, tag: "UNSET" }
  replicas: 1
  port: 8081
  resources:
    requests: { cpu: "100m", memory: "256Mi" }
    limits:   { cpu: "500m", memory: "512Mi" }

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

networkPolicy:
  enabled: true
  # ClusterIP du service kube-dns — dépend du cluster, override par environnement.
  # Récupérable via: kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.clusterIP}'
  dnsClusterIP: ""
```

**`values-staging.yaml`** — les overrides réels pour l'environnement staging :

```yaml
registry:
  host: europe-west1-docker.pkg.dev
  repository: pfe-2026-495220/registry-staging-pfe
backend:
  replicas: 1
  image: { name: hr-backend, tag: "e651d30" }
frontend:
  replicas: 1
  image: { name: hr-frontend, tag: "e651d30" }
ingress:
  enabled: false
  host: hr-staging.example.com
networkPolicy:
  enabled: true
  dnsClusterIP: "34.118.224.10"
```

`values-staging.yaml` ne redéfinit **pas** `resources` — Helm fusionne les deux fichiers de values, donc `backend.resources`/`frontend.resources` de `values.yaml` s'appliquent tels quels en staging. **Les champs `backend.image.tag` et `frontend.image.tag` de `values-staging.yaml` sont gérés automatiquement par le CI de `repo-app`** (patch `yq`, voir [lifecycle-pipeline.md](lifecycle-pipeline.md)) — ne jamais les modifier à la main, le prochain push applicatif les écrasera.

`networkPolicy.dnsClusterIP` est en revanche **à renseigner à la main par environnement** — ce n'est pas une valeur portable d'un cluster à l'autre : la ClusterIP de `kube-dns` est attribuée par GKE à la création du cluster et change si le cluster est recréé (ex: `destroy-staging` puis `apply`). `values.yaml` la laisse vide (`""`) pour que l'absence de valeur soit visible immédiatement (policy egress DNS invalide) plutôt que de pointer silencieusement sur une IP obsolète.

L'Application ArgoCD `hr-staging` référence explicitement `helm.valueFiles: [values-staging.yaml]` — c'est ArgoCD qui applique la fusion `values.yaml` (défauts du chart) + `values-staging.yaml` (overrides), exactement comme le ferait `helm template -f values.yaml -f values-staging.yaml`.

## 🚀 Les templates et leurs valeurs injectées

| Template | Ressource | `.Values.*` utilisés |
|---|---|---|
| `deployment-backend.yaml` | Deployment `hr-backend` | `backend.replicas`, `registry.host`, `registry.repository`, `backend.image.name`, `backend.image.tag`, `backend.port`, `backend.resources` |
| `deployment-frontend.yaml` | Deployment `hr-frontend` | `frontend.replicas`, `registry.host`, `registry.repository`, `frontend.image.name`, `frontend.image.tag`, `frontend.port`, `frontend.resources` |
| `service-backend.yaml` | Service ClusterIP `hr-backend` | `backend.port` (comme `targetPort` ; le `port` exposé, `8081`, est en dur) |
| `service-frontend.yaml` | Service ClusterIP `hr-frontend` | `frontend.port` (comme `targetPort` ; le `port` exposé, `80`, est en dur) |
| `ingress.yaml` | Ingress `hr-ingress` (conditionnel) | `ingress.enabled` (garde `{{- if }}`), `ingress.host` |
| `networkpolicy-default-deny.yaml` | NetworkPolicy `default-deny-all` (conditionnel) | `networkPolicy.enabled` (garde `{{- if }}`) — `podSelector: {}` sur tout le namespace |
| `networkpolicy-backend.yaml` | NetworkPolicy `hr-backend` (conditionnel) | `networkPolicy.enabled`, `backend.port`, `networkPolicy.dnsClusterIP` (ingress depuis `app: hr-frontend` + egress DNS scopé à la ClusterIP `kube-dns`) |
| `networkpolicy-frontend.yaml` | NetworkPolicy `hr-frontend` (conditionnel) | `networkPolicy.enabled`, `frontend.port`, `backend.port`, `networkPolicy.dnsClusterIP` (ingress ouvert, egress limité à `hr-backend` + DNS scopé) |

Les `resources` (requests/limits CPU et mémoire) sont injectées via :
```yaml
resources:
  {{- .Values.backend.resources | toYaml | nindent 12 }}
```
(et l'équivalent `.Values.frontend.resources` côté frontend) — jamais codées en dur dans le template, ce qui permet de les ajuster par environnement sans toucher au template lui-même.

Chaque Deployment expose une `readinessProbe`/`livenessProbe` HTTP GET sur son propre port :
- Backend : `/api/health-check` (délai initial 20 s / 40 s).
- Frontend : `/` (délai initial 10 s / 20 s).

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

`helm template` ne contacte aucun cluster — c'est un rendu purement local des manifests Kubernetes finaux, exactement ce qu'ArgoCD calcule en interne avant de comparer au cluster réel. C'est la vérification la plus rapide pour s'assurer qu'un changement de `values.yaml`/`values-staging.yaml` ou d'un template produit bien le YAML attendu, avant même de committer.

Si Helm n'est pas installé localement, le même rendu peut être obtenu via l'image officielle :
```bash
docker run --rm -v "$(pwd):/work" -w /work alpine/helm:3.14.0 \
  template hr-staging charts/hr-app -f charts/hr-app/values.yaml -f charts/hr-app/values-staging.yaml
```

## 🚀 Ingress (désactivé aujourd'hui)

`values.yaml` et `values-staging.yaml` mettent tous les deux `ingress.enabled: false` — l'application n'est donc accessible que via `kubectl port-forward svc/hr-frontend -n staging 8080:80`, jamais depuis l'extérieur du cluster. Pour l'activer, il faudrait :
1. Un ingress-nginx controller déployé sur le cluster (absent aujourd'hui).
2. Passer `ingress.enabled: true` dans `values-staging.yaml` (le `host` `hr-staging.example.com` est déjà pré-rempli).

Le template `ingress.yaml` route déjà `/` vers `hr-frontend:80` et `/api` vers `hr-backend:8081` — le routage est prêt, seule l'activation manque.

## 🔗 Pour la suite

- Comment ArgoCD applique ce chart automatiquement : [guide-argocd-gitops.md](guide-argocd-gitops.md)
- D'où viennent les tags d'image patchés dans `values-staging.yaml` : [lifecycle-pipeline.md](lifecycle-pipeline.md)
- Pourquoi Helm plutôt que le provider Terraform `kubernetes` pour déployer l'appli : [decisions-architecture.md](decisions-architecture.md)
