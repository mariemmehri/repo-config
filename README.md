# Todo Config

This repository holds the Helm chart and per-environment overlays for deploying the Todo application.

## Structure

- `apps/children/`: Argo CD child `Application` manifests (one file per environment, e.g. `staging.yaml`).
- `charts/todo-app/`: The reusable Helm chart that bundles frontend and backend workloads together.
  - `templates/`: Kubernetes manifests rendered by Helm for deployments, services, and ingress rules.
  - `values*.yaml`: Base and environment-specific values that tune images, scaling, and ingress.

## GitOps Bootstrap

- Argo CD itself and the root `apps-root` Application are bootstrapped by Terraform (`terraform-todo/argocd.tf`).
- The root app points to `apps/children/` and auto-discovers all environment applications.
- This avoids a manual bootstrap step and prevents app-of-apps self-reference issues.

## Usage

1. Validate the chart locally:
   ```sh
   helm lint charts/todo-app
   ```
2. Render the manifests for inspection (example for staging):
   ```sh
   helm template todo-staging charts/todo-app -f charts/todo-app/values.yaml -f charts/todo-app/values-staging.yaml
   ```
3. Deploy to an environment using the corresponding values file:
   ```sh
   helm upgrade --install todo-staging charts/todo-app -f charts/todo-app/values-staging.yaml
   ```

Feel free to extend the chart with additional Kubernetes objects (ConfigMaps, Secrets, HPAs) as your deployment requirements grow.
