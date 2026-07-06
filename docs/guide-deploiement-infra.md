# Guide de déploiement de l'infrastructure (Terraform)

## 🎯 Objectif

Décrire comment provisionner l'infrastructure GCP depuis zéro avec Terraform (dépôt `repo-infrastructure`) : quels modules existent, dans quel ordre les appliquer, comment lire un plan avant de l'appliquer, et comment détruire proprement. Ce document est autonome — il ne couvre que la partie Terraform ; pour la partie applicative qui vient après, voir [lifecycle-pipeline.md](lifecycle-pipeline.md).

---

## ⚙️ Les deux projets Terraform

`repo-infrastructure` contient **deux projets Terraform distincts**, avec deux states différents :

```
repo-infrastructure/
├── backend-config/          ← état LOCAL (bootstrap, exécuté une seule fois)
│   main.tf, wif.tf, variables.tf, outputs.tf
│   terraform.tfstate (committé — projet chicken-and-egg)
│
├── modules/                 ← modules réutilisés par environments/staging
│   ├── networking/
│   ├── gke/
│   ├── iam/
│   └── artifact_registry/
│
└── environments/
    └── staging/             ← état distant GCS (prefix "staging"), le seul environnement actif
        main.tf, variables.tf, outputs.tf, providers.tf
```

`backend-config/` n'a pas de bloc `backend` dans son `terraform` {} — son état reste **local**, volontairement : c'est ce projet qui crée le bucket GCS que `environments/staging` utilisera ensuite comme backend distant. On ne peut pas faire dépendre le bootstrap du backend qu'il crée lui-même (problème de l'œuf et de la poule).

## 🚀 Étape 1 — Bootstrap (`backend-config/`), une seule fois par projet GCP

Ce module crée :
- Le bucket GCS de state (`google_storage_bucket.tfstate`, nom réel `tfstate-pfe-2026-495220`, versioning activé, purge après 7 versions) et son bucket de logs d'accès (`tfstate-pfe-2026-495220-logs`).
- Le pool et le provider de **Workload Identity Federation** (`github-pool-v2` / `github-provider`), scopés par `attribute.repository`.
- Deux comptes de service : `sa-terraform-ci` (pour `repo-infrastructure`) et `sa-github-actions` (pour `repo-app`).

```bash
cd repo-infrastructure/backend-config
cp terraform.tfvars.example terraform.tfvars   # puis remplir project_id, region, bucket_name, github_owner, ...
terraform init
terraform apply

# Récupérer les valeurs à mettre dans GitHub (secrets / vars) :
terraform output workload_identity_provider   # → secret WORKLOAD_IDENTITY_PROVIDER (repo-infrastructure)
terraform output terraform_ci_sa_email         # → secret SERVICE_ACCOUNT_EMAIL (repo-infrastructure)
terraform output github_actions_sa_email       # → var GCP_SERVICE_ACCOUNT (repo-app)
terraform output bucket_name                   # → à copier dans environments/staging/backend.hcl
```

⚠️ Ce module n'est à ré-exécuter que si le projet GCP est recréé de zéro, ou si l'on ajoute un nouveau dépôt consommateur de WIF.

## 🚀 Étape 2 — Provisionner l'infra staging (`environments/staging/`)

Quatre modules sont assemblés dans `environments/staging/main.tf` (dépôt `repo-infrastructure`) :

```
module.networking ──┐
                     ├──▶ module.gke  (depends_on = [module.networking, module.iam])
module.iam ──────────┘

module.artifact_registry   (indépendant, pas de depends_on)
```

| Module | Ressources créées |
|---|---|
| `modules/networking` | VPC `vpc-staging-pfe` (`auto_create_subnetworks = false`), sous-réseau `subnet-gke-staging` avec Private Google Access et VPC Flow Logs |
| `modules/iam` | SA `sa-gke-staging-pfe` (rôles `artifactregistry.reader`, `logging.logWriter`, `monitoring.metricWriter`, `monitoring.viewer`, `stackdriver.resourceMetadata.writer`), + bindings `iam.serviceAccountUser` étroits pour que `sa-terraform-ci` puisse assigner cette SA et la SA Compute par défaut au cluster |
| `modules/artifact_registry` | Repository Docker `registry-staging-pfe` dans Artifact Registry |
| `modules/gke` | Cluster GKE zonal `gke-staging-pfe` (zone `europe-west1-b`) + node pool séparé (spot VMs, autoscaling) — détails dans [guide-securite.md](guide-securite.md) |

### En local

```bash
cd repo-infrastructure/environments/staging
cp terraform.tfvars.example terraform.tfvars   # project_id, cluster_name, node_count, node_vm_size, disk_size_gb, registry_name...

terraform init \
  -backend-config="bucket=tfstate-pfe-2026-495220" \
  -backend-config="prefix=staging"

terraform fmt -check -recursive   # obligatoire avant tout commit — la CI le vérifie
terraform validate
terraform plan
```

