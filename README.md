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

Both stacks use the same four UD-IQ4_XS language-model shards, Q4_0 MTP draft and BF16 vision projector. The four main shards total 93,672,359,738 bytes and have matching SHA256 hashes on both hosts. The previous trunk-q5k package totaled 92,329,675,424 bytes; these mixed-precision packages should not be compared by the short quantization labels alone.

```text
/srv/ai/models/qwen3.8-flash-next-unsloth-iq4xs-v31/
  Qwen3.8-Flash-Next-UD-IQ4_XS-00001-of-00004.gguf
  Qwen3.8-Flash-Next-UD-IQ4_XS-00002-of-00004.gguf
  Qwen3.8-Flash-Next-UD-IQ4_XS-00003-of-00004.gguf
  Qwen3.8-Flash-Next-UD-IQ4_XS-00004-of-00004.gguf

/srv/ai/models/qwen3.8-flash-next/
  mtp-Qwen3.8-Flash-Next-Q4_0.gguf
  mmproj-BF16.gguf
  trunk-q5k-00001-of-00003.gguf  # retained on 01 for rollback
  trunk-q5k-00002-of-00003.gguf
  trunk-q5k-00003-of-00003.gguf

/srv/ai/secrets/qwen-general-api-key
```

The language-model directory is mounted read-only at `/models`. The draft and projector have separate read-only mounts at `/draft` and `/vision`. No weights or credentials belong in this repository.

## Portainer

### ai-brain-01 — general/heavy/coding/vision

Use `portainer/qwen-general.yaml`.

- host: `192.168.1.27`
- API: `8081`
- two slots
- `524288` total context, `262144` per slot (input and output combined)
- Q8_0 target K/V cache; the draft K/V cache remains FP16
- logical batch `2048`; physical microbatch `512`
- Qwen3.8 Flash Next UD-IQ4_XS, matching ai-brain-02
- native vision enabled
- official Q4_0 MTP draft model; `draft-mtp`, maximum three draft tokens, minimum probability 0.0
- `--no-reasoning-preserve`

#### Bounded ai-brain-01 validation — 2026-10-22

The target K/V cache changes from FP16 to Q8_0, the shared context from `262144` to `524288`, and the physical microbatch from `2048` to `512`. Two slots, logical batch `2048`, model weights, native vision, MTP settings and the FP16 draft cache are preserved. The tested image ID is `sha256:40292e60060df51383f0ece220ee92e1f5465c4ddc480105d428dcda5e586e9f`; deployment must retain these image contents.

The existing FP16 service and the Q8 candidate both passed the same 13,049-token three-marker retrieval, order-preserving Python deduplication, red/blue image recognition, function-call and tool-result continuation fixtures. The baseline was serving other traffic, so these observations do not isolate a Q8 speed or quality difference.

The Q8 candidate additionally passed the following synthetic OpenAI-compatible chat requests, with native vision and MTP enabled:

| Request | Input tokens | Reused input tokens | Total time | Result |
| --- | ---: | ---: | ---: | --- |
| Cold long conversation with image | 140,115 | 0 | 617.18 s | Exact expected markers |
| Concurrent second request | 13,049 | 0 | 612.68 s | Exact expected markers |
| Cached continuation of long conversation | 140,166 | 140,111 | 1.66 s | Exact expected markers |

Both slots were observed processing at the same time. The smaller concurrent request still waited behind the large prefill; two slots do not establish independent low latency. The cold long request took roughly ten minutes before completing, whereas its cached continuation completed in under two seconds.

Minimum sampled available system RAM across these long tests was 14.72 GiB. There were no new swap-outs, kernel GPU reset/fault matches, or OOM events during the long tests. These are bounded synthetic checks, not validation of a cold full-262K prompt, arbitrary real project/tool-history shapes, or overnight/multi-day operation.

