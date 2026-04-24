# JobPilot — Architecture Document v2.0

**Date:** 2 April 2026
**Based on:** PRD v2.0
**Scope:** Data flow, control flow, component interactions, persistence layer

---

## 1. System component map

```mermaid
graph TB
    subgraph External ["External Sources"]
        JS[JSearch API<br/>RapidAPI]
        LI[LinkedIn<br/>Guest Endpoints]
        NK[Naukri<br/>Public Pages]
        CP[200 Company<br/>Career Pages]
    end

    subgraph Scheduler ["APScheduler — every 10 min / 4 hrs"]
        SCH[Scheduler<br/>Cron Jobs]
    end

    subgraph Core ["Core Pipeline — jobpilot-app.service"]
        SA[Scout Agent]
        RA[Resume Agent]
        TA[Tracker Agent]
        LLM[Cascading LLM Client]
        DB[(PostgreSQL)]
        FS[Filesystem<br/>resumes/YYYY-MM-DD/]
    end

    subgraph Notification ["Telegram Service — jobpilot-telegram.service"]
        TG[Telegram Notifier]
        CB[Callback Handler]
    end

    subgraph External2 ["External Outputs"]
        GS[Google Sheets<br/>Master Spreadsheet]
        TB[Telegram Bot<br/>User's Phone]
    end

    SCH -->|triggers poll| SA
    JS -->|job listings JSON| SA
    LI -->|scraped HTML| SA
    NK -->|scraped HTML| SA
    CP -->|scraped HTML + RSS| SA

    SA -->|LLM scoring| LLM
    SA -->|store raw posting| DB
    SA -->|scored posting >=75| TG
    SA -->|on_notify mode| RA

    RA -->|LLM keyword extract| LLM
    RA -->|LLM bullet rewrite| LLM
    RA -->|store resume metadata| DB
    RA -->|PDF + .tex| FS
    RA -->|PDF attachment| TG

    TG -->|notification + PDF| TB
    TB -->|Accept / Ignore| CB
    CB -->|on_approve: trigger| RA
    CB -->|log decision| DB
    CB -->|log to sheet| TA

    TA -->|append row| GS
    TA -->|file path reference| FS
```

---

## 2. Control flow — `on_notify` mode

Resume is generated **before** the user is notified. The PDF is attached to the first message.

```mermaid
sequenceDiagram
    participant SCH as APScheduler
    participant SA  as Scout Agent
    participant LLM as Cascading LLM
    participant DB  as PostgreSQL
    participant RA  as Resume Agent
    participant FS  as Filesystem
    participant TG  as Telegram Notifier
    participant USR as User (Telegram)
    participant TA  as Tracker Agent
    participant GS  as Google Sheets

    SCH->>SA: trigger poll (every 10 min / 4 hrs)
    SA->>SA: fetch raw listings (JSearch / Playwright)
    SA->>SA: Stage 1 - title pre-filter (no LLM)
    SA->>LLM: Stage 2 - JD content scoring
    LLM-->>SA: relevance_score, matching_skills, missing_skills
    SA->>SA: Stage 3 - geo filter
    SA->>DB: Stage 4 - dedup check (SHA-256)
    DB-->>SA: new / seen
    SA->>DB: store posting (status=pending)

    alt relevance_score >= 75
        SA->>RA: trigger resume generation
        RA->>LLM: Step 1 - JD keyword extraction
        LLM-->>RA: required_skills, ats_phrases, domain_terms
        RA->>RA: Step 2 - parse base_resume.tex
        RA->>LLM: Step 3 - bullet rewriting
        LLM-->>RA: rewritten .tex
        RA->>RA: Step 4/5 - bullet + skills reordering
        RA->>RA: Step 6 - pdflatex compilation
        RA->>RA: Step 7 - ATS score calculation
        RA->>FS: save {N}_company_role.pdf + .tex
        RA->>DB: store resume metadata (ats_score, file_path)
        RA-->>SA: PDF ready

        SA->>TG: notify(posting, PDF, apply_link)
        TG->>USR: message + PDF attachment + [Accept][Ignore][Details]

        alt User taps Accept
            USR->>TG: callback: accept:{job_id}
            TG->>DB: update status=accepted
            TG->>TA: log_accepted(job_id)
            TA->>GS: append row to YYYY-MM-DD tab
            TA->>TG: confirm message to user
        else User taps Ignore
            USR->>TG: callback: ignore:{job_id}
            TG->>DB: update status=ignored
            TG->>USR: Ignored.
        else User taps Details
            USR->>TG: callback: detail:{job_id}
            TG->>USR: full JD scoring breakdown
        end
    else relevance_score < 75
        SA->>DB: update status=filtered_out
    end
```

