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
│   terraform.tfstate (gitignoré — projet chicken-and-egg)
│
├── modules/                 ← modules réutilisés par environments/staging
│   ├── networking/
│   ├── gke/
│   ├── iam/
│   ├── artifact_registry/   ← instancié 2x (staging + prod)
│   └── cnpg_backup/         ← instancié 3x (staging/dev/prod)
│
└── environments/
    └── staging/             ← état distant GCS (prefix "staging"), la seule racine Terraform active
        main.tf, variables.tf, outputs.tf, providers.tf
```

`backend-config/` n'a pas de bloc `backend` dans son `terraform` {} — son état reste **local**, volontairement : c'est ce projet qui crée le bucket GCS que `environments/staging` utilisera ensuite comme backend distant. On ne peut pas faire dépendre le bootstrap du backend qu'il crée lui-même (problème de l'œuf et de la poule).

⚠️ Le nom `environments/staging` désigne la racine Terraform, pas un environnement applicatif unique — cette même infrastructure héberge les trois namespaces applicatifs `dev`/`staging`/`prod` (gérés côté `repo-config`/ArgoCD, pas ici). Il n'existe pas de `environments/dev` ni `environments/prod`, et il n'y a pas de fichier `moved.tf` dans cette racine.

## 🚀 Étape 1 — Bootstrap (`backend-config/`), une seule fois par projet GCP

Ce module crée :
- Le bucket GCS de state (`google_storage_bucket.tfstate`, nom = `var.bucket_name`, versioning activé, purge après 7 versions) et son bucket de logs d'accès (`<bucket_name>-logs`).
- Le pool et le provider de **Workload Identity Federation** (`github-pool-v2` / `github-provider`), scopés par `attribute.repository`.
- Deux comptes de service : `sa-terraform-ci` (pour `repo-infrastructure`) et `sa-github-actions` (pour `repo-app`).

```bash
cd repo-infrastructure/backend-config
cp terraform.tfvars.example terraform.tfvars
# variables.tf exige aussi github_owner / github_infra_repo / github_app_repo (sans défaut) —
# le .tfvars.example committé ne contient que project_id/region/bucket_name, à compléter à la main
terraform init
terraform apply

# Récupérer les valeurs à mettre dans GitHub (secrets / vars) :
terraform output workload_identity_provider   # → secret WORKLOAD_IDENTITY_PROVIDER (repo-infrastructure)
terraform output terraform_ci_sa_email         # → secret SERVICE_ACCOUNT_EMAIL (repo-infrastructure)
terraform output github_actions_sa_email       # → var GCP_SERVICE_ACCOUNT (repo-app)
```

⚠️ Ce module n'est à ré-exécuter que si le projet GCP est recréé de zéro, ou si l'on ajoute un nouveau dépôt consommateur de WIF.

## 🚀 Étape 2 — Provisionner l'infra (`environments/staging/`)

**Huit blocs de module, issus de cinq sources distinctes**, sont assemblés dans `environments/staging/main.tf` :

```
module.networking ──┐
                     ├──▶ module.gke  (depends_on = [module.networking, module.iam])
module.iam ──────────┘

