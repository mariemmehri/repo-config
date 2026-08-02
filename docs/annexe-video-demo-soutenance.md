# Annexe — Guide vidéo de démonstration (soutenance)

Déroulé minuté pour enregistrer une vidéo de démo de la plateforme SIRH (Sopra HR Software).
Contrairement à un déroulé générique, tout ce qui suit correspond à des ressources **réellement présentes** dans ce monorepo (`repo-app`, `repo-infrastructure`, `repo-config`) — vérifiées avant rédaction, y compris à partir du code actuel (pas seulement de la documentation, qui contient plusieurs sections obsolètes signalées dans les `CLAUDE.md` de chaque repo). Il n'y a **pas** de Prometheus/Grafana/Alertmanager, **pas** d'Ansible, **pas** de GitLab CI, **pas** de scaling prédictif ici (voir `docs/decisions-architecture.md` ADR-3 pour le choix explicite de ne pas utiliser Ansible) — ne pas improviser ces sections en tournage.

**Correction importante par rapport à une version précédente de ce guide** : le nom d'hôte de l'environnement staging n'est **pas** `hr-staging.local` mais **`hr4you.com`** (`charts/hr-app/values-staging.yaml`, renommé dans les commits `ba37d0f`/`e66cfc5`/`7ae0b0d`). Dev utilise `hr-dev.local`, prod utilise `hr4youp.com`. Tous les exemples `curl`/hosts ci-dessous utilisent le nom réel.

## Prérequis avant d'enregistrer

```bash
# Credentials cluster
gcloud container clusters get-credentials gke-staging-pfe --region europe-west1-b --project pfe-2026-495220

# Etat propre des 3 repos (rien d'inattendu dans le diff pendant l'enregistrement)
cd "repo-app" && git status
cd "../repo-infrastructure" && git status
cd "../repo-config" && git status

# ArgoCD Synced/Healthy sur les 3 environnements avant de commencer
kubectl get applications -n argocd

# Confirmer le nombre de noeuds actuel (necessaire pour le timing de la phase autoscaling)
kubectl get nodes -o wide
```
Si `kubectl get nodes` ne montre qu'1 seul noeud, la phase "Node autoscaling" prendra plus de temps à démarrer (le pool doit d'abord provisionner un 2e noeud spot) — prévoir une marge dans le montage vidéo plutôt que de couper en live.