---

## 3. Control flow — `on_approve` mode

Resume is generated **only after** the user taps Accept. The PDF is sent as a follow-up message.

```mermaid
sequenceDiagram
    participant SCH as APScheduler
    participant SA  as Scout Agent
    participant LLM as Cascading LLM
    participant DB  as PostgreSQL
    participant TG  as Telegram Notifier
    participant USR as User (Telegram)
    participant RA  as Resume Agent
    participant FS  as Filesystem
    participant TA  as Tracker Agent
    participant GS  as Google Sheets

    SCH->>SA: trigger poll
    SA->>SA: fetch + title filter
    SA->>LLM: JD content scoring
    LLM-->>SA: relevance_score + skills breakdown
    SA->>SA: geo filter + dedup
    SA->>DB: store posting (status=pending)

    alt relevance_score >= 75
        SA->>TG: notify(posting, apply_link) — no PDF yet
        TG->>USR: message + [Accept][Ignore][Details]

        alt User taps Accept
            USR->>TG: callback: accept:{job_id}
            TG->>USR: Generating resume...
            TG->>RA: trigger_resume_agent(job_id)

            RA->>LLM: JD keyword extraction
            LLM-->>RA: keywords
            RA->>RA: parse + rewrite + reorder
            RA->>RA: pdflatex compile
            RA->>RA: ATS score
            RA->>FS: save PDF + .tex
            RA->>DB: store resume metadata

            RA->>TG: resume_ready(job_id, pdf_path, ats_score)
            TG->>USR: Accepted message + PDF attachment + file path + sheet row

            TG->>DB: update status=accepted
            TG->>TA: log_accepted(job_id)
            TA->>GS: append row to YYYY-MM-DD tab

        else User taps Ignore
            USR->>TG: callback: ignore:{job_id}
            TG->>DB: update status=ignored
            TG->>USR: Ignored.
        end
    end
```

---

## 4. Data flow — what moves where

```mermaid
flowchart LR
    subgraph Sources
        A1[JSearch API] -->|JSON: title, company, url, jd_text, location| B
        A2[LinkedIn Guest] -->|HTML parsed: same fields| B
        A3[Naukri Playwright] -->|HTML parsed: same fields| B
        A4[Career Pages] -->|HTML/RSS/JSON parsed: same fields| B
    end

    B[Scout Agent] -->|raw_posting: title, company, url, jd_text, location, source| C1[(PostgreSQL\nraw_postings)]
    B -->|jd_text + resume_text| D[Cascading LLM]
    D -->|scoring_result JSON| B
    B -->|filtered posting: id, title, company, url, apply_link,\nrelevance_score, matching_skills, missing_skills, location| E

    subgraph Resume ["Resume Agent (conditional)"]
        E -->|posting| F1[JD Keyword Extractor]
        F1 -->|keywords JSON| F2[LaTeX Parser]
        F2 -->|intermediate repr| F3[Bullet Rewriter]
        F3 -->|rewritten .tex| F4[pdflatex]
        F4 -->|compiled PDF| F5[ATS Scorer]
    end

    F5 -->|pdf_path, tex_path, ats_score| G[(PostgreSQL\nresumes)]
    F4 -->|PDF + .tex files| H[Filesystem\nresumes/YYYY-MM-DD/\nNN_company_role.*]

    E -->|posting metadata| I[Telegram Notifier]
    F5 -->|PDF binary + ats_score| I

    I -->|Telegram message:\ntitle, company, relevance,\nATS score, skills, apply_link,\nPDF attachment| J[User]

    J -->|callback: accept/ignore/detail| K[Callback Handler]

    K -->|job_id, decision, timestamp| L[(PostgreSQL\ndecisions)]
    K -->|on accept| M[Tracker Agent]

    M -->|row: date, company, role, url, location,\nrelevance, matching_skills, missing_skills,\nATS score, resume_path, decision, timestamp| N[Google Sheets\nYYYY-MM-DD tab]
```

