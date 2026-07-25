---
name: tpu-management
description: Manage Google Cloud TPU capacity (v6e, v5p, v5e) and Gemma 4 vLLM serving on TPU VMs. Triggers include "TPU", "queued resource", "flex-start", "v6e", "vLLM on TPU", "TPU quota".
---

# TPU Management

Operate Google Cloud TPU serving infrastructure for Gemma 4: acquire capacity, run vLLM,
verify health, benchmark, and tear down. Two ways to act:

1. **Preferred — MCP agent tools.** If the `tpu-devops` MCP server is
   connected in this session, use its tools (catalog below). They wrap the correct
   `gcloud` invocations, discovery, and retry/cleanup logic.
2. **Fallback — direct `gcloud`.** If the MCP server is not connected, either offer to
   register the bundled server (see "Registering the MCP server") or run the equivalent
   `gcloud` commands from `references/tpu-guide.md`.

## Bundled files

- `mcp/server.py` — the FastMCP DevOps agent (snapshot of the repo-root `server.py`;
  the live copy at the repo root is authoritative if the two differ).
- `mcp/project-setup.sh` — one-command installer: copies this skill into a target project and
  registers the MCP server (see "Registering the MCP server").
- `mcp/startup_script_template.sh` — the TPU VM startup script the agent injects when
  creating a queued resource (pulls `vllm/vllm-tpu:nightly` and serves the model).
- `references/tpu-guide.md` — the TPU getting started guide: prerequisites,
  flex-start capacity zones per TPU family, `gcloud` creation templates for v6e/v5p/v5e,
  persistent-disk + startup-script patterns, quota metrics and request procedure,
  troubleshooting/FAQ. Read it when working without the MCP tools, diagnosing
  provisioning failures, or answering quota/capacity/billing questions.

## Registering the MCP server

Easiest path — run the bundled installer (idempotent; installs this skill into the
target project and writes the `tpu-devops` entry into the project's `.mcp.json`,
using the system `python3` — it warns if the pip deps below are missing but never
creates a venv):

```bash
mcp/project-setup.sh /path/to/project --project <gcp-project-id>   # one project
mcp/project-setup.sh --global                                      # all projects (user scope)
# from the skill repo root: 
make init TARGET=/path/to/project ARGS='--project <id>'
make skill-install                                                 # global install for Antigravity
make plugin-install                                                # register as local plugin for Antigravity
```

Run `mcp/project-setup.sh --help` for all options (`--model`, `--accelerator`, `--tp`,
`--server-name`, `--skip-deps`). Then restart your agent (Claude or Antigravity) in the target project and
approve the server when prompted; the `tpu-devops` tools should be available.

Requires: `pip install -r mcp/requirements.txt`, an authenticated
`gcloud` CLI with alpha components (`gcloud components install alpha`), and the TPU API
enabled. The server reads config from env vars: `GOOGLE_CLOUD_PROJECT` (falls back to
the active gcloud config), `GOOGLE_CLOUD_ZONE` (default `europe-west4-a`),
`GOOGLE_CLOUD_REGION`, `MODEL_NAME`, `ACCELERATOR_TYPE`, `TENSOR_PARALLEL_SIZE`.
A Hugging Face token must exist as Secret Manager secret `hf-token` (save one with the
`save_hf_token` tool) before any resource creation.

## Standard lifecycle

1. **Status first.** `get_system_status` (dashboard) or `list_queued_resources` /
   `find_gpu`. Never create before checking what already exists.
2. **Acquire capacity.**
   - Preferred (v6e/v5p): `create_tpu_vm_instance` — GCE flex-start VM with vLLM
     auto-start — or `find_tpu_vm` to sweep zones until one grants capacity (family
     quota is only discoverable by attempting creation). Then `wait_for_vllm_ready`
     polls until serving is up; `get_tpu_vm_serial_log` for manual watching.
   - Known zone, legacy API: `create_tpu_queued_resource` (non-destructive; skips if
     the resource already exists) or `manage_queued_resource` (destructive — deletes
     every other queued resource in the zone). Flex-start by default: 4h max-run;
     `reserved=True` for reservations.
   - Unknown zone: `get_zones_with_available_quota`, or `find_tpu` which sweeps every
     zone with quota, polls until ACTIVE (3 min, extended to 10 min once PROVISIONING),
     and cleans up failures. It skips zones previously marked failed in
     `~/.cache/tpu-devops/tpu_zones_status.md`.
