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
│  Application "hr-staging"  (apps/children/staging.yaml)      │
│  source.path       : charts/hr-app                            │
│  source.helm.valueFiles : [values-staging.yaml]                │
│  destination.namespace  : staging                              │
│  ignoreDifferences  : Deployment /status/terminatingReplicas,  │
│                       StatefulSet /spec/volumeClaimTemplates/0/status │
│  syncPolicy.automated : prune=true, selfHeal=true                │
│  syncOptions : CreateNamespace=true, ServerSideApply=true,       │
│                RespectIgnoreDifferences=true                    │
│                                                                   │
│  → rend le chart Helm charts/hr-app avec values-staging.yaml     │
│    et applique le résultat dans le namespace "staging"           │
└───────────────────────────────────────────────────────────┘
```

`root-app` ([apps/root-app.yaml](../apps/root-app.yaml)) est l'unique Application appliquée manuellement (par le job `bootstrap-argocd` du workflow `workflow-infra.yml`, via `kubectl apply -f apps/root-app.yaml`, jamais par ArgoCD lui-même). Son rôle unique est de surveiller le dossier `apps/children/` : **chaque fichier YAML ajouté dans ce dossier devient automatiquement une nouvelle Application ArgoCD**, sans intervention manuelle sur le cluster. Aujourd'hui, `apps/children/` ne contient qu'un seul fichier : [staging.yaml](../apps/children/staging.yaml), qui définit l'Application `hr-staging`.

Ce découpage à deux niveaux évite l'auto-référence (un `root-app` qui pointerait sur son propre dossier créerait une boucle) et centralise la source de vérité : ajouter un environnement `production` ne demande qu'un commit `apps/children/production.yaml`, jamais une commande `kubectl` ou `argocd app create` manuelle sur le cluster.

## 🚀 Étape 1 — Comprendre `syncPolicy.automated`

Les deux Applications activent :
- **`prune: true`** — toute ressource présente dans le cluster mais absente du rendu Git (chart Helm + values) est supprimée automatiquement à la prochaine synchronisation.
- **`selfHeal: true`** — toute modification manuelle faite directement sur le cluster (`kubectl edit`, `kubectl scale`...) est annulée et remplacée par l'état déclaré dans Git à la synchronisation suivante.

Concrètement, **toute modification doit passer par un commit dans `repo-config`** — un `kubectl apply` ou `kubectl edit` manuel sur le namespace `staging` sera défait par ArgoCD dans les minutes qui suivent.

`hr-staging` ajoute en plus `ignoreDifferences` sur `group: apps, kind: Deployment, jsonPointers: [/status/terminatingReplicas]` — ce champ de status fluctue naturellement pendant les rolling updates et ferait apparaître le Deployment comme "OutOfSync" sans réelle dérive ; `RespectIgnoreDifferences=true` dans `syncOptions` est nécessaire pour que ce bloc soit effectivement pris en compte lors du calcul du diff.

Même mécanisme pour le `StatefulSet` `postgres` (`charts/hr-app/templates/statefulset-postgres.yaml`) : une deuxième entrée `ignoreDifferences` cible `group: apps, kind: StatefulSet, jsonPointers: [/spec/volumeClaimTemplates/0/status]`. Dès qu'un `PersistentVolumeClaim` est créé à partir d'un `volumeClaimTemplates`, l'API server injecte un sous-objet `status` (ex: `{phase: Pending}`) dans le template lui-même — un champ absent du manifeste rendu par Helm. Sans cette exception, ArgoCD compare éternellement "rien" (Git) à "status: {phase: Pending}" (cluster) et affiche le StatefulSet comme `OutOfSync` en permanence, même quand le pod est `Healthy` et qu'aucune vraie dérive n'existe — constaté en pratique juste après le premier déploiement de `postgres-0` sur le cluster staging.

## 🚀 Étape 2 — Observer une synchronisation avec `kubectl`

Aucune UI ArgoCD n'est exposée publiquement (pas d'ingress dédié) — on l'atteint par port-forward :

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
# Vue d'ensemble des deux Applications
kubectl get applications -n argocd

# Détail de la synchronisation de root-app
kubectl get application root-app -n argocd \
  -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'

# Détail de hr-staging (celle qui déploie réellement l'app)
kubectl get application hr-staging -n argocd \
  -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
# attendu : Synced Healthy

# Historique des opérations de sync
kubectl describe application hr-staging -n argocd

# Les pods réellement déployés
kubectl get pods -n staging
kubectl get deployment hr-backend hr-frontend -n staging \
  -o jsonpath='{.items[*].spec.template.spec.containers[0].image}{"\n"}'
```

Avec la CLI `argocd` (si installée et authentifiée sur le serveur port-forwardé) :

```bash
argocd app get hr-staging
argocd app sync hr-staging          # force une synchronisation immédiate
argocd app history hr-staging       # historique des déploiements
```

## ✅ Vérifier qu'un déploiement récent a bien été pris en compte

Après un push sur `repo-app` (voir [lifecycle-pipeline.md](lifecycle-pipeline.md)), le commit automatique `ci: update image tags to <SHA7>` doit apparaître dans `repo-config` :

```bash
git -C repo-config log --oneline -3
```

Puis, sur le cluster, l'image effectivement déployée doit correspondre au même tag :

```bash
kubectl get deployment hr-backend -n staging \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
# doit se terminer par :<même SHA7 que le commit ci:>
```

Si `hr-staging` reste `OutOfSync` plus de quelques minutes, forcer une resynchronisation manuelle (`argocd app sync hr-staging` ou `kubectl patch` pour déclencher un refresh) et vérifier les événements :

```bash
kubectl get events -n staging --sort-by='.lastTimestamp'
```

## 🚀 Étape 3 — Revenir en arrière (rollback)

ArgoCD étant en mode `selfHeal`, le seul rollback fiable et durable consiste à **revenir à un commit précédent dans `repo-config`**, pas à toucher directement au cluster (ArgoCD annulerait tout changement manuel).

```bash
# 1. Identifier le dernier commit "bon" (avant le tag problématique)
git -C repo-config log --oneline -- charts/hr-app/values-staging.yaml

# 2. Revenir à ce commit sans perdre l'historique (revert, pas reset --hard)
git -C repo-config revert <sha-du-commit-cassé> --no-edit
git -C repo-config push origin main
```

ArgoCD détecte le nouveau commit (revert) et resynchronise automatiquement le namespace `staging` avec les anciens tags d'image — exactement le même mécanisme que pour un déploiement normal, dans l'autre sens.

⚠️ Ne jamais faire de `git push --force` sur `repo-config` : `selfHeal` fonctionne par comparaison d'état, pas par rejouement d'historique — un simple `revert` propre est plus sûr et laisse une trace claire de pourquoi le rollback a eu lieu.

## 🔗 Pour la suite

- Comment le chart Helm `charts/hr-app` est structuré et testé : [guide-helm-chart.md](guide-helm-chart.md)
- Comment ArgoCD lui-même est installé sur le cluster (job `bootstrap-argocd`) : [guide-deploiement-infra.md](guide-deploiement-infra.md)
- Pourquoi ArgoCD (pull-based) a été choisi plutôt qu'un déploiement direct depuis la CI : [decisions-architecture.md](decisions-architecture.md)
