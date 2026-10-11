# AI Inference Containers

Reproducible container images and Portainer stack definitions for the local AI inference hosts.

Model weights and API credentials are **not** stored in this repository. They remain on the inference hosts under `/srv/ai/models` and `/srv/ai/secrets`.

## Qwen 3.8 Flash-Next

The production Strix Halo runtime is pinned to:

- upstream `ggml-org/llama.cpp`
- commit `23b0202a189c44a54625aadcb37a946dd1d6278d` (build 11552)
- Vulkan/RADV backend
- Linux amd64 container image
- one local source patch that removes the obsolete Qwen4Exp MTP token-only assertion so native vision embeddings can use MTP

The GitHub Actions workflow builds and publishes:

`ghcr.io/phikapjames/qwen38-llama:upstream-23b0202-visionfix`

The image contains the inference engine and Vulkan userspace only. Model weights are mounted read-only by the Portainer stacks.

The former `drluoto/llama.cpp` `ba5354d46` image remains a rollback reference and is not the production runtime.

## Host model files

Both Strix Halo hosts use:

```text
/srv/ai/models/qwen3.8-flash-next/
  trunk-q5k-00001-of-00003.gguf
  trunk-q5k-00002-of-00003.gguf
  trunk-q5k-00003-of-00003.gguf
  mtp-Qwen3.8-Flash-Next-Q4_0.gguf
  mmproj-BF16.gguf

/srv/ai/secrets/qwen-general-api-key
```

The projector is required only by the vision-enabled `qwen-general` stack.

## Portainer

### ai-brain-01 — general/heavy/coding/vision

Use `portainer/qwen-general.yaml`.

- host: `192.168.1.27`
- API: `8081`
- two slots
- `262144` total context, `131072` per slot
- native vision enabled
- official Q4_0 MTP draft model
- `--no-reasoning-preserve`

### ai-brain-02 — frontier/design

Use `portainer/qwen-frontier.yaml`.

- host: `192.168.1.28`
- API: `8082`
- one slot
- full `262144` context
- text-only
- official Q4_0 MTP draft model
- `--no-reasoning-preserve`

The stacks pull the pinned GHCR image instead of depending on a host-local Docker image.

## First publish

GitHub Container Registry packages created under a personal account are private by default. After the first successful workflow run, open the `qwen38-llama` package settings and change its visibility to **Public** if anonymous pulls from Portainer are desired.

If the package remains private, authenticate Docker/Portainer to `ghcr.io` before redeploying the stacks.