3. **Wait for ACTIVE.** `describe_queued_resource`.
   Queued resources move QUEUED → PROVISIONING → ACTIVE; FAILED/SUSPENDED means
   delete and retry (the manage tool does this automatically).
4. **Serve.** The creation startup script auto-starts vLLM. Otherwise
   `manage_vllm_docker` with action `start|stop|restart|status|log|rm` (targets the
   queued resource's node by default, or a GCE VM via `instance_name` — same for
   `get_vllm_docker_logs`, `get_tpu_system_logs`, `run_vllm_benchmark`).
   **Switching the model:** call `start` with `model_name` (or any serving param) —
   that replaces the container with the new config; a plain `start` just restarts
   the existing container unchanged. It auto-picks
   load format, max-model-len, and memory utilization from the model size
   (26B/31B → `tpu_streaming_loader`, 16384 ctx, 0.80 util; smaller → `runai_streamer`,
   65536 ctx, 0.90 util). Model load can take many minutes — check
   `get_vllm_docker_logs` for "Application startup complete."
5. **Verify.** `verify_model_health`, `get_vllm_endpoint`, `get_model_details`,
   `query_queued_gemma4` (`include_stats=True` for TTFT/throughput). Health checks,
   queries, and benchmarks auto-target whatever model the server actually loaded
   (via `/v1/models`), so they keep working after a deploy-time `model_name` override.
6. **Benchmark (optional).** `run_vllm_benchmark` (runs `vllm bench serve` in a
   separate container on the VM). Pass `save_result=True` to also get the run's
   metrics as a `throughput.sweep[]` entry for the repo's benchmark report format
   (`benchmarks/serving-report.schema.json`) — use it once per concurrency level
   when building a report.
7. **Tear down.** `destroy_queued_resource` / `destroy_tpu_vm_instance`. Flex-start bills until deletion and
   cannot be paused — if the user explicitly requests teardown, proceed immediately without prompting.
   Only confirm teardown if acting on unrequested idle resources, and remind them that flex-start resources left running expire at max-run-duration.

## MCP tool catalog (by task)

**Capacity & lifecycle (GCE flex-start — recommended for v6e/v5p):**
`create_tpu_vm_instance` (creates the VM with the proven flags: 200GB boot disk,
docker-installing startup script, cloud-platform scopes), `find_tpu_vm` (zone sweep),
`wait_for_vllm_ready` (poll until serving), `list_tpu_vm_instances`,
`destroy_tpu_vm_instance`, `get_tpu_vm_serial_log`, `get_tpu_vm_endpoint`

**Capacity & lifecycle (queued resources — legacy, v5e):** `find_tpu`,
`create_tpu_queued_resource` (non-destructive),
`manage_queued_resource` (destructive cleanup), `destroy_queued_resource`,
`list_queued_resources`, `describe_queued_resource`,
`get_zones_with_available_quota`, `find_gpu`, `estimate_deployment_cost`

**Serving:** `manage_vllm_docker`, `get_vllm_endpoint`,
`get_vllm_deployment_config` (gcloud one-liner), `save_hf_token`

**Health, logs & diagnostics:** `get_system_status`, `verify_model_health`,
`get_model_details`, `get_metrics`, `get_vllm_docker_logs`, `get_tpu_system_logs`,
`get_cloud_logging_logs`, `analyze_cloud_logging` (Gemma-4-powered log triage)

**Inference & benchmarking:** `query_queued_gemma4` (`include_stats=True` for
latency/throughput), `run_vllm_benchmark`

Every agent in this repo also exposes `get_help` for its live configuration.

### Extended Model Summary Format (`get_model_details`)

When inspecting a deployment using `get_model_details`, the tool queries the serving host and outputs a structured Markdown report containing four extended diagnostic sections:

