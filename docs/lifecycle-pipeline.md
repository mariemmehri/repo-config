# Cycle de vie complet d'un changement

## 🎯 Objectif

Suivre un changement applicatif du `git push` initial jusqu'au pod qui tourne sur GKE, en nommant précisément chaque job CI, chaque fichier lu/écrit, et chaque outil utilisé à chaque flèche. Deux flux existent : le flux automatique dev/staging, et le flux de promotion prod (manuel, en deux temps). Ce document est autonome — il ne suppose pas que tu aies lu [architecture-globale.md](architecture-globale.md), même s'il le complète.

---

## ⚙️ Flux 1 — dev / staging (entièrement automatisé)

```
 DÉVELOPPEUR
     │ git push develop   (ou main)
     ▼
┌─────────────────────────────────────────────────────────────────┐
│ repo-app/.github/workflows/ci.yml                                │
│                                                                   │
│  Job 1: backend-ci ─────────┐   Job 2: frontend-ci ─────────┐    │
│  mvn verify -q              │   npm install --legacy-peer-  │    │
│  → artefact backend-jar     │     deps && npm run build     │    │
│                              │   → artefact frontend-dist    │    │
│  └─────────────┬─────────────┴───────────────┬───────────────┘    │
│                │ needs: [backend-ci, frontend-ci]                │
│                │ if: github.event_name == 'push'                 │
│                ▼                                                 │
│  Job 3: docker-build-push                                        │
│    1. auth GCP OIDC (google-github-actions/auth@v2,               │
│       vars.GCP_WORKLOAD_PROVIDER / vars.GCP_SERVICE_ACCOUNT)      │
│    2. docker build ./backend  -t <REGISTRY>/hr-backend:<SHA7>     │
│       docker build ./frontend -t <REGISTRY>/hr-frontend:<SHA7>    │
│       (SHA7 = ${GITHUB_SHA::7})                                   │
│    3. aquasecurity/trivy-action (severity: CRITICAL,               │
│       ignore-unfixed: true, exit-code: 1) sur chaque image        │
│       → une CVE CRITICAL avec fix disponible bloque tout le job   │
│    4. docker push (les deux images)                               │
│    5. gcloud artifacts docker tags list --filter="tag:<SHA7>"    │
│       → vérifie que l'image existe VRAIMENT dans le registre      │
│    6. génère un token GitHub App de courte durée (~1h,             │
│       scope repo-config uniquement — remplace l'ancien GH_PAT)    │
│    7. git clone repo-config avec ce token                          │
│    8. yq eval '.backend.image.tag = strenv(IMAGE_TAG) |           │
│                 .frontend.image.tag = strenv(IMAGE_TAG)'          │
│         -i charts/hr-app/values-<env>.yaml                        │
│       (<env> = dev si develop, staging si main)                   │
│    9. git commit -m "ci: update <env> image tags to <SHA7>"       │
│       git push origin HEAD:main   (vers repo-config)              │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
                Google Artifact Registry
     europe-west1-docker.pkg.dev/pfe-2026-495220/registry-staging-pfe/
                 hr-backend:<SHA7>  et  hr-frontend:<SHA7>
                            │
                            │ (commit poussé sur repo-config, main)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ repo-config  (aucun workflow GitHub Actions ici)                  │
│ charts/hr-app/values-<env>.yaml modifié :                         │
│   backend.image.tag  = <SHA7>                                     │
│   frontend.image.tag = <SHA7>                                     │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                            │ ArgoCD (dans le cluster) détecte le diff
                            │ Application "hr-<env>" (apps/children/<env>.yaml)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ ArgoCD — reconciliation loop                                      │
│  1. Lit charts/hr-app/ + values-<env>.yaml depuis repo-config     │
│  2. helm template (rendu interne)                                  │
│  3. Compare le rendu à l'état vivant du namespace <env>             │
│  4. syncPolicy.automated : prune=true, selfHeal=true (dev/staging   │
│     uniquement — voir Flux 2 pour prod)                             │
│  5. Applique le diff (Deployment hr-backend / hr-frontend           │
│     mis à jour avec la nouvelle image)                              │
└───────────────────────────┬───────────────────────────────────────┘
                            ▼
                Cluster GKE (gke-staging-pfe, namespace dev ou staging)
     Pods hr-backend / hr-frontend redémarrés avec la nouvelle image
```

