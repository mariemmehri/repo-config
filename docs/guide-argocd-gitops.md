# Guide ArgoCD / GitOps

## 🎯 Objectif

Expliquer comment fonctionne le pattern **App-of-Apps** dans ce projet (`apps/root-app.yaml` + `apps/children/*.yaml`), comment observer une synchronisation avec `kubectl`, et comment revenir à un état antérieur en cas de problème. Ce document est autonome — il ne suppose pas que tu aies lu les autres guides.

---

## ⚙️ App-of-Apps : les deux niveaux

```
              argocd namespace (cluster GKE)
┌───────────────────────────────────────────────────────────┐
│  Application "root-app"                                    │
│  source.repoURL : https://github.com/mariemmehri/repo-config│
│  source.path    : apps/children                             │
│  destination.namespace : argocd                             │
│  syncPolicy.automated : prune=true, selfHeal=true            │
│  syncOptions : CreateNamespace=true                           │
│                                                               │
│  → surveille le DOSSIER apps/children/, pas un fichier unique │
└─────────────────────────┬─────────────────────────────────┘
                          │ génère/gère une Application par fichier trouvé
                          ▼
┌───────────────────────────────────────────────────────────┐
│  hr-dev / hr-staging / hr-prod   (apps/children/{dev,       │
│  staging,prod}.yaml)                                        │
│  source.path       : charts/hr-app                            │
│  source.helm.valueFiles : [values-<env>.yaml]                  │
│  destination.namespace  : <env>                                │
│  ignoreDifferences  : Deployment /status/terminatingReplicas,  │
│                       Deployment /spec/replicas,                │
│                       Ingress /metadata/annotations,             │
│                       Ingress /metadata/finalizers                │
│  syncPolicy.automated : prune=true, selfHeal=true                │
│                         — SAUF hr-prod, qui n'a AUCUN bloc         │
│                         automated (sync manuel uniquement)         │
│                                                                   │
│  + Applications CNPG : cert-manager, cnpg-operator,                │
│    cnpg-plugin-barman-cloud, cnpg-cluster-<env>,                    │
│    cnpg-network-policy-<env> — une famille par composant PostgreSQL │
└───────────────────────────────────────────────────────────┘
```

`root-app` ([apps/root-app.yaml](../apps/root-app.yaml)) est l'unique Application appliquée manuellement (par le job `bootstrap-argocd` du workflow `workflow-infra.yml`, via `kubectl apply -f apps/root-app.yaml`, jamais par ArgoCD lui-même). Son rôle unique est de surveiller le dossier `apps/children/` : **chaque fichier YAML ajouté dans ce dossier devient automatiquement une nouvelle Application ArgoCD**, sans intervention manuelle sur le cluster. Aujourd'hui, `apps/children/` contient les trois Applications applicatives (`dev.yaml`/`staging.yaml`/`prod.yaml`, définissant `hr-dev`/`hr-staging`/`hr-prod`) **et** la famille d'Applications CloudNativePG (opérateur, plugin de backup, un `Cluster` Postgres et une `NetworkPolicy` par environnement, plus `cert-manager` en prérequis TLS du plugin de backup).

Ce découpage à deux niveaux évite l'auto-référence (un `root-app` qui pointerait sur son propre dossier créerait une boucle) et centralise la source de vérité : ajouter un nouvel environnement applicatif ne demande qu'un commit `apps/children/<env>.yaml` + `charts/hr-app/values-<env>.yaml`, jamais une commande `kubectl` ou `argocd app create` manuelle sur le cluster.

## 🚀 Étape 1 — Comprendre `syncPolicy.automated`

`hr-dev` et `hr-staging` activent :
- **`prune: true`** — toute ressource présente dans le cluster mais absente du rendu Git (chart Helm + values) est supprimée automatiquement à la prochaine synchronisation.
- **`selfHeal: true`** — toute modification manuelle faite directement sur le cluster (`kubectl edit`, `kubectl scale`...) est annulée et remplacée par l'état déclaré dans Git à la synchronisation suivante.

**`hr-prod` n'a aucun bloc `automated`** — ni `prune`, ni `selfHeal`. Un commit sur `values-prod.yaml` (que ce soit une promotion `promote-prod.yml` ou une modification manuelle du chart) laisse seulement `hr-prod` `OutOfSync` ; le déploiement effectif nécessite un `argocd app sync hr-prod` explicite (ou l'UI). Un hand-edit sur le namespace `prod` n'est donc pas non plus automatiquement reverté — plus petit risque que sur `dev`/`staging`, pas une protection pour autant.

Concrètement, **toute modification doit passer par un commit dans `repo-config`** — un `kubectl apply`/`kubectl edit` manuel sur `dev`/`staging` sera défait par ArgoCD dans les minutes qui suivent ; sur `prod`, il persistera jusqu'au prochain sync manuel.

