# Sandbox Documentation

This repository contains high-level usage documentation for the Sandbox workspace built from the `sandbox-*` repositories.

## Repositories in the Workspace

- `sandbox-cluster-config` - Flux GitOps cluster configuration for Minikube, environment manifests, namespaces, apps and monitoring.
- `sandbox-helm-charts` - shared Helm charts for the sandbox apps, including `sandbox-nginx`, `sandbox-redis`, and `sandbox-vllm`.
- `sandbox-env-values` - shared base values plus environment overlays used by Flux HelmReleases for dev, test, and prod.
- `wsl-config` - WSL support scripts and configuration helpers.

## What this setup does

This workspace defines a GitOps flow for a Minikube cluster using Flux. It supports:


- multiple environments: `dev`, `test`, `prod`, `monitoring`, and `llm`
- Helm chart deployments managed by Flux
- shared and environment-specific Helm values stored separately from cluster configuration
- a monitoring stack with Prometheus, Grafana, Loki/Promtail, and Alertmanager
- Traefik Ingress routes for application and monitoring access

**Warning:**  If your computer does not have a dedicated NVIDIA graphics processor that supports CUDA, do not install the `llm` and `ai-consumer` modules. The procedure is described later in this documentation.

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
minikube start \
  -p minikube \
  --driver=docker \
  --container-runtime=docker \
  --gpus=all

kubectl get node minikube \
  -o jsonpath='Allocatable: cpu={.status.allocatable.cpu} mem={.status.allocatable.memory} gpu={.status.allocatable.nvidia\.com/gpu}{"\n"}'

docker inspect minikube \
  --format 'NanoCPUs={{.HostConfig.NanoCpus}} MemoryBytes={{.HostConfig.Memory}}'

kubectl config current-context
```



### 3. Bootstrap Flux

The main Flux entrypoint is `sandbox-cluster-config/clusters/minikube`.

If you are bootstrapping Flux for the first time, use a command similar to:

```bash
export GITHUB_TOKEN=<your-github-token>
flux bootstrap github \
  --owner=RobertKustra \
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

- `dev` - `clusters/minikube/environments/dev`
- `test` - `clusters/minikube/environments/test`
- `prod` - `clusters/minikube/environments/prod`
- `monitoring` - `clusters/minikube/environments/monitoring`
- `llm` - `clusters/minikube/environments/llm`

Each environment includes a Kustomization pointing to an app overlay under `apps/overlays/<env>`.

For example, the dev environment deploys:

- `apps/base/sandbox-nginx.yaml` - shared HelmRelease definition for `sandbox-nginx`
- `apps/base/ingress.yaml` - shared Ingress definition
- `apps/overlays/dev/kustomization.yaml` - dev-specific overlay (including host `sandbox-nginx.dev.local`)

### 5. Access services

Current host names configured in the repo:

- `sandbox-nginx.dev.local`
- `sandbox-nginx.test.local`
- `sandbox-nginx.prod.local`
- `sandbox-vllm.llm.local`
- `grafana.monitoring.local`
- `alertmanager.monitoring.local`

You may need to add these entries to your `/etc/hosts` file to resolve them locally.

### 5a. vLLM Hardware Requirements

The `sandbox-vllm` service requires specific hardware to operate properly. The requirements vary depending on the model size and inference parameters.

#### Current Configuration

The sandbox setup uses:

- **Model**: `Qwen/Qwen2.5-Coder-3B-Instruct` (3B parameters)
- **GPU**: 1x NVIDIA GPU (minimum 16GB VRAM recommended)
- **CPU**: 2 cores (request) / 4 cores (limit)
- **Memory**: 4Gi (request) / 8Gi (limit)
- **Storage**: 30Gi persistent volume for model cache (`/root/.cache/huggingface`)
- **Shared Memory**: 2Gi `/dev/shm` mount (for CUDA operations)

#### Minimum Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU | 8GB VRAM | 16GB+ VRAM |
| CPU Cores | 2 | 4-8 |
| System RAM | 8Gi | 16Gi+ |
| Storage | 30Gi | 50Gi+ (for multiple models) |
| Shared Memory (/dev/shm) | 1Gi | 2Gi+ |

#### GPU Memory Considerations

- **Smaller models (1B-3B parameters)**: 8-12GB VRAM sufficient
  - Example: `Qwen/Qwen2.5-Coder-3B-Instruct`
  - Max model length: 2048-4096 tokens
  - Max concurrent sequences: 1-4

- **Medium models (7B-13B parameters)**: 16GB VRAM minimum
  - Max model length: 2048 tokens
  - Max concurrent sequences: 1-2

