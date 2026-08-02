# Annexe — Guide vidéo de démonstration (soutenance) — Version courte (15 min)

Variante resserrée de `annexe-video-demo-soutenance.md` (qui dure environ 45 minutes une fois jouée en entier). Ce fichier ne remplace pas l'original — c'est une trame alternative pour un format contraint à 15 minutes maximum, qui intègre aussi le stack de supervision (Prometheus/Grafana/Alertmanager) ajouté après la rédaction du guide long.

**Différence de fond avec le guide long** : ici, la plupart des démonstrations « live » (drain de nœud, cordon manuel, attente de la redescente HPA après ~5 min de stabilisation) sont remplacées par une explication orale + une commande `kubectl get` qui montre l'état déjà obtenu lors d'un test préalable, plutôt que rejouées intégralement en direct. Le compromis assumé : moins de suspense en direct, mais un timing tenable.

**Mise à jour importante** : contrairement à ce qu'indique le guide long, il y a désormais un stack Prometheus/Grafana/Alertmanager réel, déployé via ArgoCD (`apps/children/monitoring.yaml`, chart `kube-prometheus-stack` 88.0.1), dans le namespace `monitoring`, avec un dashboard Grafana dédié par environnement (`dev-overview`, `staging-overview`, `prod-overview`).

---

## Prérequis avant d'enregistrer

Mêmes prérequis que le guide long (credentials cluster, `git status` propre sur les 3 repos, `kubectl get applications -n argocd` propre), plus :

```bash
# Confirmer que le stack de monitoring est en place
kubectl get pods -n monitoring
kubectl get application kube-prometheus-stack -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
```
Attendu : tous les pods `Running`, et `OutOfSync Healthy` (pas `Synced`) — c'est normal, pas un bug à expliquer en tournage si la question ne vient pas spontanément (GKE réinjecte une annotation `cloud.google.com/neg` sur chaque Service du namespace `monitoring`, qu'ArgoCD ne peut jamais faire correspondre exactement ; ça ne casse rien).

Lancer les deux port-forwards avant l'enregistrement, dans deux terminaux séparés à garder ouverts :
```bash
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80

pwd grafana:rcL7MGoaafKC4VdhWuIektsn

kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090


kubectl port-forward svc/argocd-server -n argocd 9089:80
pwd argocd :BhMaFG8dwZVMlApK   

kubectl port-forward svc/hr-frontend -n dev 8090:80



```


**Préparer aussi, avant l'enregistrement, la preuve visible dev/staging/prod** (voir section CI/CD 1:45) — pour éviter de rejouer en direct un merge `develop`→`main` et une promotion taguée, ce qui ferait exploser le format 15 min :
1. Modifier `repo-app/frontend/src/app/app.component.html` ligne 2 : le logo d'en-tête `SoproRH` → `SoproRH Cloud`.
2. Pousser sur `develop`, attendre `hr-dev` `Synced`/`Healthy`, merger `develop` dans `main` (déclenche `staging`), attendre `hr-staging` `Synced`/`Healthy`.
3. Créer un tag `vX.Y.Z` sur ce commit (déclenche `promote-prod.yml`), valider le gate humain `production`, puis `argocd app sync hr-prod`.
4. Vérifier que les trois frontends affichent bien `SoproRH Cloud` avant l'enregistrement :
   ```bash
   kubectl port-forward svc/hr-frontend -n dev 8080:80 &
   curl -s http://localhost:8080 | grep -o 'SoproRH[^<]*'   # dev : port-forward direct, pas d'Ingress utilisé ici
   curl -s http://hr-staging.local | grep -o 'SoproRH[^<]*'
   curl -s http://hr-prod.local | grep -o 'SoproRH[^<]*'
   ```
5. Garder `SoproRH Live` comme **deuxième** texte, différent, pour le changement rejoué en direct sur `dev` uniquement pendant l'enregistrement — la différence entre `SoproRH Cloud` (staging/prod, propagé avant) et `SoproRH Live` (dev, en direct) est volontaire : elle prouve que chaque environnement a son propre cycle de déploiement indépendant.

---

## Minutage proposé (15:00 max)

| Temps | Séquence | Durée |
|---|---|---|
| 0:00 | Introduction | 0:30 |
| 0:30 | Architecture rapide (namespaces, ArgoCD, monitoring inclus) | 1:15 |
| 1:45 | CI/CD applicatif — GitOps live sur dev + preuve déjà obtenue staging/prod | 4:00 |
| 5:45 | Monitoring — Prometheus + Grafana par environnement | 2:15 |
| 8:00 | HPA — stress test compressé | 2:15 |
| 10:15 | Résilience en rafale — PDB / anti-affinity / node autoscaling | 2:15 |
| 12:30 | Backup/restore CNPG (mentionné, pas exécuté en direct) | 1:00 |
| 13:30 | Nettoyage et conclusion | 1:30 |

Total : 15:00.

---

### 0:00 — Introduction (30s)

Dire :
```text
Je vais présenter, en quinze minutes, une plateforme SIRH cloud-native sur Google Kubernetes Engine : Infrastructure as Code Terraform, CI/CD GitHub Actions, déploiement continu GitOps via ArgoCD, et supervision Prometheus/Grafana.
```

