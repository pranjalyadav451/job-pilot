# JobPilot — Product Requirements Document v2.0

**Author:** Pranjal Yadav
**Date:** 2 April 2026
**Status:** Draft — pending approval
**Supersedes:** PRD v1.1 Definitive (scope reduction)

---

## 1. Overview

JobPilot is a focused, autonomous job discovery and resume delivery pipeline. It monitors LinkedIn, Naukri.com, and 200 company career pages for relevant postings; scores them against the user's resume; notifies via Telegram with the tailored resume attached and the apply link included; logs approved applications to a master Google Sheet; and saves the generated resume to an organised local directory.

The system runs natively as isolated systemd services inside a Python virtualenv. Containerisation is deferred to a later production phase.

---

## 2. Goals and success criteria

### 2.1 Primary goals

- Discover relevant job postings from LinkedIn, Naukri, and 200 company career pages.
- Score each posting for relevance against the user's resume; surface only postings with score ≥ 75.
- Notify the user on Telegram with the parsed JD summary, relevance score, ATS score, apply link, and the tailored resume as a PDF attachment.
- On user approval, log the application to Google Sheets and save the resume to the correct local directory.
- On rejection, log the skip decision silently; no further action.

### 2.2 Success metrics

| Metric                                      | Target                                        |
| ------------------------------------------- | --------------------------------------------- |
| Time from posting published → user notified | < 15 min (tier-1 companies), < 4 hrs (tier-2) |
| Relevance scoring false-positive rate       | < 15%                                         |
| Resume generation time (on demand)          | < 90 seconds                                  |
| ATS keyword match rate                      | > 80% of JD keywords                          |
| System uptime                               | > 99% (supervised via systemd)                |
| Monthly LLM cost                            | < $5 realistic; $0 path via Ollama            |

---

## 3. Decisions (locked)

| Decision                  | Choice                                                                                                                                                                                                                                                         |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Job sources               | LinkedIn, Naukri.com, 200 company career pages                                                                                                                                                                                                                 |
| Source toggle             | Each source independently enabled/disabled via `config.yaml`                                                                                                                                                                                                   |
| Notification channel      | Telegram Bot API                                                                                                                                                                                                                                               |
| Resume format             | LaTeX (user provides `.tex` base)                                                                                                                                                                                                                              |
| Resume generation trigger | Configurable: `on_notify` (pre-generated) or `on_approve` (on demand)                                                                                                                                                                                          |
| Resume attachment         | PDF file attached directly to Telegram message                                                                                                                                                                                                                 |
| Application submission    | Manual — apply link provided; no auto-submission                                                                                                                                                                                                               |
| Tracker                   | Google Sheets — one master spreadsheet, one tab per day                                                                                                                                                                                                        |
| Relevance threshold       | ≥ 75 (configurable)                                                                                                                                                                                                                                            |
| LLM strategy              | Cascading: DeepSeek V3 → Gemini Flash → OpenAI GPT-4o-mini → Ollama                                                                                                                                                                                            |
| Base resume               | User-provided `.tex`; stays fixed unless manually replaced                                                                                                                                                                                                     |
| Database                  | PostgreSQL (native install)                                                                                                                                                                                                                                    |
| Deployment                | Native Python virtualenv; each process runs as a systemd service                                                                                                                                                                                               |
| Containerisation          | Deferred — not in scope for v2.0                                                                                                                                                                                                                               |
| LinkedIn / Naukri access  | No free official APIs exist. Primary: JSearch API (RapidAPI, 200 req/month free — aggregates Google for Jobs, covers both platforms). Fallback: Playwright guest-endpoint scraping (unauthenticated, rate-limited, no account login). Configurable per source. |
| Career page automation    | Playwright browser automation for all 200 companies                                                                                                                                                                                                            |

---

## 4. System architecture

### 4.1 Component overview

