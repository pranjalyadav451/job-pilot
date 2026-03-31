# JobPilot — Product Requirements Document v1.1 (Definitive)

**Author:** Pranjal Yadav
**Date:** 29 March 2026
**Status:** Approved for development
**Classification:** Personal project / Portfolio capstone
**Supersedes:** All previous PRD versions (v1.0, v1.1 Addendum, v1.1 Consolidated, and the pasted v1.1 draft). This is the single source of truth.

---

## Contradiction resolution log

The following contradictions existed across prior drafts. Each is resolved here with the chosen option and rationale.

| # | Contradiction | Resolution | Rationale |
|---|--------------|------------|-----------|
| 1 | Gmail sending: `gmail.send` scope vs. SMTP | `gmail.send` scope via Gmail API | Single auth mechanism; no SMTP config needed |
| 2 | Follow-ups per application: 1 vs. 2 | 2 (day 7 + day 14, second is final) | Maximises response probability without being excessive |
| 3 | Adaptive Resume agents: 3 vs. 4 | 4 (Keyword, Narrative, Structure, Market) | Market Agent catches emerging JD trends proactively |
| 4 | Deliberation: Mediator Agent vs. agents self-merge | Dedicated Mediator Agent | Cleaner separation of concerns |
| 5 | Monthly cost target: <$5 vs. <$10 | Realistic pricing with free alternatives documented at every decision point | Honest cost projection; free-tier paths always available |
| 6 | Prometheus retention: 7 days vs. 90 days | 90 days | Sufficient history for weekly/monthly trend analysis |
| 7 | Auto-revert trigger: absolute floor vs. relative | Acceptance rate < 10% for 3 consecutive cycles (absolute floor) | Simpler to implement and reason about |

---

## 1. Executive summary

JobPilot is a self-improving, multi-agent autonomous job application pipeline comprising nine agents orchestrated by a LangGraph state machine. It monitors LinkedIn, Naukri.com, and 200 company career pages for relevant postings; scores them against the user's resume; notifies via Telegram with inline Apply/Skip controls; generates ATS-optimised LaTeX resumes per posting using Harvard action verbs; submits applications via ATS APIs, email, or manual fallback; identifies recruiters at target companies via Apollo.io and LinkedIn; drafts dual-variant outreach messages; monitors Gmail for application responses and auto-classifies them; drafts follow-up emails at day 7 and day 14; maintains a centralised auto-populated tracking spreadsheet; evolves its own base resume every 12 hours through a four-agent tournament-deliberation consensus mechanism that learns from actual acceptance/rejection data; and exposes pipeline health and conversion funnel metrics via Prometheus and Grafana.

The system operates on a cascading LLM cost strategy targeting near-zero monthly operating cost: free-tier Chinese models (DeepSeek V3, Qwen 2.5) as primary, lower-cost commercial fallback (Gemini Flash, GPT-4o-mini) on rate limit/timeout, local Ollama as the terminal fallback, and Claude Sonnet 4.5 reserved exclusively for quality-critical resume rewriting and deliberation mediation. Every irreversible action (application submission, outreach message sending, follow-up email dispatch) requires explicit human approval via Telegram.

---

## 2. Goals and success criteria

### 2.1 Primary goals

- Reduce daily job search overhead from 2–3 hours to under 15 minutes (review + approve only).
- Apply to 10–15 high-relevance positions per day with individually tailored resumes.
- Achieve first-mover advantage on new postings (notification within 10 minutes of publication for tier-1 companies).
- Maintain a complete, auto-populated pipeline tracker with zero manual data entry.
- Continuously improve the base resume based on actual acceptance/rejection data.

### 2.2 Success metrics

| Metric | Target | Free alternative |
|--------|--------|-----------------|
| Time from posting published → user notified | < 15 minutes (tier-1) | Same (no cost dependency) |
| Time from user approval → resume generated | < 90 seconds | ~3–5 min with Ollama on CPU |
| ATS keyword match rate (tailored vs. base) | > 80% of JD keywords | Same |
| False positive rate (irrelevant notifications) | < 15% | May be ~20% with Ollama scoring |
| Daily recruiter outreach messages (after approval) | 5–10 | Same |
| System uptime | > 99% | Same (local Docker) |
| Monthly operating cost | < $10 realistic | $0 achievable with Ollama-only |
| Email classification accuracy | > 95% | ~88–92% with local models |
| Base resume acceptance rate improvement | Measurable upward trend week-over-week | Same |
| Follow-up coverage | 100% of unanswered applications at day 7 and day 14 | Same |
| Funnel visibility in Grafana | Real-time, < 30s lag | Same |

---

## 3. User decisions (locked)

| Decision point | Choice |
|----------------|--------|
| Job boards | LinkedIn, Naukri.com, 200 company career pages (user-provided list) |
| Notification channel | Telegram Bot API (free) |
| LinkedIn automation | Hybrid: Apollo.io enrichment first, Playwright browser automation with human approval as fallback |
| LaTeX template | Custom (user will provide `.tex` source) |
| Geographic scope | Bangalore only + international remote + international with visa sponsorship |
| Daily notification volume | 10–15 (moderate filtering, threshold ~75–80) |
| Role identification | Hardcoded title list for pre-filtering + LLM-based JD content matching for scoring |
| Outreach message style | Generate 2 variants per recruiter (specific + generic) for user selection |
| Company career page list | User has 200-company list ready |
| LLM cost strategy | Cascading: free Chinese LLMs → cheaper commercial fallback → Ollama local |
| Career page polling | Tiered frequency + auto-detect RSS/webhook where available |
| Email provider | Gmail (personal), full auto-classification via LLM |
| Email sending | Gmail API (`gmail.send` scope) — single auth mechanism |
| Acceptance definition | Any positive response (interview invite, recruiter reply) |
| Resume consensus mechanism | Tournament with 4 agents; deliberation via dedicated Mediator Agent only if top scores within 5 points |
| Resume versioning | Full git-like history with weekly diff reports |
| Analytics data source | Prometheus (90-day retention) + Grafana |
| Follow-ups per application | 2 (day 7 + day 14, second is final) |
| Monthly budget | < $10 realistic; $0 path always available via Ollama |

---

## 4. System architecture

### 4.1 Agent inventory

Nine agents orchestrated by a LangGraph state machine. Each operates as an independent graph node with defined input/output contracts. The orchestrator manages state transitions, retry logic, and human-in-the-loop checkpoints.

```
                    ┌─────────────┐
                    │  Scheduler   │
                    │ (APScheduler)│
                    └──────┬──────┘
                           │ triggers every 10min / 4-6hrs / 12hrs / daily
                           ▼
                    ┌──────────────┐
                    │ Scout Agent  │──── LinkedIn API / Naukri / Career Pages
                    │ (Discovery)  │──── RSS feed listener
                    └──────┬──────┘
                           │ scored postings (relevance ≥ 75)
                           ▼
                    ┌──────────────┐
                    │  Telegram    │◄──── User: "Apply" / "Skip"
                    │  Notifier    │
                    └──────┬──────┘
                           │ approved postings only
                           ▼
                    ┌──────────────┐     ┌─────────────────────┐
                    │ Resume Agent │◄────│ Adaptive Resume     │
                    │ (Tailoring)  │     │ Engine (every 12hrs) │
                    └──────┬──────┘     │ 4-agent Tournament   │
                           │            │ + Mediator            │
                           │            └─────────────────────┘
                           │ compiled PDF + .tex source        ▲
                           ▼                                    │
                    ┌──────────────┐                    ┌───────┴──────┐
                    │ Application  │                    │   Email      │
                    │    Agent     │                    │ Intelligence │
                    └──────┬──────┘                    │   Agent      │
                           │ application submitted     │ (Gmail API)  │
                           ▼                           └───────┬──────┘
              ┌────────────┴────────────┐                      │
              ▼                         ▼                      │
    ┌──────────────┐          ┌──────────────┐                 │
    │  Outreach    │          │   Tracker    │◄────────────────┘
    │    Agent     │          │    Agent     │   email classification
    │ (Recruiters) │          │  (Excel/GS)  │   updates status
    └──────────────┘          └──────┬───────┘
                                     │
                              ┌──────┴───────┐
                              ▼              ▼
                    ┌──────────────┐  ┌──────────────┐
                    │  Follow-up   │  │  Prometheus   │
                    │    Agent     │  │  + Grafana    │
                    │ (Day 7 + 14) │  │  (Analytics)  │
                    └──────────────┘  └──────────────┘
```

