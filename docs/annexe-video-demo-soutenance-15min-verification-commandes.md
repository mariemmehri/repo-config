# Annexe — Guide vidéo de démonstration (soutenance) — Version 15 min, vérification par commandes

Troisième variante, inspirée des deux guides existants (`annexe-video-demo-soutenance.md`, 45 min, et `annexe-video-demo-soutenance-15min.md`, 15 min). Ne remplace ni ne modifie les deux fichiers d'origine — c'est une trame alternative, même contrainte de 15 minutes que la version courte, mais avec deux différences de fond assumées :

1. **Vérification par commandes** : contrairement à `annexe-video-demo-soutenance-15min.md`, qui remplace plusieurs démonstrations « live » par une explication orale + une preuve obtenue avant l'enregistrement (capture d'écran, `watch` déjà en cours), ici chaque étape s'appuie sur une **commande exécutée à l'écran pendant l'enregistrement**, qui interroge l'état réel du cluster au moment T — jamais une capture pré-enregistrée présentée comme si elle était live.
2. **Vidéo divisée en deux parties distinctes, infra puis applicatif** : la Partie 1 couvre exclusivement `repo-infrastructure` (Terraform, GKE, pipeline CI/CD infra) — rien à voir avec le code applicatif. La Partie 2 couvre `repo-app`/`repo-config` (CI/CD applicatif, GitOps, résilience des pods, HPA, CNPG, monitoring). Cette séparation reflète la réalité du repo : ce sont deux pipelines GitHub Actions indépendants (`workflow-infra.yml` vs `ci.yml`/`promote-prod.yml`), un changement dans l'un ne déclenche jamais l'autre — la vidéo respecte cette frontière au lieu de mélanger les deux.

**Étape finale imposée : Grafana**, à la toute fin de la Partie 2, avec un objectif précis — afficher et vérifier le **nombre de replicas réel de chaque environnement** (dev / staging / prod), puis croiser ce chiffre avec la commande `kubectl` équivalente, pour prouver que le dashboard reflète bien l'état du cluster et pas une valeur figée.

---

## Rappel des valeurs attendues (à vérifier, pas à réciter)

D'après `charts/hr-app/values-{dev,staging,prod}.yaml` au moment de la rédaction — ces chiffres peuvent avoir changé depuis, **c'est justement pour ça que chaque étape ci-dessous revérifie par commande plutôt que de les affirmer** :

| Env | `hr-backend` replicas | Autoscaling backend | `hr-frontend` replicas | PDB backend |
|---|---|---|---|---|
| `dev` | 1 | désactivé | 1 | non |
| `staging` | 2 | activé, min 2 / max 4 | 1 | oui, `minAvailable: 1` |
| `prod` | 2 | activé, min 2 / max 5 | 2 | non défini dans `values-prod.yaml` (à vérifier en live) |

---

## Prérequis avant d'enregistrer

```bash
# Credentials cluster
gcloud container clusters get-credentials gke-staging-pfe --region europe-west1-b --project pfe-2026-495220

# Etat propre des 3 repos
cd "repo-app" && git status
cd "../repo-infrastructure" && git status
cd "../repo-config" && git status

# Toutes les Applications ArgoCD Synced/Healthy avant de commencer
kubectl get applications -n argocd

# Stack de monitoring en place
kubectl get pods -n monitoring
kubectl get application kube-prometheus-stack -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
kubectl get application monitoring-manifests -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'

# gh CLI authentifié — nécessaire pour la Partie 1 (lecture des runs GitHub Actions de repo-infrastructure)
gh auth status
```
Attendu pour `kube-prometheus-stack` : `OutOfSync Healthy` est normal (GKE réinjecte une annotation `cloud.google.com/neg` sur les Services du namespace `monitoring`, qu'ArgoCD ne peut jamais faire correspondre exactement) — ne pas s'arrêter dessus si la question ne vient pas spontanément.

Port-forwards à lancer avant l'enregistrement, dans des terminaux séparés à garder ouverts :
```bash
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80
kubectl port-forward svc/argocd-server -n argocd 9089:80
kubectl port-forward svc/hr-frontend -n dev 8080:80
```

`dev` n'a pas d'Ingress (`ingress.enabled: false` dans `values-dev.yaml`) — c'est voulu, pas un oubli ; `dev` se montre uniquement en port-forward direct. `staging` (`https://hr4you.com`, TLS cert-manager) et `prod` (`http://hr4youp.com`, pas de TLS configuré aujourd'hui) passent par l'Ingress GCE natif — vérifier que les deux entrées existent dans le fichier hosts local :
```bash
grep -E "hr4you.com|hr4youp.com" /etc/hosts
```