module.artifact_registry        (staging, indépendant)
module.artifact_registry_prod   (registre prod isolé, même source, indépendant)
module.cnpg_backup              (bucket backup staging, indépendant)
module.cnpg_backup_dev          (bucket backup dev, même source, indépendant)
module.cnpg_backup_prod         (bucket backup prod, même source, indépendant)
google_compute_global_address.ingress_ip   (for_each dev/staging/prod — pas un module, une ressource directe)
```

| Module | Ressources créées |
|---|---|
| `modules/networking` | VPC `vpc-staging-pfe` (`auto_create_subnetworks = false`), sous-réseau `subnet-gke-staging` avec Private Google Access et VPC Flow Logs |
| `modules/iam` | SA `sa-gke-staging-pfe` (rôles `artifactregistry.reader`, `logging.logWriter`, `monitoring.metricWriter`, `monitoring.viewer`, `stackdriver.resourceMetadata.writer`), + bindings `iam.serviceAccountUser` étroits pour que `sa-terraform-ci` puisse assigner cette SA et la SA Compute par défaut au cluster |
| `modules/artifact_registry` ×2 | Repository Docker `registry-staging-pfe` (build normal) et `registry-prod-pfe` (écrit uniquement par `crane copy` dans `promote-prod.yml`) |
| `modules/cnpg_backup` ×3 | Bucket GCS (`prevent_destroy = true`) + SA dédiée `sa-cnpg-<env>-backup` + binding Workload Identity, un par environnement applicatif |
| `google_compute_global_address` ×3 | IP statique globale par environnement (`ip-hr-dev`/`ip-hr-staging`/`ip-hr-prod`), consommée par l'Ingress GCE natif côté `repo-config` |
| `modules/gke` | Cluster GKE zonal `gke-staging-pfe` (zone `europe-west1-b`) + node pool séparé nommé `default` (spot VMs, autoscaling, Dataplane V2) — détails dans [guide-securite.md](guide-securite.md) |

### En local

```bash
cd repo-infrastructure/environments/staging
cp terraform.tfvars.example terraform.tfvars
# project_id, cluster_name, node_count, max_node_count, node_vm_size, disk_size_gb, registry_name,
# cnpg_backup_bucket_name, cnpg_backup_bucket_name_dev, cnpg_backup_bucket_name_prod (aucun défaut)

terraform init \
  -backend-config="bucket=<bucket créé par backend-config>" \
  -backend-config="prefix=staging"