## 🚀 Étapes détaillées — Flux 1

### 1. Push développeur → déclenchement CI

Un `git push` (ou une PR) sur `develop` ou `main` de `repo-app` déclenche `repo-app/.github/workflows/ci.yml`.

### 2. `backend-ci` et `frontend-ci` (en parallèle)

- **`backend-ci`** (working-directory `backend`) : `actions/setup-java@v4` (Java 17, Temurin) → cache Maven (`~/.m2`) → **`mvn verify -q`** → upload de `backend/target/surefire-reports/` et `backend/target/*.jar`.
- **`frontend-ci`** (working-directory `frontend`) : `actions/setup-node@v4` (Node 20) → cache `node_modules` → **`npm install --legacy-peer-deps`** → **`npm run build`** (= `ng build --configuration production`) → upload de `frontend/dist/`.

### 3. `docker-build-push` (seulement sur `push`, jamais sur PR)

1. Télécharge les artefacts `backend-jar` et `frontend-dist`.
2. S'authentifie à GCP par **OIDC** — aucune clé JSON stockée.
3. Calcule le tag d'image : `image_tag=${GITHUB_SHA::7}`.
4. `docker build` — les deux `Dockerfile` ne compilent rien (`COPY target/*.jar app.jar`, `COPY dist/hr-frontend/browser`) : *build once, promote always*.
5. Scan Trivy : `severity: CRITICAL`, `ignore-unfixed: true`, `exit-code: 1`.
6. `docker push` des deux images.
7. Vérification post-push dans Artifact Registry avant de toucher au GitOps.
8. Mappe la branche vers un environnement : `develop` → `values-dev.yaml`, `main` → `values-staging.yaml` ; toute autre ref fait échouer le job.
9. Génère un **token GitHub App** de courte durée (`actions/create-github-app-token`, scope `repo-config` uniquement, expire en ~1h) — remplace l'ancien mécanisme à base de `GH_PAT` longue durée.
10. `git clone --depth 1` de `repo-config` avec ce token.
11. Patch de `charts/hr-app/values-<env>.yaml` via `yq`.
12. `git commit -m "ci: update <env> image tags to <SHA7>"` puis push vers `repo-config`. Si le diff est vide, le job s'arrête proprement sans commit.

### 4. ArgoCD détecte et synchronise

L'Application `hr-<env>` (définie dans `apps/children/<env>.yaml`) pointe sur `source.path: charts/hr-app` avec `helm.valueFiles: [values-<env>.yaml]`. `dev` et `staging` ont `syncPolicy.automated.prune=true`/`selfHeal=true` — la synchronisation est automatique, sans intervention humaine.

### 5. Rollout sur GKE

Les Deployments `hr-backend` et `hr-frontend` (namespace `dev` ou `staging`) reçoivent la nouvelle image, sous couvert de leurs `readinessProbe`/`livenessProbe` sur `/api/health-check` (backend) et `/` (frontend).

## ✅ Vérifier que le flux 1 a bien fonctionné

