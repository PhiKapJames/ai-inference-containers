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

## Qwen 3.8 stack baseline

- Engine: upstream `ggml-org/llama.cpp`
- Commit: `23b0202a189c44a54625aadcb37a946dd1d6278d` (build 11552)
- Image: `ghcr.io/phikapjames/qwen38-llama:upstream-23b0202-visionfix`
- Local source patch: remove the obsolete Qwen4Exp MTP token-only assertion for native vision embeddings; see `Dockerfile.qwen38`
- GPU backend: Vulkan / RADV on AMD Radeon 8060S (Strix Halo)
- Model: Qwen3.8-Flash-Next, existing split `trunk-q5k` weights
- ai-brain-01: official Q4_0 MTP draft and BF16 vision projector, 262144 total context across two 131072-token slots
- ai-brain-02: text-only, speculative decoding disabled and no draft model loaded, full 262144 context in one slot; logical batch 2048 and physical microbatch 512 as the next unvalidated candidate
- Frontier MTP-off with physical microbatch 2048 passed bounded synthetic native checks but subsequently failed during a real OpenCode prompt at 23:32:56 UTC on 2026-10-13. It is not qualified for overnight development. Keep the passing observations and the later failure in README.md; do not treat the synthetic tests as end-to-end stability evidence.
- Benchmark physical microbatch 512 against the failing request shape, cache transitions and ordinary OpenCode tool continuations before qualifying it. Preserve the other Frontier runtime settings for this comparison; do not promise that smaller microbatches cure device loss.
- Both stacks keep `--no-reasoning-preserve` and DirectIO load mode
- Host model path: `/srv/ai/models/qwen3.8-flash-next`
- Endpoints and deployment commands: see `README.md` and the files under `portainer/`
- The former `drluoto/llama.cpp` `ba5354d46` runtime is a rollback reference, not the current stack baseline
