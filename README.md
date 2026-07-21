# Sandbox Documentation

This repository contains high-level usage documentation for the Sandbox workspace built from the `sandbox-*` repositories.

## Repositories in the Workspace

- `sandbox-cluster-config` — Flux GitOps cluster configuration for Minikube, environment manifests, namespaces, apps and monitoring.
- `sandbox-helm-charts` — shared Helm charts for the sandbox apps, including the `sandbox-app` example chart.
- `sandbox-env-values` — shared base values plus environment overlays used by Flux HelmReleases for dev, test, and prod.
- `wsl-config` — WSL support scripts and configuration helpers.

## What this setup does

This workspace defines a GitOps flow for a Minikube cluster using Flux. It supports:

- multiple environments: `dev`, `test`, `prod`, and `monitoring`
- Helm chart deployments managed by Flux
- shared and environment-specific Helm values stored separately from cluster configuration
- a monitoring stack with Prometheus, Grafana, Loki/Promtail, and Alertmanager
- Traefik Ingress routes for application and monitoring access

## How to use it

### 1. Prepare the workspace

Clone or open the workspace containing all `sandbox-*` `wsl-config` repositories.

For this demo, you'll need WSL set up with Ubuntu.

Run the prerequisite script to install required packages:


```bash
bash wsl-setup.sh
```

The script installs the following packages for the demo environment:

- `build-essential`
- `curl`
- `file`
- `tilix`
- `docker-ce`, `docker-ce-cli`, `containerd.io`
- Homebrew for Linux
- `git`, `wget`, `zsh`, `tmux`, `neovim`, `python`, `libpq`, `htop`, `ripgrep`, `fd`, `fzf`, `bat`, `jq`, `awscli`, `k9s`, `docker`, `minikube`, `kubectl`, `flux`
- `kubectx` and `kubens`

### 2. Start Minikube

```bash
minikube start
kubectl config current-context
```



### 3. Bootstrap or reconcile Flux

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

### 4. Deploy and verify environments

The following environments are managed separately:

- `dev` — `clusters/minikube/environments/dev`
- `test` — `clusters/minikube/environments/test`
- `prod` — `clusters/minikube/environments/prod`
- `monitoring` — `clusters/minikube/environments/monitoring`

Each environment includes a Kustomization pointing to an app overlay under `apps/overlays/<env>`.

For example, the dev environment deploys:

- `apps/base/sandbox-app.yaml` — shared HelmRelease definition for `sandbox-app`
- `apps/base/ingress.yaml` — shared Ingress definition
- `apps/overlays/dev/kustomization.yaml` — dev-specific overlay (including host `sandbox-app.dev.local`)

### 5. Access services

Current host names configured in the repo:

- `sandbox-app.dev.local`
- `sandbox-app.test.local`
- `sandbox-app.prod.local`
- `grafana.monitoring.local`
- `alertmanager.monitoring.local`

You may need to add these entries to your `/etc/hosts` file to resolve them locally.

### 5a. Test sandbox-vllm on Minikube

Use this flow to validate the LLM service exposed in the `llm` namespace.

1. Verify service and pod are running:

```bash
kubectl -n llm get svc,pods
```

Expected service name:

- `sandbox-vllm` on port `8000`

2. Start port-forward from local machine to the cluster service:

```bash
kubectl -n llm port-forward svc/sandbox-vllm 8000:8000
```

Leave this command running in a separate terminal.

3. Run quick API checks from another terminal:

```bash
curl -i http://127.0.0.1:8000/health
curl -s http://127.0.0.1:8000/v1/models | jq .
```

4. Send a sample chat completion request:

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "Qwen/Qwen2.5-1.5B-Instruct",
    "messages": [{"role": "user", "content": "Napisz jedno zdanie o Kubernetes."}],
    "temperature": 0.8,
    "max_tokens": 80
  }' | jq .
```

5. Optional stress test (parallel requests):

```bash
model="Qwen/Qwen2.5-1.5B-Instruct"
count=50
parallel=8

seq 1 "$count" | xargs -I{} -P "$parallel" bash -lc '
  i={}
  out=$(curl -sS -o /tmp/vllm_test_$i.json -w "%{http_code} %{time_total}" http://127.0.0.1:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"'"$model"'\",\"messages\":[{\"role\":\"user\",\"content\":\"Test #$i: napisz 6 slow o AI\"}],\"temperature\":0.7,\"max_tokens\":64}")
  echo "$i $out"
' | tee /tmp/vllm_parallel_results.txt

awk '{c[$2]++} END{for (k in c) print k, c[k]}' /tmp/vllm_parallel_results.txt | sort -n
```

If the status summary is mostly `200`, the endpoint is healthy under this load.

### 6. Update charts and values

To change application behavior:

- edit Helm chart definitions in `sandbox-helm-charts/charts/sandbox-app`
- edit shared values in `sandbox-env-values/base` and environment overrides in `sandbox-env-values/overlays/dev`, `sandbox-env-values/overlays/test`, or `sandbox-env-values/overlays/prod`
- update shared app manifests in `sandbox-cluster-config/apps/base`, environment overlays in `sandbox-cluster-config/apps/overlays/<env>`, or monitoring manifests in `sandbox-cluster-config/apps/monitoring`

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
- `apps/overlays/<env>/kustomization.yaml` — environment-specific overlay for `sandbox-app` resources (including Ingress host)
- `apps/overlays/<env>/kustomization.yaml` — also contains `redis-auth` Secret generation used by the Redis HelmRelease

### sandbox-helm-charts

Contains reusable chart definitions used by Flux.

Current chart:

- `charts/sandbox-app` — example NGINX-based app chart exposed via service and ingress
- `charts/redis` — Redis chart used by the Redis HelmRelease in `sandbox-cluster-config`

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

## Repository dependency diagram

The relationship between the core repositories is:

```text
sandbox-cluster-config
        |    \
        |     \
        |      \
        |       \
        v        v
sandbox-env-values   sandbox-helm-charts
```

- `sandbox-cluster-config` is the Flux GitOps entrypoint and references both:
  - `sandbox-env-values` for environment-specific Helm values
  - `sandbox-helm-charts` for Helm chart sources used by HelmReleases

This means `sandbox-cluster-config` orchestrates deployments using the other two repositories.

