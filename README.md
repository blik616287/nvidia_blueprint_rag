# NVIDIA RAG Blueprint - Spectro Cloud Palette Packs

Spectro Cloud Palette packs for deploying the NVIDIA RAG (Retrieval Augmented Generation) Blueprint on Kubernetes clusters with GPU support.

## Overview

This repository contains addon packs that deploy a complete RAG pipeline including:

- **Vector Database**: Milvus with GPU-accelerated indexing
- **LLM Inference**: NVIDIA NIM microservices with dynamic model selection
- **Document Processing**: NV-Ingest for PDF/image extraction
- **RAG Application**: Query server, ingestor, and web frontend

## Pack Structure

| Pack | Version | Priority | Description |
|------|---------|----------|-------------|
| `nvidia-rag-data-infrastructure` | 1.0.10 | 10 | Milvus, etcd, MinIO, Redis |
| `nvidia-rag-core-nims` | 1.0.8 | 11 | LLM, Embedding, Reranker NIMs |
| `nvidia-rag-application` | 1.0.14 | 12 | RAG Server, Ingestor, NV-Ingest, Frontend |
| `nvidia-rag-guardrails` | 1.0.4 | 13 | NeMo Guardrails (optional) |
| `nvidia-rag-observability` | 1.0.5 | 14 | OTEL, Prometheus, Grafana, Zipkin (optional) |

### Pack Dependencies

```
nvidia-rag-application
├── nvidia-rag-core-nims (required)
│   ├── nvidia-gpu-operator (optional)
│   └── nvidia-nim-operator (optional)
└── nvidia-rag-data-infrastructure (required)
```

## Directory Layout

```
packs/<pack-name>-<version>/
├── pack.json                    # Pack metadata, dependencies, image list
├── values.yaml                  # Default Helm values for Palette
└── charts/
    └── <chart-name>/
        ├── Chart.yaml           # Helm chart metadata
        ├── values.yaml          # Chart default values
        └── templates/           # Kubernetes manifests
            └── *.yaml

profile/
├── profile.yaml                 # Cluster profile definition
└── variables.yaml               # Variable documentation
```

## Development Guide

### Pack Versioning

When modifying a pack:
1. Increment the version in `pack.json`
2. Update the chart version in `charts/<name>/Chart.yaml`
3. Rename the pack directory to match: `packs/<name>-<new-version>/`
4. Update references in `profile/profile.yaml`

### Values Hierarchy

Values are merged in order (later overrides earlier):
1. `charts/<name>/values.yaml` - Chart defaults
2. `packs/<name>/values.yaml` - Pack defaults (includes `pack:` metadata)
3. `profile/profile.yaml` - Profile overrides per pack

### Variable Templating

Packs use Spectro Cloud variable syntax for secrets:
```yaml
ngcApiKey: "{{.spectro.var.NGC_API_KEY}}"
```

Variables are defined in the profile's `spec.variables` section and set at cluster creation time.

### Pack Metadata (pack.json)

```json
{
  "name": "nvidia-rag-core-nims",
  "version": "1.0.8",
  "layer": "addon",
  "displayName": "NVIDIA RAG Core NIMs",
  "cloudTypes": ["all"],
  "charts": ["charts/nvidia-rag-core-nims-1.0.8.tgz"],
  "constraints": {
    "dependencies": [
      {
        "packName": "nvidia-gpu-operator",
        "minVersion": "24.6.0",
        "type": "optional"
      }
    ]
  }
}
```

### Install Priority

The `spectrocloud.com/install-priority` annotation controls deployment order:
```yaml
pack:
  spectrocloud.com/install-priority: "11"
```

Lower numbers install first. Data infrastructure (10) must be ready before NIMs (11), which must be ready before the application (12).

## Key Configuration Options

### Dynamic Model Selection (Core NIMs)

The LLM NIM automatically selects a model based on available GPU memory:

| GPU Memory | Model | Tensor Parallelism |
|------------|-------|-------------------|
| 16-24 GB | Llama 3.1 8B | 1 |
| 48-79 GB | Llama 3.1 70B | 2 |
| 80-159 GB | Llama 3.1 70B | 4 |
| 160+ GB | Nemotron 49B | 4 |

Override with:
```yaml
llm:
  manualOverride:
    enabled: true
    model:
      name: "meta/llama-3.1-8b-instruct"
      gpuCount: 1
```

### Cloud vs On-Prem NIMs

Each NIM service can run locally (GPU required) or use NVIDIA cloud APIs:

```yaml
llm:
  useCloud: false      # Local NIM (default)
  # useCloud: true     # Use ai.api.nvidia.com
```

NV-Ingest document processing uses cloud NIMs by default (no GPU needed).

### Milvus Index Configuration

```yaml
milvus:
  indexConfig:
    indexType: "GPU_CAGRA"    # GPU-accelerated
    metricType: "L2"
  gpu:
    enabled: true
```

## Required Credentials

| Variable | Format | Purpose |
|----------|--------|---------|
| `NGC_API_KEY` | Base64 string | Pull images from nvcr.io |
| `NVIDIA_API_KEY` | `nvapi-...` | Cloud NIM API access |
| `MINIO_ACCESS_KEY` | Plain string | MinIO username |
| `MINIO_SECRET_KEY` | Plain string | MinIO password |

Obtain keys from:
- NGC API Key: https://org.ngc.nvidia.com/setup/api-key
- NVIDIA API Key: https://build.nvidia.com

## GPU Resource Requirements

Minimum for default configuration:

| Component | GPUs | Notes |
|-----------|------|-------|
| Milvus | 1 | Vector indexing |
| LLM NIM | 1 | Scales with model size |
| Embedding NIM | 1 | llama-3.2-nv-embedqa-1b-v2 |
| Reranker NIM | 1 | llama-3.2-nv-rerankqa-1b-v2 |
| **Total** | **4** | With 8B LLM model |

To reduce GPU usage:
- Set `useCloud: true` for LLM/embedding/reranker
- Use CPU-only Milvus (`milvus.gpu.enabled: false`, change image tag)

## Testing Locally

Charts can be templated locally for validation:

```bash
cd packs/nvidia-rag-application-1.0.14/charts/nvidia-rag-application
helm template . -f ../../values.yaml
```

## Image Registry

All NVIDIA images are pulled from `nvcr.io` and require authentication via `ngc-pull-secret`. The secret is created by the `nvidia-rag-core-nims` pack using the `NGC_API_KEY` variable.

Third-party images (Milvus, Redis, MinIO, etc.) are pulled from public registries.
