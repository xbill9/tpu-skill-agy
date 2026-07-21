# 🛠 Installation Guide — tpu-skill (Antigravity)

This guide covers setting up the **`tpu-management`** skill and the **`tpu-devops`** MCP server for **Google Antigravity**.

---

## 📋 Prerequisites

Before installing, ensure your environment is configured:

1.  **Google Cloud CLI:**
    *   Install `gcloud` and the `alpha` components:
        ```bash
        gcloud components install alpha
        ```
    *   Authenticate:
        ```bash
        gcloud auth login
        ```
        ```bash
        gcloud auth application-default login
        ```
2.  **Python 3:**
    *   The MCP server requires Python 3.8+. Install dependencies:
        ```bash
        pip install -r requirements.txt
        ```
3.  **Hugging Face Token:**
    *   The agent needs a token to pull models. Store it in Google Secret Manager:
        ```bash
        # You can use the agent's 'save_hf_token' tool later, or do it now:
        printf "your-hf-token" | gcloud secrets create hf-token --data-file=-
        ```

---

## 🚀 Method 1: One-Command Installer (Recommended)

The `project-setup.sh` script is idempotent and handles both skill installation and MCP registration.

### For a Specific Project
Installs the skill to `./.gemini/skills` and registers the server in `.mcp.json`.
```bash
./project-setup.sh . --project <your-gcp-project-id>
```

### Global Installation
Installs to your user home directory for use in all projects.
```bash
./project-setup.sh --global --project <your-gcp-project-id>
```

---

## 🏗 Method 2: Makefile Targets

If you are working from the repository clone, use the provided `Makefile`:

| Target | Description |
| :--- | :--- |
| `make skill-install` | Installs to **Antigravity** user directory (`~/.gemini/antigravity-cli/skills`) |
| `make plugin-install` | Registers the project as an **Antigravity** plugin (`agy plugin install .`) |
| `make init ARGS='--global'` | Installs globally and registers the MCP server |

---

## 🧩 Method 3: Plugin Installation

If you have cloned the repository, you can install it as a local plugin:
```bash
make plugin-install
# or directly:
agy plugin install .
```

---

## ⚙️ Manual Configuration

If you prefer manual setup, you can register the MCP server directly. Add the server to your project's `.mcp.json`:

```json
{
  "mcpServers": {
    "tpu-devops": {
      "command": "python3",
      "args": ["/absolute/path/to/server.py"],
      "env": {
        "GOOGLE_CLOUD_PROJECT": "your-project-id"
      }
    }
  }
}
```

---

## ✅ Verification

1.  **Restart your agent** (Antigravity).
2.  **Verify Skill:** Check the skill list in Antigravity.
3.  **Verify MCP:** List resources in Antigravity to see the `tpu-devops` tools.
4.  **Check Status:** Run the `get_system_status` tool to verify the GCP connection.

---

## 💡 Troubleshooting

*   **Stockouts:** If `find_tpu_vm` fails across all zones, check your `GPUS_ALL_REGIONS` quota in the GCP Console.
*   **Permission Denied:** Ensure the default compute service account (`<project-number>-compute@developer.gserviceaccount.com`) has the `roles/secretmanager.secretAccessor` role.
*   **Missing Dependencies:** If the server fails to start, ensure `mcp`, `httpx`, and `openai` are installed in your system Python.