### 0:30 — Architecture rapide (1:15)

```bash
kubectl get ns
kubectl get applications -n argocd
```

Attendu pour `get ns` : `dev`, `staging`, `prod`, `argocd`, `cnpg-system`, `cert-manager`, `monitoring`.

Dire :
```text
Trois environnements applicatifs — dev, staging, prod — comme namespaces sur un seul cluster GKE, chacun avec sa propre base Postgres CloudNativePG. ArgoCD réconcilie en continu l'état du cluster avec ce qui est décrit dans repo-config — j'y reviens dans un instant. Depuis peu, un namespace monitoring supervise les trois environnements avec Prometheus et Grafana.
```

### 1:45 — CI/CD applicatif : GitOps live sur dev + preuve déjà obtenue staging/prod (4:00)

**Dev — en direct :**
```bash
cd repo-app
git checkout develop
git pull
```
Modifier `frontend/src/app/app.component.html` ligne 2 : `SoproRH Cloud` → `SoproRH Live`.
```bash
git add frontend/src/app/app.component.html
git commit -m "demo: update header label to prove live GitOps flow"
git push origin develop
```

Dire pendant que la CI tourne :
```text
Le push construit les images backend et frontend, les scanne avec Trivy, les pousse sur Artifact Registry, puis patche automatiquement values-dev.yaml dans repo-config avec le nouveau tag d'image.
```

Après le job `deploy` :
```bash
cd ../repo-config && git pull && git log --oneline -2
kubectl get application hr-dev -n argocd
kubectl port-forward svc/hr-frontend -n dev 8080:80
```

