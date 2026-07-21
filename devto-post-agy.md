title: "Self-Hosted Gemma 4 on TPU v6e: Deployment & SRE with Antigravity"
published: false
tags: tpu, llm, vllm, antigravity, googlecloud
description: "From quota-sweeping to self-hosted SRE diagnostics: how the Antigravity TPU skill automates Gemma 4 serving on flex-start v6e chips."
---

*Measured 2026-07-21 on `vllm/vllm-tpu:nightly` (vLLM 0.23.1rc1.dev1076), a GCE flex-start `ct6e-standard-1t` (one TPU v6e chip, 32 GB HBM) in europe-west4-a, managed by the **Antigravity TPU DevOps agent**.*

## TL;DR

**The plain `google/gemma-4-E2B-it` serves beautifully on one v6e chip; the Antigravity TPU skill automates the entire provisioning and SRE lifecycle.** The 2-billion-parameter "efficient" Gemma 4 sustains 213 tok/s for a single user, scales to ~2,200 output tok/s, and handles complex tool-calling flawlessly.

This post deep-dives into the serving performance and failure modes of the QAT checkpoints, and introduces the **tpu-devops** MCP server—a specialized Antigravity skill that handles zone-sweeping for capacity, vLLM orchestration, and even uses the deployed model to analyze its own system logs.

One capability coupling to know about: with a reasoning parser configured, **schema enforcement only engages when thinking is enabled** — thinking-off requests sail through unconstrained with a 200 status. Details on this and the agent-driven deployment below.

The host is a GCE **flex-start** VM — capacity granted on request, billed until deleted, hard-stopped at a 4-hour max run. Manually, this requires a massive `gcloud compute instances create` command with specific startup scripts and IAM scopes.

With the **Antigravity TPU skill**, this is a single natural-language request: *"Find a v6e-1 in europe-west4 and serve Gemma 4 E2B."*

The agent's `find_tpu_vm` or `create_tpu_vm_instance` tools handle:
- **Quota Sweeping:** Attempting multiple zones until capacity is granted.
- **Infrastructure Setup:** Attaching a 200 GB boot disk (the default 10 GB is too small for vLLM images) and configuring `cloud-platform` scopes.
- **Secrets Management:** Securely fetching the `hf-token` from Secret Manager without embedding it in metadata.
- **Container Orchestration:** Pulling `vllm/vllm-tpu:nightly` and applying the specialized serving flags: `--max-model-len 65536 --gpu-memory-utilization 0.9 --max_num_batched_tokens 4096 --enable-auto-tool-choice --tool-call-parser gemma4 --reasoning-parser gemma4`.

Boot timeline: VM RUNNING at t+0 → Docker installed ~t+1:00 → image pulled ~t+6:00 → weights downloaded, XLA compiled, health green ~t+8:30. The agent's `wait_for_vllm_ready` tool polls the endpoint until the first 200 OK is received.

Two environment quirks the agent helps navigate:
- **IAP Tunneling:** Direct SSH often fails on corporate networks. The agent uses `gcloud compute ssh --tunnel-through-iap` for its diagnostic tools (`get_vllm_docker_logs`, `manage_vllm_docker`).
- **KV Cache Optimization:** vLLM auto-selects an **fp8_e5m2 KV cache** on v6e — the largest memory consumer is 8-bit before any weight quantization enters the picture.

## 2. Three ways the QAT checkpoints fail to load

| Checkpoint / path | Failure | Verdict |
|---|---|---|
| `-qat-w4a16-ct` · JAX | int4 compressed-tensors scheme unimplemented for E2B's `per_layer_model_projection` | ✕ no load |
| `-qat-q4_0-unquantized` · JAX | `k_norm.weight` "missing" for layers 15–34 | ✕ no load |
| `-qat-q4_0-unquantized` · torchax | identical missing-weights error via `MODEL_IMPL_TYPE=vllm` | ✕ no load |
| `gemma-4-E2B-it` (plain) · JAX | loads and serves | ✓ serves |