Les trois Applications applicatives portent en plus quatre entrées `ignoreDifferences`, load-bearing, pas décoratives :
- `Deployment /status/terminatingReplicas` — fluctue naturellement pendant les rolling updates, ferait apparaître le Deployment `OutOfSync` sans réelle dérive.
- `Deployment /spec/replicas` — nécessaire depuis l'ajout du `HorizontalPodAutoscaler` `hr-backend` (`charts/hr-app/templates/hpa-backend.yaml`, voir [guide-helm-chart.md](guide-helm-chart.md)) : `deployment-backend.yaml` déclare un `spec.replicas` statique, mais une fois le HPA actif c'est lui qui patch ce champ selon la charge CPU. Sans cette exception, `selfHeal: true` reverterait le scaling du HPA à chaque cycle de réconciliation. Cette exception s'applique à **tous** les Deployments du chart (donc aussi `hr-frontend`, qui n'a pas de HPA).
- `Ingress /metadata/annotations` et `Ingress /metadata/finalizers` — ajoutées avec l'Ingress GCE natif. Le contrôleur Ingress de GKE écrit lui-même des annotations (bookkeeping NEG/backend) et un finalizer sur l'objet `Ingress` après création — sans cette exception, chaque cycle de réconciliation verrait l'Ingress `OutOfSync` et `selfHeal` se battrait indéfiniment avec le contrôleur.

`RespectIgnoreDifferences=true` dans `syncOptions` est nécessaire pour que ces blocs soient effectivement pris en compte lors du calcul du diff.

## 🚀 Étape 2 — Observer une synchronisation avec `kubectl`

Aucune UI ArgoCD n'est exposée publiquement (pas d'ingress dédié pour ArgoCD lui-même) — on l'atteint par port-forward :

```bash
gcloud container clusters get-credentials gke-staging-pfe \
  --region europe-west1-b --project pfe-2026-495220

kubectl port-forward svc/argocd-server -n argocd 9089:80
# UI : http://localhost:9089, utilisateur admin

kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
```

Pour suivre l'état sans UI :

```bash
# Vue d'ensemble de toutes les Applications (app + CNPG)
kubectl get applications -n argocd

# Détail de la synchronisation de root-app
kubectl get application root-app -n argocd \
  -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'

# Détail d'un environnement applicatif
kubectl get application hr-staging -n argocd \
  -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
# attendu : Synced Healthy (hr-prod peut légitimement rester OutOfSync entre une
# promotion et le prochain sync manuel — ce n'est pas un bug)

# Les pods réellement déployés
kubectl get pods -n staging      # remplacer par dev / prod selon l'environnement
kubectl get deployment hr-backend hr-frontend -n staging \
  -o jsonpath='{.items[*].spec.template.spec.containers[0].image}{"\n"}'
```

Avec la CLI `argocd` (si installée et authentifiée sur le serveur port-forwardé) :

```bash
argocd app get hr-staging
argocd app sync hr-staging          # force une synchronisation immédiate
argocd app sync hr-prod             # LE mécanisme de déploiement en prod — pas juste un "forcer"
argocd app history hr-staging       # historique des déploiements
```

## ✅ Vérifier qu'un déploiement récent a bien été pris en compte

Après un push sur `repo-app` (voir [lifecycle-pipeline.md](lifecycle-pipeline.md)), le commit automatique `ci: update <env> image tags to <SHA7>` doit apparaître dans `repo-config` :

```bash
git -C repo-config log --oneline -3
```

Puis, sur le cluster, l'image effectivement déployée doit correspondre au même tag :

```bash
kubectl get deployment hr-backend -n staging \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Si `hr-dev`/`hr-staging` reste `OutOfSync` plus de quelques minutes, forcer une resynchronisation manuelle et vérifier les événements :

```bash
kubectl get events -n staging --sort-by='.lastTimestamp'
```

## 🚀 Étape 3 — Revenir en arrière (rollback)

Pour `dev`/`staging` (mode `selfHeal`), le seul rollback fiable et durable consiste à **revenir à un commit précédent dans `repo-config`**, pas à toucher directement au cluster (ArgoCD annulerait tout changement manuel). Pour `prod` (pas de `selfHeal`), la même méthode marche, mais un `argocd app sync hr-prod` reste nécessaire après le revert pour que le déploiement change réellement.

```bash
# 1. Identifier le dernier commit "bon" (avant le tag problématique)
git -C repo-config log --oneline -- charts/hr-app/values-staging.yaml

# 2. Revenir à ce commit sans perdre l'historique (revert, pas reset --hard)
git -C repo-config revert <sha-du-commit-cassé> --no-edit
git -C repo-config push origin main
```

ArgoCD détecte le nouveau commit (revert) et resynchronise automatiquement `dev`/`staging` avec les anciens tags d'image — exactement le même mécanisme que pour un déploiement normal, dans l'autre sens. Pour `prod`, exécuter ensuite `argocd app sync hr-prod`.

⚠️ Ne jamais faire de `git push --force` sur `repo-config` : `selfHeal` fonctionne par comparaison d'état, pas par rejouement d'historique — un simple `revert` propre est plus sûr et laisse une trace claire de pourquoi le rollback a eu lieu.

## 🔗 Pour la suite

- Comment le chart Helm `charts/hr-app` est structuré et testé, y compris l'Ingress GCE natif : [guide-helm-chart.md](guide-helm-chart.md)
- Comment ArgoCD lui-même est installé sur le cluster (job `bootstrap-argocd`) : [guide-deploiement-infra.md](guide-deploiement-infra.md)
- Pourquoi ArgoCD (pull-based) a été choisi plutôt qu'un déploiement direct depuis la CI : [decisions-architecture.md](decisions-architecture.md)
- Le flux complet de promotion prod (deux gates humains) : [lifecycle-pipeline.md](lifecycle-pipeline.md)