```
┌──────────────────────────────────────────────────────┐
│                   APScheduler                        │
│   (runs inside main process, supervised by systemd)  │
└────────┬─────────────────┬────────────────┬──────────┘
         │ every 10 min    │ every 10 min   │ tiered: 10min / 4hrs
         ▼                 ▼                ▼
  ┌─────────────┐   ┌────────────┐   ┌──────────────────┐
  │   LinkedIn  │   │  Naukri    │   │  Career Pages    │
  │   Source    │   │  Source    │   │  (200 companies) │
  └──────┬──────┘   └─────┬──────┘   └────────┬─────────┘
         └────────────────┴──────────────────┘
                          │ raw postings
                          ▼
                 ┌─────────────────┐
                 │  Scout Agent    │
                 │  - title filter │
                 │  - LLM scoring  │
                 │  - geo filter   │
                 │  - dedup        │
                 └────────┬────────┘
                          │ postings with score ≥ 75
                          ▼
              ┌──────────────────────────┐
              │  Resume Agent (optional) │  ← if mode = on_notify
              │  LaTeX tailoring + PDF   │
              └────────────┬─────────────┘
                           │
                           ▼
                 ┌─────────────────┐
                 │  Telegram Bot   │
                 │  (Notifier)     │
                 │  + PDF attach   │
                 └────────┬────────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
         [Accept ✓]              [Ignore ✗]
              │                       │
              ▼                       ▼
  ┌─────────────────────┐     ┌───────────────┐
  │  Resume Agent       │     │  Log: Skipped │
  │  (if on_approve)    │     │  (PostgreSQL) │
  └────────┬────────────┘     └───────────────┘
           │ PDF + .tex saved to day directory
           ▼
  ┌─────────────────────┐
  │  Google Sheets      │
  │  Tracker Agent      │
  │  (daily tab update) │
  └─────────────────────┘
```

### 4.2 Agent inventory

| Agent                 | Function                                                                             | Trigger                               | Output                   |
| --------------------- | ------------------------------------------------------------------------------------ | ------------------------------------- | ------------------------ |
| **Scout**             | Polls all sources; title filter → LLM score → geo filter → dedup                     | APScheduler (per source tier)         | Scored postings ≥ 75     |
| **Resume**            | Rewrites LaTeX bullets; compiles PDF; calculates ATS score                           | Configurable: on notify or on approve | Tailored `.tex` + `.pdf` |
| **Telegram Notifier** | Sends job notification with resume PDF + apply link; handles Accept/Ignore callbacks | Scout output                          | User decision            |
| **Tracker**           | Updates Google Sheets (daily tab); saves resume to directory                         | On Accept                             | Sheet row + saved resume |

### 4.3 Cascading LLM strategy

All LLM calls go through a single cascading client. On failure (rate limit, timeout, HTTP error), each request falls to the next tier automatically.

| Tier         | Provider                               | Cost     |
| ------------ | -------------------------------------- | -------- |
| 1            | DeepSeek V3 (50M tokens/day free)      | $0       |
| 2            | Gemini 2.0 Flash (free tier, 15 RPM)   | $0–1/mo  |
| 3            | OpenAI GPT-4o-mini                     | ~$1–2/mo |
| 4 (terminal) | Ollama local (Llama 3.2 / Qwen 2.5 7B) | $0       |

```python
class CascadingLLM:
    tiers = [
        {"provider": "deepseek", "model": "deepseek-chat",    "timeout": 30},
        {"provider": "gemini",   "model": "gemini-2.0-flash", "timeout": 20},
        {"provider": "openai",   "model": "gpt-4o-mini",      "timeout": 20},
        {"provider": "ollama",   "model": "llama3.2",         "timeout": 60},
    ]

    async def complete(self, messages, task_type="general"):
        for tier in self.tiers:
            try:
                return await call_provider(tier, messages)
            except (RateLimitError, TimeoutError, HTTPError):
                log.warning(f"{tier['provider']} failed, cascading...")
        raise AllProvidersFailedError()
```

### 4.4 Data persistence