---

## Minutage proposé (15:00 max)

| Temps | Séquence | Durée |
|---|---|---|
| 0:00 | Introduction générale | 0:30 |
| **PARTIE 1 — INFRASTRUCTURE** (`repo-infrastructure`) | | **4:30** |
| 0:30 | Architecture GKE réelle — modules Terraform, node pool, namespaces | 1:00 |
| 1:30 | CI/CD infra — pipeline Terraform vérifié par commandes | 2:30 |
| 4:00 | Node autoscaling — résilience côté infra, vérifiée par commande | 1:00 |
| **PARTIE 2 — APPLICATIF** (`repo-app` / `repo-config`) | | **10:00** |
| 5:00 | CI/CD applicatif — push live sur dev, état réel staging/prod vérifié par commandes | 3:30 |
| 8:30 | Résilience applicative — PDB / anti-affinité des pods | 1:15 |
| 9:45 | HPA — seuils et état réel vérifiés par commandes | 1:15 |
| 11:00 | Backup/restore CNPG — vérification de la configuration | 0:45 |
| 11:45 | **Monitoring / Grafana (étape finale)** — replicas par environnement, croisés avec `kubectl` | 2:15 |
| 14:00 | Checklist de vérification complète + conclusion | 1:00 |

Total : 15:00.

---

## PARTIE 1 — INFRASTRUCTURE (`repo-infrastructure`)

Tout ce qui suit ne touche que Terraform, GKE et le pipeline `workflow-infra.yml` — aucune ligne de code applicatif. Dire en ouverture de cette partie :
```text
On commence par la couche infrastructure, dans repo-infrastructure — un dépôt et un pipeline GitHub Actions totalement séparés du reste. Rien ici ne touche au code de l'application.
```

### 0:00 — Introduction générale (30s)

Dire :
```text
Je vais présenter, en quinze minutes, une plateforme SIRH cloud-native sur Google Kubernetes Engine, en deux parties : d'abord l'infrastructure — Terraform et son pipeline CI/CD dédié — puis la couche applicative — GitOps, résilience, autoscaling et supervision Prometheus/Grafana. Chaque affirmation sera vérifiée en direct par une commande, pas juste illustrée.
```

### 0:30 — Architecture GKE réelle 

```bash
cd repo-infrastructure
kubectl get ns
kubectl get nodes -o wide
```

Attendu pour `get ns` : `dev`, `staging`, `prod`, `argocd`, `cnpg-system`, `cert-manager`, `monitoring` — tous provisionnés par ce même Terraform, à l'exception des Applications ArgoCD elles-mêmes.

Dire :
```text
Un seul cluster GKE zonal, provisionné par huit blocs module Terraform répartis sur cinq sources réutilisables — networking, iam, gke, artifact_registry utilisé deux fois, cnpg_backup utilisé trois fois, une fois par environnement. Le node pool qu'on voit ici tourne en VM spot avec autoscaling, ce qu'on vérifiera concrètement dans quelques minutes.
```

## pipeline Terraform vérifié par commandes 

