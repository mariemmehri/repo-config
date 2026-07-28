# Récapitulatif sécurité

## 🎯 Objectif

Lister les mesures de sécurité réellement en place dans ce projet et, pour chacune, expliquer **quel problème concret** elle empêche — pas seulement la nommer. Document autonome, basé uniquement sur le code Terraform (`repo-infrastructure`), les Dockerfiles (`repo-app`) et le workflow CI (`repo-app/.github/workflows/ci.yml`, `repo-infrastructure/.github/workflows/workflow-infra.yml`) réellement présents.

---

## ⚙️ Authentification : Workload Identity Federation (pas de clé JSON)

**Ce que ça empêche :** une clé de compte de service GCP au format JSON, une fois committée par erreur dans un dépôt Git (même supprimée ensuite, elle reste dans l'historique) ou exfiltrée depuis un secret GitHub mal configuré, donne un accès **permanent** à qui la possède — jusqu'à révocation manuelle. Avec WIF, il n'existe **aucune clé statique à voler** : GitHub Actions échange un jeton OIDC court-lived contre un jeton GCP à chaque exécution.

**Comment c'est implémenté :**
- `backend-config/wif.tf` crée le pool `github-pool-v2` et le provider `github-provider`, avec `attribute_condition = "assertion.repository_owner == '${var.github_owner}'"` — seul le propriétaire GitHub configuré peut s'authentifier via ce pool.
- Deux comptes de service **distincts et cloisonnés par dépôt** : `sa-terraform-ci` n'accepte de jetons OIDC que depuis `attribute.repository/<owner>/repo-infrastructure` ; `sa-github-actions` uniquement depuis `attribute.repository/<owner>/repo-app`. Un jeton OIDC émis pour `repo-app` ne peut donc pas s'authentifier en tant que `sa-terraform-ci`, même si le nom du secret était deviné.
- Dans les workflows : `google-github-actions/auth@v2` avec `workload_identity_provider`/`service_account` — jamais de champ `credentials_json`.

## 🚀 Scan Trivy bloquant (CI `repo-app`)

**Ce que ça empêche :** déployer en production une image contenant une CVE **CRITICAL avec correctif disponible** — par exemple une vulnérabilité connue dans une bibliothèque OS ou Java packagée dans l'image, exploitable dès que l'image tourne dans le cluster.

**Comment c'est implémenté :** dans `docker-build-push` (`repo-app/.github/workflows/ci.yml`), `aquasecurity/trivy-action@v0.36.0` scanne chaque image juste après le `docker build`, avec `severity: CRITICAL`, `ignore-unfixed: true` (on ignore les CVE sans correctif — sinon le pipeline serait bloqué par des vulnérabilités qu'on ne peut de toute façon pas corriger) et `exit-code: 1` — le job échoue et **le `docker push` n'a jamais lieu**. L'historique de `repo-app` contient une remédiation réelle suite à ce mécanisme : `df4def4 fix: upgrade tomcat-embed-core to 10.1.55 (CVE-2026-41293, CVE-2026-43512, CVE-2026-43515)`, où le Tomcat embarqué a dû être mis à jour dans `pom.xml` (`<tomcat.version>10.1.55</tomcat.version>`) pour que le scan passe.

## 🚀 NetworkPolicy (GKE Dataplane V2 / Cilium) sur le cluster GKE

**Ce que ça empêche :** par défaut, **tous les pods d'un cluster Kubernetes peuvent dialoguer entre eux sans restriction**, quel que soit le namespace. Un pod compromis (via une dépendance vulnérable, par exemple) pourrait alors sonder ou atteindre n'importe quel autre service du cluster.