| Store                | Technology                 | Purpose                                                |
| -------------------- | -------------------------- | ------------------------------------------------------ |
| Job postings + state | PostgreSQL                 | Posting metadata, dedup hashes, user decisions         |
| Seen-posting dedup   | PostgreSQL (SHA-256 index) | Hash of `normalise(title+company+url)` — O(1) lookup   |
| Generated resumes    | Filesystem                 | `resumes/YYYY-MM-DD/{N}_{company}_{role}.pdf` + `.tex` |
| Configuration        | `config.yaml`              | All tuneable parameters                                |
| Secrets              | `.env`                     | API keys, Telegram bot token, Google credentials       |

### 4.5 Technology stack

| Component          | Technology                                                     |
| ------------------ | -------------------------------------------------------------- |
| Language           | Python 3.12                                                    |
| Scheduling         | APScheduler (in-process cron)                                  |
| Web framework      | FastAPI (Telegram webhook receiver + health endpoint)          |
| Browser automation | Playwright (async) for career pages + LinkedIn/Naukri fallback |
| LaTeX compilation  | `pdflatex` via `subprocess` (TeX Live, native install)         |
| Telegram           | `python-telegram-bot`                                          |
| Google Sheets      | `google-api-python-client` (Sheets API v4)                     |
| Database           | PostgreSQL (native install)                                    |
| Orchestration      | LangGraph (agent state machine)                                |
| Deployment         | Python virtualenv + systemd services                           |

---

## 5. Scout Agent

### 5.1 Sources and polling schedule

| Source                                | Default frequency | Method                             | Config key                 |
| ------------------------------------- | ----------------- | ---------------------------------- | -------------------------- |
| LinkedIn + Naukri (via JSearch)       | Every 10 min      | JSearch API (RapidAPI) — primary   | `sources.jsearch.enabled`  |
| LinkedIn (direct fallback)            | Every 10 min      | Playwright guest-endpoint scraping | `sources.linkedin.enabled` |
| Naukri (direct fallback)              | Every 10 min      | Playwright guest-endpoint scraping | `sources.naukri.enabled`   |
| Career pages — Tier 1 (top 30)        | Every 10 min      | Playwright + RSS auto-detect       | built-in tier assignment   |
| Career pages — Tier 2 (remaining 170) | Every 4 hrs       | Playwright + RSS auto-detect       | built-in tier assignment   |

Each source can be independently toggled in `config.yaml`:

```yaml
sources:
  jsearch:
    enabled: true           # primary LinkedIn + Naukri aggregator
    api_key: ""             # RapidAPI key
    requests_per_month: 200 # free tier limit
  linkedin:
    enabled: true           # direct Playwright fallback
    method: playwright      # playwright only — no official API
  naukri:
    enabled: true           # direct Playwright fallback
    method: playwright      # playwright only — no official API
  career_pages:
    enabled: true
    company_list: ./data/companies.yaml
```

### 5.2 LinkedIn and Naukri sourcing strategy

**Findings (researched April 2026):**

| Platform   | Official free API | Reality                                                                                  |
| ---------- | ----------------- | ---------------------------------------------------------------------------------------- |
| LinkedIn   | ❌ None            | Partner-only, enterprise-only access. Automating a personal account risks permanent ban. |
| Naukri.com | ❌ None            | Enterprise/ATS partnerships only. No public developer API.                               |

**Chosen approach — two-tier:**

**Tier 1 — JSearch API (primary, recommended):**
- JSearch (via RapidAPI) aggregates Google for Jobs, which indexes LinkedIn, Naukri, and hundreds of other job boards.
- Free tier: ~200 requests/month. Each request returns up to 10 results with full JD text, apply URL, company, location.
- Covers both LinkedIn and Naukri postings without directly scraping either platform.
- Risk: slight data delay (Google indexing lag, typically minutes to hours). Acceptable for Tier 2 / general listings. For Tier 1 companies, career-page scraping provides fresher data.
- Config key: `sources.jsearch.enabled`

