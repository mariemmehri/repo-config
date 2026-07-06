# Décisions d'architecture (ADR)

## 🎯 Objectif

Documenter, au format ADR (Architecture Decision Record) court, les choix structurants de ce projet : pourquoi Helm plutôt que le provider Terraform `kubernetes`, pourquoi ArgoCD pull-based plutôt qu'un déploiement direct depuis la CI, et pourquoi Ansible n'a pas sa place ici. Chaque décision suit le même format : contexte → alternatives envisagées → choix retenu → conséquences.

---

## ADR-1 — Helm plutôt que le provider Terraform `kubernetes` pour déployer l'application

### 🎯 Contexte

L'application (`hr-backend`, `hr-frontend`) doit être déployée sur le cluster GKE et mise à jour à chaque nouveau tag d'image, sans toucher à l'infrastructure GCP elle-même. Terraform gère déjà l'infra (`repo-infrastructure`) et propose un provider `kubernetes` capable de créer des `Deployment`/`Service` directement.

### ⚙️ Alternatives envisagées

1. **Provider Terraform `kubernetes`** — définir les Deployments/Services comme ressources Terraform, dans le même state que l'infra GCP.
2. **Manifests YAML bruts** (`kubectl apply -f`) — pas de templating, un fichier par ressource et par environnement.
3. **Helm chart** — templating paramétrable, une seule source (`charts/hr-app/`) avec des fichiers `values-*.yaml` par environnement.

### 🚀 Choix retenu

**Helm.** `environments/staging/providers.tf` (dépôt `repo-infrastructure`) contient d'ailleurs les blocs `provider "helm"` et `provider "kubernetes"` **commentés**, avec en commentaire explicite : *"Outil pour installer des charts Helm : installe ArgoCD"* / *"Outil pour parler à Kubernetes (GKE) : gère le cluster GKE"* — la trace volontaire d'avoir écarté cette option.

Un seul chart (`charts/hr-app/`) avec deux fichiers de values (`values.yaml` pour les défauts, `values-staging.yaml` pour les overrides) évite la duplication qu'imposeraient des manifests YAML bruts (un jeu de fichiers complet par environnement), tout en gardant le déploiement applicatif **complètement découplé du state Terraform de l'infra**.

### ✅ Conséquences

- Un changement d'image applicative ne touche jamais au state Terraform — seul `values-staging.yaml` change (patché par `yq` depuis le CI de `repo-app`).
- `helm lint` et `helm template` permettent de valider un changement de chart en local sans toucher à aucun cluster ni à aucun state distant (voir [guide-helm-chart.md](guide-helm-chart.md)).
- Ajouter un environnement se limite à créer un nouveau fichier `values-<env>.yaml` et une nouvelle Application ArgoCD — aucune ressource Terraform à écrire.
- Contrepartie : deux outils de templating coexistent dans le projet (Terraform pour l'infra, Helm pour l'appli) plutôt qu'un seul — jugé acceptable car leurs cycles de vie sont fondamentalement différents (l'infra change rarement, l'application change à chaque push).

---

## ADR-2 — ArgoCD pull-based plutôt qu'un déploiement direct depuis la CI

### 🎯 Contexte

Une fois l'image construite et poussée dans Artifact Registry, il faut mettre à jour le cluster GKE. Deux familles d'approche existent : la CI se connecte elle-même au cluster et applique les manifests (push-based), ou un agent tournant dans le cluster détecte les changements Git et les applique lui-même (pull-based).

### ⚙️ Alternatives envisagées

1. **Push-based depuis `repo-app`** — le job `docker-build-push` récupérerait les credentials GKE et ferait `helm upgrade --install` directement après le push d'image.
2. **Pull-based avec ArgoCD** — la CI se limite à patcher `values-staging.yaml` dans `repo-config` ; un agent ArgoCD, déployé dans le cluster, détecte le commit et applique lui-même le changement.

### 🚀 Choix retenu