- **Larger models (30B+ parameters)**: 24GB+ VRAM
  - Consider quantization (4-bit, 8-bit) for lower VRAM usage

#### Key Tuning Parameters

Located in [sandbox-cluster-config/apps/llm/sandbox-vllm.yaml](https://github.com/RobertKustra/sandbox-cluster-config/blob/development/apps/llm/sandbox-vllm.yaml):

- `--dtype half` - Use float16 to reduce memory usage (vs float32)
- `--gpu-memory-utilization 0.6` - Allocate 60% of GPU VRAM for model; adjust down (0.4-0.5) if OOM occurs
- `--max-model-len 16384` - Maximum input + output token length; reduce to 2048-4096 if memory constrained
- `--max-num-seqs 4` - Maximum concurrent sequences; reduce to 1-2 if OOM occurs
- `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` - Enable memory fragmentation mitigation

#### Out-of-Memory (OOM) Mitigation

If experiencing OOM errors with 16GB GPUs:

1. Reduce `--gpu-memory-utilization` from 0.6 to 0.4-0.5
2. Lower `--max-num-seqs` from 4 to 1-2
3. Decrease `--max-model-len` from 16384 to 2048-4096
4. Ensure `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True,max_split_size_mb:64` is set
5. Increase shared memory `/dev/shm` from 2Gi to 4Gi if available

#### Deployment Without a Dedicated GPU

> **Warning:** If your machine does not have a dedicated NVIDIA GPU that supports CUDA, do **not** deploy the `llm` environment and do **not** deploy `sandbox-ai-consumer` in any environment (`dev`, `test`, `prod`). Without GPU support vLLM will fail to start and `sandbox-ai-consumer` will loop on connection errors.

To disable these components, comment out or remove the relevant entries in the following files:

1. **Disable the `llm` environment entirely** - remove the reference from the cluster kustomization:

   - File: [sandbox-cluster-config/clusters/minikube/kustomization.yaml](https://github.com/RobertKustra/sandbox-cluster-config/blob/development/clusters/minikube/kustomization.yaml)
   ```yaml
   resources:
     # - ./environments/llm.yaml   # comment out when no GPU available
   ```

2. **Disable `sandbox-ai-consumer` from dev/test/prod overlays** - remove the resource from each overlay:

   - File: [sandbox-cluster-config/apps/overlays/dev/kustomization.yaml](https://github.com/RobertKustra/sandbox-cluster-config/blob/development/apps/overlays/dev/kustomization.yaml)
   - File: [sandbox-cluster-config/apps/overlays/test/kustomization.yaml](https://github.com/RobertKustra/sandbox-cluster-config/blob/development/apps/overlays/test/kustomization.yaml)
   - File: [sandbox-cluster-config/apps/overlays/prod/kustomization.yaml](https://github.com/RobertKustra/sandbox-cluster-config/blob/development/apps/overlays/prod/kustomization.yaml)
   ```yaml
   resources:
     # - ../../sandbox-ai-consumer/base   # comment out when no GPU / vLLM not deployed
   ```

3. **Remove the `minikube-llm` dependency from dev/test/prod Flux Kustomizations** - otherwise Flux will block reconciliation waiting for the missing llm environment:

   - File: [sandbox-cluster-config/clusters/minikube/environments/dev.yaml](https://github.com/RobertKustra/sandbox-cluster-config/blob/development/clusters/minikube/environments/dev.yaml)
   - File: [sandbox-cluster-config/clusters/minikube/environments/test.yaml](https://github.com/RobertKustra/sandbox-cluster-config/blob/development/clusters/minikube/environments/test.yaml)
   - File: [sandbox-cluster-config/clusters/minikube/environments/prod.yaml](https://github.com/RobertKustra/sandbox-cluster-config/blob/development/clusters/minikube/environments/prod.yaml)
   ```yaml
   spec:
     dependsOn:
       # - name: minikube-llm   # remove when llm environment is disabled
   ```

After committing these changes Flux will skip the GPU-dependent components and all remaining environments will reconcile normally.

#### Cluster Resource Reservation

Ensure your Minikube instance has sufficient resources:

```bash
minikube start \
  -p minikube \
  --driver=docker \
  --container-runtime=docker \
  --gpus=all

kubectl get node minikube \
  -o jsonpath='Allocatable: cpu={.status.allocatable.cpu} mem={.status.allocatable.memory} gpu={.status.allocatable.nvidia\.com/gpu}{"\n"}'

docker inspect minikube \
  --format 'NanoCPUs={{.HostConfig.NanoCpus}} MemoryBytes={{.HostConfig.Memory}}'
```

For WSL2 with NVIDIA GPU passthrough, configure in your `.wslconfig`:

```ini
[wsl2]
memory=16GB
processors=8
guiApplications=true
gpuSupport=true
```

### 5b. Test sandbox-vllm on Minikube

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
    "model": "Qwen/Qwen2.5-Coder-1.5B-Instruct",
    "messages": [{"role": "user", "content": "Write a Bash script that displays the contents of the default system variables."}],
    "temperature": 0.8,
    "max_tokens": 120
  }' | jq .
```

5. Optional stress test (parallel requests):

```bash
model="Qwen/Qwen2.5-Coder-3B-Instruct"
count=50
parallel=8

seq 1 "$count" | xargs -I{} -P "$parallel" bash -lc '
  i={}
  out=$(curl -sS -o /tmp/vllm_test_$i.json -w "%{http_code} %{time_total}" http://127.0.0.1:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"'"$model"'\",\"messages\":[{\"role\":\"user\",\"content\":\"Test #$i: Write 6 words about AI\"}],\"temperature\":0.7,\"max_tokens\":64}")
  echo "$i $out"
' | tee /tmp/vllm_parallel_results.txt

awk '{c[$2]++} END{for (k in c) print k, c[k]}' /tmp/vllm_parallel_results.txt | sort -n
```

If the status summary is mostly `200`, the endpoint is healthy under this load.

### 6. Update charts and values

To change application behavior:

- edit Helm chart definitions in `sandbox-helm-charts/charts/sandbox-nginx`, `sandbox-helm-charts/charts/sandbox-redis`, and `sandbox-helm-charts/charts/sandbox-vllm`
- edit shared values in `sandbox-env-values/base` and environment overrides in `sandbox-env-values/overlays/dev`, `sandbox-env-values/overlays/test`, or `sandbox-env-values/overlays/prod`
- update shared app manifests in `sandbox-cluster-config/apps/base`, environment overlays in `sandbox-cluster-config/apps/overlays/<env>`, LLM manifests in `sandbox-cluster-config/apps/llm`, or monitoring manifests in `sandbox-cluster-config/apps/monitoring`

Then commit and push the changes, and let Flux reconcile the cluster.

## Repository details

### sandbox-cluster-config

This repository contains the Flux GitOps configuration:

- `clusters/minikube/flux-system` - Flux resources and HelmRepository sources
- `clusters/minikube/environments/*` - environment-specific Kustomizations
- `namespaces/*` - namespace manifests
- `apps/*` - HelmRelease and Ingress manifests for each environment
- `sources/*` - GitRepository definitions for external repo sources

Important components:

- `apps/monitoring` - monitoring stack with Prometheus, Grafana, Loki-stack, Promtail, and Alertmanager
- `apps/llm` - `sandbox-vllm` HelmRelease, ingress, and smoke-test-enabled deployment configuration
- `apps/overlays/<env>/kustomization.yaml` - environment-specific overlay for `sandbox-nginx` resources (including Ingress host)
- `apps/overlays/<env>/kustomization.yaml` - also contains `sandbox-redis-auth` Secret generation used by the `sandbox-redis` HelmRelease

### sandbox-helm-charts

Contains reusable chart definitions used by Flux.

Current charts:

- `charts/sandbox-nginx` - NGINX-based app chart exposed via service and ingress
- `charts/sandbox-redis` - Redis chart used by the `sandbox-redis` HelmRelease in `sandbox-cluster-config`
- `charts/sandbox-vllm` - vLLM inference chart with a post-install/post-upgrade smoke-test hook
- `charts/sandbox-ai-consumer` - vLLM consumer worker chart using environment-driven runtime settings (`VLLM_*`)

### sandbox-env-values

Contains Helm values for each environment:

- `dev/` - values for development deployments
- `test/` - values for test deployments
- `prod/` - values for production deployments

Current rollout scope for `sandbox-ai-consumer` is `dev` and `test`.

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
- TODO: Configure Alertmanager notifications for Discord (webhook or relay) in `sandbox-cluster-config/apps/monitoring/alertmanager.yaml`.

## Repository dependency diagram

The relationship between the core repositories is:

```text
sandbox-cluster-config
        |         \
        |          \
        |           \
        |            \
        v             v
sandbox-env-values   sandbox-helm-charts
```

- `sandbox-cluster-config` is the Flux GitOps entrypoint and references both:
  - `sandbox-env-values` for environment-specific Helm values
  - `sandbox-helm-charts` for Helm chart sources used by HelmReleases

This means `sandbox-cluster-config` orchestrates deployments using the other two repositories.

