# Cycle de vie complet d'un changement

## 🎯 Objectif

Suivre un changement applicatif du `git push` initial jusqu'au pod qui tourne sur GKE, en nommant précisément chaque job CI, chaque fichier lu/écrit, et chaque outil utilisé à chaque flèche. Ce document est autonome — il ne suppose pas que tu aies lu [architecture-globale.md](architecture-globale.md), même s'il le complète.

---

## ⚙️ Schéma complet du flux

```
 DÉVELOPPEUR
     │ git push main
     ▼
┌─────────────────────────────────────────────────────────────────┐
│ repo-app/.github/workflows/ci.yml                                │
│ "CI - Build, Test and Push to GCP Artifact Registry"             │
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
│    3. aquasecurity/trivy-action@v0.36.0 (severity: CRITICAL,      │
│       ignore-unfixed: true, exit-code: 1) sur chaque image        │
│       → une CVE CRITICAL avec fix disponible bloque tout le job   │
│    4. docker push (les deux images)                               │
│    5. gcloud artifacts docker tags list --filter="tag:<SHA7>"    │
│       → vérifie que l'image existe VRAIMENT dans le registre      │
│    6. git clone repo-config (via secrets.GH_PAT)                  │
│    7. yq eval '.backend.image.tag = strenv(IMAGE_TAG) |           │
│                 .frontend.image.tag = strenv(IMAGE_TAG)'          │
│         -i charts/hr-app/values-staging.yaml                      │
│    8. git commit -m "ci: update image tags to <SHA7>"             │
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
│ charts/hr-app/values-staging.yaml modifié :                       │
│   backend.image.tag  = <SHA7>                                     │
│   frontend.image.tag = <SHA7>                                     │
└───────────────────────────┬───────────────────────────────────────┘
                            │
                            │ ArgoCD (dans le cluster) détecte le diff
                            │ Application "hr-staging" (apps/children/staging.yaml)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ ArgoCD — reconciliation loop                                      │
│  1. Lit charts/hr-app/ + values-staging.yaml depuis repo-config   │
│  2. helm template (rendu interne, équivalent à                    │
│     `helm template hr-staging charts/hr-app                       │
│        -f charts/hr-app/values.yaml                               │
│        -f charts/hr-app/values-staging.yaml`)                     │
│  3. Compare le rendu à l'état vivant du namespace `staging`        │
│  4. syncPolicy.automated : prune=true, selfHeal=true               │
│     syncOptions : CreateNamespace=true, ServerSideApply=true,      │
│                    RespectIgnoreDifferences=true                   │
│  5. Applique le diff (Deployment hr-backend / hr-frontend           │
│     mis à jour avec la nouvelle image)                              │
└───────────────────────────┬───────────────────────────────────────┘
                            ▼
                Cluster GKE (gke-staging-pfe, namespace staging)
     Pods hr-backend / hr-frontend redémarrés avec la nouvelle image
     readinessProbe / livenessProbe sur /api/health-check (backend)
                          et / (frontend)
```

## 🚀 Étapes détaillées

### 1. Push développeur → déclenchement CI

Un `git push` (ou une PR) sur la branche `main` de `repo-app` déclenche `repo-app/.github/workflows/ci.yml`. Aucun filtre de chemin (`paths:`) n'est défini — tout push sur `main` lance le workflow, quel que soit le fichier modifié.

### 2. `backend-ci` et `frontend-ci` (en parallèle)

