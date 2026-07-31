# Annexe — Guide vidéo de démonstration (soutenance)

Déroulé minuté pour enregistrer une vidéo de démo de la plateforme SIRH (Sopra HR Software).
Contrairement à un déroulé générique, tout ce qui suit correspond à des ressources **réellement présentes** dans ce monorepo (`repo-app`, `repo-infrastructure`, `repo-config`) — vérifiées avant rédaction. Il n'y a **pas** de Prometheus/Grafana/Alertmanager, **pas** d'Ansible, **pas** de GitLab CI, **pas** de scaling prédictif ici (voir `docs/decisions-architecture.md` ADR-3 pour le choix explicite de ne pas utiliser Ansible) — ne pas improviser ces sections en tournage.

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

# Confirmer le nombre de noeuds actuel (necessaire pour le timing de la Phase 6)
kubectl get nodes -o wide
```
Si `kubectl get nodes` ne montre qu'1 seul noeud, la Phase 6 (node autoscaling) prendra plus de temps à démarrer (le pool doit d'abord provisionner un 2e noeud spot) — prévoir une marge dans le montage vidéo plutôt que de couper en live.

Ajouter au fichier hosts local (si pas déjà fait) :
```
<IP_STATIQUE_ip-hr-staging>  hr-staging.local
```
(récupérer l'IP avec `gcloud compute addresses describe ip-hr-staging --global --format='value(address)'`)

---

## Minutage proposé

| Temps | Séquence |
|---|---|
| 0:00 | Introduction |
| 1:00 | Architecture Kubernetes réelle (namespaces, ArgoCD) |
| 4:00 | Terraform + CI/CD + GitOps |
| 8:00 | Application, Ingress, HTTPS |
| 11:00 | Load Balancing |
| 15:00 | HPA — stress test |
| 20:00 | PodDisruptionBudget |
| 23:00 | podAntiAffinity soft |
| 26:00 | Node autoscaling |
| 30:00 | Nettoyage final et conclusion |

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
On voit trois environnements applicatifs — dev, staging et prod — qui tournent comme des namespaces sur un seul cluster GKE, pas trois clusters séparés. Chacun a sa propre base Postgres CloudNativePG. ArgoCD, visible dans le namespace argocd, réconcilie en continu l'état du cluster avec ce qui est décrit dans le dépôt repo-config.
```

### 4:00 — Terraform, CI/CD, GitOps

Ouvrir les fichiers :
```text
repo-infrastructure/environments/staging/main.tf
repo-infrastructure/modules/gke/main.tf
repo-infrastructure/.github/workflows/workflow-infra.yml
repo-app/.github/workflows/ci.yml
repo-config/apps/root-app.yaml
repo-config/apps/children/staging.yaml
```

Dire :
```text
Terraform décrit l'infrastructure GCP : VPC, cluster GKE, IAM, Artifact Registry. Un seul workflow GitHub Actions gère tout le cycle de vie infra : validation, lint, scan de sécurité Checkov, plan, apply, puis bootstrap d'ArgoCD. Côté application, le CI de repo-app compile, teste, scanne les images avec Trivy, et pousse sur le registre via une identité fédérée OIDC — pas de clé JSON stockée. ArgoCD ensuite tire l'état désiré depuis repo-config et l'applique au cluster : c'est du GitOps pull-based, pas un déploiement poussé directement par la CI.
```

Note pour les questions du jury : ce projet n'utilise **pas** Ansible (choix documenté, ADR-3 — cluster managé GKE, aucune VM à configurer) et **pas** GitLab CI (GitHub Actions).

### 8:00 — Application, Ingress, HTTPS

```bash
# Acces direct (sans passer par le LB) pour valider que le pod repond
kubectl port-forward svc/hr-backend -n staging 8081:8081 &
curl http://localhost:8081/api/health-check

# Acces via l'Ingress externe, en HTTPS (TLS via une CA locale cert-manager)
curl -kI https://hr-staging.local/api/health-check
```

Dire :
```text
Le backend expose un endpoint de santé utilisé à la fois comme readiness et liveness probe Kubernetes. L'application est aussi exposée en HTTPS via un Ingress GKE natif, avec un certificat émis par une autorité de certification locale gérée par cert-manager — pas de domaine public ici, la résolution se fait via le fichier hosts local.
```

Nettoyage avant de continuer :
```bash
kill %1 2>/dev/null || true
```

