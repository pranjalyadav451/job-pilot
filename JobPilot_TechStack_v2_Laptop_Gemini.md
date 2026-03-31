# JobPilot — Tech Stack & Architecture (v2 - Laptop Server Edition)

**Author:** Pranjal Yadav  
**Date:** 29 March 2026  
**Hardware Constraint:** HP Gaming Laptop — Ryzen 5 5600, 16GB RAM, 500GB SSD, NVIDIA GTX 1650 (4GB VRAM)  
**Deployment Model:** 24/7 Dedicated Home Server (always plugged in)  
**Budget:** Near $0/month operational targeting  

---

## Executive Summary

This document defines the technology stack for the JobPilot Multi-Agent pipeline, aggressively optimised for your 16GB RAM constraint while leveraging your GTX 1650 GPU for local inference. It retains high-quality Chinese LLMs in the primary cascade while explicitly incorporating **Google Gemini** for premium reasoning tasks and fallback automation.

---

## 1. Core Framework & Orchestration

| Component | Technology | Justification |
|-----------|-----------|---------------|
| **Language** | Python 3.12+ | Ecosystem dominance (Ollama, AI toolkits, web scraping). |
| **Orchestrator** | LangGraph | Graph-based state machine natively supporting Human-In-The-Loop (HITL) checkpoints and complex agent cycles. |
| **Web Server** | FastAPI | Async support for scraping, fast webhook receiving for Telegram inputs. |
| **Task Scheduling** | APScheduler | In-process, lightweight cron replacement (zero external dependencies like Redis). |
| **Resume Compiler** | TeX Live (`pdflatex`) | Precise LaTeX compilation for the custom ATS resume template. |

---

## 2. LLM Cascading Strategy (Cloud + Local)

To maintain a $0-$10 monthly cost, JobPilot routes LLM requests through a sophisticated 4-tier cascade. 

> [!IMPORTANT]  
> **Gemini Integration:** Gemini 2.0 Flash is injected as the most reliable first-line fallback, while the state-of-the-art **Gemini 3.1 Pro (High)** is utilized for heavy reasoning tasks (like the Resume Adaptive Mediator).

| Tier | Provider & Model | Cost | Assigned Workflow Tasks | Reason |
|:---:|:---|:---|:---|:---|
| **Tier 1**<br>*(Primary)* | **DeepSeek V3**<br>**Qwen 2.5 Plus** | Free Tier | JD parsing, Relevance Scoring, Tracker Updates, Email Classification, Outreach Drafts. | 50M free tokens/day covers 95% of pipeline volume. |
| **Tier 2**<br>*(Fallback)* | **Google Gemini 2.0 Flash** | Free Tier | Direct fallback if Tier 1 hits rate limits or API timeouts. | Google's 15 RPM free tier is highly stable and extremely fast. |
| **Tier 3**<br>*(Premium)* | **Google Gemini 3.1 Pro**<br>*(or Claude Sonnet)* | Paid<br>(<$3/mo) | **Adaptive Resume Editor** & **Tournament Mediator**. Used exclusively for quality-critical bullet rewriting. | Highest quality reasoning required for ATS-optimised vocabulary adjustments. |
| **Tier 4**<br>*(Local)* | **Ollama**<br>(Qwen 2.5 7B Q4) | **$0**<br>(Uses GTX 1650) | Terminal fallback when no internet is available or APIs are entirely down. Highly private data processing. | Fits perfectly inside the 4GB VRAM while leaving room for the OS. |

---

## 3. Database & Persistence Layer (16GB RAM Focus)

With 16GB total system RAM, Memory management is the tightest bottleneck. 
*Playwright uses ~1GB, Ollama (7B) uses ~4-5GB, OS uses ~1.5GB.*

You have two competing options for persistence. Both are fully viable:

### Option A: PostgreSQL + `pgvector` (Recommended for scale)
- **RAM Footprint:** ~1.0 GB - 1.5 GB.
- **Pros:** Native concurrency (9 agents writing simultaneously without locks), industry-standard vector search via `pgvector`, natively supported LangGraph checkpointing.
- **Cons:** Eats 10% of your total system RAM.

### Option B: SQLite + `sqlite-vec` (Recommended for RAM savings)
- **RAM Footprint:** ~50 MB (Near zero).
- **Pros:** Massive RAM savings, single-file DB format, wildly fast for simple sequential operations, embedded vector search via `sqlite-vec`.
- **Cons:** Concurrency limits (LangGraph agents could lock the DB if trying to write at the exact same millisecond). 
- **Verdict:** If Playwright contexts cause Out-Of-Memory (OOM) errors, switch to SQLite.

---

## 4. Networking & Webhooks (Free Tunnels)

JobPilot needs to accept incoming webhooks from Telegram (When you hit `[Apply ✓]`).

> [!TIP]
> **Cloudflare Tunnel (Highly Recommended):** Easiest, most secure, and 100% free solution. It exposes your local FastAPI server to a public URL (`https://jobpilot.yourdomain.com`) without opening router ports or dealing with your dynamic IP.

Alternative Free Tunnels (No domain required):
- **Pinggy.io:** Run `ssh -p 443 -R0:localhost:8000 a.pinggy.io` for an instant public URL.
- **LocalTunnel:** Fast and free Node-based tunnel.

---

## 5. Browser Automation & Scraping

- **Agent:** Playwright (Python Async).
- **Constraints:** Hard-capped to **2 concurrent browser contexts**. (You must avoid launching 10 browsers at once, or your 16GB RAM will instantly fill up).
- **RSS Auto-detection:** Used preferentially (`feedparser`) to entirely bypass the need for Playwright on companies that expose green-house/lever RSS feeds, saving massive compute cycles.

---

## 6. Container Strategy (Hybrid Deployment)

Running the entire stack in `docker-compose` adds around ~300MB - 500MB of overhead and complicates GPU passthrough. Because you are treating the laptop as a permanent server:

1. **Bare Metal (Systemd):** Run **Ollama**, **Playwright**, and the **Python App** directly on Ubuntu 24.04. This guarantees direct 100% access to your GTX 1650 with zero docker networking loss or complicated NVIDIA-CTK setups.
2. **Docker Compose:** Run **PostgreSQL**, **Prometheus**, and **Grafana** in lightweight containers to keep data management and monitoring clean.

---

## 7. Monthly Cost Projection

Running this laptop 24/7 (plugged in without battery limits, as confirmed) results in the following:

| Expense | Cost Estimation |
|:---|:---|
| **Laptop Electricity** | ~$2 - $4 / month (estimated Indian electricity rates for 60W continuous). |
| **DeepSeek & Qwen API** | $0 (Free tiers). |
| **Google Gemini Flash API** | $0 (Free tier). |
| **Google Gemini 3.1 Pro API** | < $5.00 / month (Severely restricted usage for resume rewriting only). |
| **Ollama Local GPU** | $0. |
| **Cloudflare Tunnel** | $0. |
| **Total Target** | **< $10 / month** |

---

## 8. Summary Checklist Before Development

- [x] Hardware capability confirmed (16GB RAM requires concurrency limiting on Playwright).
- [x] OS prepared (Ubuntu 24.04 Server / headless).
- [x] Tunnel strategy confirmed (Cloudflare Tunnel).
- [x] Gemini models integrated into the fallback cascade.
- [x] Battery lifespan constraints fully acknowledged.
- [ ] Finalize Database choice (SQLite vs PostgreSQL) based on initial load testing.
