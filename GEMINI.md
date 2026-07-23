# 🤖 Gemini Workspace Context: TPU Management Skill & tpu-devops MCP Agent

This workspace context file helps **Antigravity** (and other developer tools) quickly understand the layout, goals, and integration methods of the **tpu-skill** project.

---

## 🎯 Project Overview & Role

This repository packages a **skill** (`tpu-management`) and a **Model Context Protocol (MCP) server** (`tpu-devops`) that together act as an AI DevOps/SRE agent for Google Cloud TPUs. Two main purposes:

1. **Infrastructure Operations:** Finding, provisioning, and destroying TPU capacity (flex-start VMs, queued resources) and running Gemma 4 vLLM serving on TPU VMs (v6e, v5p, v5e).
2. **Log & SRE Diagnostics:** Utilizing the self-hosted Gemma 4 model to analyze system/cloud logs and generate remediation suggestions.

---

## 📂 Quick Navigation

Key entrypoints in the codebase:

- **MCP server source:** [server.py](server.py) — the authoritative `tpu-devops` FastMCP agent (full tool catalog in `SKILL.md` / the `get_help` tool)
- **Skill definition:** [.claude/skills/tpu-management/SKILL.md](.claude/skills/tpu-management/SKILL.md) — lifecycle, tool catalog, required vLLM flags, field notes
- **Installer:** [project-setup.sh](project-setup.sh) — one-command skill install + MCP registration
- **Root Makefile:** [Makefile](Makefile) — `skill` / `skill-install` / `skill-package` / `init` targets
- **Snapshot refresher:** [refresh_skill.py](refresh_skill.py) — regenerates the bundled skill copies from the root sources
- **Plugin marketplace manifests:** [.claude-plugin/](.claude-plugin/) — makes the repo installable via the Claude Code plugin system
- **Reference guide:** `.claude/skills/tpu-management/references/tpu-guide.md` — TPU getting started guide: zones, quotas, troubleshooting
- **Serving Benchmark Reports:** [benchmarks/](benchmarks/) — JSON schema ([serving-report.schema.json](benchmarks/serving-report.schema.json)) and reporting guidelines ([benchmarks/README.md](benchmarks/README.md)) for tracking model deployment latency, throughput, and costs.

---

## 🛠 Development Workflow & Makefile Tasks

The repo-root files (`server.py`, `project-setup.sh`, `tpu.md`) are authoritative; the skill directories and zip are generated snapshots. After editing a source:

```bash
make skill                  # Regenerate skill snapshots + plugin copy
make skill-install-agy      # ...and install to Antigravity skills directory
make plugin-install         # ...and register as an Antigravity plugin (recommended)
make skill-package          # ...and rebuild dist/tpu-management-skill.zip
```

---

## 🔗 Integration with Gemini CLI via LiteLLM Proxy

You can redirect standard Gemini CLI commands to run against the self-hosted Gemma 4 model served from a TPU VM deployed by this agent. This lets developers use their own self-hosted inference engine under the hood.

### 1. Install LiteLLM Proxy

```bash
pip install 'litellm[proxy]'
```

### 2. Configure LiteLLM

Create a `litellm_config.yaml` targeting the TPU vLLM endpoint (get the IP with the agent's `get_vllm_endpoint` / `get_tpu_vm_endpoint` tools):

```yaml
model_list:
  - model_name: "gemma4-tpu"
    litellm_params:
      model: "openai/google/gemma-4-31B-it"
      api_base: "http://YOUR_TPU_IP_ADDRESS:8000/v1"
      api_key: "none"
    router_settings:
      model_group_alias:
        "gemini-2.0-flash": "gemma4-tpu"
        "gemini-2.0-flash-lite": "gemma4-tpu"
        "gemini-1.5-flash": "gemma4-tpu"
        "gemini-1.5-pro": "gemma4-tpu"
```

Adjust `model` to match the served model (`MODEL_NAME` env var of the agent), e.g. `openai/google/gemma-4-12B-it` or `openai/google/gemma-4-E4B-it`.

### 3. Run Proxy & Point Gemini CLI at It

Run the proxy locally:

```bash
litellm --config litellm_config.yaml --port 4000
```

The `model_group_alias` mapping above is what does the real work: any request the
Gemini CLI makes for a `gemini-*` model is routed to the self-hosted `gemma4-tpu`
endpoint. Then point the CLI at the proxy:

```bash
export GOOGLE_GEMINI_BASE_URL="http://localhost:4000"
export GEMINI_API_KEY="local-proxy-token"
export GEMINI_MODEL="google/gemma-4-31B-it"   # match the served model
```

> **Note:** environment-variable names vary between Gemini CLI releases — if the CLI
> ignores `GOOGLE_GEMINI_BASE_URL`, check `gemini --help` / its settings file for the
> current base-URL override; the LiteLLM config itself needs no changes.

---

## 🔧 Technical Standards for vLLM & Gemma 4 Tool Calling

When managing TPU deployments or customizing vLLM serving, ensure the following vLLM serving parameters are applied for stable Gemma 4 tool integration:

- **Optimization flags:** `--tensor-parallel-size 8` (TPU v6e-8), `--disable_chunked_mm_input`, `--max-model-len 16384`.
- **Tool Parsing:** `--enable-auto-tool-choice`, `--tool-call-parser gemma4`, and `--reasoning-parser gemma4` to enable native function calling compatibility.
- **Multimodal configuration:** `--limit-mm-per-prompt '{"image":4,"audio":1}'` and `--max_num_batched_tokens 4096`.
- **Universal SRE Help:** The agent exposes a standardized `get_help` tool providing details on active configuration environment variables and all exposed tools.

## 📊 Analysis Standards

- **Dependency Portability:** Avoid assuming third-party analysis libraries like `pandas` are installed in the workspace environment. Prefer standard libraries (e.g., `csv`, `json`) for data parsing and aggregation scripts.

## 🎨 Antigravity Best Practices (Aesthetics & Artifacts)

When using this skill with **Antigravity**, follow these UI/UX standards:

- **Premium Dashboards:** Use `write_to_file` to create rich markdown artifacts for TPU status dashboards. Use Mermaid diagrams for architecture and alerts for critical capacity warnings.
- **Dynamic Monitoring:** If a user asks to monitor a deployment, use `schedule` to set up recurring checks and report back with artifact updates.
- **Rich Visualization:** When benchmarking or analyzing logs, generate visual summaries. Use the `generate_image` tool to create mockups of monitoring dashboards if requested.
- **Actionable Plans:** Use the `/plan` command to outline complex multi-zone capacity sweeps before execution.