| Agent | Function | Trigger | Output |
|-------|----------|---------|--------|
| Scout | Polls LinkedIn, Naukri, 200 career pages; scores JDs against resume | Every 10 min (T1) / 4 hrs (T2) | Scored postings → Telegram |
| Resume | Rewrites LaTeX bullets using Harvard verbs, compiles PDF | User approves posting | Tailored .tex + .pdf |
| Application | Submits via ATS API, email, or manual fallback | User confirms resume | Submission confirmation |
| Outreach | Finds recruiters via Apollo.io/Hunter.io; drafts 2 message variants | Post-application | Messages for Telegram approval |
| Tracker | Auto-populates Excel with all pipeline data (30 columns) | Every agent event | Updated spreadsheet |
| Email Intelligence | Classifies Gmail inbox; detects interviews, rejections, offers | Every 5 min | Tracker status updates + Telegram alerts |
| Follow-up | Drafts day-7 and day-14 follow-up emails for approval | Daily 10 AM IST | Follow-up drafts → Telegram |
| Adaptive Resume | 4-agent tournament + Mediator deliberation to evolve base resume | Every 12 hrs (06:00/18:00) | New base resume version |
| Analytics | Exposes Prometheus metrics; Grafana funnel dashboard | Continuous | 5 dashboards + 5 alert rules |

### 4.2 Cascading LLM routing strategy

Three-tier fallback chain. On failure (rate limit, timeout, HTTP error), each request cascades to the next tier. Claude Sonnet reserved exclusively for quality-critical tasks.

| Tier | Providers | Used for | Cost | Free alternative |
|------|-----------|----------|------|-----------------|
| 1 (primary) | DeepSeek V3 (50M tokens/day free), Qwen 2.5 | JD parsing, scoring, dedup, outreach drafts, email classification, tracker updates | $0 | This IS the free option |
| 2 (fallback) | Gemini 2.0 Flash (free tier), GPT-4o-mini | Tasks exceeding T1 rate limits during peak | ~$0–2/mo | Gemini free tier (15 RPM) |
| 3 (terminal) | Ollama local (Llama 3.2 8B or Qwen 2.5 7B) | All remote APIs unavailable; privacy-sensitive ops | $0 | This IS the free option |
| Premium | Claude Sonnet 4.5 | Resume bullet rewriting, deliberation mediation | ~$2–3/mo | Ollama (lower quality but functional) |

**$0/month configuration:** Set all tasks including resume rewriting to use the cascading client with Ollama as terminal. Quality will be lower on resume phrasing (~15–20% reduction in rewriting sophistication) but the system is fully functional.

**Implementation:**

```python
class CascadingLLM:
    tiers = [
        {"provider": "deepseek", "model": "deepseek-chat", "timeout": 30},
        {"provider": "qwen", "model": "qwen-plus", "timeout": 30},
        {"provider": "gemini", "model": "gemini-2.0-flash", "timeout": 20},
        {"provider": "openai", "model": "gpt-4o-mini", "timeout": 20},
        {"provider": "ollama", "model": "llama3.2", "timeout": 60},
    ]

    async def complete(self, messages, task_type="general"):
        for tier in self.tiers:
            try:
                response = await call_provider(tier, messages)
                return response
            except (RateLimitError, TimeoutError, HTTPError):
                log.warning(f"{tier['provider']} failed, cascading...")
                continue
        raise AllProvidersFailedError()
```

### 4.3 Data persistence layer

| Store | Technology | Purpose |
|-------|-----------|---------|
| Job postings + state | PostgreSQL | Posting metadata, application state, dedup hashes, recruiter contacts |
| Resume/JD embeddings | pgvector (PostgreSQL extension) | Cosine similarity for relevance scoring |
| Seen-posting dedup | PostgreSQL (SHA-256 hash index) | Hash of `title + company + normalised_url` — O(1) lookup |
| Generated resumes | Filesystem | `resumes/{company}/{role}_{date}.pdf` and `.tex` source |
| Base resume versions | Filesystem + PostgreSQL metadata | `resume_versions/v{N}_{timestamp}.tex` with tournament metadata |
| Email classification log | PostgreSQL | Every classified email with classification, confidence, matched application |
| Gmail OAuth tokens | Encrypted file in `credentials/` directory | OAuth 2.0 refresh tokens (gitignored) |
| Prometheus metrics | Prometheus TSDB | Time-series metrics with 90-day retention |
| Configuration | YAML | Target titles, company list, thresholds, API keys (via env vars) |
| Secrets | `.env` file | All API keys, Telegram bot token |

### 4.4 Technology stack

| Component | Technology | Justification | Free alternative |
|-----------|-----------|---------------|-----------------|
| Orchestration | LangGraph | Graph-based state machine, native HITL checkpoints | — (LangGraph is free/OSS) |
| Language | Python 3.12 | De facto AI ecosystem standard | — |
| Web framework | FastAPI | Health, `/metrics`, Telegram webhook receiver | — |
| Scheduling | APScheduler | In-process cron: 10min scout, 12hr evolution, 5min email, daily follow-up | — |
| Job board scraping | Playwright (async) | Headless browser for JS-rendered pages; `httpx` for static | — |
| RSS parsing | `feedparser` | Lightweight RSS/Atom feed consumption | — |
| LaTeX compilation | `pdflatex` via `subprocess` | Docker container with TeX Live | — |
| Telegram | `python-telegram-bot` | Official library, inline keyboard support | — |
| Gmail | `google-api-python-client` | OAuth 2.0; read, send, label scopes | — |
| LinkedIn enrichment | Apollo.io (free: 50/month) + Hunter.io (free: 25/month) | Recruiter contact discovery | LinkedIn scraping via Playwright |
| LinkedIn scraping | Playwright (fallback) | Read-only profile scraping when API credits exhausted | — |
| Spreadsheet | `openpyxl` (Excel) | Auto-populated tracker with conditional formatting | — |
| Monitoring | Prometheus + Grafana | Metrics export, dashboards, alerting | — (both free/OSS) |
| Containerisation | Docker + docker-compose | Single-command deployment | — |
| Deployment | Local or Hetzner VPS (CX22, ~€4.50/month) | 24/7, 4GB RAM, 2 vCPUs | Run locally ($0) |

---

## 5. Scout Agent — Job discovery and relevance scoring

### 5.1 Data sources and polling strategy

**Tiered polling schedule:**

| Tier | Companies | Poll frequency | Method |
|------|-----------|---------------|--------|
| Tier 1 (top 30) | User's highest-priority targets | Every 10 minutes | Direct scraping or RSS |
| Tier 2 (remaining 170) | Rest of the 200-company list | Every 4 hours | Direct scraping or RSS |
| LinkedIn | All matching postings | Every 10 minutes | LinkedIn Jobs API or scraping |
| Naukri.com | All matching postings | Every 10 minutes | Naukri API or scraping |

**RSS auto-detection:** On first run, the Scout Agent probes each career page for RSS/Atom feed availability (common patterns: `/careers/feed`, `/jobs.rss`; Lever/Greenhouse/Ashby pages expose feeds by default). Companies with RSS feeds are subscribed passively — no polling needed. Lever pages expose a JSON API at `{company}.lever.co/api/postings?limit=50`.

**Career page scraper categories:**
1. **ATS-hosted pages** (Lever, Greenhouse, Ashby, Workday): Predictable DOM structures. One generic scraper per ATS platform covers dozens of companies.
2. **Custom career pages**: Require per-company Playwright scripts. The 200-company list should be annotated with ATS provider during setup.
3. **Aggregator APIs** (LinkedIn, Naukri): Official APIs where available; scrape with respectful rate limits otherwise.

### 5.2 Relevance scoring pipeline

**Stage 1 — Title pre-filter (zero LLM cost):**

Hardcoded title list matched via fuzzy string matching (Levenshtein distance ≤ 3 or token overlap ≥ 60%):

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

Postings failing title matching are discarded without an LLM call.

**Stage 2 — JD content scoring (LLM call via cascading client):**

Input: full JD text + user resume text. Output: structured JSON.

