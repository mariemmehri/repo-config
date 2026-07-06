# Architecture globale du projet

## 🎯 Objectif

Expliquer pourquoi ce projet est découpé en **trois dépôts Git séparés** (`repo-app`, `repo-infrastructure`, `repo-config`), ce qui appartient à chacun, et pourquoi un changement dans l'un ne déclenche jamais le pipeline d'un autre.

Ce document est autonome : il ne suppose pas que tu aies lu les autres fichiers de `docs/`. Pour le détail de chaque étape, voir [lifecycle-pipeline.md](lifecycle-pipeline.md), [guide-deploiement-infra.md](guide-deploiement-infra.md), [guide-argocd-gitops.md](guide-argocd-gitops.md) et [guide-helm-chart.md](guide-helm-chart.md).

---

## ⚙️ Les trois dépôts

```
┌─────────────────────────┐   ┌──────────────────────────────┐   ┌─────────────────────────┐
│       repo-app           │   │     repo-infrastructure      │   │      repo-config        │
│  (mariemmehri/repo-App)  │   │ (mariemmehri/repo-infra...)  │   │ (mariemmehri/repo-config)│
├───────────────────────────┤   ├──────────────────────────────┤   ├─────────────────────────┤
│ backend/  (Spring Boot)  │   │ modules/                     │   │ apps/                    │
│ frontend/ (Angular)      │   │   networking/                │   │   root-app.yaml           │
│ docker-compose.yml       │   │   gke/                       │   │   children/staging.yaml   │
│ .github/workflows/ci.yml │   │   iam/                       │   │ charts/hr-app/             │
│                           │   │   artifact_registry/         │   │   templates/*.yaml         │
│                           │   │ environments/staging/        │   │   values.yaml               │
│                           │   │ backend-config/               │   │   values-staging.yaml       │
│                           │   │ .github/workflows/            │   │                             │
│                           │   │   workflow-infra.yml          │   │                             │
└───────────┬───────────────┘   └───────────────┬──────────────┘   └────────────┬────────────────┘
            │                                   │                               │
            │  build + push image               │  provisionne le cluster GKE   │  ArgoCD lit ici
            │  patch values-staging.yaml ───────────────────────────────────────▶
            │  (git push direct, via GH_PAT)    │  + bootstrap ArgoCD          │
            │                                   │  (kubectl apply              │
            │                                   │   apps/root-app.yaml) ───────▶
            ▼                                   ▼                               ▼
   Google Artifact Registry            Cluster GKE (staging)           ArgoCD surveille repo-config
   (images hr-backend/hr-frontend)     + namespace argocd installé     et applique les Helm releases
```

Chaque dossier local (`repo-app/`, `repo-infrastructure/`, `repo-config/`) est **un `.git` séparé**, avec sa propre origine GitHub, son propre historique de commits, et son propre dossier `.github/workflows/`. Le fait qu'ils vivent dans le même dossier parent sur disque (`stage pfe/`) est une commodité de développement — sur GitHub, ce sont trois dépôts indépendants qui ne se connaissent que par convention (URLs, noms de variables partagés) et jamais par un lien technique direct (pas de submodule Git, pas de monorepo tooling).

## 🚀 Ce qui appartient à chaque dépôt

### `repo-app` — le code applicatif

- Backend Spring Boot 3.2.0 / Java 17 (package `com.example.hr`) : une mini application RH (SIRH) sans base de données, données en mémoire.
- Frontend Angular 17.
- `docker-compose.yml` pour lancer les deux en local.
- Son propre CI (`.github/workflows/ci.yml`) : compile, teste, construit les images Docker, les scanne avec Trivy, les pousse dans Artifact Registry, puis **patch directement** `values-staging.yaml` dans `repo-config`.

### `repo-infrastructure` — l'infrastructure GCP