The historical October cutover recommended LiteLLM `max_input_tokens: 262144`; the November migration below supersedes that metadata recommendation with separate input/output budgets. The backend window includes both prompt and answer. The historical automatic `openwebui` router also included the smaller `general` route, so its derived safe window could remain below the direct `heavy`/`coding` window.

#### Matching ai-brain-02 package — 2026-11-15

ai-brain-01 now uses the exact four-shard UD-IQ4_XS package already deployed on ai-brain-02. The Q4_0 draft and BF16 projector hashes match across hosts. Both use the same immutable image digest, two 262144-token slots, Q8 target K/V, default FP16 draft cache, MTP maximum three / minimum probability 0.0, logical batch 2048 and physical microbatch 512. Ports and native model names remain host-specific. No new engine image was built.

The original container, image, command and canonical source were captured before cutover; its trunk weights and pinned rollback definition remain on 01. Model files were verified against source SHA256 hashes during transfer and again from destination disk. The Compose project remains `ai-inference-containers`; Portainer uses `portainer/qwen-general.yaml`.

Short greedy fixtures used thinking disabled, a 24-token warm-up and a 768-token response limit. Client-observed decode rates exclude time to first token:

| Fixture | Original trunk-q5k, tokens/s | Matching UD-IQ4_XS, tokens/s |
| --- | ---: | ---: |
| sequence_json | 61.15 | 58.99 |
| python_module | 59.29 | 56.72 |
| design_json | 52.66 | 51.27 |

All three short assertions and the same twelve functional Python fixtures outside the generated examples passed for both packages. The fixture set covers stable deduplication order, inclusive interval overlap/touching and group counts; it does not establish broad coding quality equivalence. The matching package was about 3–4% slower on these short fixtures. The original process had run since October 27, whereas the matching process was freshly started, so memory and timing differences are not an isolated quantization experiment.

The same two cold image-bearing requests then ran together on each package. Each requested an input ceiling of 196608 and an output allowance of 32768; responses generated only the short validation payloads below, not full-length outputs.

| Package and request | Input tokens | Generated tokens | Total time | First token |
| --- | ---: | ---: | ---: | ---: |
| Original trunk-q5k: coder | 194655 | 139 | 1306.583 s | 1002.463 s |
| Original trunk-q5k: designer | 194637 | 103 | 2016.593 s | 2012.263 s |
| Matching UD-IQ4_XS: coder | 194655 | 249 | 1615.893 s | 984.469 s |
| Matching UD-IQ4_XS: designer | 194637 | 103 | 2031.170 s | 2026.732 s |

Both packages passed all large-request core checks: required Coder function-call format, Designer JSON values, all three distant markers and exact image colors/numbers. Candidate strict instruction misses: `long_coder.requested_fix_expression`. These are recorded separately from infrastructure/core checks; a core pass is not a claim that every assertion passed. The original Coder returned the exact literal `sorted(set(values))`; the IQ4_XS package's known long-context instruction-following limitation on 02 is documented below.

Long continuation checks were submitted only if an idle slot could be confirmed to hold the matching long prefix. The matching-01 run could not confirm that residency through the available slot telemetry, so these checks were skipped. Candidate skipped checks: `long_cached_continuation_designer; long_cached_continuation_coder`. The original long follow-ups were skipped after live agent traffic evicted their prefixes; the validator's own cold follow-up was cancelled without stopping other users' work. A cold 195K re-prefill is not presented as a cache benchmark. A restricted OpenCode key passed a streamed image/function/tool-result/JSON continuation through the updated `coding` pool, served by 02. A targeted 01 gateway check using the administrator credential passed the streamed vision/tool request in 4.575 seconds and its cached JSON tool-result continuation in 1.138 seconds (471 reused input tokens). These gateway checks have separate authentication and placement scopes; they do not establish a cached 195K continuation on the matching 01 package.

Candidate minimum sampled available RAM was 14.42 GiB; host swap-out counters increased by 7.99 GiB. GPU/OOM kernel matches, runtime device-loss/assertion matches and unexpected container restarts were zero at the end of validation. Host operating-system settings differ: 01's swappiness is 60 and 02's is 10; no kernel settings were changed as part of this model cutover. Native prefills can progress in sequence even with two admitted slots, and large prefill work can delay the other slot's generation.