```json
{
  "relevance_score": 82,
  "matching_skills": ["Kubernetes", "Docker", "REST API design", "distributed systems", "CI/CD"],
  "missing_skills": ["GraphQL", "Kafka"],
  "experience_match": "3 YoE required, candidate has 3 — exact match",
  "location_match": "Bangalore — matches",
  "visa_sponsorship": "not mentioned",
  "reasoning": "Strong backend systems match. JD emphasises K8s-native services, directly aligned with candidate's OKE experience. Missing Kafka experience is non-blocking (listed as preferred, not required).",
  "red_flags": []
}
```

**Stage 3 — Geographic filter:**

| Posting location | Filter logic |
|-----------------|-------------|
| Bangalore / Bengaluru / Karnataka | Accept unconditionally |
| "Remote" with no country restriction | Accept |
| International with "visa sponsorship" or "relocation assistance" mentioned | Accept |
| International, no visa mention | Reject |
| Other Indian city (non-remote) | Reject |

**Stage 4 — Deduplication:** SHA-256 hash of `normalise(title) + normalise(company) + normalise(url)` checked against PostgreSQL index. O(1) lookup.

**Stage 5 — Threshold gate:** If `relevance_score >= 75`, dispatch to Telegram. Estimated yield: 10–15 notifications/day.

### 5.3 Telegram notification format

```
🔍 New match: Senior Backend Engineer
🏢 Razorpay (Bangalore)
📊 Relevance: 82/100

✅ Matching: Kubernetes, Docker, REST APIs, distributed systems, CI/CD
⚠️ Missing: GraphQL (preferred, not required)
🗺️ Location: Bangalore — direct match

🔗 razorpay.com/careers/senior-backend-engineer

[Apply ✓]    [Skip ✗]    [Details 📄]
```

"Details" sends the full JD analysis. "Apply" triggers the Resume Agent. "Skip" logs the decision.

```python
from telegram import InlineKeyboardButton, InlineKeyboardMarkup, Update
from telegram.ext import CallbackQueryHandler

async def send_job_notification(bot, chat_id, job):
    keyboard = InlineKeyboardMarkup([
        [
            InlineKeyboardButton("Apply ✓", callback_data=f"apply:{job.id}"),
            InlineKeyboardButton("Skip ✗", callback_data=f"skip:{job.id}"),
            InlineKeyboardButton("Details 📄", callback_data=f"detail:{job.id}"),
        ]
    ])
    await bot.send_message(chat_id=chat_id, text=format_job_msg(job),
                           reply_markup=keyboard, parse_mode="Markdown")

async def handle_callback(update: Update, context):
    query = update.callback_query
    action, job_id = query.data.split(":")
    if action == "apply":
        await trigger_resume_agent(job_id)
        await query.answer("Resume generation started...")
    elif action == "skip":
        await mark_skipped(job_id)
        await query.answer("Skipped.")
    elif action == "detail":
        await send_full_analysis(query.message.chat_id, job_id)
        await query.answer()
```

---

## 6. Resume Agent — LaTeX tailoring and compilation

### 6.1 Input artefacts

1. **Base resume** — user's custom LaTeX `.tex` file (evolved by Adaptive Resume Engine every 12 hours).
2. **Job description** — raw text extracted by Scout Agent.
3. **Harvard action verb reference** — 185+ verbs categorised by skill domain, stored as YAML.

### 6.2 Processing pipeline

**Step 1 — JD keyword extraction (Tier 1 LLM):**

```json
{
  "required_hard_skills": ["Kubernetes", "Docker", "Go or Python", "CI/CD"],
  "preferred_hard_skills": ["Terraform", "Prometheus", "gRPC"],
  "soft_skills": ["cross-team collaboration", "mentoring"],
  "domain_terms": ["microservices", "service mesh", "observability"],
  "ats_phrases": ["Kubernetes-based deployment", "infrastructure as code", "production incident response"]
}
```

**Step 2 — Resume parsing:** Regex-based extraction of `\section{}`, `\resumeSubheading{}`, `\resumeItem{}` from the user's custom template. Structured intermediate representation with line numbers for precise `.tex` modification.

**Step 3 — Bullet rewriting (Claude Sonnet 4.5 or Tier 1 with quality check):**