**The forensics point at the loader, not the checkpoint.** Reading the safetensors headers of both repos: the plain export ships `self_attn.k_norm` for all 35 layers; the QAT export ships it only for the 15 non-KV-shared layers. Both configs are identical — including `num_kv_shared_layers: 20`. Layers 15–34 reuse K/V from lower layers and have no k-norm of their own, so the QAT export is the architecturally honest one; the plain checkpoint only loads because it carries those (unused) tensors anyway. Proposed fix in [#3225](https://github.com/vllm-project/tpu-inference/issues/3225): skip instantiating K/V-side parameters for KV-shared layers.

Until it lands: **serve the plain checkpoint.** At 2B parameters (~5 GB bf16 against 32 GB HBM), 4-bit weights buy little here anyway — memory pressure lives in the KV cache, which is already fp8.

## 3. What one chip is worth: the concurrency sweep

Same workload at every level — 1,024-token prompts, 128-token completions, `vllm bench serve`, random dataset.

| Concurrency | Req/s | Output tok/s | Total tok/s | TTFT med | TTFT p99 | TPOT med | Per-stream tok/s |
|---|---|---|---|---|---|---|---|
| 1 | 1.64 | 209 | 1,884 | 16 ms | 17 ms | 4.7 ms | 213 |
| 8 | 9.44 | 1,209 | 10,878 | 27 ms | 99 ms | 6.2 ms | 161 |
| 32 | 12.78 | 1,636 | 14,721 | 155 ms | 189 ms | 17.5 ms | 57 |
| 64 | 16.72 | 2,140 | 19,262 | 122 ms | 349 ms | 25.3 ms | 39 |
| 100 (burst) | 17.31 | 2,215 | 19,938 | 833 ms | 1,573 ms | 36.8 ms | 27 |

Reading the curve:

- **Prefill is effectively free at low load.** A 1,024-token prompt reaches first token in 16 ms — roughly 64K prefill tok/s for a single stream.
- **c=8 is nearly free concurrency:** six times the throughput of a single stream for +1.5 ms/token; each of 8 users still sees ~160 tok/s.
- **The knee is between 32 and 64.** Going 32→64 buys +31% throughput for +45% per-token latency; 64→burst buys +3.5% for another +45%.
- **Capacity-planning number:** run at ≤64 concurrent streams for smooth per-user experience; the ceiling is ~17 req/s at this workload shape.

(Single run per configuration — more meaningful here than usual: a kernel study on this same stack, cited in §7, measured run-to-run cv ≤ 0.3% under greedy decoding with static shapes.)

## 4. Function calling at 2B scale

Served with `--tool-call-parser gemma4 --enable-auto-tool-choice`, probed with two OpenAI-style tools at temperature 0. Five scenarios, five clean results:

| Probe | Behavior | Latency |
|---|---|---|
| Simple call | correct tool, inferred the optional `unit` arg from phrasing | 166 ms |
| Result synthesis | fed the tool result back → clean natural-language answer | 140 ms |
| No-tool restraint | answered directly, no spurious call | 97 ms |
| Parallel calls | both tool calls emitted in one turn, correct args each | 150 ms |
| Underspecified | asked "What city are you interested in?" instead of hallucinating a call | 44 ms |

A 2B model producing well-formed `tool_calls` JSON, choosing correctly between calling and answering, batching parallel calls, and asking for missing arguments — at double-digit-millisecond latency. For high-volume, low-complexity agent steps, the quality floor is higher than the parameter count suggests.

## 5. Structured output works — but only with thinking on

| Probe | Observed behavior | Verdict |
|---|---|---|
| `json_schema`, thinking off (default) | free prose with a 200 status; `strict: true`, `guided_json`, `structured_outputs` spellings all equally unenforced | ✕ silently skipped |
| `json_object`, thinking off | fenced code block, array where an object was asked, invented enum value | ± prompt-level |
| `json_schema` + `enable_thinking` | **exact schema conformance** — bare JSON object, typed integer, "ASAP" correctly mapped into the `high` enum | ✓ enforced |
| Reasoning, default | no reasoning traces on any prompt | off by default |
| Reasoning, `enable_thinking` | parser cleanly splits thinking trace from a terse answer; ~2.4× completion tokens | ✓ works |

The mechanism: with `--reasoning-parser gemma4` configured, vLLM defers grammar enforcement until the reasoning section ends. Thinking off → no reasoning terminator → the grammar never engages, and unconstrained prose ships with a 200 status. Enable thinking (`"chat_template_kwargs": {"enable_thinking": true}`) and the same request is enforced exactly.

**Operational guidance:** pair structured output with `enable_thinking: true` under a reasoning parser — or drop `--reasoning-parser` from servers that don't need it. And keep client-side validation regardless: the silent-skip failure mode means a trusting client can't tell an enforced response from a lucky one.

## 6. Vision at 2B: accurate and nearly free

One restart with `--limit-mm-per-prompt '{"image":4,"audio":1}'` makes it a vision server. COCO validation images, base64 data URIs, temperature 0:

| Probe | Answer (abridged) | Latency |
|---|---|---|
| Describe (two cats) | "Two tabby cats… on a bright pink surface… a remote control visible" | 197 ms |
| Count + attributes | 2 animals, both cats, remote + blanket identified | 421 ms |
| Scene (bear) | "A bear lying down in a grassy outdoor environment" | 329 ms |
| Room inventory | wall-mounted TV, shelving, furniture correctly enumerated | 874 ms |

An image costs ~280 prompt tokens and adds almost nothing over a text request. Two notes: server-side fetching of external image URLs proved flaky (intermittent 422s) — **base64 data URIs are the reliable path** — and the first multimodal request after boot can 422 while the processor warms; retry once.

## 7. fp8 KV cache, HBM anatomy, and a related kernel result

**fp8 vs bf16 KV:** six greedy prompts (explanation, code, listing, translation, arithmetic, summarization — 889 completion tokens) run under the default fp8_e5m2 cache, then re-run after a restart with `--kv-cache-dtype bfloat16`. **Result: 6 of 6 outputs byte-identical.** On this (small, greedy) sample the compression is genuinely free — take the fp8 cache.

**Where the 32 GB goes** (bf16-KV boot, 65,536 max context): usable HBM reports as 31.24 GiB; at 0.9 utilization vLLM works within 28.12 GiB — roughly 5.5–6 GiB weights, 16.3 GiB KV cache (8,713 blocks × 128 tokens × 15 layers × 128 KiB), ~6 GiB workspace.

The interesting physics: E2B's KV sharing is directly visible in the allocator — **only 15 of 35 layers hold KV tensors**, each with a single 256-dim KV head, so a token costs ~15 KiB of KV in bf16 (~7.5 KiB under fp8). That yields **~1.1 million tokens of resident KV** — seventeen full 65K-context conversations on-chip — which is why the sweep saturated on *compute*, never memory. Also: of the 404-second engine init, 329 seconds is XLA compilation — the dominant term in the ~10-minute cold start.

**Related work:** a kernel-substitution study by Zimbres ([DOI 10.5281/zenodo.21404069](https://doi.org/10.5281/zenodo.21404069)) shows the RPA v3 kernel's decode block-size heuristic costs 27.7–68.7% of large-batch throughput on 27B/31B models on v6e. E2B sits at the low-exposure end of that effect: at our c=64 operating point, attention is ~9% of memory traffic (vs ~41% in their regime) precisely because of the KV-sharing design above. Testing the override on E2B — smallest model, one chip, KV sharing — is a natural follow-up.

## 8. Cost breakdown

Rate verified against Google's published [Dynamic Workload Scheduler pricing](https://cloud.google.com/products/dws/pricing): **$1.35 per chip-hour** for v6e flex-start (europe-west4, us-east1, us-east5, asia-northeast1).

| Operating point | Output tok/s | $ / M output tokens |
|---|---|---|
| Saturation (burst) | 2,215 | $0.17 |
| Sweet spot (c=64) | 2,140 | $0.18 |
| Interactive (c=8) | 1,209 | $0.31 |
| Single stream | 209 | $1.79 |

- **Breakeven vs. the API:** against [Gemini 2.5 Flash-Lite](https://ai.google.dev/gemini-api/docs/pricing) at $0.40/M output tokens, self-hosting wins once you sustain ~940 output tok/s — roughly the c=8 operating point held continuously.
- **Cold start** (~8.5–10.5 min, mostly XLA compile) costs $0.19–0.24 per provisioning.
- **A full 4-hour session** costs $5.40 and, at saturation, delivers ~30M output tokens — about $12 worth at Flash-Lite prices. Flex-start fits batch bursts, not idle always-on endpoints.

## 9. Self-Hosting your AI Assistant with LiteLLM

One of the most powerful ways to use this setup is to point your existing AI tools at your TPU. The Antigravity repo includes a guide for using **LiteLLM Proxy** to redirect standard Gemini CLI requests to your self-hosted TPU endpoint.

1.  **Get the Endpoint:** Use `get_vllm_endpoint` or `get_tpu_vm_endpoint`.
2.  **Configure LiteLLM:** Map `gemini-2.0-flash` to your TPU endpoint in a `litellm_config.yaml`.
3.  **Point your CLI:** Set `GOOGLE_GEMINI_BASE_URL="http://localhost:4000"` and use your self-hosted model for code analysis or chat.

This effectively gives you a private, high-performance inference engine that costs ~$1.35/hour and can handle massive context windows (65K tokens on-chip).

## 10. The Antigravity SRE Experience

The Antigravity TPU skill doesn't just deploy; it operates. The `analyze_cloud_logging` tool uses the very model you just deployed to triage system logs. 

If a deployment fails, you can ask Antigravity: *"Why did the v6e-1 deployment in europe-west4-a fail?"*. The agent will:
1.  Fetch Cloud Logging entries for the resource.
2.  Send them to the Gemma 4 model on a *different* active TPU (or a local fallback).
3.  Report a human-readable diagnosis (e.g., "The startup script failed because the Hugging Face token secret was not accessible to the compute service account").

## 11. Reproduction with Antigravity

```bash
# 1. Install the skill (Global or Project-specific)
make skill-install-agy

# 2. Deploy (Natural Language)
# "Find a v6e-1 in europe-west4, serve gemma-4-E2B-it, and tell me when it's ready."

# 3. Monitor (Agent Tool)
# The agent will use wait_for_vllm_ready and get_vllm_docker_logs
# to ensure the ~10 minute cold-start completes successfully.

# 4. Benchmark
# "Run a throughput sweep on the active v6e for concurrency 1, 8, and 32."
# (The agent invokes run_vllm_benchmark save_result=True)
```

*Environment: vLLM 0.23.1rc1.dev1076+g5c342876a (vllm-tpu:nightly, tpu-inference backend) · TPU v6e-1 (ct6e-standard-1t, 32 GB HBM, GCE flex-start) · bf16 weights, fp8_e5m2 KV cache managed by the **Antigravity TPU DevOps agent**.*