**Tier 2 — Direct Playwright scraping (fallback when JSearch quota exhausted):**
- LinkedIn: uses unauthenticated guest job-search endpoints (`/jobs-guest/`). No account login. Rate-limited to respectful intervals (30–60s between requests). Configurable on/off.
- Naukri: Playwright-based scraping of public search results. No account login. Rate-limited.
- **Risk acknowledged:** Both platforms prohibit automated scraping in their ToS. Account bans do not apply (no login used). IP throttling is the main risk — mitigated by rate limiting and the fact that scrapers only run at 10-minute intervals, not continuously.
- Fallback is disabled by default; user enables via config after accepting the risk.

```yaml
sources:
  linkedin:
    enabled: false   # enable manually, scraping fallback only
  naukri:
    enabled: false   # enable manually, scraping fallback only
```

### 5.3 Career page scraper strategy

1. **RSS/Atom auto-detection** — on first run, probe each company URL for common feed patterns (`/careers/feed`, `/jobs.rss`). Companies with RSS feeds are subscribed passively; no Playwright needed.
2. **ATS-hosted pages** (Lever, Greenhouse, Ashby, Workday) — one generic Playwright scraper per ATS platform covers multiple companies. Lever exposes a free JSON API at `{company}.lever.co/api/postings`.
3. **Custom career pages** — per-company Playwright scripts. The `companies.yaml` file is annotated with ATS provider per entry.

### 5.3 Relevance scoring pipeline

**Stage 1 — Title pre-filter (zero LLM cost)**

Fuzzy match against hardcoded title list (token overlap ≥ 60%). Postings failing this are discarded immediately.

```yaml
target_titles:
  - "Backend Engineer"
  - "Senior Backend Engineer"
  - "Platform Engineer"
  - "Senior Platform Engineer"
  - "AI Platform Engineer"
  - "Infrastructure Engineer"
  - "Site Reliability Engineer"
  - "SRE"
  - "Software Engineer - Backend"
  - "Software Engineer - Platform"
  - "Software Engineer - Infrastructure"
  - "DevOps Engineer"
  - "Cloud Engineer"
  - "ML Infrastructure Engineer"
  - "AI Infrastructure Engineer"
  - "Software Engineer"
```

**Stage 2 — JD content scoring (cascading LLM call)**

Input: full JD text + user resume text. Output:

```json
{
  "relevance_score": 82,
  "matching_skills": ["Kubernetes", "Docker", "CI/CD"],
  "missing_skills": ["Kafka"],
  "experience_match": "3 YoE required — exact match",
  "location_match": "Bangalore — matches",
  "reasoning": "Strong backend match. Kafka listed as preferred, not required.",
  "red_flags": []
}
```

**Stage 3 — Geographic filter**

| Location                                                   | Decision |
| ---------------------------------------------------------- | -------- |
| Bangalore / Bengaluru / Karnataka                          | Accept   |
| Remote (no country restriction)                            | Accept   |
| International with visa sponsorship / relocation mentioned | Accept   |
| International, no visa mention                             | Reject   |
| Other Indian city (non-remote)                             | Reject   |

**Stage 4 — Deduplication**

SHA-256 hash of `normalise(title) + normalise(company) + normalise(url)` checked against PostgreSQL index.

**Stage 5 — Threshold gate**

`relevance_score >= 75` (configurable via `scoring.relevance_threshold` in `config.yaml`) → pass to Resume Agent or Telegram Notifier.

---

## 6. Resume Agent

### 6.1 Trigger modes

Controlled by `resume.generation_mode` in `config.yaml`:

| Mode         | Behaviour                                                                               |
| ------------ | --------------------------------------------------------------------------------------- |
| `on_notify`  | Resume generated automatically before Telegram notification; PDF attached immediately   |
| `on_approve` | Resume generated only after user taps Accept on Telegram; PDF sent as follow-up message |

### 6.2 Inputs

