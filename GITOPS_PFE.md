# Documentation GitOps - PFE

## Objectif

Mettre en place un workflow GitOps fiable avec ArgoCD sans erreur intermittente sur le CRD `Application`.

## Probleme rencontre

Quand on installe ArgoCD via Helm et qu'on cree l'objet ArgoCD `Application` dans le meme `terraform apply`, Terraform peut aller trop vite.

Le chart Helm est bien installe, mais le CRD `argoproj.io/v1alpha1` n'est pas toujours enregistre immediatement par l'API server Kubernetes.

Erreur typique:

`the server could not find the requested resource (argoproj.io/v1alpha1, Application)`

## Solution retenue (Best Practice)

Separer en **2 phases Terraform** (2 stacks/state):

1. **Phase A - Infra + ArgoCD**
2. **Phase B - Bootstrap GitOps** (`apps-root`)

Cette separation supprime la course de synchronisation CRD et rend le pipeline deterministe.

## Architecture finale

### Phase A: `terraform-todo`

- Cree AKS, ACR et prerequis infra
- Cree le namespace `argocd`
- Installe ArgoCD via `helm_release`
- **Ne cree pas** l'objet `Application` racine

Fichier principal: `terraform-todo/argocd.tf`

### Phase B: `terraform-bootstrap-gitops`

- Lit le cluster AKS existant via `data.azurerm_kubernetes_cluster`
- Configure le provider `kubernetes`
- Cree l'objet ArgoCD `apps-root` via `kubernetes_manifest`
- `apps-root` pointe vers `apps/children` dans le config repo

Fichiers principaux:

- `terraform-bootstrap-gitops/providers.tf`
- `terraform-bootstrap-gitops/main.tf`
- `terraform-bootstrap-gitops/variables.tf`

## Workflow GitOps detaille

1. Le developpeur pousse du code applicatif dans `todo-app`
2. Le pipeline CI construit et pousse les images dans ACR
3. Le pipeline met a jour les tags SHA dans le config repo (`todo-config`)
4. ArgoCD surveille le config repo
5. ArgoCD detecte le commit et synchronise automatiquement le cluster
6. `prune` supprime les ressources obsoletes, `selfHeal` corrige la derive

## Organisation des manifests ArgoCD

- `apps-root` (cree par Terraform phase B) pointe sur `apps/children`
- Les applications environnement (ex: staging) vivent dans `apps/children/*.yaml`

Cette organisation evite l'auto-reference App-of-Apps et clarifie la source de verite.

## Procedure d'execution

### Etape 1 - Phase A

```powershell
Set-Location "c:/Users/wassi/Downloads/stage pfe/terraform-todo"
terraform init
terraform apply -var-file="terraform.tfvars"
```

### Etape 2 - Phase B

```powershell
Set-Location "c:/Users/wassi/Downloads/stage pfe/terraform-bootstrap-gitops"
terraform init
terraform apply -var-file="terraform.tfvars"
```

## Migration depuis l'ancien modele

Si `kubernetes_manifest.argocd_root_app` etait deja gere dans la phase A, le retirer du state phase A avant de reappliquer:

```powershell
Set-Location "c:/Users/wassi/Downloads/stage pfe/terraform-todo"
terraform state rm kubernetes_manifest.argocd_root_app
```

Puis appliquer la phase B pour que la ressource soit geree par le bon state.

## Resultat

Le workflow GitOps est robuste, reproductible et documente:

- plus d'echec aleatoire lie au CRD ArgoCD
- separation claire infra/applications
- deploiement pilote par Git avec reconciliation automatique ArgoCD
