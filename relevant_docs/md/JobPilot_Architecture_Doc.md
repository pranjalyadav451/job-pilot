# JobPilot — Architecture & Data Flow Design Document

**Source:** [JobPilot PRD v1.1 Definitive](file:///home/pranjal/ai-projects/job-pilot/relevant_docs/md/JobPilot_PRD_v1.1_Definitive.md)
**Date:** 2 April 2026

---

## 1. System Architecture Overview

The system comprises **9 autonomous agents** orchestrated by a **LangGraph state machine**, triggered by **APScheduler** cron jobs, and gated by **Telegram human-in-the-loop** checkpoints. All data is persisted to **PostgreSQL + pgvector** and the **local filesystem**. Metrics are exported to **Prometheus** and visualised in **Grafana**.

```mermaid
graph TB
    subgraph External["External Sources"]
        LI["LinkedIn"]
        NK["Naukri.com"]
        CP["200 Career Pages"]
        RSS["RSS/Atom Feeds"]
        Gmail["Gmail Inbox"]
        Apollo["Apollo.io"]
        Hunter["Hunter.io"]
    end

    subgraph Scheduler["APScheduler Triggers"]
        T10["Every 10 min"]
        T4H["Every 4 hrs"]
        T5M["Every 5 min"]
        T12H["Every 12 hrs<br/>(06:00 / 18:00)"]
        TDAILY["Daily 10 AM"]
        TNIGHTLY["Daily 9 PM"]
    end

    subgraph Agents["9 LangGraph Agent Nodes"]
        Scout["🔍 Scout Agent"]
        Resume["📄 Resume Agent"]
        App["📤 Application Agent"]
        Outreach["📨 Outreach Agent"]
        Tracker["📊 Tracker Agent"]
        EmailIntel["📧 Email Intelligence"]
        FollowUp["⏰ Follow-up Agent"]
        Adaptive["🧬 Adaptive Resume Engine"]
        Analytics["📈 Analytics (Metrics)"]
    end

    subgraph HumanLoop["Telegram HITL"]
        TG["🤖 Telegram Bot"]
        User["👤 Pranjal"]
    end

    subgraph DataStores["Data Persistence"]
        PG["PostgreSQL 17 + pgvector"]
        FS["Filesystem<br/>(resumes/, resume_versions/)"]
        Excel["tracker/job_pipeline.xlsx"]
        Prom["Prometheus TSDB"]
    end

    subgraph Infra["Infrastructure"]
        Grafana["Grafana Dashboards"]
        FastAPI["FastAPI /metrics"]
        Ollama["Ollama (local LLM)"]
    end

    T10 --> Scout
    T4H --> Scout
    T5M --> EmailIntel
    T12H --> Adaptive
    TDAILY --> FollowUp
    TNIGHTLY --> Tracker

    Scout --> TG
    Resume --> TG
    App --> TG
    Outreach --> TG
    FollowUp --> TG
    EmailIntel -.->|"high-priority alerts"| TG
    Adaptive -.->|"weekly report"| TG
    TG <--> User

    Scout --> PG
    Resume --> FS
    App --> PG
    Outreach --> PG
    Tracker --> Excel
    EmailIntel --> PG
    Adaptive --> FS

    LI --> Scout
    NK --> Scout
    CP --> Scout
    RSS --> Scout
    Gmail --> EmailIntel
    Apollo --> Outreach
    Hunter --> Outreach

    Analytics --> FastAPI
    FastAPI --> Prom
    Prom --> Grafana
```

---

## 2. Complete Inter-Agent Data Flow

This diagram traces every data handoff between agents, showing what data is produced, consumed, and where it's stored.

```mermaid
flowchart TD
    subgraph Discovery["Phase 1 — Discovery"]
        S1["Scout Agent polls<br/>LinkedIn / Naukri / Career Pages"]
        S2["Title Pre-filter<br/>(fuzzy match, 0 LLM cost)"]
        S3["JD Scoring<br/>(Cascading LLM)"]
        S4["Geo Filter"]
        S5["Dedup<br/>(SHA-256 hash → PG)"]
        S6["Threshold Gate<br/>(≥ 75 score)"]
    end

    subgraph Notification["Phase 2 — User Decision"]
        N1["Telegram Notification<br/>score + skills + link"]
        N2{"User Decision"}
        N3["Skip → log to PG"]
    end

    subgraph Tailoring["Phase 3 — Resume Tailoring"]
        R1["JD Keyword Extraction<br/>(Tier 1 LLM)"]
        R2["Resume Parsing<br/>(regex on .tex)"]
        R3["Bullet Rewriting<br/>(Claude Sonnet / Tier 1)"]
        R4["Bullet Reordering"]
        R5["Skills Reordering"]
        R6["LaTeX Compilation<br/>(xelatex)"]
        R7["ATS Score Estimation"]
        R8["PDF + .tex → Filesystem"]
    end

    subgraph Submission["Phase 4 — Application"]
        A1{"Submission Method"}
        A2["ATS API<br/>(Lever / Greenhouse / Ashby)"]
        A3["Gmail API<br/>(gmail.send)"]
        A4["Manual Fallback<br/>(URL + reminder)"]
    end

    subgraph PostApp["Phase 5 — Post-Application"]
        O1["Outreach Agent<br/>finds recruiters"]
        O2["Draft 2 variants<br/>(specific + generic)"]
        T1["Tracker Agent<br/>updates Excel"]
    end

    S1 --> S2 --> S3 --> S4 --> S5 --> S6
    S6 -->|"scored posting"| N1
    N1 --> N2
    N2 -->|"Apply ✓"| R1
    N2 -->|"Skip ✗"| N3

    R1 --> R2 --> R3 --> R4 --> R5 --> R6 --> R7 --> R8
    R8 -->|"PDF via Telegram<br/>Confirm / Edit / Regenerate"| A1

    A1 -->|"ATS-hosted"| A2
    A1 -->|"email apply"| A3
    A1 -->|"other portals"| A4

    A2 --> T1
    A3 --> T1
    A4 --> T1

    A2 --> O1
    A3 --> O1
    O1 --> O2
    O2 -->|"Telegram approval"| T1
```

---

## 3. Cascading LLM Routing Strategy

Every LLM call in the system follows this waterfall. On failure at any tier, the request cascades to the next.

```mermaid
flowchart LR
    Request["LLM Request"] --> T1

    subgraph Cascade["Cascading Client"]
        T1["Tier 1: DeepSeek V3<br/>50M tokens/day FREE"]
        T1 -->|"rate limit / timeout"| T1b["Tier 1b: Qwen 2.5<br/>1M tokens/month FREE"]
        T1b -->|"rate limit / timeout"| T2["Tier 2: Gemini Flash<br/>15 RPM free"]
        T2 -->|"rate limit / timeout"| T2b["Tier 2b: GPT-4o-mini<br/>$0.15/M input"]
        T2b -->|"rate limit / timeout"| T3["Tier 3: Ollama Local<br/>Qwen 2.5 7B / $0"]
    end

    T1 -->|"success"| Response["Response"]
    T1b -->|"success"| Response
    T2 -->|"success"| Response
    T2b -->|"success"| Response
    T3 -->|"success"| Response
    T3 -->|"all failed"| Error["AllProvidersFailedError"]

    Premium["Premium Path<br/>(resume rewriting,<br/>deliberation only)"] --> Claude["Claude Sonnet 4.5<br/>$3/M input"]
    Claude -->|"fallback"| Cascade
```

### LLM Routing by Task

| Task                        | Primary Route    | Fallback                |
| --------------------------- | ---------------- | ----------------------- |
| JD scoring                  | Cascade (Tier 1) | Full cascade            |
| JD keyword extraction       | Cascade (Tier 1) | Full cascade            |
| Bullet rewriting            | Claude Sonnet    | Cascade (lower quality) |
| Outreach drafts             | Cascade (Tier 1) | Full cascade            |
| Email classification        | Cascade (Tier 1) | Full cascade            |
| Follow-up drafting          | Cascade (Tier 1) | Full cascade            |
| Tournament: Keyword Agent   | Cascade (Tier 1) | Full cascade            |
| Tournament: Narrative Agent | Claude Sonnet    | Cascade (lower quality) |
| Tournament: Structure Agent | Cascade (Tier 1) | Full cascade            |
| Tournament: Market Agent    | Cascade (Tier 1) | Full cascade            |
| Tournament: Judge           | Cascade (Tier 1) | Full cascade            |
| Deliberation: Mediator      | Claude Sonnet    | Cascade (lower quality) |

---

## 4. Adaptive Resume Engine — Tournament & Feedback Loop

This is the architecturally most distinctive subsystem: a closed feedback loop that evolves the base resume every 12 hours based on actual acceptance/rejection data.

```mermaid
flowchart TD
    subgraph Trigger["Trigger: APScheduler 06:00 / 18:00 IST"]
        CHECK{"≥ 10 applications<br/>with known outcomes?"}
    end

    subgraph DataCollection["Feedback Data Collection"]
        POS["Positive signals<br/>(interview, recruiter reply)<br/>weight: +0.6 to +1.0"]
        NEG["Negative signals<br/>(no response 14d, rejection)<br/>weight: -0.3 to -0.5"]
        JDS["Last 12 hrs of<br/>discovered JDs"]
    end

    subgraph Tournament["4-Agent Tournament (parallel)"]
        KW["🔑 Keyword Agent<br/>ATS keyword density<br/>(Tier 1)"]
        NA["📝 Narrative Agent<br/>verb strength + impact<br/>(Claude Sonnet)"]
        ST["📐 Structure Agent<br/>bullet/section ordering<br/>(Tier 1)"]
        MK["📊 Market Agent<br/>emerging JD trends<br/>(Tier 1)"]
    end

    subgraph Judging["Judge Scoring"]
        JU["⚖️ Judge Agent<br/>(Tier 1 or Claude)"]
        SC["Score on 5 dimensions:<br/>keyword (25%) | phrasing (25%)<br/>structure (20%) | ATS (15%)<br/>truthfulness (15%, 2x weight)"]
    end

    subgraph Deliberation["Conditional Deliberation"]
        MARGIN{"Top 2 margin<br/>< 5 points?"}
        MED["🤝 Mediator Agent<br/>(Claude Sonnet)<br/>merge top 2 proposals"]
        RESCORE["Judge re-scores<br/>merged vs originals"]
    end

    subgraph Output["Output"]
        COMPILE["Compile winning .tex<br/>with xelatex"]
        VALIDATE{"Compilation<br/>success?"}
        SAVE["Save as new base<br/>resume_versions/v{N+1}"]
        KEEP["Keep previous version<br/>(no regression)"]
        META["Write metadata.json<br/>+ changelog entry"]
    end

    CHECK -->|"Yes"| POS
    CHECK -->|"No (cold start)"| SKIP["Use original resume as-is"]
    POS --> KW & NA & ST
    NEG --> KW & NA & ST
    JDS --> MK

    KW --> JU
    NA --> JU
    ST --> JU
    MK --> JU
    JU --> SC --> MARGIN

    MARGIN -->|"Yes (~20-30%)"| MED --> RESCORE --> COMPILE
    MARGIN -->|"No"| COMPILE

    COMPILE --> VALIDATE
    VALIDATE -->|"✓"| SAVE --> META
    VALIDATE -->|"✗"| KEEP
```

### The Closed Feedback Loop

```mermaid
flowchart LR
    BASE["Base Resume<br/>(resume_versions/current)"]
    TAILOR["Resume Agent<br/>tailors per JD"]
    SUBMIT["Application Agent<br/>submits"]
    RESPONSE["Email Intelligence<br/>classifies response"]
    FEEDBACK["Feedback Store<br/>(PostgreSQL)"]
    ENGINE["Adaptive Resume<br/>Engine (12hr cycle)"]

    BASE -->|"provides template"| TAILOR
    TAILOR -->|"tailored .tex/.pdf"| SUBMIT
    SUBMIT -->|"tracks outcome"| RESPONSE
    RESPONSE -->|"outcome + weight"| FEEDBACK
    FEEDBACK -->|"positive/negative signals"| ENGINE
    ENGINE -->|"improved version"| BASE

    style BASE fill:#4CAF50,color:#fff
    style ENGINE fill:#FF9800,color:#fff
```

---

## 5. Email Intelligence & Follow-up Data Flow

```mermaid
flowchart TD
    subgraph EmailPipeline["Email Intelligence Agent (every 5 min)"]
        POLL["Gmail API poll<br/>(historyId-based incremental sync)"]
        DOMAIN["Stage 1: Domain Filter<br/>(company domains + ATS domains)<br/>0 LLM cost"]
        KEYWORD["Stage 2: Keyword Pre-filter<br/>(regex: interview, rejection, etc.)<br/>0 LLM cost"]
        LLM_CLASS["Stage 3: LLM Classification<br/>(ambiguous emails only)<br/>Cascading client"]
    end

    subgraph Classification["8 Classification Categories"]
        INT["interview_invite"]
        REC["recruiter_reply"]
        ASS["assessment_request"]
        OFF["offer"]
        FUN["follow_up_needed"]
        ACK["acknowledgement"]
        REJ["rejection"]
        UNR["unrelated → discard"]
    end

    subgraph Actions["Downstream Actions"]
        PG_UP["PostgreSQL:<br/>update application status"]
        TG_ALERT["Telegram alert<br/>(high-priority only)"]
        TG_SILENT["Silent update<br/>(ack, rejection)"]
        TRACKER_UP["Tracker Agent:<br/>update Excel row"]
        FEEDBACK["Feedback Store:<br/>outcome weight for<br/>Adaptive Engine"]
        LABEL["Gmail: apply label<br/>JobPilot/{category}"]
        CONF{"Confidence<br/>≥ 0.85?"}
    end

    POLL --> DOMAIN --> KEYWORD --> LLM_CLASS

    LLM_CLASS --> INT & REC & ASS & OFF & FUN & ACK & REJ & UNR

    INT -->|"weight: 1.0"| CONF
    REC -->|"weight: 0.8"| CONF
    ASS -->|"weight: 0.9"| CONF
    OFF -->|"weight: 1.0"| CONF
    FUN -->|"weight: 0.6"| CONF
    ACK -->|"weight: 0.0"| PG_UP
    REJ -->|"weight: -0.5"| PG_UP

    CONF -->|"≥ 0.85"| PG_UP
    CONF -->|"< 0.85"| TG_ALERT

    PG_UP --> TRACKER_UP
    PG_UP --> FEEDBACK
    PG_UP --> LABEL
    INT --> TG_ALERT
    OFF --> TG_ALERT
    FUN --> TG_ALERT
    ACK --> TG_SILENT
    REJ --> TG_SILENT
```

### Follow-up Agent

```mermaid
flowchart LR
    TRIGGER["Daily 10:00 AM IST"] --> QUERY["Query PostgreSQL:<br/>submitted > 7 days,<br/>no response,<br/>no follow-up sent"]

    QUERY --> DAY7["Day 7 candidates"]
    QUERY --> DAY14["Day 14 candidates<br/>(follow_up_1 sent)"]

    DAY7 --> DRAFT7["LLM drafts 4-sentence<br/>follow-up (Tier 1)"]
    DAY14 --> DRAFT14["LLM drafts 2-sentence<br/>final follow-up (Tier 1)"]

    DRAFT7 --> TG7["Telegram:<br/>Send / Edit / Skip / Delay"]
    DRAFT14 --> TG14["Telegram:<br/>Send / Edit / Skip / Delay"]

    TG7 -->|"Send ✓"| GMAIL7["Gmail API<br/>gmail.send"]
    TG14 -->|"Send ✓"| GMAIL14["Gmail API<br/>gmail.send"]

    GMAIL7 --> UPDATE7["Tracker: follow_up_1_sent = true"]
    GMAIL14 --> UPDATE14["Tracker: follow_up_2_sent = true<br/>status = 'No response — closed'"]

    CHECK{{"Guard: skip if<br/>• rejection received<br/>• interview scheduled<br/>• outreach already sent<br/>• batch limit (10/day)"}}

    QUERY --> CHECK
    CHECK -->|"pass"| DAY7 & DAY14
```

---

## 6. Outreach Agent Data Flow

```mermaid
flowchart TD
    TRIGGER["Post-application trigger"] --> FIND["Recruiter Discovery"]

    subgraph Discovery["3-Phase Recruiter Discovery"]
        AP["Phase 1: Apollo.io<br/>(50 free credits/mo)<br/>search by company + title"]
        HU["Phase 2: Hunter.io<br/>(25 free searches/mo)<br/>email lookup by name + domain"]
        LI["Phase 3: LinkedIn Scraping<br/>(Playwright fallback)<br/>30 profile views/day"]
    end

    FIND --> AP
    AP -->|"no email found"| HU
    HU -->|"credits exhausted"| LI

    AP --> CONTACTS["Recruiter Contacts<br/>(name, title, email, LinkedIn)"]
    HU --> CONTACTS
    LI --> CONTACTS

    CONTACTS --> STORE["Store in PostgreSQL"]
    CONTACTS --> DRAFT["Draft 2 Variants (Tier 1 LLM)"]

    subgraph Variants["Message Variants"]
        VA["Variant A: Specific<br/>(references resume accomplishments<br/>matched to JD)"]
        VB["Variant B: Generic<br/>(role + interest +<br/>years of experience)"]
    end

    DRAFT --> VA & VB

    VA --> TG["Telegram Approval:<br/>Send A / Send B / Edit / Skip"]
    VB --> TG

    TG -->|"approved"| SEND["Send via Gmail API"]
    SEND --> TRACKER["Tracker Agent:<br/>outreach_status, variant, date"]

    LIMITS{{"Rate Limits:<br/>• 20 LinkedIn requests/day<br/>• 30 emails/day<br/>• 2 recruiters/company max<br/>• 14-day cooldown/person"}}

    FIND --> LIMITS
```

---

## 7. Data Store Topology

```mermaid
flowchart TD
    subgraph PG["PostgreSQL 17 + pgvector"]
        JOBS["jobs table<br/>posting metadata,<br/>JD text, scores"]
        APPS["applications table<br/>status, dates, methods,<br/>follow-up flags"]
        DEDUP["dedup_hashes<br/>SHA-256 index"]
        RESUMES_META["resumes_metadata<br/>ATS score, base version,<br/>keywords matched"]
        RECRUITERS["recruiters table<br/>contacts, outreach<br/>status, cooldowns"]
        EMAIL_LOG["email_classifications<br/>class, confidence,<br/>matched application"]
        FEEDBACK["feedback_store<br/>outcome weights,<br/>signals for engine"]
        EMBEDDINGS["pgvector embeddings<br/>resume/JD cosine<br/>similarity"]
        HISTORY_ID["gmail_sync_state<br/>historyId for<br/>incremental sync"]
        CHECKPOINTS["langgraph_checkpoints<br/>HITL state persistence"]
    end

    subgraph FS["Filesystem"]
        TAILORED["resumes/{company}/<br/>{role}_{date}.tex/.pdf<br/>+ metadata.json"]
        VERSIONS["resume_versions/<br/>v{N}_{timestamp}.tex/.pdf<br/>v{N}_meta.json<br/>changelog.md<br/>current → symlink"]
        TEMPLATES["templates/<br/>base_resume.tex<br/>cover_letter.tex"]
        DATA["data/<br/>harvard_action_verbs.yaml<br/>companies.yaml<br/>applicant_profile.yaml"]
        CREDS["credentials/<br/>gmail_oauth.json<br/>gmail_token.json"]
    end

    subgraph External_Stores["External"]
        EXCEL["tracker/<br/>job_pipeline.xlsx"]
        PROM["Prometheus TSDB<br/>(90-day retention)"]
    end

    Scout -->|"write"| JOBS
    Scout -->|"check"| DEDUP
    Scout -->|"embed"| EMBEDDINGS
    Resume_Agent["Resume Agent"] -->|"read"| VERSIONS
    Resume_Agent -->|"write"| TAILORED
    Resume_Agent -->|"write"| RESUMES_META
    App_Agent["Application Agent"] -->|"write"| APPS
    Outreach_Agent["Outreach Agent"] -->|"write"| RECRUITERS
    Email_Agent["Email Intelligence"] -->|"write"| EMAIL_LOG
    Email_Agent -->|"update"| APPS
    Email_Agent -->|"write"| FEEDBACK
    Email_Agent -->|"read/write"| HISTORY_ID
    FollowUp_Agent["Follow-up Agent"] -->|"read"| APPS
    FollowUp_Agent -->|"update"| APPS
    Adaptive_Engine["Adaptive Engine"] -->|"read"| FEEDBACK
    Adaptive_Engine -->|"write"| VERSIONS
    Tracker_Agent["Tracker Agent"] -->|"write"| EXCEL
    Analytics_Agent["Analytics"] -->|"write"| PROM
    Orchestrator["LangGraph Orchestrator"] -->|"read/write"| CHECKPOINTS
```

---

## 8. Human-in-the-Loop (HITL) Checkpoint Map

Every irreversible action in the system requires explicit user approval via Telegram. This diagram shows all HITL checkpoints and what happens at each.

```mermaid
flowchart TD
    subgraph Checkpoints["All HITL Checkpoints"]
        CP1["🔍 Job Notification<br/><b>Apply</b> / Skip / Details"]
        CP2["📄 Resume Review<br/><b>Confirm & Apply</b> / Edit / Regenerate"]
        CP3["📤 Application Confirm<br/><b>Confirm</b> / Cancel / View Resume"]
        CP4["📨 Outreach Approval<br/><b>Send A</b> / Send B / Edit / Skip"]
        CP5["⏰ Follow-up Approval<br/><b>Send</b> / Edit / Skip / Delay 3d"]
        CP6["📧 Low-confidence Email<br/><b>Verify</b> classification"]
    end

    CP1 -->|"Apply"| RESUME["Resume Agent starts"]
    CP1 -->|"Skip"| LOG1["Log skip → Tracker"]

    CP2 -->|"Confirm"| APPLY["Application Agent starts"]
    CP2 -->|"Edit"| EDIT["User sends edit instructions<br/>→ Resume Agent re-tailors"]
    CP2 -->|"Regenerate"| REGEN["Resume Agent re-runs<br/>from scratch"]

    CP3 -->|"Confirm"| SUBMIT["Submit via ATS/Email/Manual"]

    CP4 -->|"Send A/B"| OUTREACH["Send via Gmail API"]
    CP4 -->|"Edit"| EDIT_MSG["User edits → re-draft"]
    CP4 -->|"Skip"| LOG4["Log skip → Tracker"]

    CP5 -->|"Send"| FOLLOWUP["Send via Gmail API"]
    CP5 -->|"Delay"| DELAY["Reschedule +3 days"]
    CP5 -->|"Skip"| LOG5["Log skip → Tracker"]

    CP6 -->|"Confirm"| AUTO_UPDATE["Apply classification"]
    CP6 -->|"Override"| MANUAL["User provides correct class"]

    style CP1 fill:#2196F3,color:#fff
    style CP2 fill:#4CAF50,color:#fff
    style CP3 fill:#FF9800,color:#fff
    style CP4 fill:#9C27B0,color:#fff
    style CP5 fill:#F44336,color:#fff
    style CP6 fill:#607D8B,color:#fff
```

---

## 9. Monitoring & Observability

```mermaid
flowchart LR
    subgraph App["JobPilot FastAPI"]
        COUNTERS["Counters:<br/>jobs_discovered_total<br/>applications_submitted_total<br/>llm_calls_total<br/>llm_failures_total<br/>..."]
        GAUGES["Gauges:<br/>active_applications<br/>pending_followups<br/>acceptance_rate<br/>llm_cascade_depth"]
        HISTOGRAMS["Histograms:<br/>resume_generation_seconds<br/>scout_cycle_seconds<br/>llm_latency_seconds"]
        ENDPOINT["/metrics endpoint"]
    end

    subgraph Prom["Prometheus (Docker)"]
        SCRAPE["Scrape every 15s<br/>90-day retention"]
        RULES["Alert Rules:<br/>• Scout stalled 2hrs<br/>• All LLMs failing<br/>• Acceptance < 10%<br/>• Email agent disconnected<br/>• Application backlog > 5"]
    end

    subgraph Grafana_Dash["Grafana Dashboards (Docker)"]
        D1["📊 Application Funnel<br/>discovered → applied → responded → offer"]
        D2["📈 Daily Activity<br/>jobs/hour, apps/day"]
        D3["🧬 Resume Evolution<br/>acceptance rate + version annotations"]
        D4["💰 LLM Cost & Health<br/>calls by provider, latency p50/p95"]
        D5["📨 Outreach Effectiveness<br/>response rate by variant"]
    end

    COUNTERS --> ENDPOINT
    GAUGES --> ENDPOINT
    HISTOGRAMS --> ENDPOINT
    ENDPOINT --> SCRAPE
    SCRAPE --> RULES
    RULES -->|"fire alert"| TG["📱 Telegram Alert"]
    SCRAPE --> D1 & D2 & D3 & D4 & D5
```

---

## 10. End-to-End Sequence — Happy Path

This shows the complete lifecycle of a single job posting from discovery to follow-up, with every agent interaction and data store touchpoint.

```mermaid
sequenceDiagram
    participant Sched as APScheduler
    participant Scout as Scout Agent
    participant PG as PostgreSQL
    participant TG as Telegram
    participant User as 👤 Pranjal
    participant Resume as Resume Agent
    participant FS as Filesystem
    participant App as Application Agent
    participant Outreach as Outreach Agent
    participant Apollo as Apollo.io
    participant Gmail as Gmail API
    participant Tracker as Tracker Agent
    participant Excel as Excel
    participant EmailIntel as Email Intelligence
    participant FollowUp as Follow-up Agent
    participant Adaptive as Adaptive Engine
    participant Prom as Prometheus

    Sched->>Scout: trigger (every 10 min)
    Scout->>Scout: scrape LinkedIn / career page
    Scout->>Scout: title pre-filter (fuzzy match)
    Scout->>Scout: JD scoring (Cascading LLM)
    Scout->>PG: dedup check (SHA-256 hash)
    PG-->>Scout: not seen before
    Scout->>PG: store job posting + score
    Scout->>TG: send notification (score: 82/100)
    TG->>User: 🔍 Senior BE @ Razorpay — Apply?
    User->>TG: [Apply ✓]

    TG->>Resume: trigger with job_id
    Resume->>PG: fetch JD text
    Resume->>FS: read base resume (current symlink)
    Resume->>Resume: extract JD keywords (Tier 1 LLM)
    Resume->>Resume: rewrite bullets (Claude Sonnet)
    Resume->>Resume: reorder bullets + skills
    Resume->>Resume: compile LaTeX → PDF
    Resume->>Resume: estimate ATS score (84%)
    Resume->>FS: save .tex + .pdf + metadata
    Resume->>PG: store resume metadata
    Resume->>TG: send PDF + diff summary
    TG->>User: 📄 ATS: 84% — Confirm?
    User->>TG: [Confirm & Apply]

    TG->>App: trigger with job_id + resume_path
    App->>App: detect method (Lever API)
    App->>TG: 📤 Method: Lever API — Confirm?
    TG->>User: [Confirm ✓]
    User->>TG: Confirm
    App->>App: POST /postings/{id}/apply
    App->>PG: update application status → submitted
    App->>Tracker: trigger update
    Tracker->>Excel: write/update row (30 columns)

    App->>Outreach: trigger recruiter search
    Outreach->>Apollo: search recruiters at Razorpay
    Apollo-->>Outreach: Ananya Sharma (Sr. Tech Recruiter)
    Outreach->>PG: store recruiter contact
    Outreach->>Outreach: draft 2 variants (Tier 1 LLM)
    Outreach->>TG: 📨 Variant A / B — Send?
    TG->>User: [Send Variant A]
    Outreach->>Gmail: send email to recruiter
    Outreach->>Tracker: update outreach columns

    Note over Sched,Prom: ⏳ 9 days later...

    Sched->>EmailIntel: trigger (every 5 min)
    EmailIntel->>Gmail: poll (historyId sync)
    Gmail-->>EmailIntel: new email from Razorpay
    EmailIntel->>EmailIntel: domain filter → match
    EmailIntel->>EmailIntel: keyword filter → "schedule a call"
    EmailIntel->>EmailIntel: classify → interview_invite (0.94)
    EmailIntel->>PG: update status → Interview Scheduled
    EmailIntel->>PG: store feedback (weight: +1.0)
    EmailIntel->>Gmail: label → JobPilot/Interview Invites
    EmailIntel->>TG: 🎉 Interview invite from Razorpay!
    EmailIntel->>Tracker: update Excel row

    Note over Sched,Prom: ⏳ At 06:00 or 18:00 IST...

    Sched->>Adaptive: trigger (every 12 hrs)
    Adaptive->>PG: fetch positive signals (interviews, replies)
    Adaptive->>PG: fetch negative signals (no response 14d+)
    Adaptive->>Adaptive: 4 agents run in parallel
    Adaptive->>Adaptive: Judge scores all proposals
    Adaptive->>Adaptive: Mediator merges top 2 (if margin < 5)
    Adaptive->>Adaptive: compile winner
    Adaptive->>FS: save new version + update symlink
    Adaptive->>Prom: emit resume_versions_total metric

    Note over Sched,Prom: All agents continuously emit metrics
    Scout->>Prom: jobs_discovered_total++
    App->>Prom: applications_submitted_total++
    EmailIntel->>Prom: emails_classified_total++
```

---

## 11. Scheduling Summary

| Trigger              | Frequency          | Agent(s)               | What Happens                              |
| -------------------- | ------------------ | ---------------------- | ----------------------------------------- |
| Scout Tier 1         | Every 10 min       | Scout                  | Poll top 30 companies + LinkedIn + Naukri |
| Scout Tier 2         | Every 4 hrs        | Scout                  | Poll remaining 170 companies              |
| Email poll           | Every 5 min        | Email Intelligence     | Gmail incremental sync + classify         |
| Resume evolution     | 06:00 + 18:00 IST  | Adaptive Resume Engine | 4-agent tournament → new base resume      |
| Follow-up check      | Daily 10:00 AM IST | Follow-up Agent        | Draft day-7 and day-14 follow-ups         |
| Daily summary        | Daily 9:00 PM IST  | Tracker Agent          | Pipeline stats → Telegram                 |
| Weekly resume report | Sunday 9:00 PM IST | Adaptive Resume Engine | Evolution summary → Telegram              |
| Prometheus scrape    | Every 15 sec       | Analytics (passive)    | `/metrics` scraped by Prometheus          |

---

## 12. Technology Deployment Map

```mermaid
flowchart TD
    subgraph BareMetal["Bare Metal (Ubuntu 24.04)"]
        APP["JobPilot App<br/>(Python 3.12 + FastAPI)"]
        OLLAMA["Ollama<br/>(Qwen 2.5 7B on GTX 1650)"]
        VENV[".venv<br/>(all Python deps)"]
        XELATEX["XeLaTeX<br/>(TeX Live)"]
        PLAYWRIGHT["Playwright<br/>(Chromium, headless)"]
    end

    subgraph Docker["Docker Compose"]
        PG["PostgreSQL 17<br/>+ pgvector<br/>(1.5 GB RAM limit)"]
        PROM["Prometheus v3.10.0<br/>(512 MB limit)"]
        GRAF["Grafana 12.4.2<br/>(256 MB limit)"]
    end

    subgraph Cloud["Cloud APIs"]
        DS["DeepSeek V3"]
        QW["Qwen (DashScope)"]
        GEM["Gemini Flash"]
        GPT["GPT-4o-mini"]
        CLAUDE["Claude Sonnet 4.5"]
        APOLLO["Apollo.io"]
        HUNTER["Hunter.io"]
        GMAIL_API["Gmail API"]
        TG_API["Telegram Bot API"]
    end

    APP --> PG
    APP --> OLLAMA
    APP --> XELATEX
    APP --> PLAYWRIGHT
    APP --> PROM
    APP --> DS & QW & GEM & GPT & CLAUDE
    APP --> APOLLO & HUNTER & GMAIL_API & TG_API
    PROM --> GRAF
```

> [!NOTE]
> **RAM Budget (16 GB total):** PostgreSQL ~1.5 GB, Ollama ~4 GB (VRAM) + ~1 GB (RAM), Prometheus ~200 MB, Grafana ~200 MB, Playwright ~600 MB (2 contexts max), JobPilot app ~500 MB, OS + buffers ~8 GB. Fits within 16 GB with 8 GB swap as safety net.
