# JobPilot — Environment Setup Plan

**Hardware:** HP Gaming Laptop — Ryzen 5 5600 (6C/12T), 16GB RAM, 500GB SSD, NVIDIA GTX 1650 (4GB VRAM)  
**OS:** Ubuntu 24.04 LTS  
**Deployment:** Hybrid — Docker Compose for infra services, bare-metal for app + Ollama  
**Sources:** [PRD v1.1](file:///home/pranjal/ai-projects/job-pilot/JobPilot_PRD_v1.1_Definitive.md), [TechStack Analysis](file:///home/pranjal/ai-projects/job-pilot/JobPilot_TechStack_Analysis.md), [TechStack v2](file:///home/pranjal/ai-projects/job-pilot/JobPilot_TechStack_v2_Laptop_Gemini.md)

---

## Overview

| Phase | What                                                                                                |  Est. Time   | Dependencies |
| :---: | --------------------------------------------------------------------------------------------------- | :----------: | :----------: |
|   1   | [OS Hardening for 24/7 Server](#phase-1--os-hardening-for-247-server)                               |    15 min    |      —       |
|   2   | [NVIDIA Drivers + CUDA](#phase-2--nvidia-drivers--cuda)                                             |    20 min    |   Phase 1    |
|   3   | [Docker + Docker Compose](#phase-3--docker--docker-compose)                                         |    10 min    |   Phase 2    |
|   4   | [PostgreSQL 16 + pgvector (Docker)](#phase-4--postgresql-16--pgvector-via-docker)                   |    15 min    |   Phase 3    |
|   5   | [Ollama Local LLM (Bare Metal)](#phase-5--ollama-local-llm-bare-metal)                              |    20 min    |   Phase 2    |
|   6   | [Python 3.12 + Virtual Environment](#phase-6--python-312--virtual-environment)                      |    15 min    |   Phase 1    |
|   7   | [TeX Live + XeLaTeX](#phase-7--tex-live--xelatex)                                                   |    10 min    |   Phase 1    |
|   8   | [Playwright Browser Automation](#phase-8--playwright-browser-automation)                            |    5 min     |   Phase 6    |
|   9   | [Telegram Bot Setup](#phase-9--telegram-bot-setup)                                                  |    10 min    |      —       |
|  10   | [Gmail OAuth 2.0 Setup](#phase-10--gmail-oauth-20-setup)                                            |    20 min    |   Phase 6    |
|  11   | [Cloud LLM API Keys](#phase-11--cloud-llm-api-keys)                                                 |    15 min    |      —       |
|  12   | [Networking — Cloudflare Tunnel + Tailscale](#phase-12--networking--cloudflare-tunnel--tailscale)   |    20 min    |   Phase 1    |
|  13   | [Monitoring — Prometheus + Grafana (Docker)](#phase-13--monitoring--prometheus--grafana-via-docker) |    15 min    |   Phase 3    |
|  14   | [Backup Automation](#phase-14--backup-automation)                                                   |    10 min    |   Phase 4    |
|  15   | [Systemd Services + Final Validation](#phase-15--systemd-services--final-validation)                |    15 min    |  All above   |
|       | **Total**                                                                                           | **~3.5 hrs** |              |

---

## Phase 1 — OS Hardening for 24/7 Server

> [!IMPORTANT]
> This laptop will run 24/7 with the lid closed. These steps prevent sleep/hibernation, configure swap as a RAM safety net, and enable thermal management.

### 1.1 Switch to headless mode (saves ~500MB RAM)

```bash
sudo systemctl set-default multi-user.target
# Do NOT reboot yet — complete the remaining steps in this phase first
```

### 1.2 Disable sleep, suspend, and hibernate

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

### 1.3 Disable lid-close action

```bash
sudo sed -i 's/#HandleLidSwitch=.*/HandleLidSwitch=ignore/' /etc/systemd/logind.conf
sudo sed -i 's/#HandleLidSwitchExternalPower=.*/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf
sudo systemctl restart systemd-logind
```

### 1.4 Configure swap (8GB — safety net for RAM spikes)

```bash
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make persistent across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 1.5 Thermal management

```bash
sudo apt update
sudo apt install -y thermald tlp
sudo systemctl enable thermald tlp
sudo systemctl start thermald tlp
```

### 1.6 Unattended security updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades  # Select "Yes"
```

### 1.7 Reboot into headless mode

```bash
sudo reboot
# SSH back in after reboot:
# ssh pranjal@<laptop-ip>
```

### ✅ Verification

```bash
# Confirm headless
systemctl get-default        # Expected: multi-user.target

# Confirm sleep is masked
systemctl status sleep.target  # Expected: masked

# Confirm swap
swapon --show                # Expected: /swapfile 8G

# Confirm lid switch
grep HandleLidSwitch /etc/systemd/logind.conf  # Expected: ignore
```

---

## Phase 2 — NVIDIA Drivers + CUDA

> [!IMPORTANT]
> Required for Ollama GPU inference on the GTX 1650 (4GB VRAM). Without this, Ollama falls back to CPU (~5x slower).

### 2.1 Install NVIDIA drivers

```bash
sudo apt install -y ubuntu-drivers-common
sudo ubuntu-drivers install --gpgpu

# OR install a specific version if the above fails:
# sudo apt install -y nvidia-driver-550
```

### 2.2 Reboot and verify

```bash
sudo reboot
# After reboot:
nvidia-smi
```

### 2.3 Install CUDA toolkit

```bash
sudo apt install -y nvidia-cuda-toolkit
```

### ✅ Verification

```bash
nvidia-smi
# Expected: Driver Version: 550.x+, CUDA Version: 12.x
# Expected: GTX 1650 listed with 4096 MiB memory

nvcc --version
# Expected: CUDA compilation tools, release 12.x
```

---

## Phase 3 — Docker + Docker Compose

> [!NOTE]
> Docker will host PostgreSQL + pgvector, Prometheus, and Grafana. The JobPilot app and Ollama run on bare metal for direct GPU access and maximum RAM efficiency.

### 3.1 Install Docker Engine

```bash
# Remove any old Docker versions
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Add Docker's official GPG key and repository
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3.2 Post-install (run Docker without sudo)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 3.3 Enable Docker on boot

```bash
sudo systemctl enable docker
```

### 3.4 Install NVIDIA Container Toolkit (for future Docker GPU use)

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### ✅ Verification

```bash
docker --version            # Expected: Docker version 26.x+
docker compose version      # Expected: Docker Compose version v2.x+
docker run --rm hello-world # Expected: "Hello from Docker!"
```

---

## Phase 4 — PostgreSQL 16 + pgvector (via Docker)

> [!IMPORTANT]
> PostgreSQL is the primary data store for all 9 agents — job postings, application state, dedup hashes, email classification, LangGraph checkpoints, and pgvector embeddings. Memory-capped at 1.5GB.

### 4.1 Create the project directory structure

```bash
mkdir -p /home/pranjal/ai-projects/job-pilot/{docker,scripts,credentials,resumes,resume_versions,tracker,templates,data,monitoring/grafana/{provisioning/datasources,provisioning/dashboards,dashboards}}
```

### 4.2 Create docker-compose.yml

Create the file at `/home/pranjal/ai-projects/job-pilot/docker-compose.yml`:

```yaml
version: "3.8"

services:
  postgres:
    image: pgvector/pgvector:pg16
    container_name: jobpilot-postgres
    environment:
      POSTGRES_USER: jobpilot
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: jobpilot
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init_database.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    deploy:
      resources:
        limits:
          memory: 1536M
          cpus: "2.0"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U jobpilot"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

### 4.3 Create PostgreSQL tuning config

Create `scripts/init_database.sql`:

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Tuning for 16GB system (applied at DB level)
ALTER SYSTEM SET shared_buffers = '512MB';
ALTER SYSTEM SET effective_cache_size = '2GB';
ALTER SYSTEM SET work_mem = '16MB';
ALTER SYSTEM SET maintenance_work_mem = '128MB';
ALTER SYSTEM SET max_connections = 30;
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;

SELECT pg_reload_conf();
```

### 4.4 Create .env file

```bash
cat > /home/pranjal/ai-projects/job-pilot/.env << 'EOF'
# === Database ===
POSTGRES_PASSWORD=<generate-a-strong-password>

# === Telegram ===
TELEGRAM_BOT_TOKEN=<from-phase-9>
TELEGRAM_CHAT_ID=<from-phase-9>

# === Gmail ===
# OAuth credentials handled via files in credentials/

# === LLM API Keys (Phase 11) ===
DEEPSEEK_API_KEY=
DASHSCOPE_API_KEY=
GEMINI_API_KEY=
OPENAI_API_KEY=
ANTHROPIC_API_KEY=

# === Monitoring ===
GRAFANA_PASSWORD=<choose-a-password>

# === Enrichment ===
APOLLO_API_KEY=
HUNTER_API_KEY=
EOF

chmod 600 /home/pranjal/ai-projects/job-pilot/.env
```

### 4.5 Add .env to .gitignore

```bash
cat > /home/pranjal/ai-projects/job-pilot/.gitignore << 'EOF'
.env
credentials/
resumes/
tracker/
__pycache__/
*.pyc
.venv/
*.egg-info/
dist/
build/
EOF
```

### 4.6 Start PostgreSQL

```bash
cd /home/pranjal/ai-projects/job-pilot
docker compose up -d postgres
```

### ✅ Verification

```bash
# Check container is running
docker compose ps  # Expected: jobpilot-postgres running, healthy

# Test connection
docker exec -it jobpilot-postgres psql -U jobpilot -c "SELECT 1;"
# Expected: returns 1

# Verify pgvector extension
docker exec -it jobpilot-postgres psql -U jobpilot -c "SELECT extname FROM pg_extension WHERE extname = 'vector';"
# Expected: vector

# Check tuning applied
docker exec -it jobpilot-postgres psql -U jobpilot -c "SHOW shared_buffers;"
# Expected: 512MB
```

---

## Phase 5 — Ollama Local LLM (Bare Metal)

> [!NOTE]
> Ollama runs on bare metal (not Docker) for direct GPU access without nvidia-container-toolkit complications. It serves as the terminal fallback (Tier 4) in the LLM cascade and provides local embeddings via `nomic-embed-text`.

### 5.1 Install Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### 5.2 Pull recommended models

```bash
# Primary local model — best for structured JSON output, fits 4GB VRAM
ollama pull qwen2.5:7b-instruct-q4_K_M

# Fast secondary for lightweight tasks (keyword matching, dedup)
ollama pull phi3.5:3.8b-mini-instruct-q5_K_M

# Embedding model for pgvector similarity search
ollama pull nomic-embed-text
```

> [!WARNING]
> Only **one model** can be loaded in VRAM at a time on your 4GB GPU. Model swap takes ~5–10 seconds. The cascade client should default to Qwen 2.5 7B and only swap to Phi-3.5 for lightweight tasks.

### 5.3 Enable Ollama as a systemd service

Ollama's installer typically creates a systemd service automatically. Verify:

```bash
sudo systemctl enable ollama
sudo systemctl start ollama
```

### ✅ Verification

```bash
# Check service status
systemctl status ollama  # Expected: active (running)

# Check GPU is being used
ollama run qwen2.5:7b-instruct-q4_K_M "Respond with only: GPU test passed"
# Expected: responds in ~1-2 seconds (GPU) vs 10+ seconds (CPU)

# Check API is accessible
curl http://localhost:11434/api/tags | python3 -m json.tool
# Expected: lists qwen2.5, phi3.5, nomic-embed-text

# Verify VRAM usage during inference
nvidia-smi  # Expected: ~3.8GB VRAM used by ollama
```

---

## Phase 6 — Python 3.12 + Virtual Environment

### 6.1 Confirm Python version

```bash
python3 --version
# Expected: Python 3.12.x (ships with Ubuntu 24.04)

# If not 3.12+, install it:
# sudo apt install -y python3.12 python3.12-venv python3.12-dev
```

### 6.2 Install system dependencies

```bash
sudo apt install -y python3-pip python3-venv python3-dev build-essential libpq-dev
```

### 6.3 Create virtual environment

```bash
cd /home/pranjal/ai-projects/job-pilot
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip setuptools wheel
```

### 6.4 Create requirements.txt

```bash
cat > /home/pranjal/ai-projects/job-pilot/requirements.txt << 'EOF'
# === Core Framework ===
langgraph>=0.2.0
langchain>=0.3.0
langchain-community>=0.3.0
fastapi>=0.111.0
uvicorn[standard]>=0.30.0
apscheduler>=3.10.0

# === LLM Clients ===
openai>=1.40.0           # OpenAI + DeepSeek (compatible API)
anthropic>=0.34.0        # Claude Sonnet
google-generativeai>=0.8.0  # Gemini
dashscope>=1.20.0        # Qwen (DashScope)
ollama>=0.3.0            # Ollama Python client

# === Database ===
asyncpg>=0.29.0          # Async PostgreSQL driver
sqlalchemy[asyncio]>=2.0.0
psycopg2-binary>=2.9.0   # Sync PostgreSQL (for migrations, scripts)
pgvector>=0.3.0          # pgvector Python bindings
alembic>=1.13.0          # Database migrations

# === Web Scraping ===
httpx[http2]>=0.27.0     # Async HTTP client
playwright>=1.44.0       # Browser automation
feedparser>=6.0.0        # RSS/Atom parsing
beautifulsoup4>=4.12.0   # HTML parsing
fake-useragent>=1.5.0    # User-agent rotation

# === Telegram ===
python-telegram-bot>=21.0

# === Gmail ===
google-api-python-client>=2.140.0
google-auth-oauthlib>=1.2.0
google-auth-httplib2>=0.2.0

# === Spreadsheet ===
openpyxl>=3.1.0

# === Monitoring ===
prometheus-client>=0.21.0
structlog>=24.4.0

# === Utilities ===
python-dotenv>=1.0.0
pyyaml>=6.0.0
pydantic>=2.8.0
pydantic-settings>=2.4.0
tenacity>=8.5.0          # Retry logic
click>=8.1.0             # CLI commands
rich>=13.0.0             # Rich terminal output

# === Development ===
pytest>=8.3.0
pytest-asyncio>=0.24.0
ruff>=0.6.0              # Linter + formatter
mypy>=1.11.0
EOF
```

### 6.5 Install dependencies

```bash
source .venv/bin/activate
pip install -r requirements.txt
```

### ✅ Verification

```bash
source .venv/bin/activate
python -c "import langgraph; print('LangGraph:', langgraph.__version__)"
python -c "import fastapi; print('FastAPI:', fastapi.__version__)"
python -c "import asyncpg; print('asyncpg:', asyncpg.__version__)"
python -c "import httpx; print('httpx:', httpx.__version__)"
python -c "import telegram; print('python-telegram-bot:', telegram.__version__)"
python -c "import prometheus_client; print('prometheus_client:', prometheus_client.__version__)"
```

---

## Phase 7 — TeX Live + XeLaTeX

> [!NOTE]
> XeLaTeX is chosen over pdfLaTeX for native Unicode support and system font access via `fontspec`. This allows using any `.ttf`/`.otf` font (e.g., Inter, Calibri) in resumes without complex TeX font setup. The ~1–2 second speed penalty per compile is negligible.

### 7.1 Install TeX Live with XeLaTeX

```bash
sudo apt install -y texlive-xetex texlive-latex-recommended texlive-latex-extra \
                    texlive-fonts-recommended texlive-fonts-extra
```

> [!TIP]
> This installs ~1.5GB. The `texlive-fonts-extra` package is optional but gives you many high-quality fonts out of the box.

### 7.2 (Optional) Install custom fonts

```bash
# Example: Install Inter font (popular for modern resumes)
mkdir -p ~/.local/share/fonts
# Download Inter from Google Fonts, extract .ttf files to ~/.local/share/fonts/
fc-cache -fv
```

### ✅ Verification

```bash
xelatex --version
# Expected: XeTeX 3.x / TeX Live 2024+

# Quick compilation test
cat > /tmp/test_xelatex.tex << 'EOF'
\documentclass{article}
\usepackage{fontspec}
\begin{document}
Hello from XeLaTeX — Unicode test: résumé, naïve, Zürich ✓
\end{document}
EOF

xelatex -interaction=nonstopmode -output-directory=/tmp /tmp/test_xelatex.tex
# Expected: produces /tmp/test_xelatex.pdf without errors
```

---

## Phase 8 — Playwright Browser Automation

> [!WARNING]
> Limit Playwright to **2 concurrent browser contexts max** on your 16GB system. Each context uses 300–500MB RAM. Implement a browser context pool with a semaphore in the application code.

### 8.1 Install Playwright browsers

```bash
source /home/pranjal/ai-projects/job-pilot/.venv/bin/activate

# Install Chromium only (saves space vs installing all browsers)
playwright install chromium

# Install system dependencies for headless Chromium
playwright install-deps chromium
```

### 8.2 Install stealth plugin

```bash
pip install playwright-stealth
```

### ✅ Verification

```bash
source /home/pranjal/ai-projects/job-pilot/.venv/bin/activate
python3 << 'PYEOF'
import asyncio
from playwright.async_api import async_playwright

async def test():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True, args=[
            '--disable-gpu', '--disable-dev-shm-usage', '--no-sandbox', '--single-process'
        ])
        page = await browser.new_page()
        await page.goto("https://example.com")
        title = await page.title()
        print(f"Page title: {title}")
        assert "Example" in title, "Playwright test failed!"
        await browser.close()
        print("✅ Playwright working correctly")

asyncio.run(test())
PYEOF
```

---

## Phase 9 — Telegram Bot Setup

### 9.1 Create the bot

1. Open Telegram and search for **@BotFather**
2. Send `/newbot`
3. Name: `JobPilot Bot` (or any name you prefer)
4. Username: `jobpilot_pranjal_bot` (must be unique and end with `bot`)
5. **Copy the bot token** — save it as `TELEGRAM_BOT_TOKEN` in `.env`

### 9.2 Get your chat ID

1. Send any message to your new bot
2. Visit: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
3. Find `"chat": {"id": <number>}` in the response
4. **Copy the chat ID** — save it as `TELEGRAM_CHAT_ID` in `.env`

### 9.3 Update .env

```bash
# Edit /home/pranjal/ai-projects/job-pilot/.env
# Set:
#   TELEGRAM_BOT_TOKEN=<your-token>
#   TELEGRAM_CHAT_ID=<your-chat-id>
```

### ✅ Verification

```bash
source /home/pranjal/ai-projects/job-pilot/.venv/bin/activate
python3 << 'PYEOF'
import os, asyncio
from dotenv import load_dotenv
load_dotenv("/home/pranjal/ai-projects/job-pilot/.env")

from telegram import Bot

async def test():
    bot = Bot(token=os.getenv("TELEGRAM_BOT_TOKEN"))
    msg = await bot.send_message(
        chat_id=os.getenv("TELEGRAM_CHAT_ID"),
        text="🚀 JobPilot environment setup — Telegram integration working!"
    )
    print(f"✅ Message sent! ID: {msg.message_id}")

asyncio.run(test())
PYEOF
# Expected: You receive the message in Telegram
```

---

## Phase 10 — Gmail OAuth 2.0 Setup

### 10.1 Create Google Cloud project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project: **JobPilot**
3. Enable the **Gmail API**: `APIs & Services → Library → Gmail API → Enable`

### 10.2 Create OAuth 2.0 credentials

1. Go to `APIs & Services → Credentials`
2. Click `Create Credentials → OAuth client ID`
3. Application type: **Desktop App**
4. Name: `JobPilot`
5. Download the JSON file
6. Save as: `/home/pranjal/ai-projects/job-pilot/credentials/gmail_oauth.json`

### 10.3 Configure OAuth consent screen

1. Go to `APIs & Services → OAuth consent screen`
2. User type: **External** (or Internal if using Workspace)
3. Add scopes:
   - `https://www.googleapis.com/auth/gmail.readonly`
   - `https://www.googleapis.com/auth/gmail.send`
   - `https://www.googleapis.com/auth/gmail.modify`
4. Add your email as a **test user**

### 10.4 Create the OAuth helper script

Create `scripts/setup_gmail_oauth.py`:

```python
#!/usr/bin/env python3
"""One-time Gmail OAuth 2.0 token setup for JobPilot."""

import os
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow

SCOPES = [
    "https://www.googleapis.com/auth/gmail.readonly",
    "https://www.googleapis.com/auth/gmail.send",
    "https://www.googleapis.com/auth/gmail.modify",
]

CREDENTIALS_PATH = "credentials/gmail_oauth.json"
TOKEN_PATH = "credentials/gmail_token.json"

def main():
    creds = None
    if os.path.exists(TOKEN_PATH):
        creds = Credentials.from_authorized_user_file(TOKEN_PATH, SCOPES)

    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(CREDENTIALS_PATH, SCOPES)
            creds = flow.run_local_server(port=0)

        with open(TOKEN_PATH, "w") as token:
            token.write(creds.to_json())
        os.chmod(TOKEN_PATH, 0o600)

    print("✅ Gmail OAuth setup complete. Token saved to:", TOKEN_PATH)

if __name__ == "__main__":
    main()
```

### 10.5 Run the OAuth flow

```bash
cd /home/pranjal/ai-projects/job-pilot
source .venv/bin/activate
python scripts/setup_gmail_oauth.py
# This will open a browser — sign in with pranjalyadav451@gmail.com and authorise
```

### ✅ Verification

```bash
source .venv/bin/activate
python3 << 'PYEOF'
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

creds = Credentials.from_authorized_user_file("credentials/gmail_token.json")
service = build("gmail", "v1", credentials=creds)
profile = service.users().getProfile(userId="me").execute()
print(f"✅ Gmail connected: {profile['emailAddress']} ({profile['messagesTotal']} messages)")
PYEOF
```

---

## Phase 11 — Cloud LLM API Keys

> [!TIP]
> You only need **DeepSeek** (free, 50M tokens/day) to start. The rest are fallbacks. Collect API keys as you go and add them to `.env`.

### 11.1 API key registration

| Provider                            | Registration URL                                        | Free Tier            | .env Variable       |
| ----------------------------------- | ------------------------------------------------------- | -------------------- | ------------------- |
| **DeepSeek** (Tier 1 — Primary)     | [platform.deepseek.com](https://platform.deepseek.com/) | 50M tokens/day       | `DEEPSEEK_API_KEY`  |
| **Qwen / DashScope** (Tier 1)       | [dashscope.aliyun.com](https://dashscope.aliyun.com/)   | 1M tokens/month      | `DASHSCOPE_API_KEY` |
| **Google Gemini** (Tier 2)          | [aistudio.google.com](https://aistudio.google.com/)     | 15 RPM free          | `GEMINI_API_KEY`    |
| **OpenAI** (Tier 2 — paid fallback) | [platform.openai.com](https://platform.openai.com/)     | None — $0.15/M input | `OPENAI_API_KEY`    |
| **Anthropic** (Premium)             | [console.anthropic.com](https://console.anthropic.com/) | None — $3/M input    | `ANTHROPIC_API_KEY` |
| **Apollo.io** (Enrichment)          | [apollo.io](https://www.apollo.io/)                     | 50 credits/month     | `APOLLO_API_KEY`    |
| **Hunter.io** (Email finder)        | [hunter.io](https://hunter.io/)                         | 25 searches/month    | `HUNTER_API_KEY`    |

### 11.2 Update .env with all keys

```bash
# Edit /home/pranjal/ai-projects/job-pilot/.env
# Fill in each API key as you register
```

### ✅ Verification

```bash
source /home/pranjal/ai-projects/job-pilot/.venv/bin/activate
python3 << 'PYEOF'
import os
from dotenv import load_dotenv
load_dotenv("/home/pranjal/ai-projects/job-pilot/.env")

keys = {
    "DEEPSEEK_API_KEY": "DeepSeek",
    "DASHSCOPE_API_KEY": "Qwen/DashScope",
    "GEMINI_API_KEY": "Gemini",
    "OPENAI_API_KEY": "OpenAI",
    "ANTHROPIC_API_KEY": "Anthropic",
    "APOLLO_API_KEY": "Apollo.io",
    "HUNTER_API_KEY": "Hunter.io",
}

for env_var, name in keys.items():
    val = os.getenv(env_var, "")
    status = "✅ Set" if val else "⬜ Not set (optional)"
    print(f"  {name:20s} ({env_var}): {status}")
PYEOF
```

---

## Phase 12 — Networking — Cloudflare Tunnel + Tailscale

> [!NOTE]
> **Start with Telegram long polling** (zero networking setup needed). Set up Cloudflare Tunnel later when you want faster webhook-based responses. Tailscale is for accessing Grafana/SSH from your Mac.

### 12.1 Tailscale (Mac → Server private VPN)

**On the Ubuntu server:**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# Follow the URL to authenticate with your Tailscale account
```

**On your Mac:**

Download Tailscale from the Mac App Store or [tailscale.com/download](https://tailscale.com/download).

After both devices are connected, your server gets a Tailscale IP like `100.x.y.z`. Access services from your Mac:
- SSH: `ssh pranjal@100.x.y.z`
- Grafana: `http://100.x.y.z:3000`

### 12.2 Cloudflare Tunnel (optional — for Telegram webhooks)

> [!TIP]
> Skip this initially. Use Telegram long polling (`application.run_polling()`) which requires no public URL. Set up webhooks later for instant notifications.

**Prerequisites:** A domain name (~$10/year for `.dev` or `.xyz`), Cloudflare account.

```bash
sudo apt install -y cloudflared

# Authenticate (one-time)
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create jobpilot

# Route traffic
cloudflared tunnel route dns jobpilot telegram-webhook.yourdomain.com

# Run (or create a systemd service)
cloudflared tunnel run jobpilot
```

### ✅ Verification

```bash
# Tailscale
tailscale status   # Expected: shows your server and Mac as connected
tailscale ip -4    # Expected: 100.x.y.z

# From Mac: ping the server
ping 100.x.y.z    # Expected: replies from the server

# Cloudflare Tunnel (if set up)
curl https://telegram-webhook.yourdomain.com/health
# Expected: response from your FastAPI server
```

---

## Phase 13 — Monitoring — Prometheus + Grafana (via Docker)

### 13.1 Create Prometheus config

Create `monitoring/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/alert_rules.yml

scrape_configs:
  - job_name: "jobpilot"
    static_configs:
      - targets: ["host.docker.internal:8000"]
    metrics_path: /metrics
    scrape_interval: 15s
```

### 13.2 Create alert rules (placeholder)

Create `monitoring/alert_rules.yml`:

```yaml
groups:
  - name: jobpilot_alerts
    rules:
      - alert: ScoutAgentStalled
        expr: increase(jobpilot_jobs_discovered_total[2h]) == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Scout Agent has not discovered any jobs in 2 hours"

      - alert: AllLLMsFailing
        expr: jobpilot_llm_cascade_depth > 4.5
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "All LLM providers failing — hitting Ollama on every request"
```

### 13.3 Create Grafana datasource provisioning

Create `monitoring/grafana/provisioning/datasources/prometheus.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

### 13.4 Add monitoring services to docker-compose.yml

Append to the existing `docker-compose.yml`:

```yaml
  prometheus:
    image: prom/prometheus:v2.52.0
    container_name: jobpilot-prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./monitoring/alert_rules.yml:/etc/prometheus/alert_rules.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=90d'
    extra_hosts:
      - "host.docker.internal:host-gateway"
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:11.0.0
    container_name: jobpilot-grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    depends_on:
      - prometheus
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: "1.0"
    restart: unless-stopped
```

Add to `volumes:` section:

```yaml
  prometheus_data:
  grafana_data:
```

### 13.5 Start monitoring stack

```bash
cd /home/pranjal/ai-projects/job-pilot
docker compose up -d prometheus grafana
```

### ✅ Verification

```bash
docker compose ps
# Expected: jobpilot-prometheus and jobpilot-grafana running

# Prometheus
curl http://localhost:9090/-/healthy
# Expected: "Prometheus Server is Healthy."

# Grafana (via Tailscale from Mac)
# Open: http://100.x.y.z:3000
# Login: admin / <your GRAFANA_PASSWORD>
# Expected: Grafana dashboard loads, Prometheus datasource auto-configured
```

---

## Phase 14 — Backup Automation

### 14.1 Install rclone

```bash
sudo apt install -y rclone

# Configure Google Drive remote (one-time interactive setup)
rclone config
# Follow prompts: name=gdrive, type=drive, scope=drive, auto-config=yes
```

### 14.2 Create backup script

Create `scripts/backup.sh`:

```bash
#!/bin/bash
# JobPilot daily backup script
set -euo pipefail

BACKUP_DIR="/home/pranjal/ai-projects/job-pilot/backups/$(date +%Y-%m-%d)"
mkdir -p "$BACKUP_DIR"

echo "$(date): Starting backup..."

# 1. PostgreSQL dump
docker exec jobpilot-postgres pg_dump -U jobpilot -Fc jobpilot > "$BACKUP_DIR/db.dump"
echo "  ✅ Database dumped"

# 2. Resume versions
tar czf "$BACKUP_DIR/resume_versions.tar.gz" -C /home/pranjal/ai-projects/job-pilot resume_versions/ 2>/dev/null || echo "  ⚠️ No resume_versions yet"

# 3. Config + .env
cp /home/pranjal/ai-projects/job-pilot/config.yaml "$BACKUP_DIR/" 2>/dev/null || true
cp /home/pranjal/ai-projects/job-pilot/.env "$BACKUP_DIR/"

# 4. Tracker spreadsheet
cp /home/pranjal/ai-projects/job-pilot/tracker/job_pipeline.xlsx "$BACKUP_DIR/" 2>/dev/null || true

# 5. Sync to Google Drive (if rclone is configured)
if rclone listremotes | grep -q "gdrive:"; then
    rclone sync "$BACKUP_DIR" "gdrive:JobPilot-Backups/$(date +%Y-%m-%d)/" --quiet
    echo "  ✅ Synced to Google Drive"
else
    echo "  ⚠️ rclone not configured — local backup only"
fi

# 6. Cleanup local backups older than 7 days
find /home/pranjal/ai-projects/job-pilot/backups/ -maxdepth 1 -mtime +7 -exec rm -rf {} \;

echo "$(date): Backup complete → $BACKUP_DIR"
```

```bash
chmod +x /home/pranjal/ai-projects/job-pilot/scripts/backup.sh
```

### 14.3 Schedule via cron

```bash
# Add to crontab
(crontab -l 2>/dev/null; echo "0 3 * * * /home/pranjal/ai-projects/job-pilot/scripts/backup.sh >> /home/pranjal/ai-projects/job-pilot/backups/backup.log 2>&1") | crontab -
```

### ✅ Verification

```bash
# Run backup manually
/home/pranjal/ai-projects/job-pilot/scripts/backup.sh
# Expected: creates backup directory with db.dump + config files

# Check cron
crontab -l | grep backup
# Expected: 0 3 * * * /home/pranjal/ai-projects/job-pilot/scripts/backup.sh ...
```

---

## Phase 15 — Systemd Services + Final Validation

### 15.1 Create JobPilot systemd service

Create `/etc/systemd/system/jobpilot.service`:

```ini
[Unit]
Description=JobPilot Multi-Agent Application
After=network-online.target docker.service ollama.service
Wants=network-online.target docker.service ollama.service

[Service]
Type=simple
User=pranjal
WorkingDirectory=/home/pranjal/ai-projects/job-pilot
EnvironmentFile=/home/pranjal/ai-projects/job-pilot/.env
ExecStart=/home/pranjal/ai-projects/job-pilot/.venv/bin/python -m jobpilot.main
Restart=always
RestartSec=10
MemoryMax=2G

[Install]
WantedBy=multi-user.target
```

> [!NOTE]
> Don't enable this service yet — the application code hasn't been written. This is prepared for when development reaches the deployment phase (Weekend 10 in the build sequence).

### 15.2 Create log rotation config

```bash
sudo tee /etc/logrotate.d/jobpilot << 'EOF'
/home/pranjal/ai-projects/job-pilot/logs/*.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
    create 0644 pranjal pranjal
}
EOF

mkdir -p /home/pranjal/ai-projects/job-pilot/logs
```

### 15.3 Final full-stack validation

```bash
echo "=============================="
echo "  JobPilot Environment Check  "
echo "=============================="
echo ""

echo "--- OS ---"
lsb_release -d
systemctl get-default
echo ""

echo "--- GPU ---"
nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv,noheader
echo ""

echo "--- Docker ---"
docker --version
docker compose ps --format "table {{.Name}}\t{{.Status}}"
echo ""

echo "--- Ollama ---"
systemctl is-active ollama
curl -s http://localhost:11434/api/tags | python3 -c "import sys,json; [print(f'  {m[\"name\"]}') for m in json.load(sys.stdin)['models']]"
echo ""

echo "--- Python ---"
/home/pranjal/ai-projects/job-pilot/.venv/bin/python --version
echo ""

echo "--- LaTeX ---"
xelatex --version 2>&1 | head -1
echo ""

echo "--- PostgreSQL ---"
docker exec jobpilot-postgres psql -U jobpilot -tAc "SELECT 'PostgreSQL ' || version();" 2>/dev/null | head -1
docker exec jobpilot-postgres psql -U jobpilot -tAc "SELECT 'pgvector ' || extversion FROM pg_extension WHERE extname='vector';" 2>/dev/null
echo ""

echo "--- Networking ---"
tailscale status 2>/dev/null | head -3 || echo "Tailscale: not configured"
echo ""

echo "--- Monitoring ---"
curl -s http://localhost:9090/-/healthy 2>/dev/null || echo "Prometheus: not running"
curl -s http://localhost:3000/api/health 2>/dev/null | python3 -c "import sys,json; print('Grafana:', json.load(sys.stdin).get('database','unknown'))" 2>/dev/null || echo "Grafana: not running"
echo ""

echo "=============================="
echo "  Setup Complete ✅           "
echo "=============================="
```

---

## Post-Setup: Project Directory Structure

After all phases, your project should look like this:

```
/home/pranjal/ai-projects/job-pilot/
├── .env                          # All API keys and secrets (Phase 4.4)
├── .gitignore                    # Excludes .env, credentials/, resumes/ (Phase 4.5)
├── .venv/                        # Python virtual environment (Phase 6.3)
├── docker-compose.yml            # PostgreSQL + Prometheus + Grafana (Phase 4, 13)
├── requirements.txt              # Python dependencies (Phase 6.4)
├── config.yaml                   # Application config (create during development)
│
├── credentials/                  # OAuth tokens (gitignored)
│   ├── gmail_oauth.json          # Google OAuth client (Phase 10.2)
│   └── gmail_token.json          # Gmail refresh token (Phase 10.5)
│
├── scripts/
│   ├── setup_gmail_oauth.py      # Gmail OAuth helper (Phase 10.4)
│   ├── init_database.sql         # PostgreSQL init + pgvector (Phase 4.3)
│   └── backup.sh                 # Daily backup script (Phase 14.2)
│
├── monitoring/
│   ├── prometheus.yml            # Prometheus scrape config (Phase 13.1)
│   ├── alert_rules.yml           # Alert rules (Phase 13.2)
│   └── grafana/
│       ├── provisioning/
│       │   └── datasources/
│       │       └── prometheus.yml  # Auto-provision Prometheus (Phase 13.3)
│       └── dashboards/           # Grafana dashboard JSON files
│
├── logs/                         # Application logs (Phase 15.2)
├── backups/                      # Local backup rotation (Phase 14)
├── resumes/                      # Per-application tailored resumes (gitignored)
├── resume_versions/              # Base resume evolution history
├── tracker/                      # Excel tracker output
├── templates/                    # LaTeX resume templates
├── data/                         # Static data (companies list, action verbs)
│
├── JobPilot_PRD_v1.1_Definitive.md
├── JobPilot_TechStack_Analysis.md
├── JobPilot_TechStack_v2_Laptop_Gemini.md
└── README.md
```

---

## What's Next: Development Build Sequence

With the environment fully set up, begin development following the PRD's 10-weekend build sequence:

| Weekend | Deliverable                                    | Key Files                                         |
| :-----: | ---------------------------------------------- | ------------------------------------------------- |
|  **1**  | Scout Agent + Telegram notifications           | `agents/scout.py`, `integrations/telegram_bot.py` |
|  **2**  | Resume Agent + XeLaTeX compilation             | `agents/resume.py`, `templates/base_resume.tex`   |
|  **3**  | Tracker Agent + daily reporting                | `agents/tracker.py`, `integrations/sheets.py`     |
|  **4**  | Application Agent + Gmail OAuth                | `agents/application.py`, `integrations/gmail.py`  |
|  **5**  | Email Intelligence Agent                       | `agents/email_intelligence.py`                    |
|  **6**  | Outreach Agent                                 | `agents/outreach.py`, `integrations/apollo.py`    |
|  **7**  | MCP servers + LangGraph orchestration          | `mcp_servers/`, `orchestrator/graph.py`           |
|  **8**  | Adaptive Resume Engine (4-agent tournament)    | `agents/adaptive_resume/`                         |
|  **9**  | Follow-up Agent + Prometheus metrics + Grafana | `agents/follow_up.py`, `monitoring/metrics.py`    |
| **10**  | Docker deployment + integration testing        | `Dockerfile`, `docker-compose.yml` (final)        |

> [!IMPORTANT]
> **Before starting Weekend 1, you need:**
> 1. Your custom LaTeX `.tex` resume file → place in `templates/base_resume.tex`
> 2. The 200-company list → place in `data/companies.yaml`
>
> The **top 30 tier-1 companies** will be auto-selected from your 200-company list by a setup script (`scripts/rank_companies.py`). The script uses your cascading LLM client to score each company on:
> - **Hiring velocity** — how frequently they post roles matching your target titles
> - **Role relevance** — historical match quality for Backend/Platform/Infra roles
> - **Company tier** — engineering reputation, scale, and compensation band
> - **ATS type** — companies with API-friendly ATS (Lever, Greenhouse) ranked higher for automation ease
>
> The top 30 are flagged as `tier: 1` in `data/companies.yaml` and polled every 10 minutes; the remaining 170 are `tier: 2` (polled every 4 hours). You can override any selection by editing the YAML file directly.
