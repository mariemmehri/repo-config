# Documentation GitOps - PFE

## Objectif

Mettre en place un workflow GitOps fiable avec ArgoCD sans erreur intermittente sur le CRD `Application`.

## Probleme rencontre

Quand on installe ArgoCD via Helm et qu'on cree l'objet ArgoCD `Application` dans le meme `terraform apply`, Terraform peut aller trop vite.

Le chart Helm est bien installe, mais le CRD `argoproj.io/v1alpha1` n'est pas toujours enregistre immediatement par l'API server Kubernetes.

Erreur typique:

`the server could not find the requested resource (argoproj.io/v1alpha1, Application)`

## Solution retenue (Best Practice)

Ne pas creer l'objet ArgoCD `Application` via Terraform (`kubernetes_manifest`) dans le meme apply que l'installation Helm d'ArgoCD — cela reintroduirait la course de synchronisation CRD.

A la place, un seul workflow GitHub Actions ([workflow-infra.yml](../repo-infrastructure/.github/workflows/workflow-infra.yml)) gere Terraform (infra GCP uniquement) **et** le bootstrap ArgoCD dans deux jobs sequentiels distincts, sans etat Terraform partage pour ArgoCD :

1. **Job `apply`** — Terraform provisionne l'infra GCP (VPC, GKE, Artifact Registry, IAM). Aucune ressource ArgoCD ici.
2. **Job `bootstrap-argocd`** — `helm upgrade --install argocd argo/argo-cd`, attend que les CRD et pods soient prets (`kubectl wait`), puis `kubectl apply -f apps/root-app.yaml` clone depuis le config repo.

Cette separation supprime la course de synchronisation CRD et rend le pipeline deterministe, sans necessiter un deuxieme state Terraform.

## Architecture finale

### Terraform (`repo-infrastructure/environments/staging`)

- Cree le VPC, le cluster GKE, Artifact Registry et l'IAM
- Ne cree ni le namespace `argocd`, ni l'objet `Application`
- Un seul state (`prefix=staging`), dedie a l'infra uniquement

### Bootstrap ArgoCD (job `bootstrap-argocd` de `workflow-infra.yml`)

- Installe ArgoCD via `helm upgrade --install` (chart `argo/argo-cd`)
- Attend que les CRD et pods ArgoCD soient prets
- Applique `apps/root-app.yaml` (clone depuis `repo-config`) via `kubectl apply`
- `root-app` pointe vers `apps/children` dans le config repo
- Idempotent : si `namespace argocd` et l'application `root-app` existent deja, le job skip (~5s), sauf `action=bootstrap` qui force une reinstallation

## Workflow GitOps detaille

1. Le developpeur pousse du code applicatif dans `repo-app`
2. Le pipeline CI construit et pousse les images dans Google Artifact Registry
3. Le pipeline met a jour les tags SHA dans le config repo (`repo-config`)
4. ArgoCD surveille le config repo
5. ArgoCD detecte le commit et synchronise automatiquement le cluster
6. `prune` supprime les ressources obsoletes, `selfHeal` corrige la derive

## Organisation des manifests ArgoCD

- `root-app` (applique par le job `bootstrap-argocd`) pointe sur `apps/children`
- Les applications environnement (ex: staging) vivent dans `apps/children/*.yaml`

Cette organisation evite l'auto-reference App-of-Apps et clarifie la source de verite.

## Resultat

Le workflow GitOps est robuste, reproductible et documente:

- plus d'echec aleatoire lie au CRD ArgoCD
- separation claire infra/applications, sans state Terraform partage pour ArgoCD
- deploiement pilote par Git avec reconciliation automatique ArgoCD