The client migration contract is context 262144, maximum input 196608 and maximum output 32768 for Coder, Designer and the Memory alias. A formatted request must still fit input plus requested output within its slot. The cold-run timings support the planned 45-minute request timeout (2700 seconds / 2700000 milliseconds). Hermes Agent, Hindsight and OpenCode each need their own timeout migration; changing the backend alone does not update clients. These are bounded synthetic/native and real-key gateway tests, not an overnight or arbitrary project-history qualification.

### ai-brain-02 — frontier/design

Use `portainer/qwen-frontier.yaml`.

- host: `192.168.1.28`
- API: `8082`
- two slots
- `524288` total context, `262144` per slot (input and output combined)
- Q8_0 K/V cache
- Qwen3.8 Flash Next UD-IQ4_XS weights from `/srv/ai/models/qwen3.8-flash-next-unsloth-iq4xs-v31`
- native vision with the existing BF16 projector mounted read-only from `/srv/ai/models/qwen3.8-flash-next/mmproj-BF16.gguf`
- Q4_0 MTP draft mounted read-only from `/srv/ai/models/qwen3.8-flash-next/mtp-Qwen3.8-Flash-Next-Q4_0.gguf`; `--spec-type draft-mtp --spec-draft-n-max 3 --spec-draft-p-min 0.0`, with the default FP16 draft cache
- logical batch `2048`; physical microbatch `512`
- `--no-reasoning-preserve`
- runtime pinned to `ghcr.io/phikapjames/qwen38-llama@sha256:40292e60060df51383f0ece220ee92e1f5465c4ddc480105d428dcda5e586e9f`

For the Git-managed stack, use **Pull and redeploy** after publishing to `main`. Confirm the stack commit, pinned image digest, `-c 524288`, `-np 2`, `-ctk q8_0`, `-ctv q8_0`, `--mmproj /vision/mmproj-BF16.gguf`, `-md /draft/mtp-Qwen3.8-Flash-Next-Q4_0.gguf`, `--spec-type draft-mtp`, `--spec-draft-n-max 3`, `--spec-draft-p-min 0.0`, and two API slots of 262144 tokens with vision and speculation enabled. The stack definition uses the prebuilt runtime image.

#### Bounded two-slot validation — 2026-11-15

The existing one-slot FP16 service passed the initial short-response check. The candidate used the same runtime image and IQ4_XS weights with two full-size slots and Q8 K/V. Streaming, a required function call and its JSON tool-result continuation passed through the OpenAI-compatible API. The two long requests below ran together and returned the exact three context markers. Coder also returned the required code-review function call, and Designer returned the required structured JSON.

| Request | Input tokens | Output allowance | Generated tokens | Total time | First token |
| --- | ---: | ---: | ---: | ---: | ---: |
| Concurrent Coder | 130270 | 24576 | 85 | 1459.451 s | 1455.675 s |
| Concurrent Designer | 195823 | 32768 | 67 | 1455.364 s | 875.239 s |
| Cached Coder continuation | 130389 | 256 | 12 | 2.589 s | 2.031 s |
| Cached Designer continuation | 195910 | 256 | 18 | 2.750 s | 1.877 s |

Both slots were observed active. The engine processed the large prefills in sequence, and Designer generation progressed slowly while Coder prefilled. Cached continuations reused 130266 and 195819 input tokens respectively. Established OpenCode Coder and Designer input/output allowances fit the 262144-token per-slot window.

Minimum sampled available RAM was 14.04 GiB. Host swap-out counters rose by approximately 49.67 MiB across the test, with kernel GPU/OOM matches and container restarts both zero. This validation covered bounded synthetic API requests, tool-result/cache transitions and approximately 25 minutes of concurrent load. Endurance qualification requires sustained representative workloads.

