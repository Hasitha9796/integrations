# Wazuh Decoder & Rule Creator

A FastAPI web application that intelligently generates custom Wazuh decoder and rule XML for any log format. It combines `wazuh-logtest` verification, machine learning similarity search, RAG (Retrieval-Augmented Generation), and a local LLM to produce accurate, ready-to-use Wazuh XML — without manual regex writing.

## What is included

- `app/main.py` — FastAPI backend, all API endpoints and generation logic
- `app/templates/index.html` — single-page frontend UI
- `app/static/` — JavaScript and CSS
- `app/decoder_ml.py` — ML decoder similarity model (TF-IDF)
- `app/decoder_ml_enhanced.py` — Enhanced ensemble ML model (TF-IDF 30% + SBERT 70%)
- `app/rag_engine.py` — RAG engine — ChromaDB vector store for real decoder retrieval
- `Modelfile` — Custom Ollama model config (`wazuh-decoder` built on `qwen2.5:7b`)
- `requirements.txt` — Python dependencies

---

## Deployment Modes

The app supports two deployment modes. **Mode A (on-server) is recommended.**

### Mode A — Run Directly on the Wazuh Server (Recommended)

Install and run the app on the same machine where Wazuh is installed. `wazuh-logtest` is called locally and files are written directly to `/var/ossec/`.

**When to use:** The app and Wazuh are on the same machine.

### Mode B — Run Remotely via SSH

Run the app on a separate machine (e.g. a dev laptop or VM) and connect to the Wazuh server over SSH.

**When to use:** You are developing on a different machine from where Wazuh runs.

---

## Installation

### 1. Set up the virtual environment

```bash
cd integrations/wazuh_decoder_rule_tool
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 2. Generate SSL certificates

```bash
mkdir -p certs
openssl req -x509 -newkey rsa:4096 \
  -keyout certs/localhost.key \
  -out certs/localhost.crt \
  -days 365 -nodes -subj "/CN=localhost"
```

> `certs/` is in `.gitignore` — your private keys will never be committed.

### 3. Configure environment variables

Set variables for your chosen mode. You can export them in your shell or place them in a `.env` file in the project root.

**Mode A — On-server (recommended):**

```bash
export WAZUH_REMOTE_ENABLED=false

# If the app does not run as root, enable sudo for logtest and file writes:
export WAZUH_USE_SUDO=true
# export WAZUH_SUDO_PASSWORD=yourpassword   # only if passwordless sudo is not configured

export OLLAMA_BASE_URL=http://localhost:11434
export OLLAMA_MODEL=llama3.1:latest
```

**Mode B — Remote SSH:**

```bash
export WAZUH_REMOTE_ENABLED=true
export WAZUH_SSH_HOST=192.168.56.10
export WAZUH_SSH_PORT=22
export WAZUH_SSH_USER=your_ssh_user
export WAZUH_SSH_PASSWORD=your_ssh_password
# optional — key-based auth:
# export WAZUH_SSH_KEY=/path/to/private_key

export OLLAMA_BASE_URL=http://localhost:11434
export OLLAMA_MODEL=llama3.1:latest
```

### 4. Start the application

```bash
uvicorn app.main:app \
  --host 0.0.0.0 --port 8443 \
  --ssl-certfile certs/localhost.crt \
  --ssl-keyfile certs/localhost.key
```

Open `https://localhost:8443` in your browser (or `https://<server-ip>:8443` when running on the Wazuh server).

> On first startup, the RAG vector store is built automatically in the background (~1–2 min). The app is fully usable while it builds.

---

## Wazuh Integration

### Mode A — Local (On-server)

When `WAZUH_REMOTE_ENABLED=false`, the app calls `wazuh-logtest` and writes files directly on the local machine.

If the app runs as a non-root user, enable sudo:

```bash
export WAZUH_USE_SUDO=true
# export WAZUH_SUDO_PASSWORD=yourpassword   # omit if passwordless sudo is configured
```

Override the logtest binary path if needed:

```bash
export WAZUH_LOGTEST_PATH=/var/ossec/bin/wazuh-logtest   # default
```

### Mode B — Remote Wazuh Server (SSH)

When `WAZUH_REMOTE_ENABLED=true`, the app will:
- Run `wazuh-logtest` over SSH with sudo to validate logs against your live Wazuh instance
- Write generated decoder/rule XML directly to `/var/ossec/etc/decoders/` and `/var/ossec/etc/rules/` on the remote server

```bash
export WAZUH_REMOTE_ENABLED=true
export WAZUH_SSH_HOST=192.168.56.10
export WAZUH_SSH_PORT=22
export WAZUH_SSH_USER=vagrant
export WAZUH_SSH_PASSWORD=vagrant
# optional:
export WAZUH_SSH_KEY=/path/to/private_key
```

---

## AI Provider Configuration

The app supports three AI providers. Configure one before starting:

**Ollama (local, recommended — no rate limits):**