- **Base resume**: user-provided `.tex` file; path set via `resume.base_tex_path` in `config.yaml`. Stays fixed unless manually replaced.
- **JD text**: extracted by Scout Agent.
- **Harvard action verb list**: 185+ verbs categorised by domain, stored as `data/harvard_verbs.yaml`.

### 6.3 Processing pipeline

**Step 1 — JD keyword extraction (cascading LLM)**

```json
{
  "required_hard_skills": ["Kubernetes", "Docker", "Go or Python"],
  "preferred_hard_skills": ["Terraform", "Prometheus"],
  "soft_skills": ["cross-team collaboration"],
  "domain_terms": ["microservices", "observability"],
  "ats_phrases": ["infrastructure as code", "production incident response"]
}
```

**Step 2 — Resume parsing**

Regex-based extraction of `\section{}`, `\resumeSubheading{}`, `\resumeItem{}` from the user's template. Produces structured intermediate representation with line numbers for precise `.tex` modification.

**Step 3 — Bullet rewriting (cascading LLM)**

System prompt constraints (enforced at all tiers):
1. Never fabricate experience, skills, or metrics.
2. Replace weak verbs with Harvard action verbs.
3. Inject JD keywords only where factually accurate.
4. Quantify impact where inferable; flag with `[INFERRED]`.
5. Front-load the strongest action verb.
6. Keep bullets to 1–2 lines maximum.
7. Output valid LaTeX (escape `&`, `%`, `$`, `#`, `_`, `{`, `}`).

**Step 4 — Bullet reordering** — most JD-relevant bullets first within each role.

**Step 5 — Skills reordering** — JD-mentioned skills first.

**Step 6 — LaTeX compilation**

```python
def compile_latex(tex_path: str, output_dir: str) -> str:
    result = subprocess.run(
        ["pdflatex", "-interaction=nonstopmode", "-output-directory", output_dir, tex_path],
        capture_output=True, text=True, timeout=30
    )
    if result.returncode != 0:
        errors = parse_latex_errors(result.stdout)
        if fixable(errors):
            fix_latex_source(tex_path, errors)
            return compile_latex(tex_path, output_dir)  # retry once
        raise LaTeXCompilationError(errors)
    return os.path.join(output_dir, tex_path.replace(".tex", ".pdf"))
```

**Step 7 — ATS score calculation**

`(JD keywords found in resume / total JD keywords) * 100`. Target: > 80%.

### 6.4 Output storage

Resumes are stored in day-based directories, numbered to match the row order in Google Sheets:

```
resumes/
└── 2026-04-02/
    ├── 01_razorpay_senior_backend_engineer.pdf
    ├── 01_razorpay_senior_backend_engineer.tex
    ├── 02_google_sre.pdf
    ├── 02_google_sre.tex
    └── ...
```

Configuration:

```yaml
resume:
  base_tex_path: ./data/base_resume.tex
  output_dir: ./resumes
  generation_mode: on_approve   # on_notify | on_approve
```

---

## 7. Telegram Notifier

### 7.1 Notification format

```
🔍 New match: Senior Backend Engineer
🏢 Razorpay · Bangalore
📊 Relevance: 82/100 · ATS: 84%

✅ Matching: Kubernetes, Docker, CI/CD, distributed systems
⚠️  Missing: Kafka (preferred, not required)
📍 Location: Bangalore — direct match

🔗 https://razorpay.com/careers/senior-backend-engineer

[Accept ✓]    [Ignore ✗]    [Details 📄]
```

When `generation_mode = on_notify`, the PDF is attached to this message.
When `generation_mode = on_approve`, the PDF is sent as a follow-up after Accept is tapped.

"Details" sends the full JD scoring breakdown. "Ignore" logs the decision silently.

### 7.2 Callback handler

```python
async def handle_callback(update: Update, context):
    query = update.callback_query
    action, job_id = query.data.split(":")

    if action == "accept":
        if config.resume.generation_mode == "on_approve":
            await trigger_resume_agent(job_id)          # generates + sends PDF
        await tracker_agent.log_accepted(job_id)        # Google Sheets + local save
        await query.answer("Logged. Resume saved.")

    elif action == "ignore":
        await mark_ignored(job_id)
        await query.answer("Ignored.")

    elif action == "detail":
        await send_full_analysis(query.message.chat_id, job_id)
        await query.answer()
```