**ArgoCD, pull-based.** Le job `docker-build-push` de `repo-app/.github/workflows/ci.yml` ne détient **aucun accès `container.developer`/credentials GKE au-delà de `roles/container.developer`** dans les rôles accordés à `sa-github-actions` (`backend-config/wif.tf`) — et surtout, ne fait jamais de `kubectl`/`helm upgrade` lui-même. Il se contente d'un `git push` vers `repo-config`. C'est l'Application ArgoCD `hr-staging` (`apps/children/staging.yaml`), tournant *dans* le cluster, qui détecte le nouveau commit et synchronise.

### ✅ Conséquences

- **Aucun credential d'écriture sur le cluster ne quitte jamais le cluster.** Le pipeline CI de `repo-app` n'a besoin d'aucun accès `kubectl`/GKE en écriture pour déployer — seul `GH_PAT` (écriture sur `repo-config`, un dépôt Git) et les rôles GCP limités (`artifactregistry.writer`, `storage.objectViewer`) suffisent.
- `selfHeal: true` corrige automatiquement toute dérive manuelle sur le cluster (voir [guide-argocd-gitops.md](guide-argocd-gitops.md)) — un avantage direct du modèle pull, impossible à obtenir aussi simplement en push-based.
- Contrepartie assumée et documentée dans [GITOPS_PFE.md](../GITOPS_PFE.md) : bootstraper ArgoCD lui-même nécessite un mécanisme séparé, puisqu'on ne peut pas utiliser ArgoCD pour s'auto-installer. C'est le job `bootstrap-argocd` de `workflow-infra.yml` (Helm + `kubectl apply -f apps/root-app.yaml`) qui joue ce rôle une seule fois, avant qu'ArgoCD ne prenne le relais en continu.
- Le déploiement n'est plus instantané au push : il dépend de la fréquence de polling d'ArgoCD (quelques minutes), contre un déploiement immédiat en push-based. Jugé acceptable pour ce projet — voir [guide-argocd-gitops.md](guide-argocd-gitops.md) pour forcer une synchronisation immédiate si besoin (`argocd app sync`).

---

## ADR-3 — Pas d'Ansible dans ce projet

### 🎯 Contexte

Ansible est un outil courant de configuration management, typiquement utilisé pour installer des paquets, configurer des services ou pousser des fichiers de configuration sur des machines (VM) après leur création.

### ⚙️ Alternatives envisagées

1. **Ansible** pour configurer les nodes GKE ou une éventuelle VM applicative après provisioning.
2. **Aucun outil de configuration management** — laisser GKE gérer entièrement le cycle de vie des nodes.

### 🚀 Choix retenu

**Pas d'Ansible.** Il n'existe, dans l'ensemble du projet (`repo-app`, `repo-infrastructure`, `repo-config`), **aucune VM à configurer manuellement**. Le cluster est un cluster **GKE géré** : les nodes sont provisionnés, mis à jour (`auto_upgrade = true`) et réparés (`auto_repair = true`) automatiquement par Google (`modules/gke/main.tf`), sans qu'aucune étape de configuration post-provisioning ne soit nécessaire côté projet. L'application elle-même ne tourne jamais directement sur une VM : elle est packagée en image Docker (`backend/Dockerfile`, `frontend/Dockerfile`) et exécutée par des pods Kubernetes.

### ✅ Conséquences

- Aucune step "installer les dépendances système sur le node" n'existe ni n'est nécessaire — tout ce qui doit tourner sur un node est encapsulé dans une image de conteneur, gérée par le registre (Artifact Registry) et le scheduler Kubernetes.
- Si un futur composant nécessitait une VM dédiée (hors du cluster GKE), cette décision serait à réévaluer — mais ce n'est pas le cas aujourd'hui dans ce projet.

## 🔗 Pour la suite

- Détail du chart Helm évoqué en ADR-1 : [guide-helm-chart.md](guide-helm-chart.md)
- Détail du fonctionnement pull-based d'ArgoCD évoqué en ADR-2 : [guide-argocd-gitops.md](guide-argocd-gitops.md)
- Détail des modules Terraform (aucune VM applicative) évoqués en ADR-3 : [guide-deploiement-infra.md](guide-deploiement-infra.md)
