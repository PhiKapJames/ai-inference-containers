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
- Model: Qwen3.8-Flash-Next UD-IQ4_XS; identical four-shard hashes on both hosts. Keep 01’s former trunk-q5k weights only as a rollback reference.
- ai-brain-01: matching UD-IQ4_XS package, official Q4_0 MTP draft and BF16 vision projector, 524288 total context across two 262144-token slots; Q8_0 K/V cache for the target model, unchanged FP16 draft cache, logical batch 2048 and physical microbatch 512. See the dated bounded validation in README.md; it does not establish full-window or overnight stability.
- ai-brain-02: Qwen3.8 Flash Next UD-IQ4_XS with the BF16 vision projector and Q4_0 MTP draft (`draft-mtp`, maximum three draft tokens, minimum probability 0.0, default FP16 draft cache), 524288 total context across two 262144-token slots, Q8_0 K/V, logical batch 2048 and physical microbatch 512. Use the image digest in portainer/qwen-frontier.yaml. The 2026-11-15 bounded validation in README.md records concurrent long image-bearing requests, tool calls, structured output, cached continuations, the MTP generation comparison, a Coder instruction miss reproduced with MTP off, and a cold request exceeding the existing 30-minute OpenCode timeout.
- Frontier MTP-off with physical microbatch 2048 passed bounded synthetic native checks but subsequently failed during a real OpenCode prompt at 23:32:56 UTC on 2026-10-13. It is not qualified for overnight development. Keep the passing observations and the later failure in README.md; do not treat the synthetic tests as end-to-end stability evidence.
- Benchmark physical microbatch 512 against the failing request shape, cache transitions and ordinary OpenCode tool continuations before qualifying it. Preserve the other Frontier runtime settings for this comparison; do not promise that smaller microbatches cure device loss.
- Both stacks keep `--no-reasoning-preserve` and DirectIO load mode
- Main model path on both hosts: `/srv/ai/models/qwen3.8-flash-next-unsloth-iq4xs-v31`. The unchanged Q4_0 draft and BF16 projector remain under `/srv/ai/models/qwen3.8-flash-next` and have separate read-only mounts.
- The 2026-11-15 matching-01 validation records before/after short decode rates, twelve functional coding fixtures, concurrent long vision requests, strict instruction misses, cache-eviction skips and host swap behavior. Preserve those qualifications. Both stack files use the same immutable image digest. LiteLLM/client Coder, Designer and Memory budgets are 262144 context, 196608 maximum input and 32768 maximum output; planned client request timeout is 45 minutes. The model cutover does not itself migrate every client.
- Endpoints and deployment commands: see `README.md` and the files under `portainer/`
- The former `drluoto/llama.cpp` `ba5354d46` runtime is a rollback reference, not the current stack baseline
