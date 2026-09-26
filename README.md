# AI Inference Containers

Reproducible container images and Portainer stack definitions for the local AI inference hosts.

Model weights and API credentials are **not** stored in this repository. They remain on the inference host under `/srv/ai/models` and `/srv/ai/secrets`.

## Qwen 3.8 Flash-Next

The production Qwen runtime is pinned to:

- `drluoto/llama.cpp`
- branch `strix-halo-vulkan`
- commit `ba5354d46ca63e8225c28e1331f0f7651723ad05`
- Vulkan/RADV backend
- Linux amd64 container image

The GitHub Actions workflow builds:

`ghcr.io/phikapjames/qwen38-llama:ba5354d46`

The image contains the inference engine and Vulkan userspace only. Model weights are mounted read-only by the Portainer stack.

## First publish

GitHub Container Registry packages created under a personal account are private by default. After the first successful workflow run, open the `qwen38-llama` package settings and change its visibility to **Public** if anonymous pulls from Portainer are desired.

## Portainer

Use `portainer/qwen-general.yaml` as the stack definition on `ai-brain-01`.

Required host files:

```text
/srv/ai/models/qwen3.8-flash-next/
  trunk-q5k-00001-of-00003.gguf
  trunk-q5k-00002-of-00003.gguf
  trunk-q5k-00003-of-00003.gguf
  mtp-Qwen3.8-Flash-Next-Q5_K-frspec-65k.gguf
  mmproj-BF16.gguf

/srv/ai/secrets/qwen-general-api-key
```

The stack publishes the OpenAI-compatible API on `192.168.1.6:8081`.