---

## 5. LLM cascade — decision flow per call

```mermaid
flowchart TD
    REQ[LLM Request<br/>from any agent] --> T1

    T1{DeepSeek V3<br/>Tier 1}
    T1 -->|success| RES[Return response]
    T1 -->|RateLimitError / TimeoutError / HTTPError| T2

    T2{Gemini 2.0 Flash<br/>Tier 2}
    T2 -->|success| RES
    T2 -->|failure| T3

    T3{OpenAI GPT-4o-mini<br/>Tier 3}
    T3 -->|success| RES
    T3 -->|failure| T4

    T4{Ollama local<br/>Tier 4 — terminal}
    T4 -->|success| RES
    T4 -->|failure| ERR[AllProvidersFailedError<br/>log + skip posting]
```

**LLM call inventory — which agent calls the LLM for what:**

| Call | Agent | Task | Input tokens (est.) | Output tokens (est.) |
|------|-------|------|--------------------|--------------------|
| 1 | Scout | JD content scoring | ~3,000 (JD + resume) | ~300 (JSON) |
| 2 | Resume | JD keyword extraction | ~2,000 (JD) | ~200 (JSON) |
| 3 | Resume | Bullet rewriting | ~2,500 (bullets + keywords) | ~1,500 (rewritten) |

Total per posting that passes threshold: **~3 LLM calls, ~9,500 tokens**

---

## 6. PostgreSQL schema

```mermaid
erDiagram
    raw_postings {
        uuid        id          PK
        text        title
        text        company
        text        url
        text        apply_link
        text        jd_text
        text        location
        text        source
        char64      dedup_hash  UK
        int         relevance_score
        jsonb       scoring_result
        text        status
        timestamptz discovered_at
    }

    resumes {
        uuid        id          PK
        uuid        posting_id  FK
        text        pdf_path
        text        tex_path
        int         ats_score
        text        base_resume_hash
        timestamptz generated_at
    }

    decisions {
        uuid        id          PK
        uuid        posting_id  FK
        uuid        resume_id   FK
        text        decision
        timestamptz decided_at
        bool        logged_to_sheets
        int         sheet_row_number
    }

    raw_postings ||--o| resumes : "generates"
    raw_postings ||--|| decisions : "results in"
    resumes      ||--o| decisions : "linked to"
```

---

## 7. Filesystem layout — runtime state

```
job-pilot/
│
├── data/
│   ├── base_resume.tex          <- user-provided, fixed base
│   ├── companies.yaml           <- 200-company list
│   └── harvard_verbs.yaml       <- action verb reference
│
├── resumes/                     <- generated output
│   ├── 2026-04-02/
│   │   ├── 01_razorpay_senior_backend_engineer.pdf
│   │   ├── 01_razorpay_senior_backend_engineer.tex
│   │   ├── 02_google_sre.pdf
│   │   └── 02_google_sre.tex
│   └── 2026-04-03/
│       └── ...
│
├── credentials/
│   └── google_service_account.json   <- chmod 600, gitignored
│
├── config.yaml
└── .env                         <- secrets, gitignored
```

**Naming convention:** `{row_number:02d}_{company_slug}_{role_slug}.{ext}`

Row number is assigned atomically when the posting passes the threshold gate and matches the Google Sheet row exactly.

---

## 8. Deployment topology