**Comment c'est implémenté :** `modules/gke/main.tf` active `datapath_provider = "ADVANCED_DATAPATH"` sur le cluster (GKE Dataplane V2, basé sur Cilium/eBPF) — cela **active le moteur d'application** des NetworkPolicy Kubernetes (sans ça, les objets `NetworkPolicy` créés seraient silencieusement ignorés par GKE). Ce réglage a remplacé un `network_policy { enabled = true, provider = "CALICO" }` plus ancien (voir `docs/issues-rencontrees.md` Issue 4 pour l'historique de cette migration et des itérations qui ont suivi sur la règle DNS). Le moteur est complété par trois objets `NetworkPolicy` dans `charts/hr-app/templates/`, gouvernés par `.Values.networkPolicy.enabled` (`true` dans tous les fichiers `values*.yaml`) :
- `networkpolicy-default-deny.yaml` : deny-all ingress+egress sur tous les pods du namespace de release (`podSelector: {}`) — base "zero trust", tout flux doit être explicitement autorisé par une policy plus spécifique.
- `networkpolicy-backend.yaml` : autorise l'ingress vers `hr-backend` uniquement depuis les pods `app: hr-frontend` sur le port applicatif (`.Values.backend.port`), l'egress DNS via deux règles `namespaceSelector`+`podSelector` visant `kube-dns` **et** `node-local-dns` dans `kube-system` (voir détail ci-dessous), et l'egress vers Postgres via `podSelector: {cnpg.io/cluster: <valeur de backend.database.cnpgClusterName>}` — paramétré par environnement (`pg-dev`/`pg-staging`/`pg-prod`) depuis `88aae5f`, après qu'une valeur codée en dur (`pg-staging`) ait silencieusement cassé l'accès DB en `dev`/`prod`.
- `networkpolicy-frontend.yaml` : autorise l'ingress vers `hr-frontend` depuis n'importe quelle source sur `.Values.frontend.port` (le frontend doit rester joignable, y compris via `kubectl port-forward` ou un futur ingress), et l'egress uniquement vers `hr-backend` + la même règle DNS double-selector.

Combinées, ces trois policies isolent le trafic est-ouest du namespace de chaque environnement : seul `hr-frontend → hr-backend`, `hr-backend → Postgres` (CNPG du bon environnement), et le DNS sortant vers `kube-dns`/`node-local-dns` sont permis, tout le reste (y compris un pod compromis ailleurs dans le cluster qui tenterait d'atteindre `hr-backend` directement, ou une exfiltration DNS vers un serveur externe) est bloqué par le deny-all par défaut.

**Note DNS (mise à jour) :** la règle d'egress DNS n'utilise plus d'`ipBlock`/`networkPolicy.dnsClusterIP` (cette valeur a été retirée du chart en `d9d5a19`). La cause racine finale n'était pas Calico vs Cilium mais **GKE NodeLocal DNSCache**, qui intercepte le trafic port 53 et répond depuis un pod `node-local-dns` par nœud plutôt que `kube-dns` — confirmé en live via `cilium-dbg monitor --type drop`. D'où les deux `podSelector` (`k8s-app: kube-dns` et `k8s-app: node-local-dns`), tous deux nécessaires. Voir `docs/issues-rencontrees.md` Issue 4 pour l'historique complet (quatre itérations).

**Egress DNS scopé à la ClusterIP `kube-dns` (`ipBlock` en `/32`) :** l'egress DNS des deux policies utilise `ipBlock: { cidr: <ClusterIP kube-dns>/32 }` (valeur portée par `.Values.networkPolicy.dnsClusterIP`, ex. `34.118.224.10` en staging), restreint au port 53 (UDP+TCP). Une première tentative avec un `namespaceSelector`/`podSelector` ciblant précisément les pods `kube-dns` avait cassé toute résolution DNS en pratique — investigation complète dans [issues-rencontrees.md](issues-rencontrees.md#issue-4--calico-ne-peut-pas-enforcer-une-networkpolicy-egress-contre-une-clusterip-dns-cassé). Cause : les pods résolvent le DNS via la **ClusterIP** du service `kube-dns` (visible dans `/etc/resolv.conf`), pas directement via l'IP d'un pod, et **Calico legacy sur GKE (mode iptables, sans Dataplane V2) ne sait pas relier un `podSelector`/`namespaceSelector` au trafic qui arrive en ClusterIP après DNAT** — seules les IP de pod directes sont évaluées correctement par ce type de sélecteur. Une migration vers GKE Dataplane V2 (Cilium/eBPF) avait été testée puis **abandonnée**, car elle nécessite de recréer le cluster GKE en entier — disproportionné pour ce cluster de staging PFE. Un `ipBlock` fonctionne différemment : Calico le traduit directement en règle iptables sur l'IP de destination du paquet (avant DNAT), donc cibler la ClusterIP elle-même via `ipBlock` est correctement enforcé. Vérifié en test live sur le cluster staging (auto-sync ArgoCD temporairement désactivé le temps du test, puis restauré) : avec `ipBlock` scopé à `34.118.224.10/32`, la résolution DNS interne (`hr-backend.staging.svc.cluster.local`) fonctionne de façon stable, et une requête DNS vers une IP externe hors de ce bloc (`8.8.8.8`) est bloquée (timeout), confirmant que le risque de DNS tunneling documenté auparavant (avec `ipBlock: 0.0.0.0/0`) est refermé.

**Limite résiduelle assumée :** la ClusterIP de `kube-dns` est attribuée par GKE à la création du cluster et n'est pas garantie stable dans le temps — une recréation du cluster (`destroy-staging` puis `apply`) change potentiellement cette IP, ce qui casserait le DNS jusqu'à mise à jour de `networkPolicy.dnsClusterIP` dans `values-staging.yaml`. Compromis jugé mineur pour un cluster de staging PFE qui n'est pas recréé en routine, et documentable en une ligne plutôt que de garder une ouverture `0.0.0.0/0` difficile à justifier.

**NetworkPolicy des pods CNPG (`manifests/cnpg-network-policy*/networkpolicy-postgres.yaml`, un par environnement) :** contrairement aux policies `hr-backend`/`hr-frontend` ci-dessus, l'egress des pods Postgres (`cnpg.io/cluster: pg-<env>`) est **entièrement ouvert** (`egress: [{}]`), pas scopé. Le commentaire dans le manifest explique pourquoi : l'instance manager de CNPG doit atteindre l'API server Kubernetes, la métadonnée GCE (169.254.169.254) pour Workload Identity, DNS et GCS — une règle `ipBlock` pinée sur l'IP de la metadata server a été testée et **ne fonctionne pas** (le proxy Workload Identity de GKE réécrit apparemment la destination au niveau du node avant que Calico n'évalue la policy), contre un `egress: [{}]` vérifié fonctionnel. L'ingress reste scopé (port 5432 depuis le namespace de l'env + depuis `cnpg-system`). C'est un compromis assumé, pas un oubli — à documenter dans "Ce qui n'existe pas encore" ci-dessous.

## 🚀 Utilisateur non-root dans les images Docker (partiel)

**Ce que ça empêche :** un conteneur qui tourne en `root` donne à un attaquant qui parviendrait à exécuter du code arbitraire dans le conteneur (ex: RCE via une dépendance vulnérable) un accès root **à l'intérieur du conteneur**, ce qui facilite l'exploitation de failles d'évasion de conteneur (container escape) vers le node hôte.

**Comment c'est implémenté :**
- `backend/Dockerfile` : `RUN addgroup -S app && adduser -S app -G app` puis **`USER app`** avant l'`ENTRYPOINT` — le process Java tourne bien en tant qu'utilisateur non privilégié `app`.
- `frontend/Dockerfile` : crée aussi le groupe/utilisateur `app` et fait `chown -R app:app /usr/share/nginx/html`, mais **ne déclare pas de directive `USER app`** — le processus `nginx` démarre donc avec le comportement par défaut de l'image `nginx:alpine` (root au démarrage, qui démarre ensuite des workers non-root en interne — comportement natif de l'image officielle, pas un choix explicite de ce Dockerfile).
- Aucun des deux Deployments (`charts/hr-app/templates/deployment-*.yaml`) ne définit de bloc `securityContext` Kubernetes (`runAsNonRoot`, `readOnlyRootFilesystem`, etc.) — la protection actuelle est **au niveau de l'image**, pas encore renforcée au niveau du pod Kubernetes.

## 🚀 Shielded Nodes (Secure Boot + Integrity Monitoring)

**Ce que ça empêche :** un rootkit ou un bootkit qui modifierait le firmware ou le chargeur de démarrage d'une VM node pour intercepter des secrets ou persister discrètement après un redémarrage.

**Comment c'est implémenté :** `modules/gke/main.tf`, à la fois sur le cluster et sur le node pool réel, `shielded_instance_config { enable_secure_boot = true, enable_integrity_monitoring = true }` — Secure Boot vérifie la signature du chargeur de démarrage et du noyau à chaque démarrage de VM ; Integrity Monitoring compare en continu l'état de boot mesuré à une baseline connue et alerte en cas d'écart.

## 🚀 Binary Authorization

**Ce que ça empêche :** qu'une image Docker arbitraire (poussée manuellement, ou provenant d'un registre tiers non vérifié) soit déployée sur le cluster sans passer par le pipeline attendu.

**Comment c'est implémenté :** `binary_authorization { evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE" }` dans `modules/gke/main.tf` — applique la politique Binary Authorization par défaut du projet au niveau du cluster.

## 🚀 Authentification cluster sans certificat client

**Ce que ça empêche :** un certificat client statique pour s'authentifier sur l'API Kubernetes est un secret longue-durée supplémentaire à protéger et à faire tourner — une fuite de ce certificat donne un accès direct au cluster, indépendamment de toute politique IAM GCP.

**Comment c'est implémenté :** `master_auth { client_certificate_config { issue_client_certificate = false } }` — force toute authentification à passer par IAM/OIDC (le même mécanisme WIF que la CI), pas par un certificat statique.

## 🚀 IAM à portée réduite (`modules/iam`)

**Ce que ça empêche :** qu'un compte de service compromis ou mal utilisé obtienne des permissions bien plus larges que ce dont il a réellement besoin (violation du principe de moindre privilège), ce qui augmenterait le blast radius de n'importe quelle erreur ou compromission.

**Comment c'est implémenté :**
- La SA des nodes GKE (`sa-gke-staging-pfe`) ne reçoit que 5 rôles précis et nécessaires au fonctionnement des nodes (`artifactregistry.reader`, `logging.logWriter`, `monitoring.metricWriter`, `monitoring.viewer`, `stackdriver.resourceMetadata.writer`) — pas de rôle d'administration.
- Le rôle `iam.serviceAccountUser`, habituellement risqué s'il est accordé au niveau projet (il permettrait d'usurper *n'importe quel* compte de service du projet), est ici accordé **uniquement sur deux ressources précises** : la SA des nodes GKE elle-même, et la SA Compute Engine par défaut (nécessaire au bootstrap du cluster) — jamais au niveau du projet entier.
- L'accès développeur en lecture (`container.clusterViewer`) est conditionnel (`count = var.developer_group_email != null ? 1 : 0`) — désactivable simplement en laissant la variable à `null`.

⚠️ **Limite documentée dans `.checkov.yaml`** : `sa-terraform-ci` (créé dans `backend-config/`) détient lui des rôles larges au niveau projet (`container.admin`, `iam.serviceAccountAdmin`, `resourcemanager.projectIamAdmin`...), nécessaires pour que Terraform puisse provisionner l'infrastructure de zéro. Les checks Checkov correspondants (`CKV_GCP_41`, `CKV_GCP_49`) sont explicitement *skippés* avec un commentaire renvoyant vers un futur audit ("extract to manual bootstrap") — un compromis assumé, pas un oubli.

## 🚀 Scan Checkov sur l'infrastructure Terraform (CRITICAL/HIGH bloquants)

**Ce que ça empêche :** qu'une ressource Terraform mal configurée (bucket public, chiffrement désactivé, rôle IAM trop large...) soit provisionnée sur GCP sans qu'aucune vérification automatisée ne le détecte avant l'`apply`.

**Comment c'est implémenté :** le job `security` de `workflow-infra.yml` exécute `bridgecrewio/checkov-action@v12.2847.0` avec `.checkov.yaml` : `soft-fail-on: [MEDIUM, LOW, INFO]` — donc tout finding **CRITICAL ou HIGH non explicitement skippé bloque le pipeline** avant même le `plan`. Chaque `skip-check` de `.checkov.yaml` porte un commentaire de justification (ex: `CKV_GCP_84 # SKIP staging — CSEK sur Artifact Registry non requis, chiffrement géré par Google suffisant`), ce qui rend chaque exception traçable et auditable plutôt que silencieuse.

⚠️ **Limite honnête sur `master_authorized_networks`** : `CKV_GCP_25` (restreindre les CIDR pouvant atteindre l'API Kubernetes) est skippé — le cluster autorise aujourd'hui `0.0.0.0/0` à atteindre l'API, avec le commentaire explicite dans le code : *"0.0.0.0/0 required for GitHub Actions (dynamic IPs); tighten post-PFE"*. C'est un compromis pragmatique (les runners GitHub Actions n'ont pas d'IP fixe) documenté comme dette à résorber, pas une bonne pratique définitive.

## ✅ Ce qui n'existe pas encore (pour éviter toute confusion)

Pour rester strictement fidèle à l'état actuel du code, ces éléments **ne sont pas implémentés** aujourd'hui, malgré leur proximité thématique avec les mesures ci-dessus :
- Pas de `securityContext` Kubernetes (`runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`) sur les Deployments `hr-backend`/`hr-frontend`.
- Pas de RBAC Kubernetes fin par Google Groups (`CKV_GCP_65` est explicitement skippé, motif : "requires Cloud Identity setup outside PFE scope").
- Pas de `private_cluster_config` (`CKV_GCP_64` skippé) — le cluster n'est pas un cluster privé.
- Pas de `logging_config`/`monitoring_config` explicites sur le cluster GKE (`CKV_GCP_25`/`CKV_GCP_71` skippés).
- Pas d'egress scopé sur les pods CNPG (`postgres-cnpg` NetworkPolicy, tous environnements) — egress entièrement ouvert par nécessité opérationnelle (voir ci-dessus), pas par principe de moindre privilège.

⚠️ Ce document (comme plusieurs autres dans `docs/`) a été écrit avant l'ajout des environnements `dev`/`prod` et de CloudNativePG (CNPG) — il ne couvrait à l'origine que `staging` sans base de données. Les mesures ci-dessus ont été mises à jour pour refléter les trois environnements et l'ajout de CNPG ; voir `../CLAUDE.md` (section "Known discrepancy") si d'autres sections de `docs/` semblent encore parler d'un seul environnement.

## 🔗 Pour la suite

- Détail complet du cluster GKE et des modules Terraform : [guide-deploiement-infra.md](guide-deploiement-infra.md)
- Détail du pipeline CI qui exécute le scan Trivy : [lifecycle-pipeline.md](lifecycle-pipeline.md)