1. **Model Information (`/v1/models`)**: Raw JSON payload containing active model ID, ownership, and object type.
2. **vLLM Engine Version (`/version`)**: Installed vLLM engine version string (e.g. `0.8.0.dev`).
3. **Health Status (`/health`)**: HTTP health endpoint evaluation (`Healthy` ✅ / `Unhealthy` ❌).
4. **Key Prometheus Metrics (`/metrics`)**: Extended runtime telemetry filtered for core operational metrics:
   - `vllm_num_requests_running`, `vllm_num_requests_waiting`
   - `vllm_gpu_cache_usage_perc` / `vllm_tpu_cache_usage_perc`
   - `vllm_avg_prompt_throughput_tok_per_s`, `vllm_avg_generation_throughput_tok_per_s`
   - `process_resident_memory_bytes`

## vLLM on TPU — required flags (Gemma 4)

When composing or reviewing a vLLM serve command for TPU, use:
`--tensor-parallel-size 8` (v6e-8), `--max-model-len 16384`,
`--disable_chunked_mm_input`, `--max_num_batched_tokens 4096`,
`--enable-auto-tool-choice --tool-call-parser gemma4 --reasoning-parser gemma4`,
and `--limit-mm-per-prompt '{"image":4,"audio":1}'` for multimodal
(the agent uses `{"image":0,"audio":0}` for text-only serving).
Image: `vllm/vllm-tpu:nightly`, run with `--privileged --net=host --shm-size 10gb`
and `HF_HOME=/dev/shm`.