```mermaid
graph TB
    subgraph Host ["Ubuntu Host — native install"]
        subgraph VEnv ["Python virtualenv: jobpilot-env"]
            SVC1["jobpilot-app.service\n---\nFastAPI port 8000\nAPScheduler\nScout Agent\nResume Agent\nTracker Agent\nCascading LLM Client"]
            SVC2["jobpilot-telegram.service\n---\nTelegram polling / webhook\nCallback Handler"]
        end

        PG["PostgreSQL\nnative, port 5432"]
        TX["TeX Live\npdflatex binary"]
        OL["Ollama\nport 11434\nterminal LLM fallback"]
        FS2["Filesystem\nresumes/ + credentials/"]
    end

    SVC1 <-->|SQL| PG
    SVC1 -->|subprocess| TX
    SVC1 -->|HTTP| OL
    SVC1 <-->|read/write| FS2
    SVC1 <-->|shared DB state| SVC2
    SVC2 <-->|read/write| FS2

    SVC1 -->|HTTPS| EXT1[DeepSeek / Gemini / OpenAI]
    SVC1 -->|HTTPS| EXT2[JSearch RapidAPI]
    SVC1 -->|HTTPS| EXT3[Google Sheets API]
    SVC2 <-->|HTTPS long-poll| EXT4[Telegram Bot API]
    SVC1 -->|Playwright headless Chromium| EXT5[Career Pages + LinkedIn/Naukri fallback]
```

**systemd service summary:**

| Unit | Restart policy | Log command |
|------|---------------|-------------|
| `jobpilot-app.service` | `on-failure`, max 3 | `journalctl --user -u jobpilot-app` |
| `jobpilot-telegram.service` | `on-failure`, max 3 | `journalctl --user -u jobpilot-telegram` |
| `postgresql.service` | `always` | `journalctl -u postgresql` |
| `ollama.service` | `always` | `journalctl -u ollama` |

---

## 9. Data dictionary

| Field | Type | Producer | Consumer | Description |
|-------|------|----------|----------|-------------|
| `dedup_hash` | SHA-256 hex | Scout | Scout | `normalise(title+company+url)` — prevents duplicate notifications |
| `relevance_score` | int 0–100 | LLM via Scout | Scout, Telegram, Sheets | Composite JD–resume match score |
| `matching_skills` | string[] | LLM via Scout | Telegram, Sheets | Skills present in both JD and resume |
| `missing_skills` | string[] | LLM via Scout | Telegram, Sheets | Skills in JD not found in resume |
| `ats_score` | int 0–100 | Resume Agent | Telegram, Sheets | `(JD keywords in resume / total JD keywords) × 100` |
| `pdf_path` | string | Resume Agent | Telegram, Sheets, FS | Absolute path to compiled PDF |
| `sheet_row_number` | int | Tracker Agent | Filesystem naming | Row index in daily tab, 1-based. Prefixes resume filenames |
| `apply_link` | URL | Scout | Telegram | Direct URL to job application page |
| `decision` | enum | Callback Handler | Tracker, DB | `accepted` or `ignored` |
| `base_resume_hash` | SHA-256 hex | Resume Agent | Resume Agent | Hash of base_resume.tex at generation time — audit trail |

---

## 10. Error handling and resilience

| Failure | Detection | Recovery |
|---------|-----------|----------|
| All LLM tiers fail | `AllProvidersFailedError` | Log + skip posting; do not send Telegram notification |
| `pdflatex` compile error | Non-zero exit code | Auto-fix attempt (1 retry); if still failing, log and skip |
| JSearch quota at 190/200 | Request counter in DB | Auto-switch to Playwright fallbacks if enabled in config |
| Google Sheets API failure | HTTP error on append | Retry 3× with exponential backoff; log locally if persistent |
| Telegram send timeout | `python-telegram-bot` exception | Retry 3×; decision stored in DB regardless of Telegram state |
| PostgreSQL down | SQLAlchemy `OperationalError` | App service exits; systemd restarts (max 3 attempts, then alert) |
| Playwright blocked / CAPTCHA | Exception or empty result | Log + skip company for this cycle; continue with remaining companies |