- **`backend-ci`** (working-directory `backend`) : `actions/setup-java@v4` (Java 17, Temurin) → cache Maven (`~/.m2`) → **`mvn verify -q`** → upload de `backend/target/surefire-reports/` (artefact `backend-test-report`, conservé 7 jours, même en cas d'échec) → upload de `backend/target/*.jar` (artefact `backend-jar`, conservé 1 jour).
- **`frontend-ci`** (working-directory `frontend`) : `actions/setup-node@v4` (Node 20) → cache `node_modules` → **`npm install --legacy-peer-deps`** → **`npm run build`** (= `ng build --configuration production`) → upload de `frontend/dist/` (artefact `frontend-dist`, conservé 1 jour).

### 3. `docker-build-push` (seulement sur `push`, jamais sur PR)

Ce job a `needs: [backend-ci, frontend-ci]` et `if: github.event_name == 'push'` — il ne tourne jamais sur une pull request, seulement après un push réel sur `main`.

1. Télécharge les artefacts `backend-jar` et `frontend-dist` produits par les deux jobs précédents.
2. S'authentifie à GCP par **OIDC** (`google-github-actions/auth@v2`, `workload_identity_provider: vars.GCP_WORKLOAD_PROVIDER`, `service_account: vars.GCP_SERVICE_ACCOUNT`) — aucune clé JSON stockée.
3. Calcule le tag d'image : `image_tag=${GITHUB_SHA::7}` (7 premiers caractères du SHA du commit).
4. `docker build ./backend` et `docker build ./frontend` — les deux `Dockerfile` ne compilent rien : ils copient uniquement le JAR déjà construit (`COPY target/*.jar app.jar`, base `eclipse-temurin:17-jre-alpine`) et le dossier `dist/hr-frontend/browser` déjà construit (base `nginx:alpine`). C'est le principe *build once, promote always*.
5. Scan Trivy (`aquasecurity/trivy-action@v0.36.0`) sur chaque image : `severity: CRITICAL`, `ignore-unfixed: true`, `exit-code: 1` — toute CVE CRITICAL avec correctif disponible fait échouer le job et bloque le push.
6. `docker push` des deux images vers `${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT_ID}/${GAR_REPOSITORY}`.
7. Vérification post-push : `gcloud artifacts docker tags list ... --filter="tag:${IMAGE_TAG}"` sur chaque image, pour confirmer qu'elle existe bien dans Artifact Registry avant de toucher au GitOps (évite de déployer un tag fantôme).
8. Installation de `yq` (binaire téléchargé depuis GitHub releases, v4.44.1).
9. `git clone --depth 1` de `repo-config` avec le PAT `secrets.GH_PAT` (identité `github-actions[bot]`).
10. Patch de `charts/hr-app/values-staging.yaml` :
    ```
    yq eval '.backend.image.tag = strenv(IMAGE_TAG) | .frontend.image.tag = strenv(IMAGE_TAG)' -i charts/hr-app/values-staging.yaml
    ```
11. `git commit -m "ci: update image tags to <SHA7>"` puis `git push origin HEAD:main` vers `repo-config`. Si le diff est vide (`git diff --cached --quiet`), le job s'arrête proprement sans commit.

### 4. ArgoCD détecte et synchronise

`repo-config` n'a aucun workflow GitHub Actions — c'est **ArgoCD**, qui tourne dans le cluster GKE, qui poll ce dépôt. L'Application `hr-staging` (définie dans `apps/children/staging.yaml`, elle-même détectée par l'App-of-Apps `root-app`) pointe sur `source.path: charts/hr-app` avec `helm.valueFiles: [values-staging.yaml]`. Dès que le nouveau commit est visible, ArgoCD :

1. Récupère le chart et les values depuis `repo-config`.
2. Effectue l'équivalent d'un `helm template` interne pour obtenir les manifests Kubernetes finaux.
3. Compare ce rendu à l'état réel du namespace `staging`.
4. Comme `syncPolicy.automated.prune=true` et `selfHeal=true`, applique automatiquement le diff — pas d'intervention humaine. `ServerSideApply=true` évite les conflits de field-ownership, `RespectIgnoreDifferences=true` fait respecter le bloc `ignoreDifferences` sur `/status/terminatingReplicas` des Deployments.

### 5. Rollout sur GKE

Les Deployments `hr-backend` et `hr-frontend` (namespace `staging`) reçoivent la nouvelle image. Le pod backend attend que `readinessProbe` (`GET /api/health-check`, délai initial 20 s) passe avant de recevoir du trafic ; `livenessProbe` (même endpoint, délai initial 40 s) redémarre le conteneur s'il ne répond plus. Le frontend a les mêmes probes sur `/`.

## ✅ Vérifier que le flux a bien fonctionné

```bash
# 1. Le tag a été patché dans repo-config
git -C repo-config log --oneline -5
# doit montrer "ci: update image tags to <SHA7>"

# 2. ArgoCD a synchronisé
kubectl get application hr-staging -n argocd \
  -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
# attendu : Synced Healthy

# 3. Les pods tournent avec la bonne image
kubectl get deployment hr-backend -n staging \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get pods -n staging
```

## 🔗 Pour la suite

- Vue d'ensemble des trois dépôts : [architecture-globale.md](architecture-globale.md)
- Détail du fonctionnement ArgoCD / App-of-Apps : [guide-argocd-gitops.md](guide-argocd-gitops.md)
- Détail du chart Helm et de ses values : [guide-helm-chart.md](guide-helm-chart.md)
