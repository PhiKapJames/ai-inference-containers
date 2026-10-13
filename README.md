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
- logical batch `2048`; physical microbatch `512` (candidate awaiting validation)
- `--no-reasoning-preserve`

**Frontier is not yet qualified for overnight development.** MTP-off passed the bounded native tests below, but a later real OpenCode prompt triggered another GPU reset. Disabling MTP alone was insufficient.

The next isolated candidate reduces `-ub` from `2048` to `512`, the pinned engine's default maximum physical batch size. The logical batch remains `2048`, context `262144`, one slot, MTP disabled, and the same model, image reference, flash-attention setting and load mode. Smaller physical batches may reduce temporary graph work and slow prompt processing; they do not reduce the configured context or establish a fix.

Related [AMD Vulkan reports](https://github.com/ggml-org/llama.cpp/issues/20515) describe sensitivity to microbatch size and context depth, including failures at `512`. Another [Strix Halo report](https://github.com/ggml-org/llama.cpp/issues/27306#issuecomment-5378231906) reproduces device loss with MTP disabled and large batches. These use different models or revisions and support this experiment, not a guaranteed cure.

After redeployment, confirm that no draft context is loaded, then test a cold long prompt and several continued tool calls. Record the actual model input tokens and output budget, completed responses, kernel GPU resets, memory/swap activity and client timeouts. A successful health check or short reply alone does not establish overnight stability.

The stacks pull the GHCR image built from the pinned upstream commit instead of depending on a host-local Docker image.

For the Git-managed `qwen-frontier` Portainer stack, use **Pull and redeploy** after the change is published to `main`. Confirm that the stack's Git commit advances and the recreated container uses `--spec-type none` without `-md`, `-ub 512`, `-b 2048` and `-c 262144`. Restarting the existing container alone does not apply the new command. This stack-only change does not rebuild the container image.

For the MTP comparison, record the running image ID or digest before and after redeployment. The existing `pull_policy: always` and mutable registry tag can fetch different image contents even when the image reference is unchanged. If the image changes, the result no longer isolates MTP alone.

#### Earlier bounded host validation — 2026-10-13, MTP-off with `-ub 2048`

| Native request | Input tokens | Reused tokens (`timings.cache_n`) | Total time | Generated / limit |
| --- | ---: | ---: | ---: | ---: |
| Cold prompt | 190,007 | 0 | 23m 29s | 170 / 768 |
| Cached-prefix extension | 224,993 | 190,309 | 5m 23s | 181 / 768 |

Both responses ended on EOS without truncation and passed synthetic-marker checks. Six short cached continuations completed in 1.43–2.34 seconds, generating 17 tokens each within a 128-token limit. The 225K request extended an existing cached conversation; it was not a cold 225K test. Observed generation was approximately 21 tokens/second; the table includes prompt-processing time.

Collected monitoring showed no GPU/OOM kernel matches, server PID change, or increase in swap-out activity during these tests and the subsequent short proxy check. A short streaming request through the LiteLLM Frontier route also returned HTTP 200. The API reported build `b11552-23b0202a1`, one slot and the full 262144-token context.

These tests used native `/completion` requests and synthetic records. They did not demonstrate actual tool calls, large-context requests through OpenCode, or overnight/multi-day endurance. Those checks remain outstanding. Native prompt-progress events kept the test connections active; they do not demonstrate client tolerance of a long silent prompt-processing interval.

Running image IDs or digests were not recorded before and after redeployment, so the observations cannot isolate MTP as the cause of the earlier failures.

#### Subsequent real-workload failure — 2026-10-13

At `23:32:56 UTC` (`19:32:56 America/New_York`), the kernel reported an AMDGPU `comp_1.1.0` ring timeout attributed to the same `llama-server` process that completed the tests above. The compute queue reset and the device recovered, but the server lost its Vulkan context and crashed. The supplied server log stopped prompt processing at `159725` tokens, with `truncated = 0`; the restarted process still used MTP-off, `-ub 2048`, `-b 2048` and the full `262144` context. Subsequent client requests encountered `Loading model` while the server initialized again.

The observed swap-out counter did not increase and no kernel OOM event was recorded. This establishes a fresh GPU failure, not the particular faulty shader or a general context-size limit. The earlier synthetic passes remain valid observations but did not qualify this real cached-history request. The independent OpenCode HTTP timeout correction does not prevent GPU device loss.

#### Acceptance for the `-ub 512` candidate

After redeployment, record the actual command, build and image digest. Validate the failing conversation's input, template and tool-schema shape through the OpenAI-compatible/LiteLLM path, including cold prompt processing and subsequent prefix-cache transitions. Keep the assessment read-only and within the input/output budget; do not execute project implementation or hardware actions as an infrastructure test.

Then exercise normal OpenCode tool-result continuations and sustained representative work. Record completed responses, prompt/generation timings, input/output/cache token counts, server restarts, kernel GPU faults and memory/swap activity. Any new device-loss event fails the candidate. A short health check, synthetic-marker reply or single passing long prompt is not overnight acceptance. Microbatch `512` has not yet been benchmarked on this deployment.

## First publish

GitHub Container Registry packages created under a personal account are private by default. After the first successful workflow run, open the `qwen38-llama` package settings and change its visibility to **Public** if anonymous pulls from Portainer are desired.

If the package remains private, authenticate Docker/Portainer to `ghcr.io` before redeploying the stacks.