Ouvrir (ou rafraîchir) le navigateur sur `http://localhost:8080` (port-forward direct sur le Service, pas d'Ingress utilisé ici) :

Dire :
```text
ArgoCD détecte le commit et synchronise automatiquement le namespace dev, sans intervention manuelle — et voici le changement, réellement visible dans l'en-tête : "SoproRH Live". Du push jusqu'au rendu dans le navigateur, entièrement piloté par Git.
```

**Staging et prod — le même mécanisme, preuve déjà obtenue avant l'enregistrement :**

Dire :
```text
Ce même commit, une fois mergé sur main, déclenche exactement le même pipeline pour l'environnement staging. Pour la production, c'est un tag de version qui déclenche la promotion, avec un gate humain obligatoire avant tout déploiement — je ne rejoue pas ces deux étapes en direct pour tenir le format, mais voici le résultat obtenu il y a quelques minutes avec ce même mécanisme.
```

```bash
curl -s http://hr-staging.local | grep -o 'SoproRH[^<]*'
curl -s http://hr-prod.local | grep -o 'SoproRH[^<]*'
kubectl get application hr-staging -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
kubectl get application hr-prod -n argocd -o jsonpath='{.status.sync.status} {.status.health.status}{"\n"}'
```

Attendu : `staging` et `prod` affichent `SoproRH Cloud` (la version précédente du changement, propagée avant l'enregistrement) — volontairement différent du `SoproRH Live` tout juste déployé sur dev, ce qui prouve justement que chaque environnement a son propre cycle de déploiement indépendant.

Dire :
```text
Staging est Synced et Healthy, en synchronisation automatique comme dev. Prod, lui, reste volontairement en attente d'un sync manuel après chaque promotion — c'est exactement pourquoi je ne l'ai pas rejoué en direct : chaque déploiement en production nécessite une action humaine explicite, jamais automatique.
```

### 5:45 — Monitoring : Prometheus + Grafana (2:15)

Montrer dans le navigateur (`http://localhost:3000`, déjà connecté) :
- `dev-overview`, `staging-overview`, `prod-overview` — un dashboard par environnement.

Dire :
```text
Chaque environnement a son propre dashboard Grafana : état des pods, redémarrages de conteneurs, replicas réels versus voulus, CPU et mémoire par pod, statut des PVC Postgres. Staging et prod ont en plus un panneau HPA — dev n'en a pas, l'autoscaling y est désactivé volontairement.
```

Basculer sur Prometheus (`http://localhost:9090`, onglet Graph), taper en direct :
```promql
sum(kube_pod_status_phase{phase="Running", namespace=~"dev|staging|prod"}) by (namespace)
```

Dire :
```text
Cette requête compte les pods actifs par environnement en direct — la preuve que Prometheus scrape bien les trois namespaces séparément, sans configuration supplémentaire : kube-state-metrics et les métriques kubelet couvrent déjà toute l'infrastructure. Les métriques applicatives fines — latence HTTP du backend, par exemple — ne sont pas encore câblées, ni les logs : c'est un stack métriques uniquement pour l'instant, une extension identifiée mais pas encore priorisée.
```

### 8:00 — HPA : stress test compressé (2:15)

```bash
kubectl get hpa hr-backend -n staging
```

Dire :
```text
Le seuil HPA est fixé à 70% de CPU. J'ai lancé une charge il y a quelques minutes pour ne pas attendre en direct — voici le résultat.
```

Montrer un `get hpa -w` déjà en cours depuis avant l'enregistrement (ou une capture d'écran/second flux) montrant la montée de 2 à 4 replicas, puis :
```bash
kubectl get pods -n staging -l app=hr-backend
```

Dire :
```text
Sous charge, le pourcentage dépasse le seuil et le nombre de replicas grimpe progressivement vers la limite maximale de 4. La redescente après l'arrêt de la charge prend environ 5 minutes de stabilisation — trop long pour ce format, je ne l'attends pas en direct.
```

### 10:15 — Résilience en rafale (2:15)

```bash
kubectl get pdb hr-backend -n staging
kubectl get pods -n staging -l app=hr-backend -o wide
kubectl get nodes
```

Dire (les trois mécanismes enchaînés sans re-tester chacun en direct) :
```text
Trois protections combinées ici. Le PodDisruptionBudget garantit qu'au moins un pod backend reste disponible pendant un drain de nœud volontaire — testé au préalable, Kubernetes attend qu'un remplaçant soit prêt avant de continuer. L'anti-affinité entre les deux pods backend est en mode "preferred" — souple : elle répartit sur des nœuds différents quand c'est possible, sans bloquer le déploiement si un seul nœud est disponible, ce qui compte vu le pool réduit de VM spot. Et le pool de nœuds lui-même est en autoscaling : sous forte charge, un nœud spot supplémentaire est provisionné automatiquement quand les pods en attente dépassent la capacité existante.
```

### 12:30 — Backup/restore CNPG (1:00, mentionné)
Ce script est en Bash (shebang #!/usr/bin/env bash, syntaxe [[ ]], trap, heredocs) — PowerShell ne peut pas l'exécuter nativement. Il faut l'invoquer via Git Bash (déjà présent sur ta machine, c'est ce que le tool Bash utilise) :

cd "repo-infrastructure"
bash ./scripts/cnpg-restore-single-row.sh --live


Dire :
```text
Chaque environnement CNPG sauvegarde en continu vers un bucket GCS dédié via barman-cloud, avec une identité Workload Identity séparée par environnement. Ce script, déjà validé, restaure une sauvegarde dans un cluster Postgres jetable pour récupérer une ligne supprimée par erreur, sans jamais toucher au cluster réellement servi — je ne le rejoue pas en direct pour tenir le timing, mais le rapport de drill dans docs/backup-restore-drill-report.md documente le résultat.
```

### 13:30 — Nettoyage et conclusion (1:30)

```bash
kubectl get pods -n staging -l app=hr-backend
kubectl get hpa hr-backend -n staging
kubectl get applications -n argocd
kubectl get application kube-prometheus-stack -n argocd -o jsonpath='{.status.health.status}{"\n"}'
```

Dire :
```text
Pour conclure : une plateforme SIRH sur GKE, infrastructure Terraform modulaire, déploiement GitOps par ArgoCD sur trois environnements en pattern build-once puis promote, load balancing et autoscaling natifs Kubernetes, résilience par PodDisruptionBudget et anti-affinité, sauvegarde CNPG validée par un drill, et désormais une supervision Prometheus/Grafana par environnement. L'ensemble reste piloté depuis Git, sans intervention manuelle sur le cluster en dehors des gates humaines volontaires de la production.
```

---

## Ce qui est volontairement coupé par rapport au guide long (45 min)

Pour repasser à la version longue si le format le permet un jour, ces sections existent déjà, prêtes à réinsérer telles quelles depuis `annexe-video-demo-soutenance.md` :
- Démonstration Terraform live (plan → Checkov → PR comment) — coupée ici, seule l'architecture modulaire est mentionnée à l'oral.
- Test de load balancing dédié (charge modérée, deux pods qui montent ensemble) — remplacé par la mention orale pendant HPA.
- Ingress/HTTPS en démonstration séparée (`curl` direct + via l'Ingress) — non montré ici.
- Drain de nœud et cordon manuel en direct pour prouver le PDB et l'anti-affinité — remplacés par l'explication orale + un état déjà obtenu.
- Exécution live du script de restauration CNPG (mode `--live`) — mentionné seulement.
- Merge `develop`→`main` et création/promotion du tag `vX.Y.Z` vers prod rejoués intégralement en direct — remplacés par une preuve déjà obtenue (le même texte d'en-tête, `SoproRH Cloud`, propagé sur staging et prod avant l'enregistrement ; seul dev reçoit un changement, `SoproRH Live`, en direct).
- Le Q&A complet (`## Questions probables et réponses courtes`) et la checklist finale détaillée du guide long restent valables tels quels si des questions sont posées après la vidéo — pas besoin de les dupliquer ici.

---

## Correction à ne pas oublier dans le guide long

`annexe-video-demo-soutenance.md` affirme encore (ligne 4 et section Q&A "Pourquoi pas de Prometheus/Grafana ?") qu'il n'y a pas de supervision Prometheus/Grafana — c'est devenu faux depuis l'ajout du stack `kube-prometheus-stack`. Ce fichier-ci ne corrige pas l'original (demande explicite de ne pas le modifier) ; à traiter séparément si le guide long doit rester à jour.