### 7.3 Resume delivery (on_approve mode)

After the user taps Accept, the bot sends:

```
✅ Accepted — Resume generated

🏢 Razorpay — Senior Backend Engineer
📊 ATS score: 84%
📎 [razorpay_senior_backend_engineer_2026-04-02.pdf]

Saved to: resumes/2026-04-02/01_razorpay_senior_backend_engineer.pdf
Logged to Google Sheets → tab: 2026-04-02, row: 1
```

---

## 8. Google Sheets Tracker

### 8.1 Spreadsheet structure

- One **master spreadsheet** (user provides the spreadsheet ID in `config.yaml`).
- One **new tab created per day**, named `YYYY-MM-DD`.
- Tab is auto-created at the first log of the day.
- Row numbering in the sheet matches the resume file numbering in the local directory.

### 8.2 Columns (per row)

| #   | Column                   | Source                     |
| --- | ------------------------ | -------------------------- |
| 1   | Date Discovered          | Scout                      |
| 2   | Company                  | Scout                      |
| 3   | Role Title               | Scout                      |
| 4   | Posting URL (apply link) | Scout                      |
| 5   | Location                 | Scout                      |
| 6   | Relevance Score          | Scout                      |
| 7   | Matching Skills          | Scout                      |
| 8   | Missing Skills           | Scout                      |
| 9   | ATS Score                | Resume Agent               |
| 10  | Resume File Path         | Resume Agent               |
| 11  | User Decision            | Telegram (Accept / Ignore) |
| 12  | Decision Timestamp       | Telegram                   |

### 8.3 Authentication

Google Sheets API v4 via `google-api-python-client`. Service account credentials stored in `credentials/google_service_account.json` (gitignored, 600 permissions).

```yaml
tracker:
  spreadsheet_id: "your-google-sheet-id-here"
  credentials_path: ./credentials/google_service_account.json
```

---

## 9. Deployment

### 9.1 Isolation strategy

All components run natively in a dedicated Python virtualenv (`jobpilot-env`). Each long-running process is managed as a **systemd user service** so it:
- Starts automatically on boot.
- Restarts on crash.
- Logs to `journald` (queryable with `journalctl --user -u jobpilot-*`).
- Has no impact on other system Python environments.

### 9.2 Services

| Service unit                | Process                                            | Restart policy               |
| --------------------------- | -------------------------------------------------- | ---------------------------- |
| `jobpilot-app.service`      | FastAPI app + APScheduler (Scout, Resume, Tracker) | `on-failure`, max 3 restarts |
| `jobpilot-telegram.service` | Telegram webhook/polling receiver                  | `on-failure`, max 3 restarts |

Both services bind to the same virtualenv and share the `.env` file.

### 9.3 Directory layout

```
job-pilot/
├── config.yaml                  # all tuneable parameters
├── .env                         # secrets (gitignored)
├── data/
│   ├── companies.yaml           # 200-company list with ATS provider annotations
│   ├── base_resume.tex          # user's base resume (gitignored)
│   └── harvard_verbs.yaml       # action verb reference
├── credentials/
│   └── google_service_account.json  # gitignored, chmod 600
├── resumes/
│   └── YYYY-MM-DD/              # auto-created per day
├── src/
│   ├── agents/
│   │   ├── scout.py
│   │   ├── resume.py
│   │   ├── notifier.py
│   │   └── tracker.py
│   ├── scrapers/
│   │   ├── linkedin.py
│   │   ├── naukri.py
│   │   └── career_pages/        # per-ATS scrapers + per-company overrides
│   ├── llm/
│   │   └── cascading_client.py
│   ├── db/
│   │   └── models.py
│   ├── scheduler.py
│   └── main.py                  # FastAPI entry point
├── systemd/
│   ├── jobpilot-app.service
│   └── jobpilot-telegram.service
└── requirements.txt
```

