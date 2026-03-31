# JobPilot — Tech Stack Analysis & Recommendations

**Author:** Generated for Pranjal Yadav  
**Date:** 29 March 2026  
**Hardware:** HP Gaming Laptop — Ryzen 5 5600, 16GB RAM, 500GB SSD, NVIDIA GTX 1650 (4GB VRAM)  
**OS:** Ubuntu 24.04 LTS  
**Role:** Dedicated 24/7 server  
**Budget:** ≤ $10/month for cloud APIs  

---

## Table of Contents

1. [Hardware Assessment & Resource Budget](#1-hardware-assessment--resource-budget)
2. [Operating System](#2-operating-system)
3. [Core Language & Frameworks](#3-core-language--frameworks)
4. [LLM Strategy — Local + Cloud Cascade](#4-llm-strategy--local--cloud-cascade)
5. [Database Layer](#5-database-layer)
6. [Container Strategy — Docker Compose vs Bare Metal](#6-container-strategy--docker-compose-vs-bare-metal)
7. [Networking — Dynamic IP, Tunnels & Webhooks](#7-networking--dynamic-ip-tunnels--webhooks)
8. [Web Scraping & Browser Automation](#8-web-scraping--browser-automation)
9. [Monitoring & Observability](#9-monitoring--observability)
10. [Backup & Storage](#10-backup--storage)
11. [Per-Component Resource Allocation](#11-per-component-resource-allocation)
12. [Monthly Cost Breakdown](#12-monthly-cost-breakdown)
13. [Recommended Final Stack — Summary Table](#13-recommended-final-stack--summary-table)

---

## 1. Hardware Assessment & Resource Budget

### Your hardware at a glance

| Component | Spec | JobPilot Relevance |
|-----------|------|--------------------|
| **CPU** | AMD Ryzen 5 5600 (6C/12T, 3.5–4.4GHz) | Excellent for Python async workloads, concurrent scraping, LaTeX compilation |
| **RAM** | 16GB DDR4 | Tight but workable — requires careful allocation |
| **GPU** | NVIDIA GTX 1650 (4GB VRAM, Turing arch) | Ollama inference for 7B quantized models; CUDA 7.5 compatible |
| **Storage** | 500GB SSD | Adequate for 6–12 months of operation; needs monitoring |
| **Network** | Home broadband, dynamic IP | Requires tunnel solution for inbound webhooks |

### Resource budget (16GB RAM)

> [!WARNING]
> 16GB RAM is the tightest constraint. The allocation below leaves ~1.5GB headroom for the OS and unexpected spikes. Every component must be memory-capped.

| Consumer | RAM Allocation | Notes |
|----------|---------------|-------|
| Ubuntu 24.04 OS + desktop (if any) | ~1.0 GB | Headless server recommended — saves ~500MB |
| PostgreSQL | 1.5 GB | `shared_buffers=512MB`, `work_mem=16MB` |
| Ollama (7B Q4 model loaded) | 4.5–5.0 GB | ~4GB VRAM offload + ~1GB system RAM for KV cache |
| Python application (9 agents) | 1.5–2.0 GB | Async event loop, Playwright browser contexts |
| Playwright (headless Chromium) | 0.5–1.0 GB | Per browser context; limit to 2 concurrent |
| Prometheus | 0.3–0.5 GB | 90-day retention, ~15s scrape interval |
| Grafana | 0.2–0.3 GB | Lightweight dashboard server |
| Docker Engine overhead | 0.3 GB | If using Docker; 0 if bare-metal |
| LaTeX compilation (xelatex) | 0.1–0.3 GB | Transient, only during resume generation; slightly more than pdflatex due to fontspec |
| **Total estimated** | **~10–12 GB** | |
| **Headroom** | **~4–6 GB** | Buffer for spikes, OS file cache |

### Storage budget (500GB SSD)

| Data | Estimated Size (12 months) | Growth Rate |
|------|-----------------------------|-------------|
| PostgreSQL data | 2–5 GB | ~10MB/day |
| Prometheus TSDB (90-day) | 1–3 GB | Self-pruning |
| Generated resumes (PDF + TeX) | 1–2 GB | ~5MB/day |
| Resume versions history | 200–500 MB | ~2MB/day |
| Playwright browser cache | 500 MB–1 GB | Periodic cleanup |
| Docker images (if used) | 5–10 GB | Stable after initial pull |
| Ollama models | 4–8 GB | Stable once downloaded |
| OS + packages | 15–20 GB | Slow growth |
| **Total estimated** | **~30–50 GB** | |
| **Free space** | **~450 GB** | More than sufficient |

### GPU budget (4GB VRAM)

| Model | Quantization | VRAM Usage | Speed (tokens/sec) | Quality |
|-------|-------------|------------|---------------------|---------|
| Llama 3.2 8B | Q4_K_M | ~4.0 GB | 15–25 tok/s | Good for general tasks |
| Qwen 2.5 7B | Q4_K_M | ~3.8 GB | 18–28 tok/s | Strong for code/structured output |
| Mistral 7B v0.3 | Q4_K_M | ~3.8 GB | 18–28 tok/s | Good general purpose |
| Phi-3.5 Mini (3.8B) | Q5_K_M | ~2.5 GB | 30–45 tok/s | Faster, slightly lower quality |
| Gemma 2 9B | Q4_K_M | ~4.2 GB | 12–18 tok/s | May not fully fit; fragile |

> [!TIP]
> **Recommendation:** Use **Qwen 2.5 7B Q4_K_M** as primary Ollama model — it excels at structured JSON output (critical for JD scoring, email classification) and fits comfortably in 4GB VRAM. Keep **Phi-3.5 Mini** as a fast secondary for simple tasks like keyword matching.

---

## 2. Operating System

### Ubuntu 24.04 LTS — Verdict: ✅ Keep it

Ubuntu 24.04 is an excellent choice. No change needed.

| Factor | Ubuntu 24.04 | Alternatives considered |
|--------|-------------|------------------------|
| **NVIDIA driver support** | First-class via `ubuntu-drivers` | Fedora (good), Arch (manual) |
| **Docker support** | Native `apt` install, no WSL overhead | Same on all Linux |
| **CUDA/cuDNN** | CUDA 12.x supported | Same on most distros |
| **LTS support** | Security updates until April 2029 | Fedora: 13 months only |
| **Systemd services** | Stable, well-documented | Same on all modern Linux |
| **Headless mode** | Easy: `sudo systemctl set-default multi-user.target` | Same |
| **Community/docs** | Largest Linux community | — |

> [!TIP]
> **Run Ubuntu headless (no GUI).** This saves ~500MB RAM and reduces attack surface for a 24/7 server. You can always SSH in from your Mac.
> ```bash
> # Switch to headless (no desktop environment)
> sudo systemctl set-default multi-user.target
> # Reboot to apply
> sudo reboot
> # To get GUI back temporarily:
> sudo systemctl start gdm3
> ```

### Essential OS setup for 24/7 operation

| Configuration | Why | Command |
|--------------|-----|---------|
| Disable sleep/suspend | Laptop will close lid without sleeping | `sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target` |
| Disable lid close action | Keep running with lid closed | Edit `/etc/systemd/logind.conf`: `HandleLidSwitch=ignore` |
| Enable auto-restart on crash | Services restart after power outage | Systemd `Restart=always` in service files |
| Set up unattended security updates | Critical for 24/7 server | `sudo apt install unattended-upgrades` |
| Configure swap (4–8 GB) | Safety net for RAM spikes | `sudo fallocate -l 8G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile` |
| Thermal management | Laptop cooling under sustained load | `sudo apt install thermald tlp` |

---

## 3. Core Language & Frameworks

### 3.1 Language: Python 3.12+

**Verdict:** ✅ No alternative worth considering

| Why Python | Detail |
|-----------|--------|
| LangGraph is Python-native | Core orchestration framework |
| AI/ML ecosystem dominance | LangChain, Ollama client, OpenAI SDK, Anthropic SDK |
| Async support | `asyncio` + `httpx` for concurrent scraping |
| Library availability | `python-telegram-bot`, `google-api-python-client`, `openpyxl`, `feedparser` |

Ubuntu 24.04 ships with Python 3.12. Use `pyenv` or the system Python with `venv`.

### 3.2 Orchestration Framework: LangGraph

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **LangGraph** (PRD choice) | Graph-based state machine, native HITL checkpoints, persistence, built for multi-agent | Steeper learning curve, LangChain dependency | ✅ **Recommended** |
| CrewAI | Simpler multi-agent setup | Less control over state transitions, less mature HITL | ❌ Too opinionated for this use case |
| AutoGen (Microsoft) | Good for conversational agents | Poor fit for pipeline/workflow agents | ❌ Wrong paradigm |
| Custom state machine | Full control, no dependencies | Massive engineering effort for HITL, checkpoints, retries | ❌ Unnecessary reinvention |
| Temporal.io | Production-grade workflow engine | Overkill, Java/Go oriented, heavy resource usage | ❌ Too heavy |

### 3.3 Web Framework: FastAPI

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **FastAPI** (PRD choice) | Async-native, auto OpenAPI docs, lightweight, `/metrics` endpoint | — | ✅ **Recommended** |
| Flask | Simpler, widely known | Sync by default, no native async | ❌ Async matters here |
| Django | Full-featured | Far too heavy for this use case | ❌ Over-engineered |
| Litestar | Modern, fast | Smaller community | ⚠️ Viable but less ecosystem |

### 3.4 Task Scheduling: APScheduler

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **APScheduler** (PRD choice) | In-process, cron-like, persistent job stores, async support | Limited distributed scheduling | ✅ **Recommended** for single-server |
| Celery + Redis/RabbitMQ | Distributed task queue | Requires Redis/RabbitMQ (extra RAM), overkill for single machine | ❌ Over-engineered |
| Linux cron | Dead simple | No in-process state, no async, no retry logic | ❌ Too limited |
| Huey | Lightweight Celery alternative | Still needs Redis | ❌ Unnecessary dependency |

### 3.5 LaTeX Compilation

#### pdflatex vs xelatex — Head-to-head comparison

| Dimension | pdflatex | xelatex | Winner |
|-----------|----------|---------|--------|
| **Unicode support** | Limited — needs `\usepackage[utf8]{inputenc}` and manual encoding setup; breaks on many non-Latin characters | Native UTF-8 — just works with any Unicode character | ✅ **xelatex** |
| **Font handling** | Uses TeX-specific `.tfm`/`.pfb` fonts; installing custom fonts is painful (requires `tfm` generation) | Uses system fonts directly via `fontspec` package — any `.ttf`/`.otf` font installed on the system is immediately available | ✅ **xelatex** |
| **Custom resume fonts** | Changing fonts requires finding TeX-packaged versions or complex font setup | `\setmainfont{Inter}` or `\setmainfont{Calibri}` — trivial to use any modern font | ✅ **xelatex** |
| **Compilation speed** | ~1–3 seconds per document | ~2–5 seconds per document (slightly slower due to font loading) | ✅ pdflatex (marginally) |
| **Package compatibility** | 100% — all LaTeX packages work | ~98% — rare packages using pdfTeX primitives may not work; all common resume packages work fine | ✅ pdflatex (marginally) |
| **microtype support** | Full microtypographic support (character protrusion, font expansion) | Partial — protrusion works, font expansion doesn't | ✅ pdflatex |
| **Output quality** | Excellent | Excellent (identical for most documents) | Tie |
| **Resume templates** | Most templates assume pdflatex | Most modern templates support both; some explicitly prefer xelatex for font control | Tie |
| **RAM usage** | ~100–200 MB during compilation | ~150–300 MB (slightly more due to fontspec and system font loading) | ✅ pdflatex (marginally) |
| **TeX Live install size** | ~800 MB (minimal packages) | ~1–1.5 GB (needs `texlive-xetex` + `texlive-fonts-extra`) | ✅ pdflatex |
| **Docker image size** | Smaller base image | ~200–400 MB larger (xetex engine + font packages) | ✅ pdflatex |
| **Special characters in resumes** | May break on names, companies, or skills with accents/symbols (e.g., résumé, naïve, Zürich) | Handles all Unicode seamlessly | ✅ **xelatex** |
| **Future-proofing** | Legacy engine; receives maintenance but no new features | Actively maintained, modern engine built for Unicode-era | ✅ **xelatex** |

#### Verdict: ✅ XeLaTeX recommended

For a resume generation system, **xelatex wins** on the dimensions that matter most:
- Custom fonts (use any system font like Inter, Garamond, or Calibri without friction)
- Unicode safety (company names with accents, international characters in recruiter names)
- Modern `.tex` templates increasingly target xelatex

The ~1–2 second speed penalty per compilation is negligible for JobPilot (target is <90s total, compilation is a tiny fraction). The slight RAM and disk overhead is well within your budget.

#### Compiler options summary

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **TeX Live + xelatex** | Native Unicode, system fonts via `fontspec`, modern engine | ~1.5 GB install, slightly slower than pdflatex | ✅ **Recommended** |
| TeX Live + pdflatex | Fastest compilation, smallest install, full `microtype` | Poor Unicode, painful custom fonts | ⚠️ Viable if no custom fonts needed |
| TeX Live + lualatex | Most modern engine, Lua scripting, Unicode | Slowest compilation (~3–7s), highest RAM | ⚠️ Over-engineered for resumes |
| TinyTeX + xelatex | Minimal install (~200MB), auto-installs packages on demand | May need manual package additions for resume templates | ⚠️ Viable, saves disk |
| Typst | Modern, fast, tiny | Not LaTeX-compatible; user has `.tex` template | ❌ Incompatible |
| Pandoc + LaTeX | Markdown→PDF | No fine-grained LaTeX control | ❌ Wrong abstraction |

> [!TIP]
> Install xelatex with the packages you need:
> ```bash
> sudo apt install texlive-xetex texlive-latex-recommended texlive-latex-extra texlive-fonts-recommended texlive-fonts-extra
> ```
> ~1.5 GB total. The `texlive-fonts-extra` package is optional but gives you many high-quality fonts out of the box.

> [!NOTE]
> **Migration from pdflatex to xelatex in your `.tex` template:**
> 1. Replace `\usepackage[utf8]{inputenc}` with `\usepackage{fontspec}` (if present)
> 2. Use `\setmainfont{YourFont}` instead of `\usepackage{times}` or similar
> 3. Compile with `xelatex` instead of `pdflatex` — most templates work with zero changes
> 4. The JobPilot compilation command becomes:
> ```python
> subprocess.run(["xelatex", "-interaction=nonstopmode", "-output-directory", output_dir, tex_path], ...)
> ```

---

## 4. LLM Strategy — Local + Cloud Cascade

### 4.1 Architecture: Cascading LLM client

Given your $10/month budget and 4GB VRAM, the optimal strategy is **API-first with local fallback**:

```
Request → DeepSeek V3 (free) → Qwen Cloud (free) → Gemini Flash (free tier)
       → GPT-4o-mini (paid, cheap) → Ollama local (free, GPU-accelerated)
```

### 4.2 Cloud LLM options (Tier 1 — free/near-free)

| Provider | Model | Free Tier | Cost After | Best For | Rate Limits |
|----------|-------|-----------|-----------|----------|-------------|
| **DeepSeek** | DeepSeek V3 / R1 | 50M tokens/day (insane) | $0.14/M input | JD scoring, classification, outreach | 60 RPM |
| **Qwen (DashScope)** | Qwen 2.5 Plus | 1M tokens/month free | $0.80/M input | Structured JSON output | 60 RPM |
| **Google** | Gemini 2.0 Flash | 15 RPM free | $0.075/M input | Fast fallback | 15 RPM (free), 1000 RPM (paid) |
| **Google** | Gemini 2.5 Flash | 500 req/day free | $0.15/M input | Thinking model, complex tasks | Generous free tier |
| **OpenAI** | GPT-4o-mini | No free tier | $0.15/M input | Reliable fallback | 500 RPM |
| **Anthropic** | Claude Sonnet 4.5 | No free tier | $3/M input | Resume rewriting, mediation | 50 RPM |
| **Groq** | Llama 3.x / Mixtral | 30 RPM free | — | Ultra-fast inference | 30 RPM |
| **Together AI** | Various open models | $5 credit on signup | $0.20/M input | Alternative fallback | Generous |

> [!IMPORTANT]
> **Cost estimate with your budget:**
> - DeepSeek free tier covers ~95% of all LLM calls (scoring, classification, outreach, extraction)
> - Claude Sonnet for resume rewriting + mediation: ~$2–3/month (2 cycles/day × 30 days)
> - GPT-4o-mini as paid fallback: ~$0.50–1/month (rare usage)
> - **Total: ~$3–5/month** — well within $10 budget

### 4.3 Local LLM options (Ollama — terminal fallback)

| Runtime | Pros | Cons | Verdict |
|---------|------|------|---------|
| **Ollama** | Dead simple, REST API, auto GPU detection, model management | Limited to single model loaded at a time | ✅ **Recommended** |
| llama.cpp (raw) | Maximum control, slightly faster | Manual model management, no REST API | ❌ Unnecessary complexity |
| vLLM | Production-grade, batching | Heavy, needs more VRAM | ❌ Too heavy for 4GB VRAM |
| LocalAI | OpenAI-compatible API | More complex setup than Ollama | ⚠️ Viable alternative |
| LM Studio | Nice GUI | GUI-focused, not server-oriented | ❌ Not for headless |

**Recommended Ollama models for your hardware:**

| Model | Use Case in JobPilot | VRAM | Speed |
|-------|---------------------|------|-------|
| **Qwen 2.5 7B Q4_K_M** (primary) | JD scoring, email classification, keyword extraction | 3.8 GB | ~20 tok/s |
| **Phi-3.5 Mini Q5_K_M** (fast secondary) | Simple formatting, keyword matching, dedup checks | 2.5 GB | ~35 tok/s |

> [!NOTE]
> Ollama can only load **one model at a time** in your 4GB VRAM. The cascade client should default to Qwen 2.5 7B and only swap to Phi-3.5 for lightweight tasks. Model swap takes ~5–10 seconds.

### 4.4 Embedding models (for pgvector similarity)

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **`nomic-embed-text` via Ollama** | Local, free, 768-dim, runs on GPU | Uses VRAM (but small — ~300MB) | ✅ **Recommended** |
| `sentence-transformers` (Python) | CPU-based, no VRAM needed | Slower, Python dependency | ⚠️ Good alternative |
| OpenAI `text-embedding-3-small` | High quality | $0.02/M tokens; adds API dependency | ⚠️ If budget allows |
| Gemini Embedding | Free tier available | Newer, less battle-tested | ⚠️ Viable |

---

## 5. Database Layer

### 5.1 Primary database comparison

| Option | Pros | Cons | RAM Usage | Verdict |
|--------|------|------|-----------|---------|
| **PostgreSQL 16** (PRD choice) | Robust, pgvector support, full SQL, concurrent access | Heavier than SQLite (~300–500MB RAM) | 1.0–1.5 GB | ✅ **Recommended** |
| SQLite | Zero config, zero RAM overhead, single file | No concurrent writers, no pgvector | ~0 MB | ⚠️ Viable for MVP |
| MySQL/MariaDB | Widely used | No native vector extension, no advantage over PG | ~500MB | ❌ No upside |
| MongoDB | Flexible schema | Unnecessary complexity, poor for relational data | ~500MB | ❌ Wrong paradigm |

### Why PostgreSQL over SQLite for JobPilot

| Requirement | PostgreSQL | SQLite |
|-------------|-----------|--------|
| Concurrent agent writes (9 agents) | ✅ Native MVCC concurrency | ❌ Single writer lock — agents would block each other |
| pgvector (resume-JD similarity) | ✅ `CREATE EXTENSION vector` | ❌ No vector support (would need a separate FAISS/ChromaDB) |
| Complex queries (feedback analysis, tracker) | ✅ Full SQL, window functions, CTEs | ⚠️ Supported but slower at scale |
| APScheduler persistent job store | ✅ Supported natively | ⚠️ Supported but less robust |
| LangGraph checkpoint persistence | ✅ First-class PostgreSQL support | ⚠️ SQLite support exists but less tested |

> [!IMPORTANT]
> **Verdict: PostgreSQL 16 is worth the ~1.5GB RAM cost.** The concurrent write support alone justifies it — 9 agents writing simultaneously to SQLite would cause constant lock contention.

### 5.2 Vector search: pgvector

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **pgvector** (PostgreSQL extension) | Same database, no extra service, HNSW indexes | Slightly slower than dedicated vector DBs at scale | ✅ **Recommended** |
| ChromaDB | Simple Python API | Extra service, extra RAM (~300MB), another dependency | ❌ Unnecessary |
| Qdrant | Production-grade vector DB | Heavy, overkill for <10K vectors | ❌ Over-engineered |
| FAISS (in-memory) | Fastest search | No persistence, all in RAM | ❌ Fragile |
| Milvus | Enterprise-grade | Way too heavy | ❌ Absurd for this scale |

**PostgreSQL tuning for 16GB system:**

```ini
# /etc/postgresql/16/main/postgresql.conf (key settings)
shared_buffers = 512MB          # 25% of allocated 2GB
effective_cache_size = 2GB      # Hint for query planner
work_mem = 16MB                 # Per-operation sort memory
maintenance_work_mem = 128MB    # For VACUUM, CREATE INDEX
max_connections = 30            # 9 agents + monitoring + overhead
wal_buffers = 16MB
checkpoint_completion_target = 0.9
```

---

## 6. Container Strategy — Docker Compose vs Bare Metal

### Do you need Docker Compose if running everything locally?

**Short answer:** You don't *need* it, but it significantly simplifies management. Here's the tradeoff:

### Option A: Docker Compose (Recommended) ✅

| Aspect | Detail |
|--------|--------|
| **Pros** | Single `docker-compose up -d` starts everything; isolated environments; reproducible; easy to reset/rebuild; resource limits per container; health checks built-in; standard deployment practice |
| **Cons** | ~300MB RAM overhead for Docker Engine; slightly more complex debugging; GPU passthrough needs `nvidia-container-toolkit` |
| **Best for** | Production-like deployment, complex multi-service apps (which JobPilot is) |

```yaml
# docker-compose.yml — Resource-constrained configuration
version: "3.8"

services:
  jobpilot:
    build: .
    deploy:
      resources:
        limits: { memory: 2G, cpus: "4.0" }
    depends_on: [postgres, ollama]
    restart: unless-stopped

  postgres:
    image: pgvector/pgvector:pg16
    deploy:
      resources:
        limits: { memory: 1536M, cpus: "2.0" }
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  ollama:
    image: ollama/ollama:latest
    deploy:
      resources:
        limits: { memory: 6G }
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    volumes:
      - ollama_models:/root/.ollama
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:v2.52.0
    deploy:
      resources:
        limits: { memory: 512M, cpus: "1.0" }
    restart: unless-stopped

  grafana:
    image: grafana/grafana:11.0.0
    deploy:
      resources:
        limits: { memory: 256M, cpus: "1.0" }
    restart: unless-stopped

volumes:
  pgdata:
  ollama_models:
```

### Option B: Bare Metal (systemd services)

| Aspect | Detail |
|--------|--------|
| **Pros** | Zero overhead; direct GPU access (no nvidia-container-toolkit); simpler debugging; maximum performance |
| **Cons** | Manual service management; dependency conflicts possible; harder to reproduce; manual resource limits via cgroups |
| **Best for** | Maximizing every MB of RAM on constrained hardware |

```ini
# /etc/systemd/system/jobpilot.service
[Unit]
Description=JobPilot Application
After=postgresql.service ollama.service
Wants=postgresql.service ollama.service

[Service]
Type=simple
User=jobpilot
WorkingDirectory=/opt/jobpilot
Environment=PATH=/opt/jobpilot/venv/bin:/usr/bin
ExecStart=/opt/jobpilot/venv/bin/python -m jobpilot.main
Restart=always
RestartSec=10
MemoryMax=2G

[Install]
WantedBy=multi-user.target
```

### Option C: Hybrid (Recommended for your setup) ⭐

Run **PostgreSQL, Prometheus, and Grafana** via Docker Compose (they benefit most from containerization), but run **Ollama and the JobPilot app** directly on bare metal (direct GPU access, no container overhead).

| Component | Where | Why |
|-----------|-------|-----|
| PostgreSQL + pgvector | Docker | Easy version management, volume persistence, pgvector pre-installed |
| Prometheus | Docker | Standard container deployment, config mounting |
| Grafana | Docker | Standard container deployment, dashboard provisioning |
| Ollama | Bare metal (systemd) | Direct GPU access without nvidia-container-toolkit complexity |
| JobPilot app | Bare metal (systemd) | Direct filesystem access for resumes, maximum RAM efficiency |
| Playwright | Bare metal (with app) | Browser automation works better outside containers |

> [!TIP]
> **Start with Option A (full Docker Compose)** for simplicity. If you hit RAM pressure, migrate Ollama and the app to bare metal (Option C). The switch is straightforward.

### Docker GPU setup (if using Docker for Ollama)

```bash
# Install NVIDIA Container Toolkit on Ubuntu 24.04
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

## 7. Networking — Dynamic IP, Tunnels & Webhooks

### Why you need a tunnel

JobPilot requires **inbound connections** for:

1. **Telegram Bot webhooks** — Telegram sends updates to YOUR server when you press "Apply" or "Skip". Without a public URL, you must use polling (slower, less efficient).
2. **MCP servers** — If you want to access JobPilot tools from Claude Desktop or Cursor on your Mac, the MCP server needs to be reachable.
3. **Grafana dashboard** — Viewing dashboards from your Mac remotely.

### How to check if your IP is dynamic

```bash
# Check your current public IP
curl ifconfig.me

# Check again tomorrow — if it changed, it's dynamic
# Most Indian ISPs (Airtel, Jio, ACT) assign dynamic IPs
```

### Tunnel/proxy options

| Option | Cost | Speed | Ease | Best For | Verdict |
|--------|------|-------|------|----------|---------|
| **Cloudflare Tunnel** | Free | Fast | Easy | HTTP services (Telegram webhooks, Grafana, MCP) | ✅ **Recommended** |
| **Tailscale** | Free (personal) | Fast | Very easy | Secure access from your Mac to the server | ✅ **Recommended (complementary)** |
| ngrok | Free (limited) | Good | Very easy | Quick testing | ⚠️ Free tier has session limits |
| Wireguard (self-hosted) | Free | Fastest | Complex | Full VPN to home network | ⚠️ Advanced users |
| Port forwarding (router) | Free | Direct | Medium | Static IP only | ❌ Won't work with dynamic IP |
| Pagekite | ~$3/month | Good | Easy | Persistent tunnels | ⚠️ Paid |

### Recommended setup: Cloudflare Tunnel + Tailscale

**Cloudflare Tunnel (for inbound webhooks):**
- Creates a persistent, secure tunnel from your laptop to Cloudflare's edge
- Gives you a stable public URL (e.g., `jobpilot.yourdomain.com`) even with dynamic IP
- Free plan is sufficient
- Requires a domain name (~$10/year for a `.dev` or `.xyz`)

```bash
# Install cloudflared
sudo apt install cloudflared

# Authenticate (one-time)
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create jobpilot

# Route traffic
cloudflared tunnel route dns jobpilot telegram-webhook.yourdomain.com

# Run (or create a systemd service)
cloudflared tunnel run jobpilot
```

**Tailscale (for your Mac → server access):**
- Install on both your Mac and Ubuntu server
- Creates a private VPN mesh — access Grafana, SSH, etc. from anywhere
- Zero configuration, works through NAT
- Free for personal use (up to 100 devices)

```bash
# Install on Ubuntu
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Install on Mac
# Download from Mac App Store or website
# Your server gets a Tailscale IP like 100.x.y.z
# Access Grafana from Mac: http://100.x.y.z:3000
```

### Alternative: Telegram long polling (no tunnel needed)

If you don't want to set up tunnels initially, Telegram's `python-telegram-bot` supports **long polling** — your server polls Telegram's API for updates instead of receiving webhooks. This is simpler but slightly slower (1–3 second delay vs instant webhooks).

```python
# Long polling (no public URL needed)
application = Application.builder().token(BOT_TOKEN).build()
application.run_polling()  # Polls Telegram API every ~1 second

# vs Webhook (needs public URL)
application.run_webhook(
    listen="0.0.0.0",
    port=8443,
    webhook_url="https://telegram-webhook.yourdomain.com/webhook"
)
```

> [!TIP]
> **Start with long polling** (zero setup). Migrate to webhooks via Cloudflare Tunnel later if you want faster response times.

---

## 8. Web Scraping & Browser Automation

### 8.1 HTTP client (for APIs and static pages)

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **httpx** (async) | Native async, HTTP/2, connection pooling | — | ✅ **Recommended** |
| aiohttp | Mature async HTTP | Slightly less ergonomic than httpx | ⚠️ Viable |
| requests | Simplest API | Synchronous only | ❌ Blocks the event loop |

### 8.2 Browser automation (for JS-rendered career pages + LinkedIn)

| Option | Pros | Cons | RAM per instance | Verdict |
|--------|------|------|-----------------|---------|
| **Playwright** (PRD choice) | Multi-browser, async Python, stealth plugins, auto-wait | ~300–500MB per browser context | 300–500 MB | ✅ **Recommended** |
| Selenium | Widely used, large community | Slower, less modern API, no native async | ~400–600 MB | ❌ Legacy |
| Puppeteer | Fast, Google-maintained | Node.js only, no Python | — | ❌ Wrong language |
| Scrapy | Powerful scraping framework | No JS rendering, needs Splash/Playwright integration | ~100 MB | ⚠️ For static pages only |
| curl_cffi | Fast, TLS fingerprint spoofing | No JS rendering | ~50 MB | ⚠️ For API calls only |

**Memory management for Playwright on 16GB:**

```python
# Limit concurrent browser contexts
browser = await playwright.chromium.launch(
    headless=True,
    args=[
        '--disable-gpu',           # Save GPU for Ollama
        '--disable-dev-shm-usage', # Prevent /dev/shm issues in Docker
        '--no-sandbox',
        '--disable-extensions',
        '--disable-background-networking',
        '--single-process',        # Reduces memory by ~100MB
    ]
)

# Reuse a single browser instance, create lightweight contexts
context = await browser.new_context()
page = await context.new_page()
# ... scrape ...
await context.close()  # Always close to free memory
```

> [!WARNING]
> Limit Playwright to **2 concurrent browser contexts max** on your system. Each context uses 300–500MB RAM. With 9 agents potentially scraping, implement a browser context pool with a semaphore.

### 8.3 RSS parsing

| Option | Verdict |
|--------|---------|
| **feedparser** | ✅ Lightweight, reliable, de facto standard |
| atoma | ⚠️ Faster but less battle-tested |

### 8.4 Anti-detection (for LinkedIn/Naukri scraping)

| Tool | Purpose | Cost |
|------|---------|------|
| `playwright-stealth` | Patches Playwright to avoid bot detection | Free |
| Random User-Agent rotation | Via `fake-useragent` library | Free |
| Request delays (2–5 second) | Human-like pacing | Free |
| Residential proxy (if blocked) | IP rotation | $5–15/month (only if needed) |

---

## 9. Monitoring & Observability

### 9.1 Metrics: Prometheus

| Option | Pros | Cons | RAM | Verdict |
|--------|------|------|-----|---------|
| **Prometheus** (PRD choice) | Industry standard, powerful PromQL, alerting | Pull-based (needs `/metrics` endpoint) | 300–500 MB | ✅ **Recommended** |
| InfluxDB | Push-based, good for IoT | Different query language, heavier | ~500 MB | ❌ No advantage |
| VictoriaMetrics | Drop-in Prometheus replacement, lower RAM | Smaller community | ~200 MB | ⚠️ If RAM is critical |

> [!TIP]
> If RAM becomes tight, **VictoriaMetrics** is a drop-in Prometheus replacement using 2–5x less memory. Same PromQL, same Grafana integration.

### 9.2 Dashboards: Grafana

| Option | Pros | Cons | RAM | Verdict |
|--------|------|------|-----|---------|
| **Grafana** (PRD choice) | Beautiful dashboards, Prometheus native, alerting | ~200–300MB RAM | 200–300 MB | ✅ **Recommended** |
| Grafana + Alertmanager | Full alerting pipeline | Extra component | +100 MB | ⚠️ Use Grafana's built-in alerts instead |

### 9.3 Application logging

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **Python `logging` → structured JSON → file** | Zero overhead, searchable with `jq` | No centralized search UI | ✅ **Recommended** |
| Loki + Grafana | Grafana-native log search | Extra service, more RAM | ⚠️ Nice-to-have, not essential |
| ELK Stack | Full-text log search | **Way** too heavy (~4GB RAM) | ❌ Absolutely not |

**Recommended logging setup:**

```python
import structlog

structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.BoundLogger,
)

log = structlog.get_logger()
log.info("job_scored", company="Razorpay", role="SBE", score=82)
# Output: {"timestamp":"2026-03-29T13:30:00Z","level":"info","event":"job_scored","company":"Razorpay","role":"SBE","score":82}
```

Log rotation via `logrotate` to prevent disk fill:

```
# /etc/logrotate.d/jobpilot
/var/log/jobpilot/*.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
}
```

---

## 10. Backup & Storage

### 10.1 What to back up

| Data | Priority | Size | Frequency |
|------|----------|------|-----------|
| PostgreSQL database | **Critical** | 2–5 GB | Daily |
| Resume versions (`resume_versions/`) | **Critical** | 200–500 MB | After each evolution cycle |
| Configuration (`config.yaml`, `.env`) | **Critical** | <1 MB | On change |
| Generated resumes (`resumes/`) | Medium | 1–2 GB | Weekly |
| Tracker spreadsheet | Medium | <100 MB | Daily |
| Grafana dashboards | Low | <10 MB | On change |

### 10.2 Free/cheap cloud backup options

| Service | Free Tier | Cost After | Type | Best For | Verdict |
|---------|-----------|-----------|------|----------|---------|
| **Backblaze B2** | 10 GB free | $0.006/GB/month | Object storage | Database dumps, resume archives | ✅ **Recommended** |
| **rclone + Google Drive** | 15 GB free (Gmail) | — | File sync | Config, resume versions | ✅ **Recommended** |
| **GitHub (private repo)** | Unlimited private repos | — | Git | Config, `.tex` templates, resume versions | ✅ **Recommended** |
| Cloudflare R2 | 10 GB free | $0.015/GB/month | Object storage | Alternative to B2 | ⚠️ Viable |
| AWS S3 | 5 GB (12 months) | $0.023/GB/month | Object storage | If already using AWS | ⚠️ Free tier expires |
| Hetzner Storage Box | — | €3.81/month for 1TB | Block storage | Large archives | ⚠️ If needed later |
| rsync.net | — | $0.015/GB/month | ZFS snapshots | Serious backups | ⚠️ Not free |

### 10.3 Recommended backup strategy ($0)

```bash
#!/bin/bash
# /opt/jobpilot/scripts/backup.sh — Run daily via cron

BACKUP_DIR="/opt/jobpilot/backups/$(date +%Y-%m-%d)"
mkdir -p "$BACKUP_DIR"

# 1. PostgreSQL dump
pg_dump -U jobpilot -Fc jobpilot > "$BACKUP_DIR/db.dump"

# 2. Resume versions
tar czf "$BACKUP_DIR/resume_versions.tar.gz" /opt/jobpilot/resume_versions/

# 3. Config
cp /opt/jobpilot/config.yaml "$BACKUP_DIR/"

# 4. Sync to Google Drive (via rclone)
rclone sync "$BACKUP_DIR" gdrive:JobPilot-Backups/$(date +%Y-%m-%d)/

# 5. Cleanup local backups older than 7 days
find /opt/jobpilot/backups/ -maxdepth 1 -mtime +7 -exec rm -rf {} \;
```

```crontab
# Run backup daily at 3 AM IST
0 3 * * * /opt/jobpilot/scripts/backup.sh >> /var/log/jobpilot/backup.log 2>&1
```

---

## 11. Per-Component Resource Allocation

### 11.1 Complete resource map

```
┌─────────────────────────────────────────────────────────────────┐
│                     16 GB RAM ALLOCATION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ██████████░░  OS + system        1.0 GB                       │
│  ████████████████░░░░  PostgreSQL  1.5 GB                      │
│  ████████████████████████████░░░░░░  Ollama (GPU+RAM)  5.0 GB  │
│  ████████████████░░░░  JobPilot App  2.0 GB                    │
│  ██████░░░░  Playwright (2 ctx)    1.0 GB                      │
│  ████░░  Prometheus                0.5 GB                      │
│  ███░░  Grafana                    0.3 GB                      │
│  ███░░  Docker overhead            0.3 GB                      │
│  ░░░░░░░░░░░░░░  Headroom          4.4 GB                     │
│                                                                 │
│  Total committed: ~11.6 GB  |  Headroom: ~4.4 GB              │
└─────────────────────────────────────────────────────────────────┘
```

### 11.2 CPU allocation

| Component | CPU Cores (of 12 threads) | Pattern |
|-----------|--------------------------|---------|
| JobPilot app (Python async) | 4 threads | Bursty — high during scrape cycles |
| Ollama inference | 4 threads | Bursty — high during LLM calls |
| PostgreSQL | 2 threads | Steady low, occasional query spikes |
| Playwright | 2 threads | Bursty during scraping |
| Prometheus + Grafana | 1 thread | Very low, steady |
| xelatex | 1 thread | Transient, <30s bursts (slightly slower than pdflatex but negligible) |

> [!NOTE]
> The Ryzen 5600's 6 cores / 12 threads are more than sufficient. Python's GIL limits true parallelism, but async I/O and subprocesses (Ollama, pdflatex) sidestep this effectively.

### 11.3 Disk I/O considerations

| Operation | I/O Pattern | SSD Impact |
|-----------|------------|-----------|
| PostgreSQL WAL writes | Sequential write, steady | Low |
| Prometheus TSDB compaction | Periodic sequential | Low |
| Playwright page loads | Read-heavy, cached | Minimal |
| xelatex compilation | Burst read/write | Minimal (slightly more than pdflatex due to font loading) |
| Ollama model loading | Large sequential read (4GB) on model swap | Medium (infrequent) |

Your SSD can handle this easily. No I/O bottlenecks expected.

---

## 12. Monthly Cost Breakdown

### Realistic operational cost

| Item | Monthly Cost | Notes |
|------|-------------|-------|
| **Electricity** (laptop 24/7) | ~$3–5 | HP gaming laptop ~65–80W under light load × 24/7 ≈ 47–58 kWh × ₹6–8/kWh ≈ ₹280–460 |
| **DeepSeek V3 API** | $0 | 50M tokens/day free tier — more than enough |
| **Qwen API (DashScope)** | $0 | Free tier covers overflow |
| **Gemini Flash API** | $0 | 15 RPM free tier |
| **Claude Sonnet** (resume rewriting) | $2–3 | ~60 resume rewrite cycles/month + 12–18 mediations |
| **GPT-4o-mini** (rare fallback) | $0.50–1 | Only when all free tiers exhausted |
| **Apollo.io** | $0 | 50 free credits/month |
| **Hunter.io** | $0 | 25 free searches/month |
| **Telegram Bot API** | $0 | Always free |
| **Gmail API** | $0 | Free for personal use |
| **Domain name** (for Cloudflare Tunnel) | ~$0.83 | ~$10/year for `.dev` or `.xyz` domain |
| **Internet** (existing) | $0 | Already paying for home broadband |
| | | |
| **Total** | **~$6–10/month** | Within budget ✅ |
| **$0 path** (all Ollama, no domain) | **~$3–5** | Electricity only cost |

### One-time costs

| Item | Cost | Notes |
|------|------|-------|
| Domain name (optional) | ~$10/year | For Cloudflare Tunnel; not needed if using long polling |
| UPS (recommended for 24/7) | ₹2,000–4,000 | Protects against power outages; saves SSD from sudden power loss |

---

## 13. Recommended Final Stack — Summary Table

| Layer | Technology | Version | Why This One |
|-------|-----------|---------|-------------|
| **OS** | Ubuntu 24.04 LTS (headless) | 24.04 | Already installed; LTS; first-class NVIDIA support |
| **Language** | Python | 3.12+ | AI ecosystem standard; LangGraph requires it |
| **Orchestration** | LangGraph | Latest | Graph-based state machine with native HITL checkpoints |
| **Web framework** | FastAPI | 0.111+ | Async, `/metrics` endpoint, webhook receiver |
| **Scheduling** | APScheduler | 3.10+ | In-process cron, async support, PostgreSQL job store |
| **Database** | PostgreSQL 16 + pgvector | PG 16, pgvector 0.7+ | Concurrent writes, vector search, LangGraph persistence |
| **Local LLM runtime** | Ollama | Latest | Simple, GPU-accelerated, REST API |
| **Local LLM model** | Qwen 2.5 7B Q4_K_M | — | Best structured JSON output at 7B; fits 4GB VRAM |
| **Cloud LLM (free)** | DeepSeek V3 → Qwen Cloud → Gemini Flash | — | Cascading free tiers cover 95%+ of calls |
| **Cloud LLM (paid)** | Claude Sonnet 4.5 (resume), GPT-4o-mini (fallback) | — | Quality-critical tasks only |
| **Embeddings** | nomic-embed-text via Ollama | — | Local, free, good quality |
| **Browser automation** | Playwright (async Python) | 1.44+ | JS rendering, LinkedIn scraping, stealth plugins |
| **HTTP client** | httpx (async) | 0.27+ | Async, HTTP/2, connection pooling |
| **RSS** | feedparser | 6.x | Lightweight, reliable |
| **Telegram** | python-telegram-bot | 21+ | Official library, inline keyboards |
| **Gmail** | google-api-python-client | 2.x | OAuth 2.0, read/send/label |
| **LaTeX** | TeX Live + xelatex | 2024 | Native Unicode, system fonts via `fontspec`, modern engine |
| **Spreadsheet** | openpyxl | 3.1+ | Excel auto-population with conditional formatting |
| **Monitoring** | Prometheus + Grafana | Prom 2.52, Graf 11.0 | Standard, lightweight, beautiful dashboards |
| **Logging** | structlog → JSON files + logrotate | — | Zero overhead, structured, searchable |
| **Containers** | Docker + Docker Compose | 26.x, Compose V2 | Start here; migrate Ollama/app to bare metal if needed |
| **Tunnel (inbound)** | Cloudflare Tunnel or Telegram long polling | — | Free, stable public URL |
| **VPN (Mac → server)** | Tailscale | — | Free, zero-config, mesh VPN |
| **Backup** | rclone → Google Drive + pg_dump | — | Free, automated, daily |
| **GPU drivers** | NVIDIA 550+ + CUDA 12.x | — | Required for Ollama GPU inference |

### Architecture diagram

```
┌─── Your Mac (anywhere) ─────────────────────────────────────────┐
│  SSH / VS Code Remote / Tailscale → 100.x.y.z                  │
│  Grafana (browser) → http://100.x.y.z:3000                     │
│  Telegram app → interact with bot                               │
└─────────────────────────┬───────────────────────────────────────┘
                          │ Tailscale mesh VPN
                          ▼
┌─── HP Laptop (Ubuntu 24.04 headless, 24/7) ─────────────────────┐
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Docker Compose                                          │   │
│  │  ├── PostgreSQL 16 + pgvector     (1.5 GB RAM)          │   │
│  │  ├── Prometheus                   (0.5 GB RAM)          │   │
│  │  └── Grafana                      (0.3 GB RAM)          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Bare Metal (systemd)                                    │   │
│  │  ├── Ollama (Qwen 2.5 7B Q4)    (5.0 GB RAM + GPU)     │   │
│  │  ├── JobPilot App (Python 3.12)  (2.0 GB RAM)           │   │
│  │  │   ├── LangGraph orchestrator                          │   │
│  │  │   ├── 9 agents (Scout, Resume, Application, ...)     │   │
│  │  │   ├── FastAPI (/metrics, /webhook)                    │   │
│  │  │   ├── APScheduler (cron jobs)                         │   │
│  │  │   └── Playwright (headless Chromium)                   │   │
│  │  └── Cloudflare Tunnel           (minimal RAM)          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  GPU: GTX 1650 → Ollama inference (Qwen 2.5 7B, ~20 tok/s)    │
│  SSD: 500 GB → ~50 GB used, ~450 GB free                       │
│  RAM: 16 GB → ~11.6 GB committed, ~4.4 GB headroom             │
└──────────────────────────────────────────────────────────────────┘
         │
         │ Cloudflare Tunnel (outbound connection, no port forwarding)
         ▼
┌─── External Services ──────────────────────────────────────────┐
│  Telegram Bot API (webhooks or long polling)                    │
│  DeepSeek API / Qwen API / Gemini API / OpenAI / Anthropic    │
│  Gmail API (OAuth 2.0)                                          │
│  Apollo.io / Hunter.io (recruiter enrichment)                   │
│  LinkedIn / Naukri.com (scraping targets)                       │
│  Google Drive (backup via rclone)                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Appendix A: Quick Start Checklist

After finalizing the tech stack, here's the setup order:

1. [ ] Switch Ubuntu to headless mode
2. [ ] Install NVIDIA drivers + CUDA
3. [ ] Install Docker + Docker Compose
4. [ ] Install Ollama, pull `qwen2.5:7b-instruct-q4_K_M`
5. [ ] Install Python 3.12, create venv
6. [ ] Install TeX Live with xelatex (`texlive-xetex` + recommended packages)
7. [ ] Start PostgreSQL + Prometheus + Grafana via Docker Compose
8. [ ] Install Tailscale on server + Mac
9. [ ] Set up Cloudflare Tunnel (or start with Telegram long polling)
10. [ ] Configure systemd services for Ollama + JobPilot
11. [ ] Configure lid-close, sleep, swap, thermal management
12. [ ] Set up backup cron job with rclone
13. [ ] Begin JobPilot development (Weekend 1: Scout Agent)

---

## Appendix B: Alternative Stacks Compared

### "Maximum Free" stack ($0 API cost)

Replace all cloud LLM calls with Ollama. Lower quality on resume rewriting (~15–20% reduction) but fully functional.

| Change | Impact |
|--------|--------|
| Claude Sonnet → Ollama Qwen 2.5 7B | Resume bullets less polished |
| GPT-4o-mini → Ollama | Slightly slower fallback |
| No domain → Telegram long polling | 1–3s delay on notifications |
| **Total cost: electricity only (~$3–5/month)** | |

### "Maximum Quality" stack (~$15–20/month)

| Change | Impact |
|--------|--------|
| Claude Sonnet for all resume tasks | Better phrasing across all resumes |
| GPT-4 for complex scoring | Higher accuracy on edge-case JDs |
| Hetzner VPS backup ($4.50) | Off-site redundancy |
| Residential proxy ($5) | Never get IP-blocked on LinkedIn |
| **Total cost: ~$15–20/month** | |

### "Cloud-first" stack (no self-hosting)

If you ever want to move off the laptop:

| Component | Cloud Option | Cost |
|-----------|-------------|------|
| Server | Hetzner CX22 (4GB, 2vCPU) | €4.50/month |
| Database | Supabase (free: 500MB) | $0 |
| GPU inference | Groq / Together AI / Replicate | $2–5/month |
| Monitoring | Grafana Cloud (free: 10K metrics) | $0 |
| **Total** | | **~$7–12/month** |

> [!NOTE]
> The cloud-first stack lacks GPU for local inference but is more reliable for 24/7 uptime (no power outages, no overheating laptop). Consider this as a future migration path.
