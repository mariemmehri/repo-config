# Problèmes rencontrés pendant le développement

## 🎯 Objectif

Documenter des problèmes réels rencontrés pendant la construction de ce projet, retrouvés dans l'historique Git et dans les notes déjà présentes dans le dépôt (`GITOPS_PFE.md`) — symptôme observé, cause racine, solution appliquée. Format court par problème, pour servir de mémoire collective en cas de récurrence.

---

## 🚀 Issue 1 — Course de synchronisation sur le CRD `Application` d'ArgoCD

**Symptôme observé**

Lors de l'installation d'ArgoCD via Helm suivie immédiatement de la création d'un objet `Application` (le CRD custom d'ArgoCD), l'erreur suivante apparaissait de façon intermittente :

```
the server could not find the requested resource (argoproj.io/v1alpha1, Application)
```

**Cause racine**

Le chart Helm d'ArgoCD s'installait correctement, mais le CRD `argoproj.io/v1alpha1` qu'il enregistre n'était pas toujours **immédiatement disponible** dans l'API server Kubernetes au moment où l'étape suivante tentait de créer l'objet `Application`. Le problème était initialement envisagé avec le provider Terraform `kubernetes_manifest`, qui nécessite que le CRD existe déjà au moment du `plan` — une dépendance séquentielle que Terraform ne peut pas garantir dans un seul `apply` quand le CRD est créé par le même apply.

**Solution appliquée**

Ne jamais créer l'objet ArgoCD `Application` via Terraform dans le même `apply` que l'installation d'ArgoCD. À la place, deux jobs GitHub Actions **séquentiels et distincts** dans `workflow-infra.yml` :
1. Job `apply` — Terraform provisionne uniquement l'infra GCP (VPC, GKE, Artifact Registry, IAM), aucune ressource ArgoCD.
2. Job `bootstrap-argocd` — `helm upgrade --install argocd argo/argo-cd`, puis `kubectl wait` explicite pour que le CRD `applications.argoproj.io` soit `established` **avant** d'exécuter `kubectl apply -f apps/root-app.yaml`.

Cette séparation supprime la dépendance implicite entre l'installation du chart et la disponibilité du CRD, sans introduire de second state Terraform pour ArgoCD. Détail complet dans [GITOPS_PFE.md](../GITOPS_PFE.md).

---

## 🚀 Issue 2 — Nodes GKE spot + autoscaling : `kubectl wait` échoue juste après le provisioning

**Symptôme observé**

Juste après un `terraform apply` créant le cluster GKE, l'étape `kubectl wait --for=condition=Ready nodes --all --timeout=300s` du job `apply` échouait avec :

```
error: no matching resources found
```

**Cause racine**

Le node pool utilise des **spot VMs** avec **autoscaling** (`min_node_count`/`max_node_count`). Le node pool peut exister côté GCP avant qu'aucun node n'ait encore fini de s'enregistrer auprès de l'API server Kubernetes — dans cette fenêtre, `kubectl get nodes` ne retourne aucun objet, et `kubectl wait --all` échoue immédiatement avec "no matching resources found" plutôt que d'attendre, puisqu'il n'y a littéralement aucune ressource à surveiller.

L'historique montre aussi une itération intermédiaire sur ce sujet : les spot VMs ont été temporairement désactivées (`infra: disable spot VMs to recover CPU allocatable`) puis réactivées deux jours plus tard (`infra: re-enable spot VMs to restore node CPU allocatable`) avant que l'autoscaling ne soit ajouté (`fix:add autoscaling`) — signe que le compromis coût (spot) vs stabilité (CPU allocatable disponible pour les pods applicatifs) a été ajusté plusieurs fois avant la solution retenue.

**Solution appliquée**