#### Bounded vision and two-slot validation — 2026-11-15

The existing BF16 vision projector reports a 2560-dimensional projection matching the IQ4_XS language model. The candidate adds its read-only file mount and `--mmproj` while using the validated two-slot Q8 configuration. Two concurrent image checks correctly recognized red/blue panels labeled 17/42 and green/orange panels labeled 3/8. The same images were included at the start of the long requests below. Both long replies returned the correct colors, numbers and three distant text markers, with the required function call or structured JSON.

| Request | Input tokens | Output allowance | Generated tokens | Total time | Reused input tokens |
| --- | ---: | ---: | ---: | ---: | ---: |
| Concurrent Coder with image | 129098 | 24576 | 139 | 1441.663 s | 0 |
| Concurrent Designer with image | 194637 | 32768 | 103 | 1444.603 s | 0 |
| Cached Coder with image | 129271 | 256 | 12 | 2.705 s | 129094 |
| Cached Designer with image | 194760 | 256 | 18 | 2.824 s | 194633 |

Minimum sampled available RAM was 15.25 GiB. Host swap-out counters rose by approximately 50.29 MiB, with kernel GPU/OOM matches and container restarts both zero. The concurrent cold run took approximately 24 minutes; image-bearing cached continuations completed in under three seconds. This bounded validation used deterministic 512-by-256 color/number fixtures and the established Coder/Designer input budgets. Requests specified the listed output allowances and generated short validation responses. A restricted OpenCode key also passed image input plus a required function call through LiteLLM’s `design` alias in 6.013 seconds, followed by a cached JSON tool-result continuation in 1.343 seconds. Both responses identified ai-brain-02 as the serving endpoint.

#### Bounded MTP comparison — 2026-11-15

MTP was re-enabled with the existing Q4_0 sidecar, three draft tokens and minimum probability 0.0. The image ID remained `sha256:40292e60060df51383f0ece220ee92e1f5465c4ddc480105d428dcda5e586e9f` for the off/on comparison. The IQ4_XS target weights, vision projector, two 262144-token slots, Q8 target cache, logical batch 2048 and physical microbatch 512 were retained.

Three short greedy fixtures used thinking disabled, a 24-token warm-up and a 768-token output limit. Client-observed generation rates exclude time to first token:

| Fixture | MTP off, tokens/s | MTP on, tokens/s | Rate ratio |
| --- | ---: | ---: | ---: |
| sequence_json | 28.54 | 58.63 | 2.05x |
| python_module | 28.54 | 56.31 | 1.97x |
| design_json | 28.40 | 51.21 | 1.80x |

All short fixture assertions passed. The integer-sequence output was identical, and the Python function bodies had identical syntax trees. Example assertions and design prose differed, so this is not a claim of byte-identical output or general quality equivalence. These short rates do not measure large cold-prefill performance.

Two image-bearing requests then ran together, each with a 196608-token input ceiling and 32768-token output allowance. Coder used 194655 input tokens and generated 249 tokens; native server time was 1621.593 seconds. Designer used 194637 input tokens and generated 103 tokens in 2051.152 seconds. Both returned the exact image colors/numbers and all three distant text markers, with the required function call or structured JSON.

**The original long test did not pass every assertion.** Coder returned a precomputed list of integers 0 through 30 in `fix`, instead of the requested literal expression `sorted(set(values))`. A cached replay repeated this miss. The same original Coder request was then run with MTP disabled, using the same model, image, vision and slot configuration. It also returned the precomputed list, in 874.357 seconds. This miss is not unique to MTP; this single control does not establish general quality equivalence or isolate cold-prefill speed, since the control ran alone. Excluding time to first token, the same 249-token reply yielded approximately 22.87 tokens/s with MTP off and 32.16 tokens/s on the MTP-enabled cached replay. Those rates describe isolated generation at this context depth, not concurrent cold-prefill throughput.