Dire :
```text
Ce pipeline est indépendant du CI/CD applicatif qu'on verra dans la deuxième partie : il ne se déclenche que sur un changement dans environments/, modules/, backend-config/, ou sur un dispatch manuel.
```

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

Dire pendant que la CI tourne (job `validate` → `lint` → `security` → `plan`) :
```text
Le pipeline valide le format, lint avec tflint, scanne avec Checkov, puis exécute un plan avec code de sortie détaillé. Comme c'est une ressource purement additive, le plan ne doit montrer qu'une création : google_compute_global_address.ingress_ip["dev"].
```

Montrer :
- le commentaire automatique posté sur la PR (tableau Create/Update/Destroy — ici 1 Create, 0 Update, 0 Destroy) ;
- le rapport Checkov (SARIF, dans l'onglet Security du repo, ou la sortie console du job) — doit rester vert, aucune nouvelle alerte CRITICAL/HIGH sur une ressource déjà couverte par le même skip-list que l'IP `ip-hr-staging` existante.

puis on merge()

---

## PARTIE 2 — APPLICATIF (`repo-app` / `repo-config`)

Transition à dire :
```text
On bascule maintenant sur la couche applicative — un pipeline CI/CD totalement distinct de celui qu'on vient de voir, dans repo-app et repo-config.
```

### 5:00 — CI/CD applicatif : push live sur dev + vérification réelle staging/prod (3:30)



**Dev — en direct :**
```bash
cd repo-app
git checkout develop
git pull
```
Modifier `frontend/src/app/app.component.html` ligne 2 (le libellé du logo d'en-tête), commit et push :
```bash
git add frontend/src/app/app.component.html
git commit -m "demo: update header label to prove live GitOps flow"
git push origin develop
```

Dire pendant que la CI tourne :
```text
Le push construit les images backend et frontend, les scanne avec Trivy, les pousse sur Artifact Registry, puis patche automatiquement values-dev.yaml dans repo-config avec le nouveau tag d'image.
```

Après le job `deploy`, vérifier — pas supposer — que le commit est bien arrivé et appliqué :
```bash
cd ../repo-config && git pull && git log --oneline -2
git show HEAD -- charts/hr-app/values-dev.yaml   # le tag doit correspondre au commit qui vient d'être poussé
kubectl get application hr-dev -n argocd
kubectl get deployment hr-backend -n dev -o jsonpath="{.spec.template.spec.containers[0].image}{'\n'}"
kubectl port-forward svc/hr-frontend -n dev 8080:80 &
curl -s http://localhost:8080 
```

Dire :
```text
ArgoCD a détecté le commit et synchronisé automatiquement le namespace dev, sans intervention manuelle. Le tag d'image du Deployment correspond exactement au commit qu'on vient de voir dans repo-config, et le contenu servi par le pod reflète le changement — la chaîne complète, du push au rendu, vérifiée à chaque maillon plutôt qu'affirmée.
```

**Staging et prod — même mécanisme, état réel vérifié par commande (pas rejoué en direct pour tenir le format) :**on fait tous puis decouper de demo

```bash
kubectl get application hr-staging -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
kubectl get application hr-prod -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
kubectl get deployment hr-backend -n staging -o jsonpath="{.spec.template.spec.containers[0].image}{'\n'}"
kubectl get deployment hr-backend -n prod -o jsonpath="{.spec.template.spec.containers[0].image}{'\n'}"
curl -sk https://hr4you.com/api/health-check
curl -s http://hr4youp.com/api/health-check
```

Dire :
```text
Staging est synchronisé automatiquement comme dev. Prod reste volontairement en attente d'un sync manuel après chaque promotion — c'est une contrainte du pipeline, pas une panne, et les deux health-check confirment que les deux environnements répondent malgré tout.
```



###  — HPA : seuils et état réel (1:15)

```bash
kubectl get hpa hr-backend -n staging
kubectl get hpa hr-backend -n prod
kubectl describe hpa hr-backend -n staging | grep -A3 "Metrics:"


#HPA : stress test compressé


kubectl get hpa hr-backend -n staging
cd "C:\Users\wassi\Downloads\stage pfe\repo-app\scripts"
.\load-test.ps1 -DurationSeconds 120 -Concurrency 20 -Namespace staging
kubectl get hpa hr-backend -n staging -w
kubectl get pods -n staging -l app=hr-backend



###- Backup/restore CNPG

```bash
kubectl get application cnpg-cluster-dev -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
kubectl get cluster.postgresql.cnpg.io -A
kubectl get backup.postgresql.cnpg.io -A --sort-by=.metadata.creationTimestamp | tail -5
```
pour tester le backup fonctionnel:

cd "repo-infrastructure"
bash ./scripts/cnpg-restore-single-row.sh --live


Dire :
```text
Chaque environnement CNPG sauvegarde en continu vers un bucket GCS dédié via barman-cloud, avec une identité Workload Identity séparée par environnement. Ces commandes montrent les clusters Postgres réels et la dernière sauvegarde effective par environnement — la preuve que le mécanisme tourne, sans avoir besoin de rejouer une restauration complète pour tenir le format.
```

### 11:45 — Monitoring / Grafana — étape finale : replicas par environnement (2:15)

Ouvrir `http://localhost:3000` (déjà connecté via le port-forward des prérequis).

Pour chaque environnement, ouvrir le dashboard correspondant (`Dev Overview`, `Staging Overview`, `Prod Overview`) et pointer le panneau **« Replicas — hr-backend / hr-frontend (réel vs voulu) »** — présent sur les trois dashboards, alimenté par les requêtes PromQL `kube_deployment_status_replicas{namespace="<env>"}` et `kube_deployment_spec_replicas{namespace="<env>"}`. Staging et prod ont en plus un panneau **« HPA hr-backend (réel / min / max) »** ; dev affiche à la place un panneau **« Pas de HPA en dev »**, cohérent avec l'autoscaling désactivé.

Dire :
```text
Voici le nombre de replicas réel de chaque environnement, tel que rapporté par kube-state-metrics et affiché dans Grafana : un dashboard dédié par environnement, avec un panneau replicas réel versus voulu, un panneau HPA sur staging et prod, l'état des pods, les redémarrages de conteneurs, le CPU et la mémoire par pod, et le statut des PVC Postgres.
```

Puis, immédiatement, croiser ce que Grafana affiche avec la commande `kubectl` équivalente — c'est le point central de cette version du guide, ne pas sauter cette étape :
```bash
kubectl get deployment hr-backend hr-frontend -n dev
kubectl get deployment hr-backend hr-frontend -n staging
kubectl get deployment hr-backend hr-frontend -n prod
kubectl get hpa hr-backend -n staging
kubectl get hpa hr-backend -n prod
```

Dire :
```text
Le chiffre affiché dans Grafana pour chaque environnement correspond exactement à la colonne READY de cette commande kubectl, exécutée à l'instant — la preuve que le dashboard reflète l'état réel du cluster, et pas une valeur mise en cache ou figée dans un exemple de dashboard.
```

Basculer sur Prometheus (`http://localhost:9090`, onglet Graph) pour montrer la requête brute derrière le panneau, tapée en direct :
```promql
sum(kube_deployment_status_replicas{namespace=~"dev|staging|prod", deployment=~"hr-backend|hr-frontend"}) by (namespace, deployment)
```

Dire :
```text
Cette requête confirme, namespace par namespace et déploiement par déploiement, le nombre de pods réellement actifs — la même donnée que Grafana affiche, interrogée directement dans Prometheus. Les métriques applicatives fines et les logs ne sont pas encore câblés : un stack métriques uniquement pour l'instant, une extension identifiée mais pas encore priorisée.
```

### 14:00 — Checklist de vérification complète + conclusion (1:00)

```bash
# Replicas réels par environnement — le chiffre à annoncer à l'oral
kubectl get deployment -n dev -o custom-columns=NAME:.metadata.name,READY:.status.readyReplicas,DESIRED:.spec.replicas
kubectl get deployment -n staging -o custom-columns=NAME:.metadata.name,READY:.status.readyReplicas,DESIRED:.spec.replicas
kubectl get deployment -n prod -o custom-columns=NAME:.metadata.name,READY:.status.readyReplicas,DESIRED:.spec.replicas

# HPA revenu à ses bornes normales
kubectl get hpa -A

# PDB toujours actif
kubectl get pdb -A

# Les trois Applications ArgoCD applicatives + monitoring
kubectl get applications -n argocd

# Santé des trois environnements
curl -s http://localhost:8080 >/dev/null && echo "dev: OK"
curl -sk https://hr4you.com/api/health-check
curl -s http://hr4youp.com/api/health-check

# Nettoyage des éventuels pods de charge/test
kubectl get pods -A | grep -i loadgen || echo "aucun pod de charge résiduel"
```

Tout doit être propre :
```text
replicas dev  : hr-backend 1/1, hr-frontend 1/1
replicas staging : hr-backend entre 2/2 et 4/4 selon charge, hr-frontend 1/1
replicas prod : hr-backend entre 2/2 et 5/5 selon charge, hr-frontend 2/2
HPA dans ses bornes (pas de Pending faute de capacité)
PDB staging actif (ALLOWED DISRUPTIONS >= 0)
hr-dev / hr-staging Synced+Healthy, hr-prod Synced ou OutOfSync (normal hors sync manuel) + Healthy
kube-prometheus-stack Healthy (OutOfSync toléré, cf. prérequis)
health-check des 3 environnements 200/UP
aucun pod de charge résiduel
```

Dire :
```text
Pour conclure : une plateforme SIRH sur GKE, avec deux pipelines CI/CD distincts — l'un pour l'infrastructure Terraform, l'autre pour l'application — et un déploiement continu par ArgoCD sur trois environnements en pattern build-once puis promote. Résilience par PodDisruptionBudget et anti-affinité, autoscaling à la fois des pods et des noeuds, sauvegarde CNPG continue, et une supervision Prometheus/Grafana par environnement. Chaque affirmation faite dans cette vidéo a été appuyée par une commande exécutée en direct — le nombre de replicas annoncé pour dev, staging et prod n'est pas un chiffre de slide, c'est celui que Grafana et kubectl viennent de confirmer, ensemble, à l'instant.
```

---

## Ce qui reste volontairement compressé par rapport aux deux guides existants

- Démonstration Terraform live complète (nouvelle branche, PR, attente du plan, merge) — présente dans le guide long 45 min ; ici, remplacée par la preuve d'exécutions déjà réelles (`gh run list`/`gh run view`) + une revalidation locale à l'identique des jobs `validate`/`lint`, pour rester dans le temps imparti sans sacrifier la vérification par commande.
- Merge `develop`→`main` et création/promotion du tag `vX.Y.Z` — non rejoués en direct ; remplacés ici par une vérification par commande de l'état déjà en place, plutôt que par une preuve préparée en amont (différence assumée avec `annexe-video-demo-soutenance-15min.md`).
- Stress test HPA complet (montée + attente de la redescente ~5 min) — seul l'état courant est vérifié par commande ; si aucune charge n'a été lancée avant l'enregistrement, les commandes montreront simplement l'état au repos, ce qui reste une vérification honnête, pas un problème à cacher.
- Exécution live du script de restauration CNPG — seule la configuration et la dernière sauvegarde réelle sont vérifiées par commande, pas une restauration complète.
- Le Q&A complet et les ADR restent valables tels quels depuis `annexe-video-demo-soutenance.md` si des questions sont posées après la vidéo.