### 11:00 — Load Balancing

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
for i in $(seq 1 200); do curl -s -o /dev/null -k https://hr-staging.local/api/employees; done
```

Dire :
```text
Si le load balancing ne fonctionnait pas, un seul pod absorberait tout le trafic. Ici les deux montent en charge quasi simultanément — c'est la répartition réelle, pas juste deux pods qui existent côte à côte.
```

### 15:00 — HPA — stress test

```bash
# Terminal A — observer le HPA et les pods en direct
kubectl get hpa hr-backend -n staging -w
kubectl get pods -n staging -l app=hr-backend -w

# Terminal B — generateur de charge local (repo-app/scripts/load-test.sh)
./repo-app/scripts/load-test.sh --url https://hr-staging.local/api/employees --concurrency 20 --duration 180
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

### 20:00 — PodDisruptionBudget

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

### 23:00 — podAntiAffinity soft

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

### 26:00 — Node autoscaling

```bash
kubectl get nodes -w
kubectl get events -n staging --sort-by=.lastTimestamp | grep -iE "FailedScheduling|TriggeredScaleUp"
```

Dire :
```text
En poussant le stress test plus fort ou plus longtemps que pour la simple démo HPA, les 4 pods backend au maximum dépassent la capacité CPU des noeuds existants. Des pods restent en attente le temps que le pool de noeuds, lui aussi en autoscaling, en provisionne un nouveau — ici un noeud spot supplémentaire.
```

(Réutiliser la boucle `kubectl run loadgen-*` de la Phase HPA, avec un `end=$(($(date +%s)+400))` plus long, pour maintenir la pression le temps que le nouveau noeud apparaisse.)

### 30:00 — Nettoyage final et conclusion

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

### Pourquoi GKE managé plutôt que des VM à configurer avec Ansible ?
```text
Le cluster est entièrement managé par Google — les noeuds sont provisionnés, mis à jour et réparés automatiquement. Il n'y a aucune VM applicative à configurer après coup, donc Ansible n'apporterait rien ici. C'est un choix documenté, pas un oubli.
```

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
La configuration ArgoCD ignore explicitement le champ spec.replicas des Deployments (ignoreDifferences). Sans ça, le mode selfHeal comparerait le nombre de pods réel au nombre déclaré dans Git et le forcerait en arrière à chaque synchronisation, ce qui annulerait le travail du HPA.
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
Ce n'est pas encore implémenté — c'est documenté comme tel dans le README. La supervision actuelle repose sur les probes readiness/liveness Kubernetes et sur Cloud Monitoring/Logging de GCP, déjà câblés via les rôles IAM du cluster. Ajouter une stack Prometheus/Grafana est une amélioration identifiée, pas encore priorisée.
```

### Pourquoi ArgoCD pull-based plutôt qu'un déploiement direct depuis la CI ?
```text
Le cluster n'a jamais besoin d'exposer de credentials à la CI pour se faire déployer dessus — c'est ArgoCD, à l'intérieur du cluster, qui va chercher l'état désiré dans Git. Ça réduit la surface d'attaque et donne un historique Git complet de tout ce qui a été déployé.
```

### Pourquoi un cluster zonal et des VM spot plutôt qu'un cluster régional avec des VM standards ?
```text
Choix de coût pour un environnement de staging/démo. Les VM spot peuvent être préemptées à tout moment — c'est un compromis assumé, pas adapté à un vrai environnement de production sans redondance supplémentaire.
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
curl -sk https://hr-staging.local/api/health-check
```

Tout doit être propre :
```text
pods Ready dans les 3 namespaces
HPA revenu à son minimum (2/4 en staging)
PDB actif (ALLOWED DISRUPTIONS: 1)
les 3 Applications ArgoCD Synced + Healthy
health-check retourne 200/UP
aucun pod loadgen residuel
```

---

## Résumé final à dire

```text
Cette vidéo montre une plateforme SIRH déployée sur Google Kubernetes Engine, avec une infrastructure décrite en Terraform et un déploiement continu piloté par ArgoCD en GitOps. Le CI/CD GitHub Actions automatise la construction, le scan de sécurité et la mise à jour de la configuration. Le load balancing natif de l'Ingress GKE répartit le trafic entre plusieurs pods API. Le HorizontalPodAutoscaler ajuste le nombre de pods selon la charge CPU observée, et le pool de noeuds s'adapte lui-même en dessous quand c'est nécessaire. Un PodDisruptionBudget protège la disponibilité pendant les opérations de maintenance, et une contrainte d'anti-affinité répartit les pods entre noeuds sur une base de compromis. L'ensemble reste cohérent avec ce qui est versionné dans Git, sans intervention manuelle sur le cluster.
```
