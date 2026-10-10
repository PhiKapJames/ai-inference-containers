# AGENTS.md

## Purpose

This repository builds reproducible inference runtime images and stores Portainer stack definitions for the user's local AI infrastructure.

## Safety and scope

- Never commit model weights, API keys, access tokens, certificates, or private host configuration.
- Model weights remain on the inference host under `/srv/ai/models`.
- Secrets remain on the inference host under `/srv/ai/secrets`.
- Keep runtime images pinned to specific upstream commits. Do not replace pinned refs with `latest`.
- Do not change model quantization, context size, speculative-decoding settings, or Vulkan flags without benchmarking the change first.
- Portainer stack files should reference prebuilt registry images rather than Compose `build:` directives for remote Agent environments.

## Qwen 3.8 production baseline

- Engine: `ggml-org/llama.cpp`
- Branch: `master`
- Commit: `23b0202a189c44a54625aadcb37a946dd1d6278d`
- Temporary workaround: remove the obsolete Qwen4Exp MTP token-only assertion described in `ggml-org/llama.cpp#30307`
- GPU backend: Vulkan / RADV on AMD Radeon 8060S (Strix Halo)
- Model: Qwen3.8-Flash-Next AgenticRequant Q5_K
- Draft: official full-vocabulary Q4_0 MTP
- Vision: BF16 mmproj
- Shared context: 262144 across 2 slots
- Reasoning history: `--no-reasoning-preserve`
- Image token allocation: projector/model default
- Host model path: `/srv/ai/models/qwen3.8-flash-next`
- Production host: `ai-brain-01`
- Service endpoint: `192.168.1.27:8081`

## Rollback baseline

- Image: `ghcr.io/phikapjames/qwen38-llama:ba5354d46`
- Engine: `drluoto/llama.cpp`
- Branch: `strix-halo-vulkan`
- Commit: `ba5354d46ca63e8225c28e1331f0f7651723ad05`
- Draft: FR-Spec MTP Q5_K, 65K
