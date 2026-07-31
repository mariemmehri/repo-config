# Architecture globale du projet

## 🎯 Objectif

Expliquer pourquoi ce projet est découpé en **trois dépôts Git séparés** (`repo-app`, `repo-infrastructure`, `repo-config`), ce qui appartient à chacun, et pourquoi un changement dans l'un ne déclenche jamais le pipeline d'un autre.

Ce document est autonome : il ne suppose pas que tu aies lu les autres fichiers de `docs/`. Pour le détail de chaque étape, voir [lifecycle-pipeline.md](lifecycle-pipeline.md), [guide-deploiement-infra.md](guide-deploiement-infra.md), [guide-argocd-gitops.md](guide-argocd-gitops.md) et [guide-helm-chart.md](guide-helm-chart.md).

---

## ⚙️ Les trois dépôts

```
┌─────────────────────────┐   ┌──────────────────────────────┐   ┌─────────────────────────┐
│       repo-app           │   │     repo-infrastructure      │   │      repo-config        │
│  (mariemmehri/repo-app)  │   │ (mariemmehri/repo-infra...)  │   │ (mariemmehri/repo-config)│
├───────────────────────────┤   ├──────────────────────────────┤   ├─────────────────────────┤
│ backend/  (Spring Boot)  │   │ modules/                     │   │ apps/                    │
│ frontend/ (Angular)      │   │   networking/                │   │   root-app.yaml           │
│ docker-compose.yml       │   │   gke/                       │   │   children/{dev,staging,  │
│ .github/workflows/       │   │   iam/                       │   │     prod}.yaml + cnpg-*.yaml│
│   ci.yml                 │   │   artifact_registry/ (x2)    │   │ charts/hr-app/             │
│   promote-prod.yml       │   │   cnpg_backup/ (x3)          │   │   templates/*.yaml         │
│                           │   │ environments/staging/        │   │   values.yaml               │
│                           │   │ backend-config/               │   │   values-{dev,staging,prod}.yaml│
│                           │   │ .github/workflows/            │   │                             │
│                           │   │   workflow-infra.yml          │   │                             │
└───────────┬───────────────┘   └───────────────┬──────────────┘   └────────────┬────────────────┘
            │                                   │                               │
            │  build + push image               │  provisionne le cluster GKE   │  ArgoCD lit ici
            │  patch values-<env>.yaml ─────────────────────────────────────────▶
            │  (git push via token GitHub App)   │  + bootstrap ArgoCD          │
            │                                   │  (kubectl apply              │
            │                                   │   apps/root-app.yaml) ───────▶
            ▼                                   ▼                               ▼
   Google Artifact Registry            Cluster GKE (un seul cluster,     ArgoCD surveille repo-config
   (staging/dev + registre prod        3 namespaces dev/staging/prod)    et applique les Helm releases
    isolé, écrit par crane copy)       + namespace argocd installé       par environnement
```