Ajouter au fichier hosts local (si pas déjà fait) :
```
<IP_STATIQUE_ip-hr-staging>  hr4you.com
```
(récupérer l'IP avec `gcloud compute addresses describe ip-hr-staging --global --format='value(address)'`)

Note : seuls `staging` et `prod` ont aujourd'hui une IP statique globale réservée (`ip-hr-staging`, `ip-hr-prod`) — `dev` a été volontairement retiré de cette liste (commit `cd72e35`, "Drop dev from reserved static ingress IPs") et reçoit une IP éphémère GKE à chaque recréation d'Ingress. C'est précisément ce qui sert d'exemple concret dans la partie Terraform ci-dessous.

---

## Minutage proposé

| Temps | Séquence |
|---|---|
| 0:00 | Introduction |
| 1:00 | Architecture Kubernetes réelle (namespaces, ArgoCD) |
| 4:00 | Terraform — structure modulaire |
| 9:00 | Terraform — démonstration live (ajout d'une ressource) |
| 13:00 | CI/CD applicatif multi-environnements (dev / staging / prod) |
| 18:00 | GitOps — ArgoCD pull-based |
| 20:00 | Application, Ingress, HTTPS |
| 23:00 | Load Balancing |
| 26:00 | HPA — stress test |
| 31:00 | PodDisruptionBudget |
| 34:00 | podAntiAffinity soft |
| 37:00 | Node autoscaling |
| 40:00 | Backup et restore CNPG |
| 44:00 | Nettoyage final et conclusion |

---

### 0:00 — Introduction

Dire :
```text
Je vais présenter une plateforme SIRH cloud-native sur Google Kubernetes Engine, avec Infrastructure as Code Terraform, CI/CD GitHub Actions, et déploiement continu via ArgoCD (GitOps).
```

Montrer :
```bash
kubectl get ns
```
Attendu : `dev`, `staging`, `prod`, `argocd`, `cnpg-system`, `cert-manager`.

### 1:00 — Architecture Kubernetes réelle

```bash
kubectl get pods -A
kubectl get svc -A
kubectl get ingress -A
kubectl get hpa -A
kubectl get applications -n argocd
```

Dire :
```text
On voit trois environnements applicatifs — dev, staging et prod — qui tournent comme des namespaces sur un seul cluster GKE, pas trois clusters séparés. Chacun a sa propre base Postgres CloudNativePG, gérée elle aussi par ArgoCD comme une Application séparée. ArgoCD, visible dans le namespace argocd, réconcilie en continu l'état du cluster avec ce qui est décrit dans le dépôt repo-config.
```

---

### 4:00 — Terraform : structure modulaire

Ouvrir le fichier :
```text
repo-infrastructure/environments/staging/main.tf
```

Dire :
```text
On a utilisé une structure Terraform modulaire : cinq modules réutilisables, certains instanciés plusieurs fois pour couvrir les trois environnements, tous appelés depuis un seul fichier racine.
```

Montrer et décrire brièvement chaque module (table à l'écran ou énoncée) :

| Module | Rôle | Instanciation dans `main.tf` |
|---|---|---|
| `networking` | Crée le VPC dédié et le sous-réseau GKE, avec Private Google Access et VPC Flow Logs. | 1x — `module "networking"` |
| `artifact_registry` | Crée un dépôt Docker Artifact Registry. | 2x — `module "artifact_registry"` (registre staging/dev) et `module "artifact_registry_prod"` (registre prod isolé, alimenté uniquement par `promote-prod.yml`) |
| `iam` | Crée le compte de service des noeuds GKE (rôles least-privilege) et autorise `sa-terraform-ci` à l'attacher au cluster. | 1x — `module "iam"` |
| `gke` | Provisionne le cluster GKE zonal et son node pool autoscalé en VM spot. | 1x — `module "gke"`, dépend explicitement de `networking` et `iam` (`depends_on`) |
| `cnpg_backup` | Crée le bucket GCS de sauvegarde CNPG (barman-cloud) + un compte de service dédié par environnement, lié en Workload Identity au KSA du cluster Postgres. | 3x — `module "cnpg_backup"` (staging), `module "cnpg_backup_dev"`, `module "cnpg_backup_prod"` |

Dire :
```text
Seul le module gke a une vraie dépendance Terraform explicite envers networking et iam — il consomme leurs outputs, le vpc_name, le subnet id, et l'email du compte de service des noeuds. Tous les autres blocs — artifact_registry, artifact_registry_prod, et les trois cnpg_backup — sont totalement indépendants entre eux : l'ordre dans le fichier ne reflète pas une chaîne de dépendances.
```

Montrer aussi la dernière ressource du fichier, qui n'est pas un module mais une ressource directe avec `for_each` :
```hcl
resource "google_compute_global_address" "ingress_ip" {
  for_each = toset(["staging", "prod"])
  name     = "ip-hr-${each.key}"
  project  = var.project_id
}
```

Dire :
```text
Une IP statique globale par environnement exposé via Ingress — consommée côté repo-config par l'annotation global-static-ip-name du template Ingress. Aujourd'hui seuls staging et prod en ont une réservée ; dev a été volontairement retiré de cette liste il y a peu, pour économiser une IP statique sur un environnement qui n'en a pas besoin en continu.
```

### 9:00 — Terraform : démonstration live (ajout d'une ressource)

Objectif : montrer un vrai cycle `plan` → `checkov` → PR comment sur une ressource simple et sûre, en remettant `dev` dans la liste des IP statiques.

```bash
cd repo-infrastructure
git checkout -b demo/ajout-ip-dev
```

Éditer `environments/staging/main.tf` :
```diff
- for_each = toset(["staging", "prod"])
+ for_each = toset(["dev", "staging", "prod"])
```

```bash
terraform fmt -recursive
terraform init -backend=false -input=false
terraform validate
git add environments/staging/main.tf
git commit -m "demo: add dev back to reserved ingress static IPs"
git push -u origin demo/ajout-ip-dev
gh pr create --title "demo: add dev ingress static IP" --body "Démo soutenance : ajout d'une ressource Terraform additive."
```

Dire pendant que la CI tourne (job `validate` → `lint` → `security` → `plan`) :
```text
Le pipeline valide le format, lint avec tflint, scanne avec Checkov, puis exécute un plan avec code de sortie détaillé. Comme c'est une ressource purement additive, le plan ne doit montrer qu'une création : google_compute_global_address.ingress_ip["dev"].
```

Montrer :
- le commentaire automatique posté sur la PR (tableau Create/Update/Destroy — ici 1 Create, 0 Update, 0 Destroy) ;
- le rapport Checkov (SARIF, dans l'onglet Security du repo, ou la sortie console du job) — doit rester vert, aucune nouvelle alerte CRITICAL/HIGH sur une ressource déjà couverte par le même skip-list que l'IP `ip-hr-staging` existante.

```bash
# Optionnel : ne merger que si vous voulez réellement déclencher l'apply
# (crée une vraie IP statique GCP, réversible en repassant le for_each à ["staging","prod"])
gh pr merge --squash
```

Dire :
```text
Selon le temps disponible, on peut soit s'arrêter au plan pour ne pas toucher à l'infrastructure réelle, soit merger pour voir le job apply provisionner réellement l'adresse — dans les deux cas, c'est le même pipeline validate/lint/security/plan/apply que pour n'importe quel changement d'infrastructure.
```

---

### 13:00 — CI/CD applicatif multi-environnements

Ouvrir les fichiers :
```text
repo-app/.github/workflows/ci.yml
repo-app/.github/workflows/promote-prod.yml
repo-config/apps/children/dev.yaml
repo-config/apps/children/staging.yaml
repo-config/apps/children/prod.yaml
```

Dire :
```text
Trois environnements, un seul pattern : build-once puis promote. Dev et staging sont construits automatiquement à chaque push ; prod n'est jamais reconstruite, elle reçoit une image déjà validée en staging.
```

| Env | Déclencheur | Build ? | Fichier values patché | Sync ArgoCD |
|---|---|---|---|---|
| `dev` | push sur `develop` | oui, tag = SHA court (7 car.) | `values-dev.yaml` | automatique (`prune`+`selfHeal`) |
| `staging` | push/merge sur `main` | oui, tag = SHA court | `values-staging.yaml` | automatique (`prune`+`selfHeal`) |
| `prod` | tag `v*.*.*` sur un commit `main` | **non** — `crane copy` de l'image déjà buildée (digest préservé) vers un registre prod isolé, re-taggée `vX.Y.Z` | `values-prod.yaml` | **manuel uniquement** (pas de `syncPolicy.automated`) + gate GitHub Environment `production` |

Démonstration concrète — un push sur `develop` :
```bash
cd repo-app
git checkout develop
git pull
# petit changement anodin pour déclencher la CI, ex. un commentaire dans HealthController.java
git commit -am "demo: trivial change to trigger dev CI"
git push origin develop
```

Dire pendant que le workflow GitHub Actions tourne :
```text
backend-ci lance mvn verify, frontend-ci build Angular en parallèle. Une fois les deux artefacts prêts, docker-build-push s'authentifie à GCP par OIDC — aucune clé JSON stockée — construit les deux images taguées avec le SHA court, les scanne avec Trivy en sévérité CRITICAL, puis les pousse sur le registre Artifact Registry.
```

Après le job `deploy`, montrer la sortie réelle côté repo-config :
```bash
cd ../repo-config
git pull
git log --oneline -3
```
Attendu : un commit `ci: update dev image tags to <SHA7>` généré automatiquement par la CI (token GitHub App de courte durée, scope limité à ce dépôt).

```bash
git show --stat HEAD          # values-dev.yaml modifié
git show HEAD -- charts/hr-app/values-dev.yaml   # diff du tag d'image
```

Puis sur le cluster :
```bash
kubectl get application hr-dev -n argocd
kubectl get deployment hr-backend -n dev -o jsonpath="{.spec.template.spec.containers[0].image}"
```

Dire :
```text
ArgoCD détecte le nouveau commit sur repo-config et synchronise automatiquement le namespace dev, sans aucune intervention manuelle : c'est le chemin complet, du push jusqu'au pod, entièrement piloté par Git.
```

Mentionner (sans forcément l'exécuter en vidéo, gate humaine) la promotion prod :
```text
Pour la prod, il n'y a jamais de nouveau build : on tague un commit main déjà passé par la CI, un reviewer doit approuver l'environnement GitHub production, puis crane copy recopie l'image bit-à-bit vers un registre prod isolé. Le commit qui patch values-prod.yaml laisse hr-prod OutOfSync : le déploiement réel exige un argocd app sync hr-prod manuel, volontairement découplé de la promotion.
```

---

### 18:00 — GitOps : ArgoCD pull-based

```bash
kubectl get application hr-staging -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
```

Dire :
```text
ArgoCD, à l'intérieur du cluster, tire en continu l'état désiré depuis repo-config et l'applique : c'est du GitOps pull-based, pas un déploiement poussé directement par la CI. Le cluster n'a jamais besoin d'exposer de credentials à la CI pour se faire déployer dessus, ce qui réduit la surface d'attaque et donne un historique Git complet de tout ce qui a été déployé.
```

### 20:00 — Application, Ingress, HTTPS

```bash
# Acces direct (sans passer par le LB) pour valider que le pod repond
kubectl port-forward svc/hr-backend -n staging 8081:8081 &
curl http://localhost:8081/api/health-check

# Acces via l'Ingress externe, en HTTPS (TLS via une CA locale cert-manager)
curl -kI https://hr4you.com/api/health-check
```

Dire :
```text
Le backend expose un endpoint de santé utilisé à la fois comme readiness et liveness probe Kubernetes. L'application est aussi exposée en HTTPS via un Ingress GKE natif, avec un certificat émis par une autorité de certification locale gérée par cert-manager — pas de domaine public résolu publiquement ici, la résolution se fait via le fichier hosts local.
```

Nettoyage avant de continuer :
```bash
kill %1 2>/dev/null || true
```

### 23:00 — Load Balancing

```bash
kubectl get pods -n staging -l app=hr-backend -o wide
kubectl get endpoints hr-backend -n staging
```

Dire :
```text
Le Service hr-backend a une annotation NEG — Network Endpoint Group — qui fait que le load balancer externe de l'Ingress route le trafic directement vers les IP des pods, sans passer par kube-proxy. On voit ici deux pods, donc deux cibles réelles derrière une seule URL.
```

Preuve comportementale (CPU des deux pods qui montent ensemble sous trafic modéré, sans déclencher le HPA) :
```bash
# Terminal A — observer la charge par pod en direct
kubectl top pods -n staging -l app=hr-backend

# Terminal B — trafic modere via l'Ingress externe (le vrai chemin du LB)
for i in $(seq 1 200); do curl -s -o /dev/null -k https://hr4you.com/api/employees; done
```

Dire :
```text
Si le load balancing ne fonctionnait pas, un seul pod absorberait tout le trafic. Ici les deux montent en charge quasi simultanément — c'est la répartition réelle, pas juste deux pods qui existent côte à côte.
```

### 26:00 — HPA — stress test

```bash
# Terminal A — observer le HPA et les pods en direct
kubectl get hpa hr-backend -n staging -w
kubectl get pods -n staging -l app=hr-backend -w

# Terminal B — generateur de charge local (repo-app/scripts/load-test.sh)
./repo-app/scripts/load-test.sh --url https://hr4you.com/api/employees --concurrency 20 --duration 180
```

Dire :
```text
Le seuil du HPA est fixé à 70% d'utilisation CPU par rapport à la requête de 100 millicores. Sous cette charge, le pourcentage dépasse le seuil et le nombre de replicas grimpe progressivement de 2 vers la limite maximale de 4.
```

**Si `load-test.sh` ne suffit pas à dépasser 70%** (limité par le CPU/réseau du poste local) — générer la charge depuis l'intérieur du cluster à la place, avec plusieurs pods éphémères en parallèle :
```bash
for i in $(seq 1 15); do
  kubectl run loadgen-$i --image=curlimages/curl:8.9.1 --restart=Never -n staging -- \
    sh -c 'end=$(($(date +%s)+180)); while [ $(date +%s) -lt $end ]; do curl -s -o /dev/null http://hr-backend.staging.svc.cluster.local:8081/api/employees; done' &
done
wait
```
Ceci tape directement le Service en DNS interne, avec 15 pods concurrents indépendants du poste local — nettement plus de débit qu'un seul `load-test.sh` local.

Nettoyage :
```bash
for i in $(seq 1 15); do kubectl delete pod loadgen-$i -n staging --ignore-not-found; done
```

Après l'arrêt de la charge, montrer la redescente (attendre ~5 minutes — stabilisation par défaut de Kubernetes sur le scale-down) :
```bash
kubectl get hpa hr-backend -n staging -w
```

### 31:00 — PodDisruptionBudget

```bash
kubectl get pdb hr-backend -n staging
```

Dire :
```text
Ce PodDisruptionBudget garantit qu'au moins un pod backend reste disponible pendant une perturbation volontaire — un drain de noeud, ou une mise à jour automatique de GKE. Il ne protège pas contre la préemption spot, qui est une éviction involontaire.
```

Test :
```bash
NODE=$(kubectl get pod -n staging -l app=hr-backend -o jsonpath='{.items[0].spec.nodeName}')
kubectl drain "$NODE" --ignore-daemonsets --delete-emptydir-data
```

Dire :
```text
L'éviction respecte le budget : Kubernetes attend qu'un pod de remplacement soit prêt ailleurs avant de continuer, au lieu de vider tous les pods d'un coup.
```

Remettre le noeud en service après la démo :
```bash
kubectl uncordon "$NODE"
```

### 34:00 — podAntiAffinity soft

```bash
kubectl get pods -n staging -l app=hr-backend -o wide
```

Dire :
```text
Les deux pods backend portent une contrainte d'anti-affinité de type "preferred" — best effort — qui essaie de les répartir sur des noeuds différents, sans l'exiger absolument.
```

Pour prouver que c'est bien "soft" et pas "hard" (isoler artificiellement un seul noeud disponible) :
```bash
for n in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  [ "$n" != "$NODE" ] && kubectl cordon "$n"
done
kubectl delete pod -n staging -l app=hr-backend
kubectl get pods -n staging -o wide   # les deux pods finissent sur le MEME noeud, pas de Pending
```

Dire :
```text
Avec une contrainte "required", le second pod resterait en attente faute d'alternative. Ici il démarre quand même sur le même noeud — la preuve que c'est un compromis de placement, pas une garantie dure.
```

Remettre en état :
```bash
kubectl get nodes -o jsonpath='{.items[*].metadata.name}' | xargs -n1 kubectl uncordon
```

### 37:00 — Node autoscaling

```bash
kubectl get nodes -w
kubectl get events -n staging --sort-by=.lastTimestamp | grep -iE "FailedScheduling|TriggeredScaleUp"
```

Dire :
```text
En poussant le stress test plus fort ou plus longtemps que pour la simple démo HPA, les 4 pods backend au maximum dépassent la capacité CPU des noeuds existants. Des pods restent en attente le temps que le pool de noeuds, lui aussi en autoscaling, en provisionne un nouveau — ici un noeud spot supplémentaire.
```

(Réutiliser la boucle `kubectl run loadgen-*` de la phase HPA, avec un `end=$(($(date +%s)+400))` plus long, pour maintenir la pression le temps que le nouveau noeud apparaisse.)

---

### 40:00 — Backup et restore CNPG

Ouvrir :
```text
repo-infrastructure/scripts/cnpg-restore-single-row.sh
repo-infrastructure/docs/backup-restore-drill-report.md
```

Dire :
```text
Chaque environnement CNPG sauvegarde en continu vers un bucket GCS dédié, via le plugin barman-cloud, avec une identité GCP séparée par environnement en Workload Identity — jamais de clé de service exportée. Ce script valide de bout en bout qu'une ligne supprimée par erreur dans pg-dev est récupérable depuis cette sauvegarde, sans jamais toucher au cluster Postgres réellement servi.
```

Lancer en temps réel, en mode dry-run (par défaut — ne touche jamais `pg-dev`) :
```bash
cd repo-infrastructure/scripts
./cnpg-restore-single-row.sh
```

Dire pendant l'exécution :
```text
Le script crée une sauvegarde CNPG à la demande, restaure cette sauvegarde dans un cluster Postgres jetable via bootstrap.recovery, extrait la ligne cible — ici la ligne id=1 de la table employees — vérifie qu'elle est bien présente dans la copie restaurée, puis détruit le cluster jetable. Aucune écriture sur la base pg-dev réelle en mode dry-run.
```

Mentionner l'option `--live` sans nécessairement l'exécuter en vidéo :
```text
En mode --live, le script va plus loin : il supprime réellement la ligne dans pg-dev, puis la réinsère à partir de la copie restaurée — c'est la preuve complète d'un scénario de perte de données. Le rapport de drill, dans docs/backup-restore-drill-report.md, est le gabarit à remplir avec les mesures de RTO/RPO obtenues.
```

Limites à mentionner si la question est posée : le script ne couvre que `pg-dev`, une seule ligne à la fois — pas une perte complète de cluster, pas de PITR sans sauvegarde à la demande, et ne teste pas le failover HA (seul `pg-prod`, à 3 instances, en aurait besoin).

### 44:00 — Nettoyage final et conclusion

```bash
kubectl delete pod -n staging -l run --ignore-not-found 2>/dev/null || true
for i in $(seq 1 15); do kubectl delete pod loadgen-$i -n staging --ignore-not-found; done
kubectl get pods -n staging -l app=hr-backend
kubectl get hpa hr-backend -n staging
kubectl get pdb hr-backend -n staging
kubectl get applications -n argocd
```

Dire :
```text
Je termine en confirmant le retour à l'état normal : deux pods backend, HPA revenu à son minimum, budget de disruption toujours actif, et les trois applications ArgoCD toujours Synced et Healthy.
```

---

## Questions probables et réponses courtes

### Pourquoi deux pods backend minimum en staging ?
```text
Pour simuler la haute disponibilité. Même avant charge, deux pods API permettent de montrer le load balancing et d'éviter un point unique de défaillance applicatif.
```

### Différence entre load balancing et autoscaling ?
```text
Le load balancing répartit le trafic entre les pods existants. L'autoscaling ajoute ou retire des pods selon la charge observée.
```

### Le HPA fait varier le nombre de replicas — pourquoi ArgoCD ne le corrige pas en arrière vers la valeur du chart ?
```text
La configuration ArgoCD ignore explicitement le champ spec.replicas des Deployments (ignoreDifferences), dans les trois Applications dev/staging/prod. Sans ça, le mode selfHeal comparerait le nombre de pods réel au nombre déclaré dans Git et le forcerait en arrière à chaque synchronisation, ce qui annulerait le travail du HPA.
```

### Pourquoi un PodDisruptionBudget ?
```text
Il garantit une disponibilité minimale pendant les perturbations volontaires — un drain de noeud ou une mise à jour automatique de GKE — sans bloquer indéfiniment la maintenance du cluster.
```

### Pourquoi une anti-affinité "soft" et pas "hard" ?
```text
Le pool de noeuds est petit et en autoscaling avec des VM spot. Une contrainte stricte pourrait laisser un pod en attente faute de noeud disponible. Le mode "preferred" reste un compromis : il répartit quand c'est possible, sans bloquer le déploiement quand ce ne l'est pas.
```

### Pourquoi pas de Prometheus/Grafana ?
```text
Ce n'est pas encore implémenté — c'est documenté comme tel. La supervision actuelle repose sur les probes readiness/liveness Kubernetes et sur Cloud Monitoring/Logging de GCP, déjà câblés via les rôles IAM du cluster. Ajouter une stack Prometheus/Grafana est une amélioration identifiée, pas encore priorisée.
```

### Pourquoi ArgoCD pull-based plutôt qu'un déploiement direct depuis la CI ?
```text
Le cluster n'a jamais besoin d'exposer de credentials à la CI pour se faire déployer dessus — c'est ArgoCD, à l'intérieur du cluster, qui va chercher l'état désiré dans Git. Ça réduit la surface d'attaque et donne un historique Git complet de tout ce qui a été déployé.
```

### Pourquoi la prod n'est-elle jamais reconstruite ?
```text
Pour garantir qu'on déploie en production exactement l'artefact déjà validé en staging — même digest d'image, copié bit-à-bit avec crane copy. Reconstruire à partir du même code source ne donnerait pas nécessairement le même artefact binaire (dépendances, horodatages de build), et casserait la traçabilité entre ce qui a été testé et ce qui tourne en prod.
```

### Pourquoi un cluster zonal et des VM spot plutôt qu'un cluster régional avec des VM standards ?
```text
Choix de coût pour un environnement de staging/démo. Les VM spot peuvent être préemptées à tout moment — c'est un compromis assumé, pas adapté à un vrai environnement de production sans redondance supplémentaire.
```

### Pourquoi un compte de service GCP séparé par environnement pour les sauvegardes CNPG ?
```text
Principe du moindre privilège appliqué par ressource : le compte de sauvegarde de dev ne peut écrire que dans le bucket de dev, celui de staging que dans celui de staging, etc. Une fuite d'identité sur un environnement n'expose pas les sauvegardes des deux autres.
```

---

## Checklist finale avant d'envoyer la vidéo

```bash
kubectl get pods -n dev
kubectl get pods -n staging
kubectl get pods -n prod
kubectl get hpa -A
kubectl get pdb -A
kubectl get applications -n argocd
curl -sk https://hr4you.com/api/health-check
```

Tout doit être propre :
```text
pods Ready dans les 3 namespaces
HPA revenu à son minimum (2/4 en staging)
PDB actif (ALLOWED DISRUPTIONS: 1)
les 3 Applications ArgoCD Synced + Healthy
health-check retourne 200/UP
aucun pod loadgen residuel
branche/PR de démo Terraform nettoyée (mergée ou fermée)
```

---

## Résumé final à dire

```text
Cette vidéo montre une plateforme SIRH déployée sur Google Kubernetes Engine, avec une infrastructure modulaire décrite en Terraform et un déploiement continu piloté par ArgoCD en GitOps sur trois environnements — dev, staging et prod — suivant un pattern build-once puis promote. Le CI/CD GitHub Actions automatise la construction, le scan de sécurité et la mise à jour de la configuration pour dev et staging, tandis que la prod exige une promotion explicite et un déploiement manuel. Le load balancing natif de l'Ingress GKE répartit le trafic entre plusieurs pods API. Le HorizontalPodAutoscaler ajuste le nombre de pods selon la charge CPU observée, et le pool de noeuds s'adapte lui-même en dessous quand c'est nécessaire. Un PodDisruptionBudget protège la disponibilité pendant les opérations de maintenance, une contrainte d'anti-affinité répartit les pods entre noeuds sur une base de compromis, et une procédure de sauvegarde/restauration CNPG valide la récupérabilité des données. L'ensemble reste cohérent avec ce qui est versionné dans Git, sans intervention manuelle sur le cluster.
```
