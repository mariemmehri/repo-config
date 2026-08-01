# Guide observabilité — Prometheus / Grafana

## 🎯 Objectif

Documenter la stack d'observabilité (`kube-prometheus-stack`, déployée 100% en GitOps via ArgoCD) : ce qui est réellement en place, pourquoi chaque choix de dimensionnement/scope a été fait, et — aussi important — ce qui n'existe **pas encore**, pour éviter toute fausse impression de couverture. Même esprit que [guide-securite.md](guide-securite.md) : pas de description abstraite, chaque choix renvoie à un fichier réel.

---

## ⚙️ Ce qui est déployé

Deux Applications ArgoCD dans `apps/children/`, toutes deux `sync-wave: "-1"`/`"0"` — même famille que `cnpg-operator.yaml`/`cnpg-cluster-staging.yaml`, pour la même raison (CRD avant consommateur) :

- **`monitoring.yaml`** (wave `-1`) — chart Helm `kube-prometheus-stack` (`prometheus-community`, version épinglée `87.21.0`), namespace `monitoring`. Contient Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, et les CRD (`ServiceMonitor`, `PrometheusRule`, ...).
- **`monitoring-manifests.yaml`** (wave `0`, `directory.recurse: true`) — Application "manifests bruts" pointant sur `manifests/monitoring/`, même mécanisme que `cnpg-network-policy.yaml`. Contient :
  - `servicemonitor-hr-backend.yaml` — scrape `hr-backend` (namespace **`staging` uniquement**, voir plus bas)
  - `networkpolicy-monitoring.yaml` — 5 objets `NetworkPolicy` pour le namespace `monitoring`
  - `dashboards/configmap-hr-platform-dashboard.yaml` (+ `dashboards/hr-platform-dashboard.json`, exclu du parsing ArgoCD via `directory.exclude` — ce n'est pas un manifest Kubernetes)

Côté application (`repo-app`) : `backend/pom.xml` ajoute `spring-boot-starter-actuator` + `micrometer-registry-prometheus` ; `application.properties` expose `/actuator/prometheus` sur le même port (8081) que le reste de l'API — additif, `HealthController`/`/api/health-check` (la sonde K8s) n'est pas touché.

## ⚙️ Scope volontairement limité à `staging`

`servicemonitor-hr-backend.yaml`'s `namespaceSelector.matchNames` ne contient que `staging`, pas `dev`/`prod`. Décision explicite (pas un oubli) : la capacité réelle du node pool (`NODE_VM_SIZE=e2-standard-2`, `NODE_COUNT=3`, `MAX_NODE_COUNT=5`, valeurs GitHub `vars.*` confirmées) n'avait jamais été vérifiée avant l'ajout de cette stack. Avec `dev`+`staging`+`prod` sur le même cluster (3 CNPG en QoS *Guaranteed*, cf. `cnpg-cluster-*.yaml`), la capacité **agrégée** suffit largement, mais rien ne garantit la répartition par nœud sous préemption spot (`spot = true`). Valider l'usage réel en `staging` seule avant d'élargir. Pour étendre à `dev`/`prod` : ajouter les namespaces dans `servicemonitor-hr-backend.yaml`'s `matchNames` **et** les namespaces symétriques dans `networkpolicy-monitoring.yaml`'s règle `monitoring-egress-hr-backend-staging` (à renommer/dupliquer) — les deux doivent rester en lockstep, sinon Prometheus a une cible configurée qu'il ne peut pas atteindre.

## ⚙️ Dimensionnement (resources)

| Composant | Requests | Limits |
|---|---|---|
| Prometheus | 100m / 512Mi | 500m / 1Gi |
| Alertmanager | 50m / 64Mi | 100m / 128Mi |
| Grafana | 50m / 128Mi | 200m / 256Mi |
| node-exporter (par nœud) | 50m / 50Mi | 100m / 100Mi |
| kube-state-metrics | 50m / 100Mi | 100m / 200Mi |
| **Total (hors node-exporter, qui tourne par nœud)** | **~250m / ~804Mi** | |

Comparé à la capacité allouable d'un nœud `e2-standard-2` (~1930m CPU / ~5.7-6.1Gi mémoire selon la formule de réservation GKE — non vérifié en live via `kubectl describe node`, à confirmer avant un déploiement réel) et au reste de la charge déjà existante sur le cluster (3× CNPG *Guaranteed* 250m/512Mi, hr-backend/hr-frontend ×3 environnements, ArgoCD/cert-manager/cnpg-operator non dimensionnés explicitement ailleurs dans ce repo), l'ajout reste largement dans le budget agrégé du cluster (~28% CPU / ~21% mémoire supplémentaires). Voir "Ce qui n'existe pas encore" ci-dessous pour la réserve sur la répartition par nœud.

`prometheus.prometheusSpec.retention: 7d` + un `volumeClaimTemplate` de `10Gi` sur `standard-rwo` (même storageClass que CNPG, `cnpg-cluster-staging.yaml:cluster.storage.storageClass`).

## ⚙️ Pourquoi ces composants sont désactivés

- `kubeControllerManager`, `kubeScheduler`, `kubeEtcd` : composants du control plane GKE, géré par Google — aucun endpoint accessible depuis ce cluster à scraper. Les laisser activés produirait des alertes "target down" en permanence pour des cibles qui n'ont jamais été atteignables.
- `kubeProxy` : GKE Dataplane V2 (`modules/gke`'s `datapath_provider = "ADVANCED_DATAPATH"`, cf. `repo-infrastructure/CLAUDE.md`) remplace kube-proxy par Cilium/eBPF — pas de pod kube-proxy à scraper non plus.
- `prometheusOperator.admissionWebhooks` : désactivés — aucun cert-manager n'est câblé pour ces webhooks dans cette installation minimale, et les activer ajoute un chemin d'ingress supplémentaire (apiserver → pod operator) à raisonner pour peu de bénéfice ici.

## ⚙️ Secret Grafana (jamais en clair)

`grafana.adminPassword` n'est **jamais** défini dans les values commitées — même principe que `terraform.tfvars`/`backend.hcl` restant gitignorés dans `repo-infrastructure`. `monitoring.yaml` référence `grafana.admin.existingSecret: grafana-admin-credentials`. À créer manuellement, une fois, avant le premier sync (ou avant que Grafana ne redémarre) :

```bash
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic grafana-admin-credentials \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='<mot-de-passe-fort-généré>'
```

Ce Secret n'est géré par aucune Application ArgoCD (comme les autres secrets de ce repo) — perdu si le cluster est recréé, à recréer manuellement dans ce cas.

## ⚙️ NetworkPolicy

Comme les trois autres namespaces (`dev`/`staging`/`prod`, via `charts/hr-app/templates/networkpolicy-default-deny.yaml`), `monitoring` a maintenant son propre default-deny-all + règles scopées explicites (`networkpolicy-monitoring.yaml`) :
- Trafic **interne** au namespace (Prometheus ↔ Alertmanager, Grafana → Prometheus, Prometheus → kube-state-metrics/node-exporter) : laissé ouvert pod-à-pod, **pas** scopé par label exact de composant. Choix assumé — les labels `app.kubernetes.io/name` exacts du chart version `87.21.0` n'ont pas été vérifiés contre un rendu live, et une policy mal scopée ici échoue **silencieusement** (dashboards vides, pas d'erreur) plutôt que bruyamment — pire que pas de policy du tout. La frontière du namespace, elle, reste zero-trust. Compromis du même esprit que l'egress totalement ouvert de CNPG (`manifests/cnpg-network-policy*/networkpolicy-postgres.yaml`), documenté là aussi comme un choix, pas un oubli.
- Egress DNS : même pattern à deux `podSelector` (`kube-dns` + `node-local-dns`) que `charts/hr-app/templates/networkpolicy-{backend,frontend}.yaml` — ne pas revenir à un `ipBlock`, voir `docs/issues-rencontrees.md` Issue 4.
- Egress vers `kube-apiserver` (namespace `default`, port 443) : nécessaire à l'opérateur Prometheus, à kube-state-metrics, et au job de scrape `kubernetes` de Prometheus lui-même. Scopé par namespace de destination plutôt que par label de pod, même raisonnement que ci-dessus.
- Egress vers `hr-backend` en `staging` uniquement (port 8081) — symétrique du `ServiceMonitor`.
- `networkpolicy-backend.yaml` (`charts/hr-app`) a gagné une règle d'ingress supplémentaire : namespace `monitoring` → `hr-backend` sur `backend.port`, à côté des règles déjà existantes (frontend, health-check GCE).

`kubectl port-forward` vers Grafana **n'est pas bloqué** par le default-deny — le port-forward passe par l'apiserver/kubelet, pas par le chemin réseau CNI que les `NetworkPolicy` gouvernent.

---

## ✅ Ce qui n'existe pas encore (pour éviter toute confusion)

- **Rétention 7 jours seulement, pas de stockage long terme** — pas de Thanos/Cortex/Mimir. Au-delà de 7 jours, l'historique est perdu.
- **Pas de haute disponibilité** — `replicas: 1` partout (Prometheus, Alertmanager, Grafana). Une préemption spot du nœud qui les héberge = coupure de la stack de monitoring jusqu'à réordonnancement, pas juste une dégradation.
- **Prometheus/Grafana sont en QoS *Burstable* (request < limit), alors que les CNPG Postgres tournent en *Guaranteed* (request = limit)** — sous pression mémoire réelle, le kubelet évince les pods Burstable en premier. Le monitoring peut donc être la première chose à disparaître exactement quand le cluster est sous stress, c'est-à-dire au moment où on voudrait le regarder. Pas de `PriorityClass` ni de passage en Guaranteed pour l'instant — accepté pour rester dans le dimensionnement demandé, à revisiter si ça se produit en pratique.
- **Alertmanager n'a aucun receiver externe configuré** (pas de Slack/email/PagerDuty) — les règles d'alerte par défaut du chart existent et s'évaluent, mais ne notifient personne. Une alerte qui se déclenche n'est visible que dans l'UI Alertmanager/Grafana, pas poussée nulle part.
- **Accès Grafana uniquement via `kubectl port-forward`** — pas d'Ingress public, cohérent avec le pattern déjà utilisé pour ArgoCD (`argocd_portforward` dans `repo-infrastructure/environments/staging/outputs.tf`).
  ```bash
  kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
  ```
- **Scope limité à `staging`** — `dev`/`prod` ne sont pas scrapés. Voir "Scope volontairement limité à `staging`" ci-dessus pour la procédure d'extension.
- **Pas de métrique CNPG (connexions actives, etc.)** — `cnpg-cluster-staging.yaml`'s `cluster.monitoring.enabled` est `false`, laissé tel quel volontairement (ce fichier était hors du périmètre autorisé de cette tâche). Le panneau correspondant dans le dashboard Grafana (`hr-platform-dashboard.json`, panel "Connexions actives CNPG") est un texte explicatif, pas un graphique vide.
- **Le dashboard ConfigMap et le fichier `.json` sont synchronisés à la main** — aucun outil de templating (Kustomize/Helm) n'est en place sur ce chemin de manifests bruts pour générer l'un depuis l'autre automatiquement.
- **Les composants déjà en place avant cette tâche restent non dimensionnés explicitement** — `cert-manager`, `cnpg-operator`, et les composants ArgoCD lui-même n'ont pas de bloc `resources` défini dans ce repo. Le calcul de capacité ci-dessus est donc une approximation : il ignore leur consommation réelle (non nulle) faute de requests à additionner.

## 🔗 Pour la suite

- Contrôles de sécurité déjà en place (NetworkPolicy, WIF, Checkov...) : [guide-securite.md](guide-securite.md)
- Historique des incidents réseau (DNS/NetworkPolicy) : [issues-rencontrees.md](issues-rencontrees.md)
- HPA de `hr-backend` (utilisé par le panel "Replicas" du dashboard) : [guide-hpa.md](guide-hpa.md)
- Capacité réelle du node pool GKE : `repo-infrastructure/CLAUDE.md` (section GKE hardening) — à vérifier en live (`kubectl describe nodes`) avant de faire confiance aux chiffres théoriques ci-dessus.
