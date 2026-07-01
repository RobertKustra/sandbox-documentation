# Sandbox Documentation

This repository contains high-level usage documentation for the Sandbox workspace built from the `sandbox-*` repositories.

## Repositories in the Workspace

- `sandbox-cluster-config` — Flux GitOps cluster configuration for Minikube, environment manifests, namespaces, apps and monitoring.
- `sandbox-helm-charts` — shared Helm charts for the sandbox apps, including the `sandbox-app` example chart.
- `sandbox-env-values` — environment-specific values used by Flux HelmReleases for dev, test, and prod.
- `wsl-config` — WSL support scripts and configuration helpers.

## What this setup does

This workspace defines a GitOps flow for a Minikube cluster using Flux. It supports:

- multiple environments: `dev`, `test`, `prod`, and `monitoring`
- Helm chart deployments managed by Flux
- environment-specific values stored separately from cluster configuration
- a monitoring stack with Prometheus, Grafana, Loki/Promtail, and Alertmanager
- Traefik Ingress routes for application and monitoring access

## How to use it

### 1. Prepare the workspace

Clone or open the workspace containing all `sandbox-*` repositories.

### 2. Start Minikube

```bash
minikube start
kubectl config current-context
```

### 3. Install Flux CLI

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux --version
```

### 4. Bootstrap or reconcile Flux

The main Flux entrypoint is `sandbox-cluster-config/clusters/minikube`.

If you are bootstrapping Flux for the first time, use a command similar to:

```bash
export GITHUB_TOKEN=<your-github-token>
flux bootstrap github \
  --owner=<github-owner> \
  --repository=sandbox-cluster-config \
  --branch=development \
  --path=./clusters/minikube \
  --personal \
  --ssh-key-algorithm=rsa \
  --ssh-hostname=github.com
```

If Flux is already installed and the repo is connected, verify the sync:

```bash
kubectl get pods -n flux-system
kubectl get kustomizations -A
kubectl get helmreleases -A
```

### 5. Deploy and verify environments

The following environments are managed separately:

- `dev` — `clusters/minikube/environments/dev`
- `test` — `clusters/minikube/environments/test`
- `prod` — `clusters/minikube/environments/prod`
- `monitoring` — `clusters/minikube/environments/monitoring`

Each environment includes a Kustomization pointing to an app package under `apps/<env>`.

For example, the dev environment deploys:

- `apps/dev/sandbox-app.yaml` — HelmRelease for `sandbox-app`
- `apps/dev/ingress.yaml` — Traefik Ingress for `sandbox-app.dev.local`

### 6. Access services

Current host names configured in the repo:

- `sandbox-app.dev.local`
- `sandbox-app.test.local`
- `sandbox-app.prod.local`
- `grafana.monitoring.local`
- `alertmanager.monitoring.local`

You may need to add these entries to your `/etc/hosts` file to resolve them locally.

### 7. Update charts and values

To change application behavior:

- edit Helm chart definitions in `sandbox-helm-charts/charts/sandbox-app`
- edit environment-specific values in `sandbox-env-values/dev`, `sandbox-env-values/test`, or `sandbox-env-values/prod`
- update HelmRelease manifests in `sandbox-cluster-config/apps/<env>` or `sandbox-cluster-config/apps/monitoring`

Then commit and push the changes, and let Flux reconcile the cluster.

## Repository details

### sandbox-cluster-config

This repository contains the Flux GitOps configuration:

- `clusters/minikube/flux-system` — Flux resources and HelmRepository sources
- `clusters/minikube/environments/*` — environment-specific Kustomizations
- `namespaces/*` — namespace manifests
- `apps/*` — HelmRelease and Ingress manifests for each environment
- `sources/*` — GitRepository definitions for external repo sources

Important components:

- `apps/monitoring` — monitoring stack with Prometheus, Grafana, Loki-stack, Promtail, and Alertmanager
- `apps/<env>/ingress.yaml` — environment-specific Ingress resources for `sandbox-app`

### sandbox-helm-charts

Contains reusable chart definitions used by Flux.

Current chart:

- `charts/sandbox-app` — example NGINX-based app chart exposed via service and ingress

### sandbox-env-values

Contains Helm values for each environment:

- `dev/` — values for development deployments
- `test/` — values for test deployments
- `prod/` — values for production deployments

### wsl-config

Contains WSL-specific setup instructions and helper scripts for using the workspace under Windows Subsystem for Linux.

## Notes

- Traefik is used as the Ingress controller in this setup.
- The monitoring stack is isolated in the `monitoring` environment and does not automatically deploy into dev/test/prod unless the corresponding Kustomization is updated.
- Flux applies the configuration from `sandbox-cluster-config` and references external sources from `sandbox-helm-charts` and `sandbox-env-values`.

## Next steps

- Add or modify application charts in `sandbox-helm-charts`
- Manage environment-specific settings in `sandbox-env-values`
- Add more environments or services under `sandbox-cluster-config`
- Use the existing `monitoring` app package to extend alerts and dashboards