```bash
export OLLAMA_BASE_URL=http://localhost:11434
export OLLAMA_MODEL=llama3.1:latest
```

A custom `wazuh-decoder` model tuned for Wazuh OS_Regex is also available via `Modelfile`:

```bash
ollama create wazuh-decoder -f Modelfile
export OLLAMA_MODEL=wazuh-decoder
```

**DashScope (Qwen):**

```bash
export DASHSCOPE_API_KEY=your_key_here
```

**OpenRouter:**

```bash
export OPENROUTER_API_KEY=your_key_here
```

Priority: Ollama → DashScope → OpenRouter. Ollama is always preferred when `OLLAMA_BASE_URL` is set.

---

## ML Decoder Learning

The app builds a similarity model from official Wazuh decoders cached from:

```
https://github.com/wazuh/wazuh.git
```

Configuration:

```bash
export WAZUH_REPO_URL=https://github.com/wazuh/wazuh.git
export WAZUH_REPO_CACHE_DIR=/path/to/cache/wazuh_repo   # default: data/wazuh_repo
export WAZUH_REPO_DECODER_SUBPATH=ruleset/decoders
```

API:

- `GET /api/ml/status` — show model status and cache location
- `POST /api/ml/refresh` — pull latest Wazuh decoders and rebuild the ML model and RAG store

### Training a fine-tuned SBERT model

```bash
# 1. Ensure the Wazuh repo cache exists (run the app once or POST /api/ml/refresh)

# 2. Build training dataset
python scripts/build_dataset.py
# Outputs: data/datasets/train.jsonl, val.jsonl

# 3. Train SBERT
python scripts/train_similarity.py
# Outputs: data/models/decoder-sbert/final/
```

The app automatically uses the fine-tuned model if `data/models/decoder-sbert/final/` exists, otherwise falls back to TF-IDF.

---

## AI-Powered Generation

The app uses a hybrid approach:

1. **wazuh-logtest analysis** — the log is tested against Wazuh's built-in decoders first
2. **Programmatic base generation** — generates syntactically correct Wazuh XML using heuristics and ML
3. **AI review** — an LLM reviews the generated XML and improves osregex patterns

This ensures the XML structure is always valid while leveraging LLM capabilities for pattern refinement.

### AI Generation Flow

1. User provides log samples and requests specific fields to extract
2. App runs `wazuh-logtest` to check what built-in decoders already match
3. Fields already decoded by built-in decoders are skipped
4. The app programmatically generates decoder XML using heuristic + ML analysis
5. The LLM receives the analysis results and the programmatic XML
6. The LLM reviews and improves regex patterns while keeping the structure intact

---

## Optional File Output

The `/api/test` endpoint supports `install_mode="write_files"` and writes generated files to:

- `/var/ossec/etc/decoders/local_<appname>_decoder_<stamp>.xml`
- `/var/ossec/etc/rules/local_<appname>_rule_<stamp>.xml`

Override the output directories:

```bash
export WAZUH_DECODERS_DIR=/custom/decoders
export WAZUH_RULES_DIR=/custom/rules
```

---

## Environment Variable Reference

| Variable | Default | Description |
|---|---|---|
| `WAZUH_REMOTE_ENABLED` | `false` | `true` = SSH mode (Mode B), `false` = local/on-server mode (Mode A) |
| `WAZUH_USE_SUDO` | `false` | Use `sudo` for local logtest and file writes (Mode A, non-root) |
| `WAZUH_SUDO_PASSWORD` | *(none)* | Sudo password — omit if passwordless sudo is configured |
| `WAZUH_LOGTEST_PATH` | `/var/ossec/bin/wazuh-logtest` | Path to wazuh-logtest binary |
| `WAZUH_SSH_HOST` | *(none)* | SSH host for remote Wazuh server (Mode B only) |
| `WAZUH_SSH_PORT` | `22` | SSH port (Mode B only) |
| `WAZUH_SSH_USER` | *(none)* | SSH username (Mode B only) |
| `WAZUH_SSH_PASSWORD` | *(none)* | SSH password (Mode B only) |
| `WAZUH_SSH_KEY` | *(none)* | Path to SSH private key (Mode B only) |
| `WAZUH_DECODERS_DIR` | `/var/ossec/etc/decoders` | Output directory for decoder XML |
| `WAZUH_RULES_DIR` | `/var/ossec/etc/rules` | Output directory for rule XML |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama API base URL |
| `OLLAMA_MODEL` | `llama3.1:latest` | Ollama model name |
| `DASHSCOPE_API_KEY` | *(none)* | DashScope API key |
| `OPENROUTER_API_KEY` | *(none)* | OpenRouter API key |
| `WAZUH_REPO_URL` | `https://github.com/wazuh/wazuh.git` | Wazuh repo for ML training data |
| `WAZUH_REPO_CACHE_DIR` | `data/wazuh_repo` | Local cache for Wazuh repo |