Ajout d'une étape de polling explicite avant le `kubectl wait` proprement dit : boucle de 30 tentatives (10 secondes d'intervalle, soit 5 minutes max) qui vérifie `kubectl get nodes --no-headers | wc -l` jusqu'à ce qu'au moins un node soit enregistré, avec un échec explicite (`::error::No GKE nodes registered after 5 minutes`) si le délai est dépassé — seulement après cela, le `kubectl wait --for=condition=Ready nodes --all --timeout=300s` original s'exécute en toute sécurité sur un ensemble de nodes non vide.

---

## 🚀 Issue 3 — CVE critiques détectées par Trivy sur le Tomcat embarqué

**Symptôme observé**

Le job `docker-build-push` de `repo-app` échouait à l'étape de scan Trivy de l'image backend, bloquant le `docker push` — le scan avait détecté des CVE de sévérité CRITICAL sur une dépendance embarquée dans l'image.

**Cause racine**

La version de Tomcat embarqué par Spring Boot 3.2.0 par défaut contenait plusieurs CVE avec correctif disponible (`CVE-2026-41293`, `CVE-2026-43512`, `CVE-2026-43515`) — comme la configuration Trivy du pipeline utilise `ignore-unfixed: true` mais `severity: CRITICAL` et `exit-code: 1`, toute CVE CRITICAL corrigeable bloque explicitement le pipeline plutôt que de laisser passer une image vulnérable.

**Solution appliquée**

Épingler explicitement une version corrigée de Tomcat dans `backend/pom.xml`, indépendamment de la version par défaut du parent Spring Boot :

```xml
<properties>
    <java.version>17</java.version>
    <tomcat.version>10.1.55</tomcat.version>
</properties>
```

Commit correspondant : `df4def4 fix: upgrade tomcat-embed-core to 10.1.55 (CVE-2026-41293, CVE-2026-43512, CVE-2026-43515)`. Ce mécanisme (scan bloquant + override de version ciblé) est le comportement attendu du pipeline — voir [guide-securite.md](guide-securite.md) pour le détail de la configuration Trivy.

## 🚀 Issue 4 — Calico ne peut pas enforcer une NetworkPolicy egress contre une ClusterIP (DNS cassé)

**Symptôme observé**

Après l'ajout des trois `NetworkPolicy` dans `charts/hr-app/templates/` (deny-all + isolation `hr-backend`/`hr-frontend`), la résolution DNS depuis les pods applicatifs se bloquait complètement :

```
;; connection timed out; no servers could be reached
```

Le trafic `hr-frontend → hr-backend` (autorisé explicitement) fonctionnait, mais toute résolution DNS échouait — cassant l'application en pratique, pas seulement le test.

**Cause racine**

La règle d'egress DNS initiale utilisait un `namespaceSelector` (`kubernetes.io/metadata.name: kube-system`), syntaxe Kubernetes NetworkPolicy standard et correcte. Or les pods résolvent le DNS via la **ClusterIP** du service `kube-dns` (`/etc/resolv.conf` pointe vers `34.118.224.10`, pas vers une IP de pod). Test isolant le problème sur le cluster live : une requête DNS directe vers l'IP d'un pod `kube-dns` (`10.52.2.2:53`) réussissait, mais la même requête vers la ClusterIP timeout — confirmant que **Calico legacy (non-Dataplane V2) sur ce cluster GKE ne peut pas enforcer une règle d'egress `NetworkPolicy` dont la destination est une ClusterIP**, seulement des IP de pod directes. Ajouter un `podSelector: k8s-app=kube-dns` en complément du `namespaceSelector` n'a rien changé — le problème n'est pas le selector, c'est le mécanisme d'enforcement lui-même face à un DNAT de Service.

**Tentative de solution root-cause (abandonnée)**

Une migration vers GKE Dataplane V2 (`datapath_provider = "ADVANCED_DATAPATH"` dans `modules/gke/main.tf`, à la place de `network_policy { enabled = true, provider = "CALICO" }`) a été préparée : Cilium (eBPF) résout correctement le trafic vers une ClusterIP jusqu'à ses pods backend avant d'appliquer la policy, contrairement à Calico/iptables. Ce changement nécessite de **recréer le cluster GKE** (le `datapath_provider` ne se bascule pas en place) via `destroy-staging` puis `apply`, et déclenchait aussi un nouveau finding Checkov (`CKV_GCP_12`, faux positif — le check ne reconnaît que le bloc `network_policy{enabled=true}` de Calico). Décision : **revert complet** de cette migration (`git reset --hard` sur `repo-infrastructure` + force-push) — trop de disruption pour ce cluster de staging PFE pour le bénéfice obtenu.