terraform fmt -check -recursive   # obligatoire avant tout commit — la CI le vérifie
terraform validate
terraform plan
```

### ✅ Lire un plan avant de l'appliquer

`terraform plan` affiche un résumé `Plan: X to add, Y to change, Z to destroy.` — avant tout `apply`, vérifie en particulier :
- Aucune destruction inattendue de `google_container_cluster.gke` ou `google_container_node_pool.default` (une destruction de cluster est irréversible pour les workloads en cours, dans les trois namespaces à la fois).
- Les changements sur `modules/iam` (bindings IAM) — vérifier qu'aucun rôle n'est élargi par accident.
- Les changements sur `modules/networking` (VPC/subnet) — un changement de CIDR peut casser la connectivité des nodes existants.
- Toute suppression accidentelle des buckets `cnpg_backup` — ils portent `prevent_destroy = true` justement pour bloquer ça, mais un `plan` qui tente de les recréer (changement de nom) mérite d'être relu deux fois.

```bash
terraform plan -out=tfplan.binary
terraform show tfplan.binary       # relire en détail avant d'appliquer
terraform apply tfplan.binary       # applique EXACTEMENT le plan relu, pas un nouveau plan
```

### Dans la CI (`workflow-infra.yml`)

Le workflow orchestre exactement cette séquence automatiquement, avec des garde-fous supplémentaires :
1. **`validate`** — `terraform fmt -check -recursive` puis `terraform init -backend=false && terraform validate` sur les 7 dossiers module (y compris `modules/cnpg_backup`).
2. **`lint`** — `tflint --recursive --format compact`.
3. **`security`** — Checkov (`.checkov.yaml`), CRITICAL/HIGH bloquants, SARIF uploadé vers l'onglet Security de GitHub. Détail dans [guide-securite.md](guide-securite.md).
4. **`plan`** — `terraform plan -detailed-exitcode -out=tfplan.binary`, artefact uploadé (1 jour de rétention), commentaire automatique sur la PR si `exitcode == 2` (changements détectés). Le `terraform.tfvars` runtime est écrit depuis les `vars.*` GitHub, y compris les trois `cnpg_backup_bucket_name*` qui n'ont aucun défaut Terraform.
5. **`apply`** — gardé par l'environnement GitHub `staging-apply` ; applique **le binaire de plan exact téléchargé**, jamais un nouveau plan ; attend ensuite qu'au moins un node GKE soit `Ready` avant de terminer.
6. **`bootstrap-argocd`** — installe/vérifie ArgoCD sur le cluster tout juste prêt (voir [guide-argocd-gitops.md](guide-argocd-gitops.md)) ; tourne même si `apply` a été sauté (`if: always()`).

Déclenchement manuel : `workflow_dispatch` avec l'input `action` = `plan` (défaut), `apply`, `destroy-staging` ou `bootstrap`.

⚠️ **Il n'existe plus de déclencheur `schedule`, plus de job `detect-drift`, plus de job `unlock` dispatchable.** Ces trois éléments ont existé plus tôt dans le projet et ont été supprimés/commentés au fil des itérations — ne pas les chercher dans le YAML actuel (le bloc `unlock` est encore présent mais entièrement commenté, donc invisible pour GitHub Actions). Il n'y a aujourd'hui **aucune exécution périodique non surveillée** de ce workflow.

## 🚀 Détecter la dérive (drift) — manuellement, en local

Sans job `detect-drift` dans la CI, la vérification de dérive se fait à la main :
```bash
cd repo-infrastructure/environments/staging
terraform plan -refresh-only
```
Aucune issue GitHub n'est ouverte automatiquement — si une automatisation de ce type est souhaitée à nouveau, elle est à reconstruire depuis zéro (l'ancien job a été supprimé, pas juste désactivé).

## 🚀 Détruire proprement l'environnement

⚠️ Action destructive et gardée par l'environnement GitHub `staging-destroy`. Ne se déclenche que manuellement (`workflow_dispatch`, `action=destroy-staging`), jamais sur push.

Le job `destroy` fait, dans l'ordre :
1. **Retire d'abord les finalizers des CR `clusters.postgresql.cnpg.io`** et les supprime — doit précéder la suppression des Applications ArgoCD (dont `cnpg-operator`), sinon le `Cluster` reste bloqué en `Terminating` (plus d'opérateur vivant pour traiter son finalizer).
2. Nettoyage GitOps : retire les finalizers de toutes les Applications ArgoCD restantes, les supprime, désinstalle le chart ArgoCD, force la suppression des CRDs et du namespace `argocd`.
3. `terraform destroy -auto-approve` sur `environments/staging`.

⚠️ **Cette dernière étape échouera** sur les trois buckets `google_storage_bucket.cnpg_backup` (`lifecycle { prevent_destroy = true }`) — le job ne les retire pas du state et ne lève pas ce lifecycle avant l'appel. Pour un destroy complet il faut gérer ces trois buckets séparément (`terraform state rm` + suppression manuelle via `gcloud storage buckets delete`, ou override temporaire du lifecycle).

En local (à n'utiliser qu'en environnement de test, jamais sans confirmation) :
```bash
cd repo-infrastructure/environments/staging
terraform destroy
```

## ✅ Débloquer un state verrouillé

Il n'existe plus de job `unlock` dispatchable dans la CI. `plan` et `apply` s'auto-libèrent en cas d'échec (lecture de l'ID de lock depuis `gs://<bucket>/staging/default.tflock`, puis `terraform force-unlock -force <id>` inline). En dernier recours, la même commande fonctionne manuellement en local après avoir récupéré l'ID du lock (affiché dans le message d'erreur `Error acquiring the state lock`).

## 🔗 Pour la suite

- Comment ArgoCD prend le relais une fois le cluster prêt : [guide-argocd-gitops.md](guide-argocd-gitops.md)
- Détail des mesures de sécurité du cluster (Workload Identity, Shielded Nodes, Binary Authorization, Dataplane V2...) : [guide-securite.md](guide-securite.md)
- Pourquoi ArgoCD n'est pas bootstrapé via le provider Terraform `kubernetes` : [decisions-architecture.md](decisions-architecture.md)
