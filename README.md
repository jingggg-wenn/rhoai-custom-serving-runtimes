# Custom Serving Runtimes for Red Hat OpenShift AI

Created: 2026-08-20
Last Modified: 2026-08-20

A collection of custom vLLM ServingRuntime configurations for serving the latest model architectures on Red Hat OpenShift AI (RHOAI).

---

## Table of Contents

1. [The Problem](#the-problem)
2. [How to Find the Required vLLM Version](#how-to-find-the-required-vllm-version)
3. [How to Create a Custom ServingRuntime](#how-to-create-a-custom-servingruntime)
4. [Working Example: Gemma 4 12B FP8-Dynamic](#working-example-gemma-4-12b-fp8-dynamic)
5. [Available Runtimes](#available-runtimes)
6. [Troubleshooting](#troubleshooting)

---



## The Problem

RHOAI ships a **bundled vLLM image** that is tied to each platform release. According to the official [Red Hat OpenShift AI: Supported Configurations](https://access.redhat.com/articles/rhoai-supported-configs-3.x) and the [Red Hat AI Supported Product and Hardware Configurations](https://docs.redhat.com/en/documentation/red_hat_ai/3/html-single/supported_product_and_hardware_configurations/index) documentation, the bundled vLLM versions are:


| RHOAI Version | Bundled vLLM (CUDA) | Container Image                                  |
| ------------- | ------------------- | ------------------------------------------------ |
| 3.0           | v0.11.0             | `registry.redhat.io/rhaii/vllm-cuda-rhel9:3.0`   |
| 3.2           | v0.11.2             | `registry.redhat.io/rhaii/vllm-cuda-rhel9:3.2`   |
| 3.3           | v0.13.0             | `registry.redhat.io/rhaii/vllm-cuda-rhel9:3.3`   |
| 3.4           | v0.18.0             | `registry.redhat.io/rhaii/vllm-cuda-rhel9:3.4.0` |


These images are certified and supported by Red Hat, but they only cover model architectures that were available at the time of release.

New model architectures land in the **upstream vLLM project faster** than Red Hat can certify and bundle them. Here are some popular examples:


| Model                                                                                     | Architecture                   | Minimum vLLM Required          | RHOAI 3.4 Bundled |
| ----------------------------------------------------------------------------------------- | ------------------------------ | ------------------------------ | ----------------- |
| [Gemma 4 12B/27B](https://recipes.vllm.ai/Google/gemma-4-12B-it)                          | Gemma4ForConditionalGeneration | v0.23.0+                       | v0.18.0           |
| [GLM-4.5V / GLM-4.6V](https://recipes.vllm.ai/zai-org/GLM-4.5V)                           | GLM4VForCausalLM               | v0.12.0+                       | v0.18.0           |
| [Kimi K2 / K2.5](https://docs.vllm.ai/projects/recipes/en/stable/moonshotai/Kimi-K2.html) | MoE (Moonshot)                 | v0.10.0+ (K2), v0.17.0+ (K2.5) | v0.18.0           |
| [Kimi K2.6](https://docs.discoverer.bg/kimi-k26-vllm-h200-guide.html)                     | MoE (Moonshot)                 | v0.19.1                        | v0.18.0           |
| [Qwen3-Coder-Next](https://huggingface.co/RedHatAI/Qwen3-Coder-Next-NVFP4)                | Qwen3NextForCausalLM           | v0.14.1+                       | v0.18.0           |


While some of these models (like GLM-4.5V and Kimi K2) technically fall within the bundled v0.18.0 version range, newer variants (Kimi K2.6, Gemma 4) require versions that exceed what RHOAI ships. As new model families continue to emerge at a rapid pace, this gap will only grow.

When you attempt to serve a model on a runtime that does not support its architecture, you typically see errors such as:

- `KeyError: unsupported model architecture`
- `NotImplementedError` for missing attention backends or kernels
- Crashes during model weight loading due to incompatible quantization formats

One approach is to create a custom `ServingRuntime` resource that uses an upstream (or community) vLLM container image while keeping everything else compatible with the OpenShift AI dashboard and KServe.

---

## Important: Support Scope and Intended Use

> **Custom serving runtimes are NOT covered by Red Hat support.**
>
> While OpenShift AI allows adding custom serving runtimes (for both generative and predictive AI model serving), any customization beyond the preinstalled, Red Hat-certified runtimes falls outside the scope of Red Hat support. If you encounter issues with a custom runtime, Red Hat support will not be able to assist with troubleshooting the runtime itself.

**The purpose of this repository** is to help customers who want to **pilot, test, and evaluate** newly released models that are not yet supported by the bundled RHOAI serving runtimes. It is intended for:

- **Proof-of-concept and demo environments** where you need to showcase the latest models
- **Development and testing** where teams want to evaluate a new model architecture before it lands in a supported RHOAI release
- **Bridging the gap** between upstream model releases and the next RHOAI version that bundles support for them

For production workloads, the recommendation is to use the preinstalled, Red Hat-certified vLLM serving runtimes and wait for the model architecture to be included in a future RHOAI release.

---



## How to Find the Required vLLM Version

Before creating a custom runtime, determine which vLLM version supports your target model. Check these sources in priority order:

### 1. HuggingFace Model Card (RedHatAI Repository)

The [Red Hat AI organization on HuggingFace](https://huggingface.co/RedHatAI) hosts 700+ optimized models ready for deployment on OpenShift AI. This is the first place to check when planning a model deployment:

- **Quantized variants** -- Red Hat AI publishes FP8-Dynamic, INT8, INT4, and NVFP4 quantizations of popular models (Llama, Qwen, Gemma, DeepSeek, GLM, Kimi, and others). These optimized weights reduce GPU memory requirements by 50-75% while maintaining accuracy, enabling models to run on fewer or smaller GPUs than the original BF16 weights require.
- **Accuracy benchmarks** -- Each model card includes evaluation results from Language Model Evaluation Harness, showing the accuracy recovery percentage compared to the original model (typically 98-105%).
- **Capacity planning data** -- Models are benchmarked with GuideLLM for throughput and latency metrics across various hardware configurations.
- **vLLM compatibility metadata** -- Each card states the exact vLLM version the model was validated on.

Navigate to a model page on [huggingface.co/RedHatAI](https://huggingface.co/RedHatAI) and look for:

- **"Validated on vLLM"** -- the exact version tested by the publisher
- **"Validated on RHOAI"** -- the RHOAI release it was tested against
- **"ModelCar Storage URI"** -- the OCI image URI for direct deployment on OpenShift AI

For example, the [RedHatAI/Qwen3-Coder-Next-NVFP4](https://huggingface.co/RedHatAI/Qwen3-Coder-Next-NVFP4) model card states:


| Field                | Value                                                               |
| -------------------- | ------------------------------------------------------------------- |
| Validated on vLLM    | 0.14.1                                                              |
| Validated on RHOAI   | 3.4 EA1                                                             |
| ModelCar Storage URI | `oci://registry.redhat.io/rhai/modelcar-qwen3-coder-next-nvfp4:3.0` |


If the "Validated on vLLM" version is **newer** than what your RHOAI release bundles, you need a custom serving runtime.

### 2. vLLM Recipes

The vLLM project maintains per-model pages with exact version requirements and recommended Docker tags:

- **URL**: [https://recipes.vllm.ai](https://recipes.vllm.ai)
- Example: The [Gemma 4 12B page](https://recipes.vllm.ai/Google/gemma-4-12B-it) states **"vLLM 0.23.0+"** and recommends a pinned Docker image.



### 3. vLLM Supported Models Documentation

The official list of all supported architectures and when support was added:

- **URL**: [https://docs.vllm.ai/en/latest/models/supported_models.html](https://docs.vllm.ai/en/latest/models/supported_models.html)



### Upstream Docker Image Tags

Once you know the required version, select the correct container image. The upstream images follow this naming convention:


| Image Tag                                      | CUDA Version      | Notes                                                     |
| ---------------------------------------------- | ----------------- | --------------------------------------------------------- |
| `vllm/vllm-openai:v<VERSION>`                  | CUDA 13 (default) | Use if your GPU driver supports CUDA 13                   |
| `vllm/vllm-openai:v<VERSION>-cu129-ubuntu2404` | CUDA 12.9         | Use for older CUDA 12.x drivers                           |
| `vllm/vllm-openai:<model-tag>`                 | Varies            | Pinned builds for specific models (e.g. `gemma4-unified`) |


---



## How to Create a Custom ServingRuntime



### Step 1: Check Your CUDA Driver Version

The vLLM image you choose must match the CUDA runtime on your GPU nodes. Check with:

```bash
# List GPU nodes and their CUDA version
oc get nodes -l nvidia.com/gpu.present=true \
  -o custom-columns='NODE:.metadata.name,CUDA:.metadata.labels.nvidia\.com/cuda\.runtime-version\.full,GPU:.metadata.labels.nvidia\.com/gpu\.product'
```

Example output:

```
NODE                                        CUDA   GPU
ip-10-0-36-13.us-east-2.compute.internal    13.0   NVIDIA-L4
```

- **CUDA 13.0** -- use the default image tag (e.g. `vllm/vllm-openai:v0.24.0`)
- **CUDA 12.x** -- use the `cu129` variant (e.g. `vllm/vllm-openai:v0.24.0-cu129-ubuntu2404`)



### Step 2: Create the ServingRuntime YAML

A custom ServingRuntime for upstream vLLM on OpenShift AI differs from the default RHOAI runtime in several ways.

**Command and args**: Upstream vLLM v0.20+ images use `vllm serve` as the entrypoint. The default RHOAI runtime uses `python -m vllm.entrypoints.openai.api_server`. Using the wrong command results in `executable file not found` errors.

```yaml
# Upstream vLLM (v0.20+)
command:
  - vllm
  - serve
args:
  - /mnt/models
  - --port
  - "8080"
  - --served-model-name
  - '{{.Name}}'
  - --max-model-len
  - "4096"
  - --gpu-memory-utilization
  - "0.95"
```

```yaml
# Default RHOAI runtime (do NOT use with upstream images)
command:
  - python
  - -m
  - vllm.entrypoints.openai.api_server
args:
  - '--port=8080'
  - '--model=/mnt/models'
  - '--served-model-name={{.Name}}'
```

**Required environment variables**: OpenShift runs containers with an arbitrary UID that does not exist in `/etc/passwd`. Upstream vLLM (via PyTorch) calls `getpass.getuser()` during initialization, which fails with `KeyError: 'getpwuid(): uid not found'`. Set these env vars to work around it:

```yaml
env:
  - name: HF_HOME
    value: /tmp/hf_home
  - name: HOME
    value: /tmp
  - name: USER
    value: vllm
  - name: LOGNAME
    value: vllm
```

**GPU memory tuning**: Adjust `--max-model-len` and `--gpu-memory-utilization` based on your GPU type and model size. A rough guide:


| GPU                | VRAM  | Suggested `--gpu-memory-utilization` |
| ------------------ | ----- | ------------------------------------ |
| NVIDIA L4          | 24 GB | 0.90 - 0.95                          |
| NVIDIA A10G        | 24 GB | 0.90 - 0.95                          |
| NVIDIA A100 (40GB) | 40 GB | 0.90                                 |
| NVIDIA A100 (80GB) | 80 GB | 0.90                                 |


For a complete working example with all annotations and fields, see `[runtimes/gemma4-vllm-v0.24.0/servingruntime.yaml](runtimes/gemma4-vllm-v0.24.0/servingruntime.yaml)`.

### Step 3: Apply the Runtime

You can apply a custom serving runtime using either the CLI or the OpenShift AI UI.

**Option A: CLI**

```bash
# Namespace-scoped (available only in this project)
oc apply -f servingruntime.yaml -n <your-namespace>
```

To make the runtime available globally across all projects, ensure the YAML includes the annotation `opendatahub.io/serving-runtime-scope: global`. The runtime still resides in a namespace, but the dashboard will surface it in all projects. Alternatively, use the UI approach below, which handles scoping automatically.

**Option B: OpenShift AI Dashboard (UI)**

1. Log in to the OpenShift AI dashboard with an administrator account
2. Navigate to **Settings > Model resources and operations > Serving runtimes**
3. You have two choices:
  - **Duplicate an existing runtime** -- click the kebab menu on an existing runtime (e.g. "vLLM NVIDIA GPU ServingRuntime for KServe") and select **Duplicate**. Then edit the duplicated YAML to change the image, command, and env vars.
  - **Add from scratch** -- click **Add serving runtime**, select **Start from scratch**, and paste the full ServingRuntime YAML.
4. Save the runtime. It will appear in the list and be available when deploying models in any project.



### Step 4: Deploy a Model Using the Custom Runtime

Create an `InferenceService` that references the custom runtime by name:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: gemma-4-12b-it-fp8
  annotations:
    serving.kserve.io/deploymentMode: Standard
    opendatahub.io/model-type: generative
  labels:
    opendatahub.io/dashboard: "true"
    opendatahub.io/genai-asset: "true"
spec:
  predictor:
    minReplicas: 1
    maxReplicas: 1
    model:
      modelFormat:
        name: vLLM
      runtime: upstream-vllm-0240-runtime   # <-- must match ServingRuntime metadata.name
      storageUri: "oci://registry.redhat.io/rhai/modelcar-gemma-4-12b-it-fp8-dynamic:3.0"
      resources:
        limits:
          nvidia.com/gpu: "1"
        requests:
          nvidia.com/gpu: "1"
    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
```

Apply with:

```bash
oc apply -f inferenceservice.yaml -n <your-namespace>
```

---



## Working Example: Gemma 4 12B FP8-Dynamic

The `[runtimes/gemma4-vllm-v0.24.0/servingruntime.yaml](runtimes/gemma4-vllm-v0.24.0/servingruntime.yaml)` file contains a working ServingRuntime for serving Gemma 4 12B FP8-Dynamic on a single NVIDIA L4 GPU.

**Why a custom runtime is needed**: Gemma 4 uses the `Gemma4ForConditionalGeneration` architecture, which requires vLLM 0.23.0 or later. RHOAI 3.4 bundles an older vLLM version that does not support this architecture.

**What was customized**:

- **Image**: `vllm/vllm-openai:v0.24.0` (upstream, CUDA 13 compatible)
- **Command**: `vllm serve` instead of `python -m vllm.entrypoints.openai.api_server`
- **Env vars**: `HOME=/tmp`, `USER=vllm`, `LOGNAME=vllm` for OpenShift UID compatibility
- **Args**: `--max-model-len 4096` and `--gpu-memory-utilization 0.95` tuned for a 24GB L4

---



## Troubleshooting



### `executable file 'python' not found in $PATH`

**Cause**: Upstream vLLM v0.20+ images no longer have `python` in the default PATH. The entrypoint is `vllm serve`.

**Fix**: Change the ServingRuntime `command` field:

```yaml
# Wrong (RHOAI default)
command: ["python", "-m", "vllm.entrypoints.openai.api_server"]

# Correct (upstream vLLM v0.20+)
command: ["vllm", "serve"]
```



### `KeyError: 'getpwuid(): uid not found: 1000910000'`

**Cause**: OpenShift assigns an arbitrary UID to containers. PyTorch calls `getpass.getuser()` which tries to resolve this UID via `/etc/passwd` and fails.

**Fix**: Add these environment variables to the ServingRuntime:

```yaml
env:
  - name: HOME
    value: /tmp
  - name: USER
    value: vllm
  - name: LOGNAME
    value: vllm
```



### CUDA Version Mismatch / Image Pull Errors

**Cause**: The default upstream image (`vllm/vllm-openai:v0.24.0`) requires CUDA 13. If your GPU nodes have CUDA 12.x drivers, vLLM will crash or fail to initialize.

**Fix**: Check your CUDA version and use the correct image tag:

```bash
oc get nodes -l nvidia.com/gpu.present=true \
  -o jsonpath='{range .items[*]}{.metadata.name}: CUDA {.metadata.labels.nvidia\.com/cuda\.runtime-version\.full}{"\n"}{end}'
```

- CUDA 13.x -- use `vllm/vllm-openai:v0.24.0`
- CUDA 12.x -- use `vllm/vllm-openai:v0.24.0-cu129-ubuntu2404`



### Model Architecture Not Supported

**Cause**: The vLLM version you selected does not yet support the model's architecture.

**Fix**: Check the [vLLM Supported Models page](https://docs.vllm.ai/en/latest/models/supported_models.html) or the model's [vLLM Recipes page](https://recipes.vllm.ai) for the minimum required version. You may need a newer release or a nightly build.

---



## Contributing

To add a new runtime:

1. Create a new directory under `runtimes/` with the naming convention `<model>-vllm-v<version>/`
2. Add a `servingruntime.yaml` with full YAML header comments documenting the configuration choices
3. Update the [Available Runtimes](#available-runtimes) table in this README