- Modules Terraform (`modules/networking`, `modules/gke`, `modules/iam`, `modules/artifact_registry`) qui créent le VPC, le cluster GKE, les comptes de service et le registre d'images.
- `environments/staging/` — le seul environnement Terraform actif aujourd'hui, qui assemble les quatre modules.
- `backend-config/` — un projet Terraform à état **local**, qui bootstrape une seule fois le bucket GCS de state distant et la Workload Identity Federation (WIF).
- Son propre workflow (`workflow-infra.yml`) qui fait `terraform plan`/`apply` **et** installe/bootstrape ArgoCD sur le cluster.

### `repo-config` — la source de vérité GitOps

- `charts/hr-app/` — le chart Helm unique qui décrit les Deployments/Services/Ingress de l'application.
- `values.yaml` (valeurs par défaut du chart) et `values-staging.yaml` (overrides pour l'environnement staging, dont les tags d'image patchés automatiquement par le CI de `repo-app`).
- `apps/root-app.yaml` et `apps/children/staging.yaml` — les manifests ArgoCD `Application` (App-of-Apps).
- Aucun code applicatif, aucun Terraform : uniquement des manifests déclaratifs que ArgoCD lit et applique.

## ✅ Pourquoi un changement applicatif ne déclenche jamais Terraform

Les trois dépôts sont séparés **au niveau CI** parce que chaque `.github/workflows/*.yml` n'écoute que les événements Git de son propre dépôt :

- Un `git push` sur `repo-app` déclenche uniquement `repo-app/.github/workflows/ci.yml`. Ce workflow ne clone jamais `repo-infrastructure`, n'a pas les permissions IAM pour toucher à GCP au-delà de `artifactregistry.writer`/`container.developer`/`storage.objectViewer` (rôles du compte de service `sa-github-actions`), et se contente de faire un `git push` HTTPS vers `repo-config` pour patcher un fichier YAML.
- Un `git push` sur `repo-infrastructure` déclenche uniquement `workflow-infra.yml`, gardé par des chemins déclenchants précis (`environments/**`, `modules/**`, `backend-config/**`, le fichier workflow lui-même, `.checkov.yaml`). Ce workflow ne connaît rien du code applicatif.
- Un commit dans `repo-config` (que ce soit le patch automatique du CI de `repo-app`, ou une modification manuelle du chart Helm) ne déclenche **aucun workflow GitHub Actions** — `repo-config` n'a pas de dossier `.github/workflows/`. Le seul consommateur de ces commits est **ArgoCD**, qui poll le dépôt en continu depuis l'intérieur du cluster GKE.

Concrètement, il n'existe **aucun mécanisme technique** (webhook, `workflow_call`, déclencheur croisé) qui relierait les trois pipelines entre eux. La seule passerelle entre `repo-app` et `repo-config` est le `git push` explicite fait par le job `docker-build-push` (avec le secret `GH_PAT`) ; la seule passerelle entre `repo-infrastructure` et `repo-config` est la lecture faite une fois par le job `bootstrap-argocd` (`kubectl apply -f apps/root-app.yaml` après un `checkout` de `repo-config`). Après ce bootstrap initial, `repo-infrastructure` ne retouche plus jamais `repo-config` — c'est ArgoCD, tournant dans le cluster, qui prend le relais et surveille `repo-config` en continu.

Cette séparation garantit qu'un bug dans le code applicatif ne peut jamais, même par erreur, relancer un `terraform apply` sur l'infrastructure GCP — et inversement, qu'une modification de l'infrastructure ne redéploie jamais l'application par accident.

## 🔗 Pour la suite

- Le détail complet du flux, fichier par fichier et job par job : [lifecycle-pipeline.md](lifecycle-pipeline.md)
- Comment provisionner l'infra depuis zéro : [guide-deploiement-infra.md](guide-deploiement-infra.md)
- Comment fonctionne ArgoCD ici : [guide-argocd-gitops.md](guide-argocd-gitops.md)
- Comment fonctionne le chart Helm : [guide-helm-chart.md](guide-helm-chart.md)