Chaque dossier local (`repo-app/`, `repo-infrastructure/`, `repo-config/`) est **un `.git` séparé**, avec sa propre origine GitHub, son propre historique de commits, et son propre dossier `.github/workflows/` (sauf `repo-config`, qui n'en a aucun). Le fait qu'ils vivent dans le même dossier parent sur disque (`stage pfe/`) est une commodité de développement — sur GitHub, ce sont trois dépôts indépendants qui ne se connaissent que par convention (URLs, noms de variables partagés) et jamais par un lien technique direct (pas de submodule Git, pas de monorepo tooling).

## 🚀 Ce qui appartient à chaque dépôt

### `repo-app` — le code applicatif

- Backend Spring Boot 3.2.0 / Java 17 (package `com.example.hr`) : mini application RH (SIRH) avec un vrai domaine métier — `Employee` et `LeaveRequest` sont des entités JPA persistées dans **PostgreSQL** (pas de données en mémoire). Le domaine « bulletins de paie » n'existe côté backend que dans la documentation historique — aucun contrôleur `Payslip` n'existe, ces appels frontend échouent en 404.
- Frontend Angular 17.
- `docker-compose.yml` pour lancer les deux en local avec une Postgres locale.
- Deux workflows : `ci.yml` (push `develop`/`main` → build, test, scan Trivy, push image, patch `repo-config`) et `promote-prod.yml` (tag `v*.*.*` → copie l'image déjà construite vers un registre prod isolé, jamais de rebuild).

### `repo-infrastructure` — l'infrastructure GCP

- Modules Terraform (`modules/networking`, `modules/gke`, `modules/iam`, `modules/artifact_registry`, `modules/cnpg_backup`) qui créent le VPC, le cluster GKE, les comptes de service, les registres d'images et les buckets de backup Postgres. `artifact_registry` et `cnpg_backup` sont chacun instanciés plusieurs fois (2x et 3x respectivement) pour couvrir staging+prod et dev+staging+prod.
- `environments/staging/` — le seul environnement Terraform actif, qui assemble 8 blocs de module. Ce nom ("staging") désigne la racine Terraform, pas les environnements applicatifs qui tournent dessus — le même cluster héberge aussi les namespaces `dev` et `prod`.
- `backend-config/` — un projet Terraform à état **local**, qui bootstrape une seule fois le bucket GCS de state distant et la Workload Identity Federation (WIF).
- Son propre workflow (`workflow-infra.yml`) qui fait `terraform plan`/`apply` **et** installe/bootstrape ArgoCD sur le cluster.

### `repo-config` — la source de vérité GitOps

- `charts/hr-app/` — le chart Helm unique qui décrit les Deployments/Services/Ingress/NetworkPolicy/HPA de l'application, paramétré par environnement via `values-{dev,staging,prod}.yaml`.
- `apps/root-app.yaml` et `apps/children/{dev,staging,prod}.yaml` — les manifests ArgoCD `Application` de l'app (App-of-Apps). `apps/children/` contient aussi les Applications CNPG (`cert-manager`, `cnpg-operator`, `cnpg-plugin-barman-cloud`, `cnpg-cluster-<env>`, `cnpg-network-policy-<env>`) qui déploient PostgreSQL.
- Aucun code applicatif, aucun Terraform : uniquement des manifests déclaratifs que ArgoCD lit et applique.

## ✅ Pourquoi un changement applicatif ne déclenche jamais Terraform

Les trois dépôts sont séparés **au niveau CI** parce que chaque `.github/workflows/*.yml` n'écoute que les événements Git de son propre dépôt :

- Un `git push` sur `repo-app` déclenche uniquement `repo-app/.github/workflows/{ci,promote-prod}.yml`. Ces workflows ne clonent jamais `repo-infrastructure`, n'ont pas les permissions IAM pour toucher à GCP au-delà de `artifactregistry.writer`/`container.developer`/`storage.objectViewer` (rôles du compte de service `sa-github-actions`), et se contentent d'un `git push` HTTPS (via un token GitHub App de courte durée, scope limité à `repo-config`) pour patcher un fichier `values-<env>.yaml`.
- Un `git push` sur `repo-infrastructure` déclenche uniquement `workflow-infra.yml`, gardé par des chemins déclenchants précis (`environments/**`, `modules/**`, `backend-config/**`, le fichier workflow lui-même, `.checkov.yaml`). Ce workflow ne connaît rien du code applicatif.
- Un commit dans `repo-config` (patch automatique de la CI, promotion prod, ou modification manuelle du chart Helm) ne déclenche **aucun workflow GitHub Actions** — `repo-config` n'a pas de dossier `.github/workflows/`. Le seul consommateur de ces commits est **ArgoCD**, qui poll le dépôt en continu depuis l'intérieur du cluster GKE.

Concrètement, il n'existe **aucun mécanisme technique** (webhook, `workflow_call`, déclencheur croisé) qui relierait les trois pipelines entre eux. La seule passerelle entre `repo-app` et `repo-config` est le `git push` explicite fait par `docker-build-push`/`promote-prod.yml` ; la seule passerelle entre `repo-infrastructure` et `repo-config` est la lecture faite une fois par le job `bootstrap-argocd` (`kubectl apply -f apps/root-app.yaml` après un `checkout` de `repo-config`). Après ce bootstrap initial, `repo-infrastructure` ne retouche plus jamais `repo-config` — c'est ArgoCD, tournant dans le cluster, qui prend le relais et surveille `repo-config` en continu.

Cette séparation garantit qu'un bug dans le code applicatif ne peut jamais, même par erreur, relancer un `terraform apply` sur l'infrastructure GCP — et inversement, qu'une modification de l'infrastructure ne redéploie jamais l'application par accident.

## 🔗 Pour la suite

- Le détail complet du flux, fichier par fichier et job par job (dev/staging automatisé + promotion prod) : [lifecycle-pipeline.md](lifecycle-pipeline.md)
- Comment provisionner l'infra depuis zéro : [guide-deploiement-infra.md](guide-deploiement-infra.md)
- Comment fonctionne ArgoCD ici, App-of-Apps à trois environnements : [guide-argocd-gitops.md](guide-argocd-gitops.md)
- Comment fonctionne le chart Helm, y compris l'Ingress GCE natif : [guide-helm-chart.md](guide-helm-chart.md)
