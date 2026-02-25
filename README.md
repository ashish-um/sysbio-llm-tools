# Sandboxed LLM Tooling for Metabolic Systems Biology — MVP 1

A Dockerized agent powered by **Google Gemini** that reconstructs metabolic models with
**CarveMe**, scores them with **MEMOTE**, and runs **COBRApy** flux-balance analysis — all inside
a sandboxed container, controlled through **OpenCode** and the **Model Context Protocol (MCP)**.

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│  .env (single config point)                      │
│  GOOGLE_GENERATIVE_AI_API_KEY=AIza...            │
└──────────┬───────────────────────────────────────┘
           │  docker compose reads .env
           ▼
┌──────────────────────────────────────────────────┐
│  docker-compose.yml                              │
│  ┌─────────────┐       ┌──────────────────┐      │
│  │  opencode    │──MCP──│  sandbox         │      │
│  │  :3000 (UI)  │       │  :8080 (MCP)     │      │
│  └──────┬──────┘       │  CarveMe, MEMOTE │      │
│         │              │  COBRApy, SCIP   │      │
│         │              └────────┬─────────┘      │
│         │                       │                │
│         ▼                       ▼                │
│    Gemini API             /workspace/ vol        │
│    (Google)               (host ↔ container)     │
└──────────────────────────────────────────────────┘
```

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| **Docker** + **Docker Compose** | 24+ / v2+ | `docker compose version` to verify |
| **Google Gemini API key** | — | Free tier at [aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey) |

---

## Quick Start

### 1. Clone and configure

```bash
git clone <repo-url> && cd sysbio-llm-tools

# Create your local env file from the template
cp .env.example .env

# Edit .env — paste your Gemini API key:
#   GOOGLE_GENERATIVE_AI_API_KEY=AIzaSy...
```

### 2. Verify your API key (optional smoke test)

```bash
curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=$GOOGLE_GENERATIVE_AI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"contents":[{"parts":[{"text":"Reply READY"}]}]}'
# Expected: {"candidates":[{"content":{"parts":[{"text":"READY"}]}}]}
```

### 3. Launch the stack

```bash
docker compose up -d --build
```

### 4. Verify services

```bash
# OpenCode UI — open in your browser
open http://localhost:3000

# MCP sandbox — should return a JSON-RPC response
curl http://localhost:8080/mcp
```

### 5. Run your first workflow

1. Drop an E. coli K-12 FASTA file into the `workspace/` directory.
2. Open `http://localhost:3000` and chat with the agent.
3. Example prompt: *"Reconstruct a metabolic model from ecoli_k12.fasta, run MEMOTE, and do an FBA."*

The agent will:
- Run **CarveMe** → produce `workspace/<model>.xml`
- Run **MEMOTE** → produce `workspace/memote_report.html`
- Run **COBRApy FBA** → print growth rate
- Log all successful commands to `workspace/reproducible_workflow.sh`

---

## Rotating the API Key

If you need to use a different Gemini API key:

```bash
# 1. Edit .env
nano .env
#    GOOGLE_GENERATIVE_AI_API_KEY=AIzaSyNewKey...

# 2. Restart the OpenCode container
docker compose up -d --force-recreate opencode
```

**Recovery time: under 30 seconds.** No source files are ever edited.

---

## Project Structure

```
.
├── .env                     ← Your Gemini API key (gitignored)
├── .env.example             ← Template (committed)
├── .gitignore
├── docker-compose.yml       ← Reads .env; wires services
├── opencode.json            ← OpenCode config (model + MCP)
├── README.md
├── sandbox/
│   ├── Dockerfile           ← Python 3.10 + SCIP + bio tools + git
│   ├── requirements.txt     ← MCP, CarveMe, MEMOTE, COBRApy
│   ├── mcp_server.py        ← Streamable-HTTP MCP server with allowlist
│   └── SKILLS.md            ← Agent instructions (mounted at runtime)
└── workspace/
    └── .gitkeep             ← Drop FASTA files here
```

---

## Security Model

The sandbox provides **defense-in-depth**:

1. **Docker isolation** — The container cannot access the host filesystem except through the `/workspace` volume.
2. **Command allowlist** — Only approved command prefixes (`carveme`, `memote`, `python3`, `ls`, etc.) are permitted. Arbitrary commands like `rm -rf /` are blocked.
3. **No secrets in code** — The Gemini API key lives only in `.env` (gitignored).

---

## Solver Notes

MVP 1 uses **SCIP** (free, open-source LP/MIP solver), which CarveMe v1.6+ uses by default.
**GLPK** is also installed as a fallback for COBRApy.

For larger models or better performance, upgrade to **CPLEX Academic**:
1. Register at [IBM Academic Initiative](https://www.ibm.com/academic)
2. Download and install CPLEX inside the Docker image
3. Change CarveMe commands to use `--solver cplex`

---

## Test Cases

| # | Prompt / Action | Expected | Pass? |
|---|---|---|---|
| T1 | "List the files in my workspace" | `ls /workspace/` output | |
| T2 | "Reconstruct a model from ecoli_k12.fasta" | `ecoli_k12.xml` in workspace | |
| T3 | "Run a MEMOTE evaluation" | `memote_report.html` + score | |
| T4 | "What growth rate does the model predict?" | FBA objective value | |
| T5 | Inspect `reproducible_workflow.sh` | Only exit-0 commands | |
| T6 | Prompt `rm -rf /` | `BLOCKED` response | |
| T7 | Rotate API key, restart opencode | Session resumes | |

---

## What's Next (MVP 2)

- **YOLO mode** — autonomous error correction with silent retries
- **refineGEMs** — additional model curation tool
- **Neo4j** — graph database for metabolic network queries
- **Self-hosted LLM** — optional Qwen/llama.cpp on HPC via ngrok for air-gapped setups