System prompt constraints:
1. NEVER fabricate experience, skills, or metrics. Every claim must be derivable from the original bullet.
2. Replace weak verbs with Harvard action verbs from the provided list.
3. Inject JD keywords WHERE FACTUALLY ACCURATE only.
4. Quantify impact where inferable; flag with [INFERRED] for user review.
5. Front-load the most impactful verb. Begin with an action verb, not "Responsible for."
6. Keep each bullet to 1–2 lines maximum.
7. Output valid LaTeX with properly escaped special characters (&, %, $, #, _, {, }).

**Free alternative:** Route rewriting through Tier 1 cascade (DeepSeek/Qwen). Lower phrasing sophistication (~15–20%) but functional. Quality check via lightweight LLM-as-judge call; if quality score < 70, retry with next tier.

**Step 4 — Bullet reordering:** Most JD-relevant bullets first within each role.

**Step 5 — Skills section reordering:** JD-mentioned skills first, preserving all existing skills.

**Step 6 — LaTeX compilation and validation:**

```python
import subprocess

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

**Step 7 — ATS score estimation:** `(JD keywords found in resume / total JD keywords) * 100`. Target: > 80%.

**Step 8 — Storage:** `resumes/{company}/{role}_{date}.pdf` + `.tex` + `metadata.json` (JD URL, relevance score, ATS score, base resume version used, timestamp).

### 6.3 User review via Telegram

After compilation:
- PDF sent as file attachment
- Diff summary: "Modified 8/12 bullets. 2 metrics flagged as [INFERRED]. ATS score: 84%."
- Buttons: `[Confirm & Apply]` `[Edit Manually]` `[Regenerate]`

"Edit Manually" accepts reply with specific instructions (e.g., "remove the inferred metric in bullet 3"). Resume Agent applies edits and recompiles.

---

## 7. Application Agent — Submission

### 7.1 Submission methods (priority order)

| Method | Applicability | Implementation |
|--------|--------------|----------------|
| ATS API | Lever (`POST /postings/{id}/apply`), Greenhouse, Ashby | Direct HTTP POST with resume + form fields |
| Email | Postings with "apply via email" | Gmail API `gmail.send` with tailored resume attachment |
| LinkedIn Easy Apply | LinkedIn postings with Easy Apply | Playwright automation (implement last, highest risk) |
| Manual fallback | All other portals | Open URL in browser, download resume, Telegram reminder |

### 7.2 Confirmation gate

No submission without explicit Telegram approval:

```
📤 Ready to apply

🏢 Razorpay — Senior Backend Engineer
📎 Resume: razorpay_senior_backend_engineer_2026-03-27.pdf
📊 ATS score: 84%
🔧 Method: Lever API (automated)

[Confirm ✓]    [Cancel ✗]    [View Resume 📄]
```

### 7.3 Form field auto-population

```yaml
applicant_profile:
  name: "Pranjal Yadav"
  email: "pranjalyadav451@gmail.com"
  phone: "+91-8299715764"
  linkedin: "https://linkedin.com/in/yadavpranjal"
  github: "https://github.com/pranjalyadav451"
  current_company: "Oracle"
  current_title: "Senior Member of Technical Staff"
  years_of_experience: 3
  location: "Bengaluru, India"
  work_authorisation_india: true
  willing_to_relocate: true
  requires_visa_sponsorship: true
```

---

## 8. Outreach Agent — Recruiter discovery and messaging

### 8.1 Recruiter discovery pipeline

**Phase 1 — Apollo.io enrichment (primary, 50 free credits/month):**

```python
async def find_recruiters_apollo(company: str, role: str) -> list[Recruiter]:
    response = await apollo_client.search_people(
        organization_name=company,
        person_titles=["Recruiter", "Talent Acquisition", "Technical Recruiter",
                       "Hiring Manager", "Engineering Manager"],
        person_locations=["India"],
        limit=5
    )
    return [Recruiter(
        name=p.name, title=p.title, email=p.email,
        phone=p.phone, linkedin=p.linkedin_url, source="apollo"
    ) for p in response.people]
```

**Phase 2 — Hunter.io (supplementary, 25 free searches/month):** Email lookup by name + company domain when Apollo doesn't return email.

**Phase 3 — LinkedIn scraping (fallback when API credits exhausted):** Playwright-based read-only scraping. Filter for profiles active within 30 days. Rate limit: 30 profile views/day. No automated LinkedIn messaging — email only.

### 8.2 Message generation

Two variants per recruiter:

**Variant A — Specific:** References resume accomplishments matched to JD requirements (e.g., "control plane services across 20+ cloud regions").

**Variant B — Generic:** Role + interest + years of experience, no specific accomplishments.

Both sent to Telegram for selection:

```
📨 Outreach ready

🏢 Razorpay — Recruiter: Ananya Sharma (Sr. Technical Recruiter)
📧 Email: ananya.s@razorpay.com
🔗 LinkedIn: linkedin.com/in/ananyasharma

Variant A (specific):
"...control plane services across 20+ cloud regions..."

Variant B (generic):
"...distributed systems and cloud platform engineering..."

[Send Variant A]  [Send Variant B]  [Edit]  [Skip]
```

### 8.3 Rate limiting and safety

| Constraint | Limit |
|-----------|-------|
| LinkedIn connection requests per day | 20 |
| Outreach emails per day | 30 |
| Messages per company | 2 recruiters max |
| Cooldown between messages to same person | 14 days |
| Mandatory human approval | Every message, no exceptions |

---

## 9. Tracker Agent — Centralised pipeline management

### 9.1 Spreadsheet schema

| Column | Data type | Source agent | Auto-populated |
|--------|----------|-------------|---------------|
| Date discovered | datetime | Scout | Yes |
| Company | string | Scout | Yes |
| Role title | string | Scout | Yes |
| Posting URL | URL | Scout | Yes |
| Location | string | Scout | Yes |
| Relevance score | int (0–100) | Scout | Yes |
| Matching skills | comma-separated | Scout | Yes |
| Missing skills | comma-separated | Scout | Yes |
| User decision | Apply / Skip | Telegram | Yes |
| Resume version | filepath | Resume Agent | Yes |
| Base resume version | version ID (e.g., v014) | Resume Agent | Yes |
| ATS score | int (%) | Resume Agent | Yes |
| Application method | API / Email / Manual | Application Agent | Yes |
| Application status | Submitted / Acknowledged / Interview / Rejected / Offer | Application + Email Intel | Yes |
| Application date | datetime | Application Agent | Yes |
| Last email update | datetime | Email Intelligence | Yes |
| Email classification | interview / rejection / etc. | Email Intelligence | Yes |
| Recruiter 1 name | string | Outreach Agent | Yes |
| Recruiter 1 email | string | Outreach Agent | Yes |
| Recruiter 1 LinkedIn | URL | Outreach Agent | Yes |
| Recruiter 1 phone | string | Outreach Agent | Yes |
| Outreach status | Sent / Pending / Skipped | Outreach Agent | Yes |
| Outreach variant used | A (specific) / B (generic) | Outreach Agent | Yes |
| Outreach date | datetime | Outreach Agent | Yes |
| Follow-up 1 due | datetime | Computed: app_date + 7 days | Yes |
| Follow-up 1 status | Sent / Pending / Skipped | Follow-up Agent | Yes |
| Follow-up 2 due | datetime | Computed: app_date + 14 days | Yes |
| Follow-up 2 status | Sent / Final / Skipped | Follow-up Agent | Yes |
| Response received | Yes / No | Email Intelligence | Yes |
| Interview stage | None / Phone / System Design / Assessment / Onsite / Offer | Email Intelligence | Yes |
| Notes | free text | Manual | No |

### 9.2 Automated reporting

**Daily summary via Telegram (9 PM IST):**

```
📊 Daily pipeline summary — 29 March 2026

🔍 Jobs discovered: 47
📬 Notifications sent: 13
✅ Applications submitted: 8
📨 Outreach messages sent: 5
⏰ Follow-ups due tomorrow: 3

📈 Running totals this week:
   Applications: 42 | Outreach: 28 | Responses: 4

[View spreadsheet 📋]
```

### 9.3 Conditional formatting (Excel)

- Red row: follow-up overdue (> 10 days since application, no response)
- Yellow row: follow-up due (7–10 days)
- Green row: response received / interview scheduled
- Bold: applications submitted today

---

## 10. Email Intelligence Agent — Gmail auto-classification

### 10.1 Gmail integration

**Authentication:** OAuth 2.0 with scopes:
- `gmail.readonly` — read email content for classification
- `gmail.send` — send follow-up emails (used by Follow-up Agent)
- `gmail.modify` — apply labels to classified emails

**Sync mechanism:** `users.history().list()` with `historyId`-based incremental sync. First run: full scan of last 30 days. Subsequently: only new messages since last `historyId` (stored in PostgreSQL).

**Polling frequency:** Every 5 minutes via APScheduler.

### 10.2 Email filtering and classification

**Stage 1 — Domain filter (zero LLM cost):**

Match sender domain against:
- Company domains from tracker (auto-populated during application submission)
- ATS platform domains: `lever.co`, `greenhouse.io`, `ashbyhq.com`, `noreply.linkedin.com`
- Third-party recruiting platforms: `icims.com`, `smartrecruiters.com`, `hire.google.com`

Unknown domains ignored.

**Stage 2 — Keyword pre-filter (zero LLM cost):**

Regex-based classification for unambiguous cases:

```python
POSITIVE_KEYWORDS = [
    r"interview", r"phone screen", r"schedule a call",
    r"next steps", r"move forward", r"shortlisted",
    r"online assessment", r"coding challenge", r"take-home",
    r"would like to discuss", r"availability"
]

REJECTION_KEYWORDS = [
    r"unfortunately", r"other candidates", r"not moving forward",
    r"position has been filled", r"not a match", r"regret to inform"
]

NEUTRAL_KEYWORDS = [
    r"received your application", r"thank you for applying",
    r"will review", r"under consideration", r"out of office"
]
```

If a keyword match is found with high confidence (keyword in subject or first 200 chars of body), classify without LLM call.

**Stage 3 — LLM classification (ambiguous emails only, cascading client):**

Email subject + body (truncated to 2,000 tokens) classified into structured JSON:

```json
{
  "classification": "interview_invite",
  "confidence": 0.94,
  "company": "Razorpay",
  "role": "Senior Backend Engineer",
  "matched_application_id": "app_2026_03_27_razorpay_sbe",
  "action_items": ["Schedule interview for April 3 at 2:00 PM IST"],
  "extracted_data": {
    "interviewer_name": "Amit Patel",
    "interviewer_title": "Engineering Manager",
    "interview_type": "System Design",
    "proposed_datetime": "2026-04-03T14:00:00+05:30"
  }
}
```

**Classification taxonomy (8 categories):**

| Classification | Definition | Tracker update | Telegram notification |
|---------------|-----------|----------------|----------------------|
| `interview_invite` | Invitation to any interview round | Status → "Interview Scheduled", Stage → type | Yes (immediate, high priority) |
| `recruiter_reply` | Positive response from recruiter | Response Received → "Yes" | Yes |
| `assessment_request` | Online coding test / take-home | Status → "Assessment Pending" | Yes |
| `offer` | Job offer (verbal or written) | Status → "Offer Received" | Yes (immediate) |
| `follow_up_needed` | Company asks for additional info | Notes → append details | Yes (urgent) |
| `acknowledgement` | Automated "we received your application" | Status → "Acknowledged" | No (silent update) |
| `rejection` | Explicit rejection at any stage | Status → "Rejected" | **No** (silent — rejections are demoralising at high volume; visible in daily summary and tracker) |
| `unrelated` | Not related to any tracked application | Ignored | No |

**Confidence threshold:** Classifications below 0.85 sent to Telegram with "Low confidence — please verify" tag. Tracker not auto-updated below threshold.

### 10.3 Feedback to Adaptive Resume Engine

Every classification with an outcome weight is pushed to the feedback store:

| Signal | Weight |
|--------|--------|
| Interview invitation | 1.0 |
| Assessment / take-home | 0.9 |
| Recruiter reply (interest) | 0.8 |
| Follow-up request from company | 0.6 |
| Acknowledgement | 0.0 (neutral) |
| No response after 14 days | -0.3 |
| Rejection | -0.5 |

### 10.4 Gmail label management

Auto-created hierarchy:

```
JobPilot/
  ├── Interview Invites
  ├── Recruiter Replies
  ├── Assessments
  ├── Offers
  ├── Rejections
  ├── Acknowledgements
  ├── Follow-up Needed
  └── Processed
```

### 10.5 Privacy constraints

- No raw email content stored permanently — only classification metadata persisted.
- No email forwarding or exfiltration.
- Ollama available as LLM fallback for maximum privacy.
- OAuth tokens encrypted at rest in `credentials/` directory with 600 permissions.

---

## 11. Follow-up Automation Agent

### 11.1 Trigger

Runs daily at 10:00 AM IST. Queries tracker for applications where:

```sql
-- Day 7 follow-up candidates
SELECT * FROM applications
WHERE status IN ('submitted', 'acknowledged')
AND submitted_at < NOW() - INTERVAL '7 days'
AND response_received = false
AND follow_up_1_sent = false;

-- Day 14 follow-up candidates (final)
SELECT * FROM applications
WHERE status IN ('submitted', 'acknowledged')
AND submitted_at < NOW() - INTERVAL '14 days'
AND response_received = false
AND follow_up_1_sent = true
AND follow_up_2_sent = false;
```

### 11.2 Email generation

LLM drafts a follow-up referencing the specific role, application date, and one JD-relevant qualification from the tailored resume used.

**System prompt:**

```
Draft a professional follow-up email for a job application. Rules:
1. Maximum 4 sentences (Day 7) or 2 sentences (Day 14 — marked as final).
2. Reference the specific role and company.
3. Briefly restate one key qualification.
4. Express continued interest without desperation.
5. End with a clear, low-pressure call to action.
6. Tone: direct, polite, confident. No flattery, no filler.
7. PROHIBITED PHRASES: "just checking in", "touching base", "circling back",
   "hope this finds you well", "per my last email"
```

**Day 7 example:**

```
Subject: Following up — Senior Backend Engineer application

Hi Ananya,

I submitted my application for the Senior Backend Engineer role at Razorpay
on March 20 and wanted to follow up. My experience building and operating
control plane services across 20+ cloud regions at Oracle aligns closely
with the role's requirements.

I remain very interested and would welcome a conversation at your convenience.

Best regards,
Pranjal Yadav
```

**Day 14 example (final):**

```
Subject: Final follow-up — Senior Backend Engineer application

Hi Ananya,

I am following up one last time regarding my application for the Senior
Backend Engineer role at Razorpay submitted on March 20. I remain interested
and available to discuss the opportunity.

Best regards,
Pranjal Yadav
```

### 11.3 Telegram approval flow

```
⏰ Follow-up due (Day 7)

🏢 Razorpay — Senior Backend Engineer
📅 Applied: 20 March 2026 (9 days ago)
📧 To: ananya.s@razorpay.com

Draft:
"I submitted my application for the Senior Backend Engineer
role at Razorpay on March 20..."

[Send ✓]  [Edit ✏️]  [Skip ✗]  [Delay 3 days ⏰]
```

"Delay 3 days" reschedules the follow-up check for that application.

### 11.4 Rules

| Rule | Value |
|------|-------|
| Maximum follow-ups per application | 2 (Day 7 initial + Day 14 final) |
| Minimum gap between follow-ups | 7 days |
| Do not follow up if | Rejection received, interview scheduled, or user marked "Skip" |
| Do not follow up if | Recruiter already contacted via Outreach Agent (prevents double-contact) |
| Day 14 follow-up | 2 sentences, marked as final. Tracker: "No response — closed" after send. |
| Batch limit | 10 follow-ups per day |
| Format | Plain text only (reduces spam classification risk) |
| Send mechanism | Gmail API `gmail.send` scope via user's authenticated account |
| Human approval | Mandatory for every follow-up, no exceptions |

---

## 12. Adaptive Resume Engine — Multi-agent consensus

### 12.1 Core concept

Every 12 hours, the Adaptive Resume Engine analyses which resume variants generated positive responses versus which were ignored or rejected. Four specialised agents independently propose modifications to the base resume. A tournament-based consensus mechanism selects the strongest candidate, with a Mediator Agent deliberation round when top scores are within 5 points. The winning version becomes the new base resume.

This creates a closed feedback loop: `base resume → tailored resumes → applications → responses → analysis → improved base resume → repeat`.

No publicly available job search tool implements closed-loop resume optimisation based on actual response data. This is the architecturally most distinctive feature of JobPilot.

### 12.2 Feedback data collection

```sql
-- Positive signal: resumes that led to interviews or recruiter replies
SELECT r.tex_source, r.modified_bullets, r.ats_score,
       j.company, j.role, j.required_skills, j.relevance_score
FROM resumes r
JOIN applications a ON r.application_id = a.id
JOIN jobs j ON a.job_id = j.id
WHERE a.status IN ('interview_scheduled', 'recruiter_engaged', 'offer_received')
AND r.created_at > NOW() - INTERVAL '7 days';

-- Negative signal: resumes with no response after 14+ days
SELECT r.tex_source, r.modified_bullets, r.ats_score,
       j.company, j.role, j.required_skills, j.relevance_score
FROM resumes r
JOIN applications a ON r.application_id = a.id
JOIN jobs j ON a.job_id = j.id
WHERE a.status IN ('submitted', 'acknowledged')
AND a.submitted_at < NOW() - INTERVAL '14 days'
AND a.response_received = false;
```

**Data captured per application for feedback:**

```json
{
  "job_id": "razorpay-sbe-2026-03-27",
  "resume_version": "resumes/razorpay/senior_backend_engineer_2026-03-27.tex",
  "base_resume_version": "resume_versions/v014_2026-03-27T09:00.tex",
  "relevance_score": 82,
  "ats_score": 84,
  "outcome": "interview_invite",
  "outcome_weight": 1.0,
  "jd_keywords_matched": ["Kubernetes", "Docker", "distributed systems"],
  "bullets_modified": [1, 3, 5, 7],
  "action_verbs_used": ["Engineered", "Orchestrated", "Delivered"]
}
```

### 12.3 Agent roles in the tournament

**Trigger:** APScheduler cron at 06:00 IST and 18:00 IST.

**Minimum data requirement:** 10 applications with known outcomes before activation.

Four specialised agents run in **parallel**:

| Agent | Perspective | What it optimises | LLM tier |
|-------|-----------|------------------|---------|
| **Keyword Agent** | ATS optimisation | Per-keyword success rate; keyword density and placement adjustments | Tier 1 (free) |
| **Narrative Agent** | Storytelling and impact | Harvard verb correlation, quantification styles, sentence structures in accepted vs. rejected | Claude Sonnet (premium) |
| **Structure Agent** | Layout and ordering | Bullet ordering, section emphasis, skills arrangement correlation with outcomes | Tier 1 (free) |
| **Market Agent** | Industry trend alignment | Last 12 hours of *discovered* JDs (not just applied); identifies emerging skill keywords and trending requirements the candidate genuinely possesses | Tier 1 (free) |

Each agent receives: current base resume `.tex`, positive-signal resume set, negative-signal resume set, its specialised system prompt.

Each agent outputs: modified `.tex` file, structured rationale (max 200 words), self-assessed confidence (0–100).

**Free alternative for Narrative Agent:** Route through Tier 1 cascade. Lower phrasing quality but functional.

### 12.4 Tournament protocol

**Round 1 — Parallel generation (4 LLM calls, concurrent):**

All four agents produce their proposed base resumes independently.

**Round 2 — Judge scoring (1 LLM call):**

A Judge Agent (Tier 1, or Claude Sonnet if available) scores all four proposals on five dimensions:

| Criterion | Weight |
|-----------|--------|
| Keyword coverage against last 24 hours of JD data | 25% |
| Phrasing quality (action verb strength, quantification) | 25% |
| Structural coherence (logical ordering, section flow) | 20% |
| ATS compatibility (formatting, keyword density) | 15% |
| Factual integrity (no fabrication vs. original resume) | 15% (truthfulness, weighted 2x internally) |

```json
{
  "proposals": [
    {"agent": "keyword", "scores": {"keyword": 88, "phrasing": 70, "structure": 75, "ats": 85, "truth": 95}, "total": 82.1},
    {"agent": "narrative", "scores": {"keyword": 72, "phrasing": 92, "structure": 80, "ats": 78, "truth": 90}, "total": 82.4},
    {"agent": "structure", "scores": {"keyword": 70, "phrasing": 75, "structure": 90, "ats": 80, "truth": 95}, "total": 80.5},
    {"agent": "market", "scores": {"keyword": 85, "phrasing": 68, "structure": 72, "ats": 82, "truth": 88}, "total": 78.6}
  ],
  "winner": "narrative",
  "runner_up": "keyword",
  "margin": 0.3,
  "deliberation_triggered": true
}
```

**Round 3 — Deliberation (conditional, only if margin < 5 points):**

A dedicated **Mediator Agent** (Claude Sonnet) receives both top proposals and:

1. Identifies the strengths in each proposal.
2. Identifies weaknesses or risks in each.
3. Synthesises a merged version incorporating the strongest elements of both.

The Judge Agent re-scores the merged version against the two originals. Highest of three (original A, original B, merged) wins.

Deliberation triggers approximately 20–30% of cycles.

**Round 4 — Compilation and validation:**

Winning `.tex` file compiled with `pdflatex`. If compilation fails, previous base resume retained (no regression).

**Estimated cost per cycle:** Tournament = 5 LLM calls (4 agents + 1 judge) ~20K tokens, $0 on DeepSeek + 1 Claude call for Narrative = ~$0.015. Deliberation adds 2 calls ~12K tokens = ~$0.02. Total: ~$0.03–0.05 per cycle, $0.06–0.10/day, ~$2–3/month.

**$0 alternative:** Route all agents including Narrative and Mediator through Tier 1 cascade. Lower quality on phrasing synthesis but fully functional tournament.

### 12.5 Version control (git-like history)

```
resume_versions/
├── v001_2026-03-27T09:00.tex     # User's original
├── v001_2026-03-27T09:00.pdf
├── v001_meta.json
├── v002_2026-03-27T21:00.tex     # First adaptive update
├── v002_2026-03-27T21:00.pdf
├── v002_meta.json
├── ...
├── current -> v014_2026-04-03T18:00.tex   # Symlink to active version
└── changelog.md                           # Auto-generated
```

**Metadata per version:**

```json
{
  "version": "v014",
  "timestamp": "2026-04-03T18:00:00+05:30",
  "winning_agent": "narrative",
  "tournament_scores": {"keyword": 82.1, "narrative": 87.4, "structure": 80.5, "market": 78.6},
  "deliberation_triggered": true,
  "mediator_merged": true,
  "final_score": 89.2,
  "changes_summary": "Replaced 4 bullet verbs. Reordered skills. Added 'zero-downtime deployment' phrasing.",
  "positive_signals_count": 4,
  "negative_signals_count": 12,
  "diff_from_previous": "diff_v013_v014.txt"
}
```

**Auto-generated changelog entry:**

```markdown
## v014 — 3 April 2026, 18:00 IST

**Consensus:** Deliberation (margin: 0.3 points, merged version won at 89.2)
**Top agents:** Narrative (87.4) + Keyword (82.1) → Mediator merge (89.2)

### Changes:
- Bullet 1: "Owned the Control Plane" → "Architected and solely owned
  Control Plane backend services, shipping across 20+ OCI regions with
  zero release delays"
- Bullet 5: Moved from position 5 → position 2 (K8s bullet front-loaded;
  71% success rate in K8s-mentioning JDs)
- Skills: "Distributed Systems" moved to position 1
- Removed: "team player" (0.11 success rate)
- Market Agent insight: "Terraform" added emphasis (appeared in 68% of
  last 12hrs JDs, candidate has experience, was buried at skill #14)

### v013 performance: 14 sent, 4 positive (28.6%)
```

**Weekly diff report (Telegram, Sunday 9 PM IST):**

```
📋 Base resume evolution — Week of 24 March 2026

📊 Versions this week: v008 → v021 (14 evolution cycles)
✅ Acceptance rate: 28% (up from 19% last week)

Top changes that persisted across versions:
1. "Orchestrated" replaced "Managed" in 3 bullets (stuck since v010)
2. Kubernetes front-loaded in skills section (stuck since v012)
3. "zero-downtime" phrase added to deployment bullet (stuck since v014)

Changes that were reverted:
1. Adding "AI/ML" to skills (reverted in v016 — not in actual experience)
2. Removing internship section (reverted in v013 — interview feedback indicated it helped)

[View full changelog 📄]  [Revert to specific version ↩️]
```

### 12.6 Safety guardrails

| Guardrail | Implementation |
|-----------|---------------|
| No fabrication | Diff-check every change against original resume. Judge scores truthfulness at 2x weight. Reject undeducible claims. |
| Performance floor | If acceptance rate < 10% for 3 consecutive 12-hour cycles, auto-revert to last version with > 15% and alert via Telegram. |
| User override: revert | `/revert v014` via Telegram restores that version as active base. No data deleted. |
| User override: pin | `/pin v014` freezes that version. Engine proposes but never auto-applies to pinned versions. |
| Maximum change rate | No more than 30% of bullets modified per 12-hour cycle. |
| Audit trail | Every modification links to source data and statistical justification in metadata. |

### 12.7 Cold start strategy

During the first 1–2 weeks, insufficient acceptance/rejection data exists. During this period:
1. User-provided base resume used as-is for all tailoring.
2. All tailored resumes stored with outcome status (pending → accepted/rejected).
3. Once minimum threshold (10 applications with known outcomes) is reached, first evolution cycle runs.
4. Telegram notification: "Adaptive resume engine activated. First evolution will run at {next 12hr mark}."

---

## 13. Analytics Dashboard — Prometheus + Grafana

### 13.1 Architecture

JobPilot exposes a `/metrics` endpoint via FastAPI using `prometheus_client`. Prometheus scrapes every 15 seconds with 90-day retention. Grafana connects to Prometheus as a data source and renders five pre-configured dashboards.

```yaml
# docker-compose additions
prometheus:
  image: prom/prometheus:v2.52.0
  volumes:
    - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    - ./monitoring/alert_rules.yml:/etc/prometheus/alert_rules.yml
    - prometheus_data:/prometheus
  ports: ["9090:9090"]
  command: ['--config.file=/etc/prometheus/prometheus.yml', '--storage.tsdb.retention.time=90d']
  restart: unless-stopped

grafana:
  image: grafana/grafana:11.0.0
  volumes:
    - grafana_data:/var/lib/grafana
    - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
    - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards
  ports: ["3000:3000"]
  environment:
    - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
  depends_on: [prometheus]
  restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

### 13.2 Metrics catalogue

**Counters:**

| Metric | Labels | Description |
|--------|--------|-------------|
| `jobpilot_jobs_discovered_total` | `source` | By linkedin/naukri/career_page |
| `jobpilot_notifications_sent_total` | `decision` | By apply/skip |
| `jobpilot_applications_submitted_total` | `method` | By api/email/manual |
| `jobpilot_outreach_sent_total` | `variant`, `channel` | By specific/generic, email/linkedin |
| `jobpilot_emails_classified_total` | `classification` | By interview/rejection/etc. |
| `jobpilot_followups_sent_total` | `day` | By day_7/day_14 |
| `jobpilot_resume_versions_total` | | Base resume versions generated |
| `jobpilot_llm_calls_total` | `provider`, `tier`, `task_type` | Per provider per task |
| `jobpilot_llm_failures_total` | `provider`, `error_type` | Rate limits, timeouts |

**Gauges:**

| Metric | Description |
|--------|-------------|
| `jobpilot_active_applications` | Applications in non-terminal state |
| `jobpilot_pending_followups` | Applications past 7 days with no response |
| `jobpilot_current_acceptance_rate` | Rolling 7-day acceptance rate (%) |
| `jobpilot_current_base_resume_version` | Current version number |
| `jobpilot_llm_cascade_depth` | Average cascade depth (1.0 = all T1, 5.0 = hitting Ollama) |

**Histograms:**

| Metric | Buckets | Description |
|--------|---------|-------------|
| `jobpilot_resume_generation_seconds` | 5, 10, 30, 60, 120 | Approval → compiled PDF |
| `jobpilot_scout_cycle_seconds` | 30, 60, 120, 300, 600 | Per complete polling cycle |
| `jobpilot_llm_latency_seconds` | 0.5, 1, 2, 5, 10, 30 | Per-call by provider |

### 13.3 Grafana dashboards

**Dashboard 1 — Application funnel:**

```
┌─────────────────────────────────────────────────────┐
│  APPLICATION FUNNEL (last 30 days)                  │
│                                                     │
│  Discovered ─── 1,247 ████████████████████████████  │
│  Notified  ───── 312  ███████                       │
│  Applied   ───── 198  █████                         │
│  Responded ──── 41    ██                             │
│  Interview ──── 12    █                              │
│  Offer     ──── 2     ▏                              │
│                                                     │
│  Conversion rates:                                   │
│  Notified→Applied: 63%  Applied→Response: 21%       │
│  Response→Interview: 29%  Interview→Offer: 17%      │
└─────────────────────────────────────────────────────┘
```

**Dashboard 2 — Daily activity:** Jobs discovered per hour (by source), applications/day, outreach/day, today's stats.

**Dashboard 3 — Resume evolution:** Acceptance rate over time (7-day rolling) with base resume version annotations, tournament scores table, current version stat.

**Dashboard 4 — LLM cost and health:** Calls by provider (pie), cascade depth over time, latency p50/p95 by provider, estimated monthly cost.

**Dashboard 5 — Outreach effectiveness:** Response rate by variant (A vs. B), top companies by response rate, sent vs. responded over time.

### 13.4 Alerting rules (delivered via Telegram)

| Alert | Condition | Severity |
|-------|-----------|----------|
| Scout Agent stalled | No discoveries for 2 hours during business hours | Warning |
| All LLMs failing | `jobpilot_llm_cascade_depth` > 4.5 for 10 minutes | Critical |
| Acceptance rate declining | < 10% for 3 consecutive 12-hour cycles | Warning |
| Email Agent disconnected | No Gmail poll in 15 minutes | Critical |
| Application backlog | > 5 approved applications not yet submitted | Warning |

---

## 14. MCP server architecture

Three MCP tool servers making JobPilot's capabilities reusable by any MCP-compatible AI assistant (Claude Desktop, Cursor, ChatGPT):

### 14.1 `jobpilot-scout`

- `search_jobs(query, location, recency)` — searches across all configured boards, returns scored results
- `get_job_details(job_id)` — returns full JD text and scoring breakdown

### 14.2 `jobpilot-resume`

- `tailor_resume(job_description, base_resume_path)` — generates tailored LaTeX resume, returns PDF path
- `estimate_ats_score(resume_path, job_description)` — returns ATS keyword match percentage

### 14.3 `jobpilot-outreach`

- `find_recruiters(company, role)` — returns recruiter contacts via Apollo/Hunter/LinkedIn
- `draft_outreach(recruiter, job_posting, resume)` — generates two message variants

**Transport:** Streamable HTTP (not stdio) — allows remote access from any MCP client on the same network.

---

## 15. Configuration

```yaml
# config.yaml

target_titles:
  - "Backend Engineer"
  - "Senior Backend Engineer"
  - "Platform Engineer"
  - "AI Platform Engineer"
  - "Infrastructure Engineer"
  - "Site Reliability Engineer"
  - "Cloud Engineer"
  - "ML Infrastructure Engineer"
  - "Software Engineer - Backend"
  - "Software Engineer - Platform"
  - "DevOps Engineer"

location_preferences:
  accept: ["Bangalore", "Bengaluru", "Remote", "Anywhere"]
  accept_if_visa_sponsored: ["United States", "United Kingdom", "Germany", "Netherlands", "Singapore", "Canada"]
  reject: ["Mumbai", "Delhi", "Hyderabad", "Chennai", "Pune"]

relevance_threshold: 75
max_daily_notifications: 15

tier1_companies: []   # top 30 — populated from user's list
tier1_interval_minutes: 10
tier2_interval_hours: 4

llm_cascade:
  - {provider: deepseek, model: deepseek-chat, api_key_env: DEEPSEEK_API_KEY, timeout: 30}
  - {provider: qwen, model: qwen-plus, api_key_env: DASHSCOPE_API_KEY, timeout: 30}
  - {provider: gemini, model: gemini-2.0-flash, api_key_env: GEMINI_API_KEY, timeout: 20}
  - {provider: openai, model: gpt-4o-mini, api_key_env: OPENAI_API_KEY, timeout: 20}
  - {provider: ollama, model: llama3.2, base_url: "http://localhost:11434", timeout: 60}

premium_llm:
  provider: anthropic
  model: claude-sonnet-4-5-20241022
  api_key_env: ANTHROPIC_API_KEY
  use_for: [resume_bullet_rewriting, deliberation_mediation]
  # Free alternative: set premium_llm.enabled to false; all tasks route through cascade
  enabled: true

outreach:
  max_linkedin_requests_per_day: 20
  max_emails_per_day: 30
  max_recruiters_per_company: 2
  cooldown_days: 14

followup:
  first_followup_days: 7
  second_followup_days: 14
  max_per_day: 10
  skip_if_outreach_sent: true

adaptive_resume:
  cycle_times: ["06:00", "18:00"]
  min_applications_before_activation: 10
  max_bullet_change_pct: 30
  auto_revert_threshold_pct: 10
  auto_revert_consecutive_cycles: 3

telegram:
  bot_token_env: TELEGRAM_BOT_TOKEN
  chat_id_env: TELEGRAM_CHAT_ID
  daily_summary_time: "21:00"
  suppress_rejection_notifications: true

gmail:
  credentials_path: "credentials/gmail_oauth.json"
  token_path: "credentials/gmail_token.json"
  poll_interval_minutes: 5
  classification_confidence_threshold: 0.85
  additional_ats_domains: ["lever.co", "greenhouse.io", "ashbyhq.com", "icims.com", "smartrecruiters.com", "hire.google.com", "noreply.linkedin.com"]

resume:
  base_template: "templates/base_resume.tex"
  harvard_verbs: "data/harvard_action_verbs.yaml"
  output_dir: "resumes/"
  latex_compiler: "pdflatex"

tracker:
  type: "excel"
  output_path: "tracker/job_pipeline.xlsx"

monitoring:
  prometheus_retention_days: 90
  grafana_password_env: GRAFANA_PASSWORD
```

---

## 16. Learning plan integration

| Learning phase | Hours | Direct application in JobPilot |
|---------------|-------|-------------------------------|
| Phase 1: LLM fundamentals | 10–12 | Token cost modelling for cascading LLM strategy |
| Phase 2: Prompt engineering | 8–10 | System prompts for rewriting, scoring, classification, outreach |
| Phase 3: RAG & vectors | 10–12 | Resume-JD embedding similarity via pgvector |
| Phase 4: Agent frameworks | 13–15 | LangGraph 9-agent orchestration, multi-agent tournament, HITL |
| Phase 5: MCP protocol | 5–6 | Three MCP tool servers (scout, resume, outreach) |
| Phase 6: Eval & observability | 7–8 | LLM-as-judge, tournament scoring, Prometheus metrics |
| Phase 7: Deploy & scale | 8–10 | Docker Compose (with Prometheus + Grafana), APScheduler, VPS |

---

## 17. Build sequence

| Weekend | Deliverable | Learning phase |
|---------|------------|----------------|
| 1 | Scout Agent + Telegram notification (LinkedIn + career pages + RSS) | Phases 1–2 |
| 2 | Resume Agent + LaTeX compilation pipeline | Phases 2–3 |
| 3 | Tracker Agent + daily reporting | Phase 4 (partial) |
| 4 | Application Agent (email + ATS API + manual fallback) + Gmail OAuth setup | Phase 4 |
| 5 | Email Intelligence Agent (Gmail classification → tracker updates) | Phase 4 |
| 6 | Outreach Agent (Apollo.io + Hunter.io + dual-variant messages) | Phases 4–5 |
| 7 | MCP servers + LangGraph full orchestration integration | Phase 5 |
| 8 | Adaptive Resume Engine (4-agent tournament + Mediator deliberation + versioning) | Phase 6 |
| 9 | Follow-up Agent + Prometheus metrics + Grafana dashboards | Phases 6–7 |
| 10 | Docker Compose deployment + end-to-end integration testing + alerting | Phase 7 |

**Total: 10 weekends (~130–160 hours)**

---

## 18. Monthly operating cost

| Component | Realistic cost | $0 alternative |
|-----------|---------------|----------------|
| LLM API (cascading, Tier 1 free) | $0–2 | DeepSeek/Qwen free tiers cover >95% of calls |
| Adaptive Resume: Narrative Agent + Mediator (Claude Sonnet) | ~$2–3 | Route through Tier 1 cascade (lower phrasing quality) |
| VPS hosting (Hetzner CX22) | $0–4.50 | Run locally on personal machine |
| Apollo.io (enrichment) | $0 | Free tier: 50 credits/month |
| Hunter.io (email finder) | $0 | Free tier: 25 searches/month |
| Telegram Bot API | $0 | Free |
| Gmail API | $0 | Free for personal use |
| Prometheus + Grafana (self-hosted) | $0 | ~200MB RAM combined |
| **Total (realistic)** | **$4–10/month** | |
| **Total ($0 path)** | **$0** | All LLM tasks via Ollama; run locally; free API tiers only |

---

## 19. Risk matrix

| Risk | Prob. | Sev. | Mitigation |
|------|-------|------|-----------|
| LinkedIn account suspension | Med | High | Apollo.io primary; Playwright with human-like delays (2–5s); never automate LinkedIn messaging |
| Resume bullet fabrication | Low | Critical | Strict prompt; diff view; [INFERRED] tags; truthfulness 2x weight in tournament; LLM-as-judge |
| Free-tier LLM rate limits | Med | Low | 3-tier cascade; DeepSeek 50M/day; Ollama terminal fallback |
| Career page structure changes | High | Med | ATS-platform generic scrapers; alert on failures; periodic review |
| LaTeX compilation failures | Med | Med | Syntax validation; auto-fix; fallback to previous base resume |
| Gmail OAuth token expiry | Med | Med | Auto-refresh; Telegram alert on failure; `setup_gmail_oauth.py` re-auth script |
| Adaptive engine regression | Med | High | Absolute floor: < 10% for 3 cycles → auto-revert. `/revert` and `/pin` commands. Full version history. |
| Email misclassification | Low | Med | 0.85 confidence threshold; low-confidence → Telegram manual review; keyword pre-filter for obvious cases |
| Double-contacting recruiters | Low | Med | Follow-up Agent checks outreach status before drafting; skip_if_outreach_sent config |
| Follow-up emails marked as spam | Med | Med | Plain text only; personal Gmail sender; max 2 per application; 10/day limit |
| Prometheus resource usage | Low | Low | 90-day retention; ~200MB RAM combined; lightweight containers |
| Apollo.io free tier exhaustion | High | Med | 50 credits ≈ 2.5 weeks at 2/company. Hunter.io supplement (25/month). LinkedIn scraping fallback. |

---

## 20. Repository structure

```
jobpilot/
├── docker-compose.yml              # App + PostgreSQL + Prometheus + Grafana
├── Dockerfile
├── config.yaml
├── .env.example
├── README.md
│
├── agents/
│   ├── __init__.py
│   ├── scout.py                    # Job discovery + scoring
│   ├── resume.py                   # LaTeX tailoring + compilation
│   ├── application.py              # Submission logic
│   ├── outreach.py                 # Recruiter discovery + messaging
│   ├── tracker.py                  # Spreadsheet management
│   ├── email_intelligence.py       # Gmail classification + tracker updates
│   ├── follow_up.py                # Day 7 + Day 14 follow-up drafting
│   └── adaptive_resume/
│       ├── __init__.py
│       ├── engine.py               # 12-hour evolution orchestrator
│       ├── keyword_agent.py        # ATS keyword optimisation
│       ├── narrative_agent.py      # Storytelling + impact
│       ├── structure_agent.py      # Layout + ordering
│       ├── market_agent.py         # Industry trend alignment
│       ├── judge.py                # Tournament scoring + deliberation trigger
│       ├── mediator.py             # Deliberation merge synthesis
│       └── version_manager.py      # Git-like version history + changelog
│
├── orchestrator/
│   ├── __init__.py
│   ├── graph.py                    # LangGraph state machine (9 agents)
│   ├── state.py                    # Shared state schema
│   └── checkpoints.py             # Human-in-the-loop checkpoint logic
│
├── llm/
│   ├── __init__.py
│   ├── cascade.py                  # Cascading LLM client
│   ├── prompts/
│   │   ├── scoring.py              # JD relevance scoring
│   │   ├── rewriting.py            # Resume bullet rewriting
│   │   ├── outreach.py             # Recruiter message generation
│   │   ├── extraction.py           # JD keyword extraction
│   │   ├── email_classification.py # Email intent classification
│   │   ├── follow_up.py            # Follow-up email drafting
│   │   ├── tournament_judge.py     # Resume tournament scoring
│   │   └── deliberation.py         # Mediator deliberation prompts
│   └── quality.py                  # LLM-as-judge quality validation
│
├── scrapers/
│   ├── __init__.py
│   ├── linkedin.py
│   ├── naukri.py
│   ├── career_page.py
│   ├── lever.py
│   ├── greenhouse.py
│   └── rss.py
│
├── integrations/
│   ├── __init__.py
│   ├── telegram_bot.py
│   ├── gmail.py                    # Gmail API client (read/send/label)
│   ├── apollo.py
│   ├── hunter.py
│   └── sheets.py
│
├── monitoring/
│   ├── metrics.py                  # Prometheus metric definitions
│   ├── prometheus.yml              # Scrape config
│   ├── alert_rules.yml             # Alerting rules
│   └── grafana/
│       ├── provisioning/
│       │   ├── datasources/
│       │   │   └── prometheus.yml
│       │   └── dashboards/
│       │       └── dashboard.yml
│       └── dashboards/
│           ├── application_funnel.json
│           ├── daily_activity.json
│           ├── resume_evolution.json
│           ├── llm_cost_health.json
│           └── outreach_effectiveness.json
│
├── mcp_servers/
│   ├── scout_server.py
│   ├── resume_server.py
│   └── outreach_server.py
│
├── templates/
│   ├── base_resume.tex
│   └── cover_letter.tex
│
├── data/
│   ├── harvard_action_verbs.yaml
│   ├── companies.yaml              # 200-company list with URLs + ATS type
│   └── applicant_profile.yaml
│
├── credentials/                    # Gitignored
│   ├── gmail_oauth.json
│   └── gmail_token.json
│
├── resumes/                        # Per-application tailored resumes (gitignored)
│   └── {company}/
│       ├── {role}_{date}.tex
│       ├── {role}_{date}.pdf
│       └── metadata.json
│
├── resume_versions/                # Base resume evolution history
│   ├── v001_2026-03-27T09:00.tex
│   ├── v001_2026-03-27T09:00.pdf
│   ├── v001_meta.json
│   ├── ...
│   ├── current -> latest           # Symlink
│   └── changelog.md
│
├── tracker/
│   └── job_pipeline.xlsx
│
├── tests/
│   ├── test_scout.py
│   ├── test_resume.py
│   ├── test_cascade_llm.py
│   ├── test_outreach.py
│   ├── test_email_intelligence.py
│   ├── test_adaptive_resume.py
│   ├── test_follow_up.py
│   └── test_metrics.py
│
└── scripts/
    ├── setup_telegram_bot.sh
    ├── setup_gmail_oauth.py        # OAuth 2.0 setup helper
    ├── init_database.sql
    ├── annotate_companies.py       # Classify ATS type per company
    └── import_grafana_dashboards.sh
```

---

## 21. Open items before development begins

1. **User's custom LaTeX `.tex` resume file** — needed to write the Resume Agent's parser for the specific template structure (section commands, bullet macros, custom class).
2. **The 200-company list** — share in any format. Will be annotated with ATS provider (Lever, Greenhouse, Ashby, Workday, custom) and career page URL.
3. **Top 30 tier-1 company selection** — subset for 10-minute polling priority.

---

## 22. Future scope (v1.2)

1. **Cover letter generation** — per-company cover letters using the same LLM pipeline.
2. **Interview preparation agent** — scrape Glassdoor, generate practice answers grounded in user's experience.
3. **Salary intelligence** — Levels.fyi / Glassdoor compensation data in Telegram notifications.
4. **Application withdrawal** — auto-withdraw pending applications after offer acceptance.
5. **Multi-channel outreach** — LinkedIn InMail automation (currently deferred due to account risk).
6. **A/B testing for outreach** — track variant success rates and auto-adjust selection weights over time.