### 9.4 Native dependencies

| Dependency          | Install method                                                  |
| ------------------- | --------------------------------------------------------------- |
| Python 3.12         | System package or `pyenv`                                       |
| PostgreSQL          | System package (`postgresql`)                                   |
| TeX Live            | System package (`texlive-latex-base texlive-fonts-recommended`) |
| Playwright browsers | `playwright install chromium` (within virtualenv)               |
| Ollama              | Official installer (`curl -fsSL https://ollama.com/install.sh   | sh`) |

---

## 10. Configuration reference (`config.yaml`)

```yaml
# Sources
sources:
  jsearch:
    enabled: true             # primary aggregator (LinkedIn + Naukri via Google for Jobs)
    api_key: ""               # RapidAPI key — set in .env as JSEARCH_API_KEY
    requests_per_month: 200   # free tier limit; system halts JSearch and activates fallbacks at 190
  linkedin:
    enabled: false            # direct Playwright fallback; enable manually after accepting ToS risk
  naukri:
    enabled: false            # direct Playwright fallback; enable manually after accepting ToS risk
  career_pages:
    enabled: true
    company_list: ./data/companies.yaml

# Scoring
scoring:
  relevance_threshold: 75
  daily_notification_cap: 15  # max Telegram notifications per day

# Resume
resume:
  base_tex_path: ./data/base_resume.tex
  output_dir: ./resumes
  generation_mode: on_approve  # on_notify | on_approve

# Tracker
tracker:
  spreadsheet_id: ""           # provide Google Sheet ID here
  credentials_path: ./credentials/google_service_account.json

# LLM cascade
llm:
  tiers:
    - provider: deepseek
      model: deepseek-chat
      timeout: 30
    - provider: gemini
      model: gemini-2.0-flash
      timeout: 20
    - provider: openai
      model: gpt-4o-mini
      timeout: 20
    - provider: ollama
      model: llama3.2
      timeout: 60

# Database
database:
  url: postgresql://jobpilot:password@localhost:5432/jobpilot

# Telegram
telegram:
  bot_token: ""               # set in .env as TELEGRAM_BOT_TOKEN
  chat_id: ""                 # set in .env as TELEGRAM_CHAT_ID
```

---

## 11. Out of scope (v2.0)

The following features from PRD v1.1 are explicitly deferred:

| Feature                                          | Status                                      |
| ------------------------------------------------ | ------------------------------------------- |
| Application Agent (ATS API / email submission)   | Deferred                                    |
| Outreach Agent (recruiter discovery + messaging) | Deferred                                    |
| Email Intelligence Agent (Gmail classification)  | Deferred                                    |
| Follow-up Automation Agent                       | Deferred                                    |
| Adaptive Resume Engine (tournament consensus)    | Deferred                                    |
| Prometheus + Grafana monitoring                  | Deferred                                    |
| MCP server architecture                          | Deferred                                    |
| Docker / docker-compose                          | Deferred (containerise after MVP is stable) |

---

## 12. Open items

| #   | Item                                                                    | Status                                                                                                                                    | Owner    |
| --- | ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| 1   | Provide 200-company list (`companies.yaml`) annotated with ATS provider | ⏳ Pending                                                                                                                                 | User     |
| 2   | Provide base resume (`base_resume.tex`)                                 | ⏳ Pending                                                                                                                                 | User     |
| 3   | Provide Google Sheet ID                                                 | ⏳ Pending                                                                                                                                 | User     |
| 4   | LinkedIn API access                                                     | ✅ Resolved — no free official API; JSearch (RapidAPI) as primary aggregator, Playwright guest scraping as opt-in fallback                 | Research |
| 5   | Naukri API access                                                       | ✅ Resolved — no free official API; covered by JSearch aggregator (Google for Jobs indexes Naukri), Playwright scraping as opt-in fallback | Research |