Upstream references: [vLLM TPU docs](https://docs.vllm.ai/projects/tpu/en/latest/),
[Recommended Models & Features](https://docs.vllm.ai/projects/tpu/en/latest/recommended_models_features/)
(the support matrix — check it before serving quantized checkpoints),
[vLLM Recipes](https://recipes.vllm.ai) (per-model deployment guides), and the
[tpu-inference GitHub repo](https://github.com/vllm-project/tpu-inference)
([releases](https://github.com/vllm-project/tpu-inference/releases) track newly
landed quantization/model support).

Known-broken (verified on `vllm-tpu:nightly`, Jul 2026): the Gemma 4 **E2B and E4B QAT**
checkpoints do not load on JAX/TPU in any form:
- **Quantized (`-qat-w4a16-ct`):** fails with `compressed-tensors scheme for layer 'per_layer_model_projection' is not yet supported in the JAX path`.
- **Unquantized (`-qat-q4_0-unquantized`):** fails during weight loading check with `ValueError: Following weights were not initialized from checkpoint: {layers.XX.self_attn.k_norm.weight}`. This occurs because the model loader demands normalization parameters for KV-shared layers that the QAT checkpoint legitimately omits.

**Workaround:** Serve the plain unquantized base models (e.g. `google/gemma-4-E2B-it` or `google/gemma-4-E4B-it`) instead. The JAX engine automatically saves memory by running the KV cache in FP8 (`fp8_e5m2`).

## Model Sizing & Context Length Constraints

- **Gemma 4 2B / 4B on `v6e-1` (32 GB HBM):**
  - Weights occupy ~5.75 GB to 8.00 GB. Supports full **65,536 context length** (`--max-model-len 65536`) with ample FP8 KV Cache room.
- **Gemma 4 12B on `v6e-1` (32 GB HBM):**
  - Weights occupy ~24 GB, leaving only ~3.56 GB for KV Cache.
  - **Must cap context:** Set `--max-model-len 8192` for single-chip serving. Attempting `--max-model-len 65536` or `12288` will trigger a KV cache memory `ValueError` OOM on startup.
  - **Full Context Workaround:** For 65,536 context length on 12B, deploy on a 4-chip pod slice (`v6e-4`, 128 GB total HBM) with `--tensor-parallel-size 4`.



## Field notes — GCE flex-start path (`gcloud compute instances create`)

Verified on a live v6e-1 deployment (Jul 2026). When creating TPU VMs as GCE
instances (the reference guide's template) rather than queued resources, the
guide's command as written will fail; apply all of these:

- **Boot disk:** the default is only 10 GB (hyperdisk-balanced) — `vllm/vllm-tpu:nightly`
  overflows it during layer extraction ("no space left on device"). Add
  `--boot-disk-size=200GB`. If already created, recover without losing flex-start
  capacity: `gcloud compute disks resize <name> --size=200GB` then
  `gcloud compute instances reset <name>` (never delete/recreate — that forfeits
  the capacity grant and restarts the max-run clock).
- **Docker:** not preinstalled on the `ubuntu-accel-2204-amd64-tpu-v5e-v5p-v6e`
  image (unlike TPU runtime images). The bundled startup script template now
  installs `docker.io` when missing; custom scripts must do the same.
- **Secrets at boot:** add `--scopes=cloud-platform` at creation and grant the
  default compute SA `roles/secretmanager.secretAccessor` on `hf-token`
  (`gcloud secrets add-iam-policy-binding hf-token --member=serviceAccount:<project-number>-compute@developer.gserviceaccount.com --role=roles/secretmanager.secretAccessor`).
  The bundled startup template fetches the token at boot via the metadata server +
  Secret Manager REST API with a retry loop (~30 min) so an IAM grant applied after
  creation still lands — the token is never written into instance metadata. Custom
  scripts should do the same. Symptom of a missing grant/scope: the fetch 403s forever.
- **Watch boot via serial console, not SSH:** SSH is often blocked by firewall
  policy. The startup template mirrors its log to `/dev/console`; follow it with
  `gcloud compute instances get-serial-port-output <name>`. Grep for the final
  "vLLM application startup complete." line — the earlier "Waiting for
  'Application startup complete.'" echo is a false-positive match.
- **When direct SSH times out, tunnel through IAP:** even with a VPC rule
  allowing tcp:22, an org policy or the client network may drop direct port-22
  traffic (symptom: `gcloud compute ssh` hangs then "Connection timed out").
  `gcloud compute ssh <name> --tunnel-through-iap` rides over HTTPS instead and
  needs only the standard IAP firewall rule (source `35.235.240.0/20`) plus
  `roles/iap.tunnelResourceAccessor`. Note the MCP agent's SSH-based tools
  (`manage_vllm_docker`, `get_vllm_docker_logs`, `run_vllm_benchmark`,
  `get_tpu_system_logs`) do not use IAP; on such networks run their documented
  equivalents manually with `--tunnel-through-iap`.
- **Quota is per region AND per TPU family:** creation fails immediately with
  `Quota 'TPUS_PER_TPU_FAMILY' exceeded. Limit: 0.0` in regions without CT6E
  quota (observed: us-east5 = 0, europe-west4 OK). This dimensioned quota is not
  visible via `gcloud compute regions describe` — attempt creation (fails fast)
  or check the console. Failure sequence for a v6e-1: boot ~1 min → docker
  install ~1 min → image pull ~5 min → model download/compile ~5-10 min.

## Cautions

> [!CAUTION]
> Before calling destructive endpoints (`destroy_queued_resource`, `destroy_tpu_vm_instance`, or `manage_queued_resource`), ensure the user explicitly requested teardown/deletion. If teardown was explicitly requested, proceed immediately without prompting. Only prompt for confirmation if destructive action is an implicit side-effect or unrequested.

- `destroy_queued_resource` and `manage_queued_resource` delete infrastructure —
  `manage_queued_resource` deletes ALL queued resources in the zone other than the
  named primary. Confirm with the user before invoking against a zone holding other resources
  unless teardown/cleanup was explicitly requested.
- Flex-start requests expire (`--valid-until-duration`) and instances self-delete at
  `--max-run-duration`; data on the VM is lost. Persist data on a separate disk or GCS
  (see the reference guide).
- Stuck in `WAITING_FOR_RESOURCES`/`PROVISIONING` or `STOCKOUT`: usually the
  `GPUS_ALL_REGIONS` global quota is 0 — see the Troubleshooting section of
  `references/tpu-guide.md` before retrying other zones.
- v5e uses the legacy queued-resources API and separate quota metrics; v6e/v5p use GCE
  machine types (`ct6e-standard-4t`, `ct5p-hightpu-4t`). Zone/family table is in the
  reference guide.
