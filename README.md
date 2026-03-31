# Todo Config

This repository holds the Helm chart and per-environment overlays for deploying the Todo application.

## Structure

- `apps/`: High-level Helm configuration entries used to install the chart in each environment. Point the `helm upgrade --install` command at these files to keep environment settings centralized.
- `charts/todo-app/`: The reusable Helm chart that bundles frontend and backend workloads together.
  - `templates/`: Kubernetes manifests rendered by Helm for deployments, services, and ingress rules.
  - `values*.yaml`: Base and environment-specific values that tune images, scaling, and ingress.

## Usage

1. Validate the chart locally:
   ```sh
   helm lint charts/todo-app
   ```
2. Render the manifests for inspection (example for staging):
   ```sh
   helm template todo-staging charts/todo-app -f charts/todo-app/values.yaml -f charts/todo-app/values-staging.yaml
   ```
3. Deploy to an environment using the corresponding file under `apps/`:
   ```sh
   helm upgrade --install todo-prod charts/todo-app -f apps/prod.yaml
   ```

Feel free to extend the chart with additional Kubernetes objects (ConfigMaps, Secrets, HPAs) as your deployment requirements grow.