**Solution retenue (v1) — `ipBlock: 0.0.0.0/0`**

Dans `networkpolicy-backend.yaml` et `networkpolicy-frontend.yaml`, la règle d'egress DNS a d'abord utilisé un `ipBlock` ouvert plutôt qu'un selector :

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 0.0.0.0/0
    ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

Calico peut enforcer un `ipBlock` correctement, y compris contre une ClusterIP — contrairement à un `podSelector`/`namespaceSelector`. Mais cette première version acceptait une contrepartie non idéale : la règle autorisait l'egress port 53 vers **n'importe quelle IP**, pas seulement `kube-dns` — un pod compromis aurait pu exfiltrer des données via DNS tunneling vers un serveur externe sans être bloqué.

**Solution retenue (v2) — `ipBlock` scopé à la ClusterIP `kube-dns` en `/32`**

Le point bloquant de la v1 n'était pas l'usage d'un `ipBlock` en soi, mais son ouverture à `0.0.0.0/0`. Puisque Calico enforce correctement un `ipBlock` contre une ClusterIP (contrairement à un selector), il suffisait de restreindre le CIDR à la ClusterIP réelle du service `kube-dns` :

```yaml
egress:
  - to:
      - ipBlock:
          cidr: {{ .Values.networkPolicy.dnsClusterIP }}/32
    ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

`networkPolicy.dnsClusterIP` est une nouvelle valeur du chart (`values.yaml` : `""` par défaut, `values-staging.yaml` : `"34.118.224.10"`, récupérée via `kubectl get svc kube-dns -n kube-system -o jsonpath='{.spec.clusterIP}'`), documentée dans [guide-helm-chart.md](guide-helm-chart.md).

**Validation en test live (cluster staging, avant commit)** — pour éviter qu'ArgoCD `selfHeal` n'écrase un test manuel, l'auto-sync a été temporairement désactivé sur `root-app` **et** `hr-staging` (l'App-of-Apps propage `selfHeal` en cascade : patcher seulement `hr-staging` ne suffit pas, `root-app` le restaure depuis Git), puis restauré immédiatement après :
- Un pod de debug labellisé `app: hr-frontend` (héritant de sa NetworkPolicy) résout `hr-backend.staging.svc.cluster.local` sans problème avec l'`ipBlock` scopé au `/32` — stable sur 8 requêtes répétées.
- Une requête DNS vers une IP hors de ce bloc (`8.8.8.8`) timeout systématiquement — confirmant que l'ouverture `0.0.0.0/0` de la v1 est bien refermée.

Conclusion de l'investigation initiale à revoir : l'échec du `namespaceSelector`/`podSelector` ne prouvait pas que **tout** ciblage de la ClusterIP échoue avec Calico legacy — seulement que ce mécanisme de *selector* ne sait pas relier un pod matché au trafic post-DNAT. Un `ipBlock`, lui, matche directement l'IP de destination du paquet (avant DNAT côté kube-proxy), donc cibler la ClusterIP via `ipBlock` fonctionne. Voir [guide-securite.md](guide-securite.md) pour le détail à jour de cette mesure de sécurité.

## 🔗 Pour la suite

- Détail du pipeline CI où le scan Trivy s'exécute : [lifecycle-pipeline.md](lifecycle-pipeline.md)
- Détail du cluster GKE (spot VMs, autoscaling) : [guide-deploiement-infra.md](guide-deploiement-infra.md)
- Détail du bootstrap ArgoCD : [guide-argocd-gitops.md](guide-argocd-gitops.md)