An explicit tool-result correction with MTP enabled returned the required expression and preserved all image and marker fields in 7.205 seconds. The subsequent image-bearing Coder JSON/tool-result continuation completed in 2.210 seconds with 194978 cached input tokens. Designer's image-bearing JSON continuation completed in 3.360 seconds with 194633 cached input tokens.

Minimum sampled available RAM during the concurrent MTP run was 12.91 GiB. Host swap-out counters rose by 3.41 GiB. Kernel GPU/OOM matches, runtime device-loss/assertion matches and container restarts were zero. Both slots reported speculation enabled and accepted draft tokens were observed. The concurrent cold run took about 34 minutes, exceeding the existing 30-minute OpenCode request timeout. Prefills progressed in sequence, and the other slot's generation slowed while a large prefill ran. No client context limits or timeouts were changed.

A restricted OpenCode key passed image input with a streamed required function call and a streamed JSON tool-result continuation through LiteLLM's existing `design` alias after MTP was restored. These bounded fixtures do not qualify arbitrary real project histories, full-window output generation or overnight operation. The recorded instruction miss and cold-start timeout exposure should remain visible when assessing a larger shared pool.

#### Historical one-slot tuning — October 2026

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

Then exercise normal OpenCode tool-result continuations and sustained representative work. Record completed responses, prompt/generation timings, input/output/cache token counts, server restarts, kernel GPU faults and memory/swap activity. Any new device-loss event fails the candidate. A short health check, synthetic-marker reply or single passing long prompt is not overnight acceptance. At that stage, microbatch `512` had not yet been benchmarked on this deployment; the later bounded checks above extend validation without establishing overnight acceptance.

## First publish

GitHub Container Registry packages created under a personal account are private by default. After the first successful workflow run, open the `qwen38-llama` package settings and change its visibility to **Public** if anonymous pulls from Portainer are desired.

If the package remains private, authenticate Docker/Portainer to `ghcr.io` before redeploying the stacks.

### Deployed ai-brain-02 runtime — 2026-11-25

The `rollout/brain02-72cda4a` branch is deployed to ai-brain-02 through Portainer. Its image is `ghcr.io/phikapjames/qwen38-llama@sha256:d0975bfe47c47a157e2590ed40ec211b815b85bc2e3068f08d5fa477ea6371a4`, built from llama.cpp `72cda4a2228c254f95a556fa238488a1324b04d8` (build 12704). Its Ubuntu 26.04 runtime uses signed Kisak Mesa `26.3.1~kisak2~r` and Vulkan loader `1.4.341.0-1`. The builder uses glslc `2026.1-1`, Vulkan headers `1.4.341.0-1` and SPIR-V headers `1.6.1+1.4.341.0-1`. The obsolete local vision assertion patch was removed. The image's Vulkan summary detected the Radeon 8060S through RADV; this does not constitute an inference test.

The deployed stack enables reasoning preservation and explicitly retains the 8192 MiB RAM prompt cache and 32 context checkpoints per slot. Its model mounts, MTP settings, slot and context sizes, cache quantization and batch sizes are unchanged. The user explicitly waived the benchmark-before-change requirement and requested short operational checks only; no endurance or performance validation is claimed. Rollback remains the prior image `sha256:40292e60060df51383f0ece220ee92e1f5465c4ddc480105d428dcda5e586e9f` and pre-rollout stack configuration. The process started with the model, BF16 vision projector, MTP draft, two slots and 262144 tokens per slot. A template-only check rendered an existing reasoning marker once. LiteLLM routes the physical deployment via `http://100.113.74.79:8082/v1`; the canonical ai-brain-01 later-migration URL is `http://100.66.68.56:8081/v1`. DirectIO was requested but the engine reported buffered-I/O fallback on some files. Shared persistent slot ownership, warm-slot waiting, full client reasoning replay and end-to-end tool continuation still require implementation or verification; the rollout is incomplete. Streaming keepalives and cancellation from a queued client have passed a short test. The user approved interrupting in-flight work for the rollout.