```bash
git -C repo-config log --oneline -5    # doit montrer "ci: update <env> image tags to <SHA7>"

kubectl get application hr-staging -n argocd \
  -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'   # attendu : Synced Healthy

kubectl get deployment hr-backend -n staging \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get pods -n staging
```
(remplacer `staging`/`hr-staging` par `dev`/`hr-dev` pour l'autre flux automatisé)

---

## ⚙️ Flux 2 — promotion prod (manuel, en deux temps)

```
 DÉVELOPPEUR / RELEASE MANAGER
     │ git tag v1.2.0 && git push origin v1.2.0   (le commit taggé doit déjà avoir tourné en CI sur main)
     ▼
┌─────────────────────────────────────────────────────────────────┐
│ repo-app/.github/workflows/promote-prod.yml                       │
│                                                                   │
│  Job promote — gate GitHub Environment "production"                │
│  (approbation humaine #1, requise avant que le job ne démarre)     │
│                                                                   │
│    1. Vérifie que les images du SHA taggé existent déjà dans      │
│       registry-staging-pfe — si non : échec explicite             │
│       ("tag a commit that went through CI"), jamais de rebuild    │
│    2. crane copy hr-backend / hr-frontend                          │
│       registry-staging-pfe → registry-prod-pfe (registre isolé)    │
│       digest préservé, re-tag v1.2.0                                │
│    3. git clone repo-config (token GitHub App, même mécanisme      │
│       que le Flux 1)                                                │
│    4. yq patch charts/hr-app/values-prod.yaml                      │
│       .backend.image.tag = .frontend.image.tag = "v1.2.0"          │
│    5. git commit -m "ci: promote v1.2.0 to prod"                   │
│       git push origin HEAD:main   (vers repo-config)                │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
                Google Artifact Registry — registry-prod-pfe
     hr-backend:v1.2.0 / hr-frontend:v1.2.0 (octets identiques à staging)
                            │
                            ▼
              repo-config : values-prod.yaml modifié
                            │
                            │ hr-prod n'a PAS de syncPolicy.automated
                            │ → reste OutOfSync, ArgoCD n'applique rien seul
                            ▼
              DÉPLOIEMENT RÉEL = approbation humaine #2
              argocd app sync hr-prod   (ou via l'UI ArgoCD)
                            │
                            ▼
                Cluster GKE, namespace prod
     Pods hr-backend (2 replicas, HPA 2-5) / hr-frontend (2 replicas)
     redémarrés avec l'image v1.2.0
```

### Points clés du flux 2

- **Jamais de rebuild en prod** — `promote-prod.yml` échoue explicitement si le SHA taggé n'a pas d'image correspondante dans `registry-staging-pfe`. Prod exécute l'octet-pour-octet identique de ce qui a tourné en staging.
- **`crane copy` préserve le digest** — la promotion est une copie de manifeste/layers entre registres, pas un nouveau build Docker.
- **Deux gates humains distincts, volontairement découplés** : l'environnement GitHub `production` gate la *promotion* (le commit sur `values-prod.yaml`) ; `argocd app sync hr-prod` gate le *déploiement* effectif. On peut promouvoir un vendredi et déployer un lundi.
- **`hr-prod` n'a pas de `selfHeal`** — contrairement à `dev`/`staging`, un hand-edit sur le namespace `prod` n'est pas automatiquement annulé (il n'y a simplement personne qui surveille en continu), ce qui n'en fait pas pour autant une pratique recommandée.

### ✅ Vérifier que le flux 2 a bien fonctionné

```bash
git -C repo-config log --oneline -3    # doit montrer "ci: promote v1.2.0 to prod"

kubectl get application hr-prod -n argocd \
  -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
# OutOfSync juste après la promotion est normal — attendu jusqu'au sync manuel

argocd app sync hr-prod
kubectl get deployment hr-backend -n prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'   # doit se terminer par :v1.2.0
```

## 🔗 Pour la suite

- Vue d'ensemble des trois dépôts : [architecture-globale.md](architecture-globale.md)
- Détail du fonctionnement ArgoCD / App-of-Apps à trois environnements : [guide-argocd-gitops.md](guide-argocd-gitops.md)
- Détail du chart Helm et de ses values : [guide-helm-chart.md](guide-helm-chart.md)
