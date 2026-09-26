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

- Engine: `drluoto/llama.cpp`
- Branch: `strix-halo-vulkan`
- Commit: `ba5354d46ca63e8225c28e1331f0f7651723ad05`
- GPU backend: Vulkan / RADV on AMD Radeon 8060S (Strix Halo)
- Keep `GGML_VK_DISABLE_GDN_CACHE_FUSION=1`; this is part of the validated Strix Halo baseline
- Model: Qwen3.8-Flash-Next AgenticRequant Q5_K
- Draft: FR-Spec MTP Q5_K, 65K
- Vision: BF16 mmproj
- Shared context: 262144 across 2 slots
- Host model path: `/srv/ai/models/qwen3.8-flash-next`
- Service endpoint: `192.168.1.6:8081`
