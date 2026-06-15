# todo-config — Source de vérité GitOps

Ce dépôt est la **source de vérité GitOps** de la plateforme. ArgoCD surveille ce dépôt en continu et réconcilie l'état du cluster Kubernetes vers l'état déclaré ici. Aucune modification directe via `kubectl` ou `helm` ne doit être faite — tout passe par Git.

---

## Contenu

```
todo-config/
├── apps/
│   ├── root-app.yaml              # (commenté — géré par Terraform bootstrap)
│   └── children/
│       └── staging.yaml           # Application ArgoCD pour l'environnement staging
│
└── charts/todo-app/               # Helm chart de l'application Todo
    ├── Chart.yaml                 # Métadonnées du chart (name, version, appVersion)
    ├── values.yaml                # Valeurs par défaut (non spécifiques à un env)
    ├── values-staging.yaml        # Overrides staging (registry, tags SHA, replicas)
    └── templates/
        ├── deployment-backend.yaml
        ├── deployment-frontend.yaml
        ├── service-backend.yaml
        ├── service-frontend.yaml
        └── ingress.yaml           # Conditionnel (enabled: false par défaut)
```

---

## Comment fonctionne le GitOps

### Flux de mise à jour

```
CI Pipeline (todo-app)
       │  yq patch values-staging.yaml
       │  git commit "ci: update image tags to <SHA>"
       │  git push → ce dépôt
       ▼
ArgoCD détecte le changement (polling toutes les 3 min ou webhook)
       │
       ▼
ArgoCD calcule la diff entre état Git et état cluster
       │
       ▼
helm upgrade todo-staging (avec values-staging.yaml)
       │
       ▼
Nouveaux pods déployés avec la nouvelle image
```

### App-of-Apps

Le `root-app` (créé par Terraform bootstrap) surveille le dossier `apps/children/`. Chaque fichier YAML dans ce dossier devient une `Application` ArgoCD gérée automatiquement.

Pour ajouter un nouvel environnement : créer `apps/children/production.yaml` et committer — ArgoCD le détecte et crée automatiquement la nouvelle application.

---

## Manifests ArgoCD

### `apps/children/staging.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mariemmehri/repo-config
    targetRevision: main
    path: charts/todo-app
    helm:
      valueFiles:
        - values-staging.yaml    # surcharge les valeurs par défaut
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      prune: true       # supprime les ressources absentes de Git
      selfHeal: true    # reverts les changements manuels sur le cluster
    syncOptions:
      - CreateNamespace=true     # crée le namespace staging si absent
      - ServerSideApply=true     # évite les conflits de merge K8s
      - RespectIgnoreDifferences=true
```

**Points importants :**
- `prune: true` — si une ressource est supprimée de Git, elle est supprimée du cluster
- `selfHeal: true` — si quelqu'un modifie manuellement un pod, ArgoCD rétablit l'état Git
- `ServerSideApply` — recommandé pour éviter les conflits de field ownership

### `apps/root-app.yaml`

Ce fichier est commenté — le root-app est géré par Terraform (`bootstrap-gitops/main.tf`). Il sert de documentation de référence pour la structure de l'objet.

---

## Helm Chart — `charts/todo-app/`

### `Chart.yaml`

```yaml
apiVersion: v2
name: todo-app
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### Structure des values

**`values.yaml`** — valeurs par défaut (non déployables sans override) :

```yaml
registry:
  host: ""           # à surcharger par environnement
  repository: ""

backend:
  image:
    name: todo-backend
    tag: "latest"
  replicas: 1
  port: 8081

frontend:
  image:
    name: todo-frontend
    tag: "latest"
  replicas: 1
  port: 80

ingress:
  enabled: false
  host: ""
```

**`values-staging.yaml`** — overrides staging (mis à jour par le CI) :

```yaml
registry:
  host: europe-west1-docker.pkg.dev
  repository: pfe-2026-495220/registry-staging-pfe

backend:
  image:
    name: todo-backend
    tag: "0c933e4"    # ← mis à jour automatiquement par yq dans le CI

frontend:
  image:
    name: todo-frontend
    tag: "0c933e4"    # ← idem

ingress:
  enabled: false
  host: todo-staging.example.com
```

### Construction de l'URL d'image

Le template `deployment-backend.yaml` construit l'URL complète :

```yaml
image: {{ .Values.registry.host }}/{{ .Values.registry.repository }}/{{ .Values.backend.image.name }}:{{ .Values.backend.image.tag }}
# → europe-west1-docker.pkg.dev/pfe-2026-495220/registry-staging-pfe/todo-backend:0c933e4
```

### Templates Kubernetes

| Template | Ressource créée | Détail |
|----------|-----------------|--------|
| `deployment-backend.yaml` | Deployment `todo-backend` | `SPRING_PROFILES_ACTIVE=prod` |
| `deployment-frontend.yaml` | Deployment `todo-frontend` | Image Nginx |
| `service-backend.yaml` | Service ClusterIP | port 8081, sélecteur `app: todo-backend` |
| `service-frontend.yaml` | Service ClusterIP | port 80, sélecteur `app: todo-frontend` |
| `ingress.yaml` | Ingress (conditionnel) | Routes `/` → frontend, `/api` → backend |

### Ingress

L'Ingress est désactivé (`ingress.enabled: false`) dans `values-staging.yaml`. Pour l'activer :

```yaml
# values-staging.yaml
ingress:
  enabled: true
  host: todo-staging.example.com
```

Nécessite un Ingress Controller déployé sur le cluster (ex: `ingress-nginx`).

---

## Utilisation locale

### Vérifier le chart

```bash
helm lint charts/todo-app
```

### Rendu des manifests (dry-run)

```bash
# Staging
helm template todo-staging charts/todo-app \
  -f charts/todo-app/values.yaml \
  -f charts/todo-app/values-staging.yaml

# Avec namespace
helm template todo-staging charts/todo-app \
  -f charts/todo-app/values-staging.yaml \
  --namespace staging
```

### Installation locale (test)

```bash
# Créer le namespace
kubectl create namespace staging

# Installer
helm upgrade --install todo-staging charts/todo-app \
  -f charts/todo-app/values-staging.yaml \
  --namespace staging

# Vérifier
kubectl get pods -n staging
kubectl get svc -n staging
```

### Désinstaller

```bash
helm uninstall todo-staging --namespace staging
```

---

## Mise à jour des tags d'image

Le CI (GitHub Actions dans `todo-app`) met à jour les tags automatiquement via `yq` :

```bash
yq eval '
  .backend.image.tag = strenv(IMAGE_TAG) |
  .frontend.image.tag = strenv(IMAGE_TAG)
' -i charts/todo-app/values-staging.yaml
```

**Ne jamais modifier les tags manuellement** — ils sont gérés par le pipeline et tout commit manuel sera écrasé au prochain push applicatif.

---

## Limitations actuelles

- **Ingress désactivé** — l'application n'est pas accessible de l'extérieur du cluster sans `kubectl port-forward`.
- **Pas de resource limits/requests** dans les templates — à ajouter pour la stabilité en production.
- **Pas de liveness/readiness probes** — Kubernetes ne sait pas si les pods sont vraiment opérationnels.
- **GITOPS_PFE.md** contient des références à Azure (AKS, ACR) — reliquat d'une version antérieure, à mettre à jour.