### ✅ Lire un plan avant de l'appliquer

`terraform plan` affiche un résumé `Plan: X to add, Y to change, Z to destroy.` — avant tout `apply`, vérifie en particulier :
- Aucune destruction inattendue de `google_container_cluster.gke` ou `google_container_node_pool.default` (une destruction de cluster est irréversible pour les workloads en cours).
- Les changements sur `modules/iam` (bindings IAM) — vérifier qu'aucun rôle n'est élargi par accident.
- Les changements sur `modules/networking` (VPC/subnet) — un changement de CIDR peut casser la connectivité des nodes existants.

```bash
terraform plan -out=tfplan.binary
terraform show tfplan.binary       # relire en détail avant d'appliquer
terraform apply tfplan.binary       # applique EXACTEMENT le plan relu, pas un nouveau plan
```

### Dans la CI (`workflow-infra.yml`)

Le workflow orchestre exactement cette séquence automatiquement, avec des garde-fous supplémentaires :
1. **`validate`** — `terraform fmt -check -recursive` puis `terraform init -backend=false && terraform validate` sur chaque module et sur `environments/staging`.
2. **`lint`** — `tflint --recursive --format compact`.
3. **`security`** — Checkov (`.checkov.yaml`), CRITICAL/HIGH bloquants, SARIF uploadé vers l'onglet Security de GitHub. Détail dans [guide-securite.md](guide-securite.md).
4. **`plan`** — `terraform plan -detailed-exitcode -out=tfplan.binary`, artefact uploadé (`tfplan-infra`, 1 jour de rétention), commentaire automatique sur la PR si `exitcode == 2` (changements détectés).
5. **`apply`** — gardé par l'environnement GitHub `staging-apply` (nécessite une approbation manuelle si configuré ainsi côté GitHub) ; applique **le binaire de plan exact téléchargé**, jamais un nouveau plan ; attend ensuite qu'au moins un node GKE soit `Ready` avant de terminer.
6. **`bootstrap-argocd`** — installe/vérifie ArgoCD sur le cluster tout juste prêt (voir [guide-argocd-gitops.md](guide-argocd-gitops.md)).

Déclenchement manuel : `workflow_dispatch` avec l'input `action` = `plan` (défaut), `apply`, `destroy-staging`, `drift`, `bootstrap` ou `unlock`.

## 🚀 Étape 3 — Détecter la dérive (drift)

Le job `detect-drift` tourne automatiquement tous les jours à 13h30 UTC (`schedule: cron "30 13 * * *"`), ou manuellement via `workflow_dispatch` avec `action=drift`. Il exécute `terraform plan -refresh-only -detailed-exitcode` et ouvre une issue GitHub labellée `terraform-drift` si un écart est détecté (en évitant les doublons si une issue avec ce label est déjà ouverte).

En local :
```bash
cd repo-infrastructure/environments/staging
terraform plan -refresh-only
```

## 🚀 Étape 4 — Détruire proprement l'environnement staging

⚠️ Action destructive et gardée par l'environnement GitHub `staging-destroy`. Ne se déclenche que manuellement (`workflow_dispatch`, `action=destroy-staging`), jamais sur push.

Le job `destroy` fait, dans l'ordre :
1. Nettoyage GitOps : retire les finalizers de toutes les Applications ArgoCD, les supprime, désinstalle le chart ArgoCD (`helm uninstall argocd -n argocd --timeout 120s`), force la suppression des CRDs `applications.argoproj.io` / `applicationsets.argoproj.io` / `appprojects.argoproj.io` et du namespace `argocd`.
2. `terraform destroy -auto-approve` sur `environments/staging`.

Cet ordre évite que Terraform ne reste bloqué à essayer de supprimer un cluster GKE encore référencé par des ressources ArgoCD/Kubernetes.

En local (à n'utiliser qu'en environnement de test, jamais sans confirmation) :
```bash
cd repo-infrastructure/environments/staging
terraform destroy
```

## ✅ Débloquer un state verrouillé

Si un job a échoué en plein `plan`/`apply` et laissé le state GCS verrouillé, le job `unlock` (`workflow_dispatch`, `action=unlock`) lit l'ID du lock depuis `gs://<bucket>/staging/default.tflock` et exécute `terraform force-unlock -force <LOCK_ID>`. En local, la même commande fonctionne après avoir récupéré l'ID du lock (affiché dans le message d'erreur `Error acquiring the state lock`).

## 🔗 Pour la suite

- Comment ArgoCD prend le relais une fois le cluster prêt : [guide-argocd-gitops.md](guide-argocd-gitops.md)
- Détail des mesures de sécurité du cluster (Workload Identity, Shielded Nodes, Binary Authorization...) : [guide-securite.md](guide-securite.md)
- Pourquoi ArgoCD n'est pas bootstrapé via le provider Terraform `kubernetes` : [decisions-architecture.md](decisions-architecture.md)
