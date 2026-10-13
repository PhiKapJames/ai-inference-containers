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

The MTP sidecar and projector are loaded only by the vision-enabled `qwen-general` stack. These files can remain stored on both hosts; the Frontier stack does not load them.

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
- speculative decoding disabled; no draft model loaded
- `--no-reasoning-preserve`

MTP is disabled on Frontier to isolate repeated Vulkan GPU device-loss failures during long requests. This is a stability candidate awaiting host validation, not a demonstrated cure. The model, image reference, context size, slot count, batching and load mode are unchanged. Generation speed may change when speculative decoding is disabled.

After redeployment, confirm that no draft context is loaded, then test a cold long prompt and several continued tool calls. Record the actual model input tokens and output budget, completed responses, kernel GPU resets, memory/swap activity and client timeouts. A successful health check or short reply alone does not establish overnight stability.

The stacks pull the GHCR image built from the pinned upstream commit instead of depending on a host-local Docker image.

For the Git-managed `qwen-frontier` Portainer stack, use **Pull and redeploy** after the change is published to `main`. Confirm that the stack's Git commit advances and the recreated container uses `--spec-type none` without `-md`. Restarting the existing container alone does not apply the new command. This stack-only change does not rebuild the container image.

For the MTP comparison, record the running image ID or digest before and after redeployment. The existing `pull_policy: always` and mutable registry tag can fetch different image contents even when the image reference is unchanged. If the image changes, the result no longer isolates MTP alone.

## First publish

GitHub Container Registry packages created under a personal account are private by default. After the first successful workflow run, open the `qwen38-llama` package settings and change its visibility to **Public** if anonymous pulls from Portainer are desired.

If the package remains private, authenticate Docker/Portainer to `ghcr.io` before redeploying the stacks.
