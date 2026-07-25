# 🚀 tpu-skill — TPU Management Skill & MCP Agent (Antigravity)

This repository packages the **`tpu-management` skill** and the **`tpu-devops` MCP server**: a specialized AI DevOps/SRE agent for managing Google Cloud TPUs, provisioning resources, and running self-hosted Gemma 4 models. Optimized for **Google Antigravity**.

> [!TIP]
> See [INSTALL.md](file:///home/xbill/tpu-skill-agy/INSTALL.md) for detailed setup instructions.

It finds and provisions TPU capacity (flex-start VMs, queued resources), starts and debugs vLLM, verifies model health, runs benchmarks, analyzes logs with the self-hosted Gemma 4 model, and tears everything down safely.

**GitHub:** https://github.com/xbill9/tpu-skill-agy

---

## ⚡ Quick Start — set up a project in one command

`project-setup.sh` installs the `tpu-management` skill **and** registers the `tpu-devops` MCP server for any project (idempotent — re-run to refresh). It targets the **Antigravity** installation paths:

```bash
./project-setup.sh /path/to/project --project <gcp-project-id>   # one project (.mcp.json + .gemini/skills)
./project-setup.sh --global                                      # all projects (~/.gemini/antigravity-cli/skills)
make init TARGET=/path/to/project ARGS='--project <id>'          # same, refreshing skill snapshots first
```

It uses the system `python3` (warning with a `pip install -r requirements.txt` hint if server dependencies are missing — it never creates a venv), merges the server entry into the project's `.mcp.json` or Antigravity configuration without touching other servers, and prints the remaining manual steps (restart the agent, gcloud auth, HF token). See `./project-setup.sh --help` for all options. The installer is also bundled inside the skill itself (`mcp/project-setup.sh`), so an unzipped `dist/tpu-management-skill.zip` is self-installing.

This repo's **own** configuration files are gitignored (they embed your GCP project id) and are generated automatically: `./init.sh` registers the server on first run (leaving an existing entry untouched), or regenerate it any time with `./project-setup.sh . --project <gcp-project-id> [--model ... --accelerator ... --tp ...]`.

---

## 📦 Installing the `tpu-management` Skill

The agent auto-discovers any skill folder containing a `SKILL.md` in two places:

- **Project-level:** `<project>/.gemini/skills/tpu-management/` — available only in that project.
- **User-level:** `~/.gemini/antigravity-cli/skills/tpu-management/` — available in every project on the machine.

Pick the install path that fits:

| Goal | Command |
| :--- | :--- |
| This machine, all projects (Antigravity) | `make skill-install` |
| Register as local plugin (Antigravity) | `make plugin-install` |
| One specific project (skill **and** `tpu-devops` MCP server) | `make init TARGET=/path/to/project ARGS='--project <gcp-project-id>'` |
| All projects + user-scope MCP registration | `make init ARGS='--global'` |
| Another machine | `make skill-package`, copy `dist/tpu-management-skill.zip`, unzip into `~/.gemini/antigravity-cli/skills/` |

### Install from GitHub

**Option A — clone and install** (all projects on this machine):

```bash
git clone https://github.com/xbill9/tpu-skill-agy
cd tpu-skill-agy
make skill-install                                   # skill only
./project-setup.sh --global                          # skill + user-scope tpu-devops MCP server
```

**Option B — zip install, no clone** (straight from the packaged zip):

```bash
curl -L -o /tmp/tpu-management-skill.zip \
  https://github.com/xbill9/tpu-skill-agy/raw/main/dist/tpu-management-skill.zip
mkdir -p ~/.gemini/antigravity-cli/skills && unzip -o /tmp/tpu-management-skill.zip -d ~/.gemini/antigravity-cli/skills/
~/.gemini/antigravity-cli/skills/tpu-management/mcp/project-setup.sh --global   # optional: register the MCP server
```

All of these first run `make skill` (`refresh_skill.py`), which regenerates the bundled snapshots from the repo-root sources: `server.py`, `project-setup.sh`, and `requirements.txt` are copied into the skill's `mcp/` folder, and `references/tpu-guide.md` is rebuilt from `tpu.md` with the embedded screenshots stripped. `SKILL.md` and `mcp/startup_script_template.sh` are hand-maintained and never overwritten.

After installing (or updating), **restart the agent** (Antigravity) or start a new session so it picks up the skill. Verify by checking your skill list in Antigravity — `tpu-management` should be listed.

Because installs are refresh-and-copy (not symlinks), an installed copy goes stale when `server.py`, `tpu.md`, or `SKILL.md` changes — rerun `make skill-install` (or `make init ...`) after editing those files.

---

## 📂 Repository Layout

| Path | Purpose |
| :--- | :--- |
| `server.py` | The `tpu-devops` FastMCP server — the authoritative source (full tool catalog in `SKILL.md` / the `get_help` tool) |
| `project-setup.sh` | One-command installer: skill + MCP registration for a target project |
| `refresh_skill.py` | Regenerates the bundled skill snapshots from the repo-root sources |
| `requirements.txt` | Python dependencies for the MCP server |
| `Makefile` | `skill` / `skill-install` / `skill-package` / `init` targets (see below) |
| `.gemini/skills/tpu-management/` | Project-level skill: `SKILL.md`, `mcp/` (server snapshot, installer, startup script template), `references/tpu-guide.md` |
| `skills/tpu-management/` | Plugin-layout copy of the skill (synced by `make skill`) |
| `dist/tpu-management-skill.zip` | Packaged skill for zip installs (built by `make skill-package`) |
| `init.sh`, `set_env.sh`, `set_adc.sh` | GCP environment / credentials setup helpers |
| `tpu.md` | TPU getting started guide source (gitignored; the stripped, vendor-neutral text copy ships in `references/tpu-guide.md`) |

---

## 🛠 Features & Capabilities

The `tpu-devops` MCP server covers the full TPU serving lifecycle (catalog with usage guidance in [SKILL.md](.gemini/skills/tpu-management/SKILL.md), live listing via the `get_help` tool):

- **Capacity discovery & provisioning:** sweep zones for available capacity (`find_tpu_vm` for flex-start VMs, `find_tpu` for queued resources), check quotas (`get_zones_with_available_quota`), estimate cost, create flex-start TPU VMs (v6e/v5p) or legacy queued resources (v5e) with an auto-serving startup script, then `wait_for_vllm_ready` until the model is up.
- **Serving stack control:** manage the vLLM Docker container (`manage_vllm_docker` — works on both flex-start VMs and queued-resource nodes), fetch endpoints and the gcloud deployment one-liner, store the HF token in Secret Manager.
- **Health, logs & diagnostics:** system status dashboard covering both serving paths, model health verification, vLLM/docker/system/serial logs, Cloud Logging retrieval, and Gemma-4-powered log triage (`analyze_cloud_logging`).
- **Inference & benchmarking:** query the deployed Gemma 4 endpoint (optional TTFT/throughput stats), run `vllm bench serve` for benchmark metrics.
- **Universal SRE help:** a standardized `get_help` tool describing the active configuration and all exposed tools.

---

## 🏗 Makefile Usage

```bash
make skill         # Refresh skill snapshots from server.py / tpu.md (also syncs the plugin copy in skills/)
make skill-install # Refresh + copy the skill to ~/.gemini/antigravity-cli/skills (Antigravity skills)
make skill-package # Refresh + build dist/tpu-management-skill.zip
make init TARGET=/path/to/project [ARGS='--project my-gcp-id']
                   # Refresh + install skill AND register the tpu-devops MCP server
```

Edit the repo-root sources (`server.py`, `tpu.md`, `project-setup.sh`), then run the appropriate target — never edit the snapshot copies directly.

---

## ⚙️ Configuration

The server reads its configuration from environment variables: `GOOGLE_CLOUD_PROJECT` (falls back to the active gcloud config), `GOOGLE_CLOUD_ZONE` (default `europe-west4-a`), `GOOGLE_CLOUD_REGION`, `MODEL_NAME`, `ACCELERATOR_TYPE`, `TENSOR_PARALLEL_SIZE`. Prerequisites: `pip install -r requirements.txt`, an authenticated `gcloud` CLI with alpha components, the TPU API enabled, and a Hugging Face token stored as Secret Manager secret `hf-token` (the `save_hf_token` tool does this for you).

---

## 🔒 Security & Credentials

When deploying to Google Cloud or Hugging Face, secure credentials using:
- **Hugging Face Access Token:** Saved locally or to Google Secret Manager.
- **Application Default Credentials (ADC):** Set up using GCP credentials helper scripts (`set_adc.sh`).

## 📖 Related Documentation

- [SKILL.md](.gemini/skills/tpu-management/SKILL.md) — the skill itself: lifecycle, tool catalog, required vLLM flags, field notes, cautions
- [GEMINI.md](GEMINI.md) — Gemini CLI integration via a LiteLLM proxy pointed at the self-hosted Gemma 4 TPU endpoint
- `references/tpu-guide.md` — TPU getting started guide: flex-start zones, quotas, troubleshooting

## Credits

Google Cloud credits are provided for this project.

#AgenticArchitect #GoogleAntigravity
