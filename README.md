# DevSynt Hiring Automation — AI-Powered Recruitment Pipeline

**Built with n8n, Supabase, and Google Gemini** · Internship Practical Evaluation Project

An end-to-end automation system that catches job applications from two different channels, screens them against job requirements using AI, and routes candidates to the right outcome — all without manual intervention.

---

## Table of Contents

- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [Architecture Overview](#architecture-overview)
- [Tech Stack](#tech-stack)
- [Workflow 1 — Application Intake, Screening & Acknowledgment](#workflow-1--application-intake-screening--acknowledgment)
- [Workflow 2 — Review & Routing](#workflow-2--review--routing)
- [Database Schema](#database-schema)
- [AI Scoring Logic](#ai-scoring-logic)
- [Error Handling](#error-handling)
- [Screenshots](#screenshots)
- [Setup & Reproduction](#setup--reproduction)
- [Repository Contents](#repository-contents)
- [Production Readiness Notes](#production-readiness-notes)
- [Author](#author)

---

## The Problem

Job applications arrive through multiple, disconnected channels (email, web forms), and reviewing each one manually — reading the resume, judging fit, and replying — doesn't scale. The result is typically:

- Applications sitting unread for days
- Inconsistent, reviewer-dependent screening quality
- Strong candidates lost to slower-moving competitors simply due to response delay
- No centralized, structured record of who applied, for what, and how they were assessed

## The Solution

Two connected n8n workflows automate the entire intake-to-decision pipeline:

1. **Workflow 1** catches applications the instant they arrive (email or form), normalizes the data, stores it in a database, sends an immediate acknowledgment, and scores the resume against the job requirements using AI.
2. **Workflow 2** runs on a schedule, reads scored candidates, and automatically routes each one to the correct outcome: shortlist, manual human review, or a respectful rejection — updating the database so nothing is processed twice.

---

## Architecture Overview

```mermaid
flowchart TD
    A[Email to careers@devsynt.com] -->|IMAP Trigger| C[Normalize Data]
    B[Google Form Submission] -->|Sheets Trigger| C
    C --> D[Merge & Validate]
    D --> E[(Supabase: candidates table)]
    D --> F[Send Acknowledgment Email]
    D --> G[AI Resume Scoring - Gemini]
    G --> H[Classify: Strong / Average / Weak]
    H --> E

    E -.->|every 15 min, status=pending_review| I[Workflow 2: Get Pending Candidates]
    I --> J{Route by Classification}
    J -->|Strong| K[Shortlist Email] --> N[(Update status: notified)]
    J -->|Average| L[Notify Hiring Manager] --> O[(Update status: manual_review)]
    J -->|Weak| M[Rejection Email] --> N
```

The two workflows are **not directly wired together** in n8n — they are decoupled and communicate only through the shared Supabase `candidates` table. Workflow 1 writes rows; Workflow 2 independently reads and acts on them. This means either workflow can be modified, tested, or temporarily disabled without breaking the other.

---

## Tech Stack

| Layer | Tool | Purpose |
|---|---|---|
| Automation engine | [n8n](https://n8n.io) (self-hosted via Docker) | Orchestrates the entire pipeline |
| Database | [Supabase](https://supabase.com) (PostgreSQL) | Stores candidate records, scores, and status |
| AI scoring | Google Gemini API (`gemini-2.5-flash`) | Reads resumes and produces an explainable ATS score |
| Email intake | IMAP | Catches applications sent by email |
| Form intake | Google Sheets (linked to Google Forms) | Catches applications submitted via web form |
| Email delivery | SMTP | Sends acknowledgment, shortlist, rejection, and manager-notification emails |

---

## Workflow 1 — Application Intake, Screening & Acknowledgment

**File:** `workflow1_application_intake.json`

| Node | Type | What it does |
|---|---|---|
| Email Applications (IMAP) | IMAP Trigger | Watches the inbox continuously; fires on any new email |
| Google Form Submissions (Sheets) | Google Sheets Trigger | Polls the linked response spreadsheet every minute for new rows |
| Normalize Email Data | Code | Parses sender name/email/subject from the raw email into a common shape |
| Normalize Form Data | Code | Maps the form's column headers into the same common shape |
| Merge Applications | Merge | Combines both intake sources into a single stream |
| Validate Required Fields | Filter | Drops/flags records missing a name or email, so malformed data doesn't break the pipeline |
| Insert Candidate (Supabase) | Supabase — Create | Writes the normalized record into the `candidates` table with `status = pending_review` |
| Send Acknowledgment Email | Email Send | Immediately confirms receipt to the candidate |
| AI Resume Scoring (Gemini) | HTTP Request | Sends the resume + job requirements to Gemini, requests a structured JSON score |
| Parse AI Score & Classify | Code | Parses the AI's response and applies scoring bands (see below) |
| Update Candidate Score (Supabase) | Supabase — Update | Writes `ats_score` and `classification` back to the candidate's row |

**Outcome:** every application — regardless of source — ends up as a structured, scored row in Supabase within seconds of arriving, and the candidate has already received a reply.

---

## Workflow 2 — Review & Routing

**File:** `workflow2_review_routing.json`

| Node | Type | What it does |
|---|---|---|
| Every 15 Minutes | Schedule Trigger | Runs the review cycle on a fixed interval |
| Get Pending Candidates (Supabase) | Supabase — Get All | Reads all rows where `status = pending_review` |
| Has Classification? | Filter | Skips any candidate whose AI scoring hasn't completed yet (safety check) |
| Route by Classification | Switch | Branches into three paths based on the `classification` field |
| Send Shortlist Email | Email Send | **Strong** → sends an HTML confirmation/shortlist email |
| Notify Hiring Manager | Email Send | **Average** → sends candidate details to a human for manual review (never auto-decided) |
| Send Rejection Email | Email Send | **Weak** → sends a respectful rejection |
| Update Status (×3) | Supabase — Update | Marks the row `notified` (Strong/Weak) or `manual_review` (Average) so it's never processed twice |

**Outcome:** every scored candidate reaches a final, correct outcome automatically, with **borderline candidates always escalated to a human** rather than auto-rejected — a deliberate design choice to avoid unfair automated decisions.

---

## Database Schema

```sql
create table candidates (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  email text not null,
  phone text,
  source text not null check (source in ('email', 'form')),
  role_applied text,
  resume_text text,
  resume_url text,
  applied_at timestamptz default now(),
  ats_score numeric,
  classification text check (classification in ('Strong', 'Average', 'Weak')),
  status text default 'pending_review' check (status in ('pending_review', 'notified', 'manual_review')),
  manager_notes text,
  unique (email, role_applied)
);
```

The `unique (email, role_applied)` constraint prevents duplicate applications for the same role from creating duplicate rows.

---

## AI Scoring Logic

Each resume is evaluated by Gemini against the specific job's requirements and returned as structured JSON (`{"ats_score": number, "reasoning": string}`), then classified using the following bands:

| ATS Score | Classification | Outcome |
|---|---|---|
| 75–100 | **Strong** | Auto-shortlisted |
| 50–74 | **Average** | Escalated to hiring manager for human judgment |
| Below 50 | **Weak** | Respectfully declined |

These thresholds are configurable in the "Parse AI Score & Classify" code node and were chosen to keep the automation conservative — only clearly weak or clearly strong candidates are auto-decided; anything in between goes to a human.

---

## Error Handling

- **Missing required fields** (name/email) are filtered out before ever reaching the database, rather than causing a crash mid-pipeline.
- **AI scoring failures** (rate limits, malformed responses) are caught and the candidate is safely flagged for manual review with a score of 0, rather than the pipeline halting or silently dropping the candidate.
- **Duplicate applications** are prevented at the database level via a unique constraint on `(email, role_applied)`.
- **Retry logic** is enabled on all external API calls (Supabase, SMTP, Gemini) to absorb transient network failures.
- **Idempotent status updates** — each candidate's `status` field is updated immediately after action, ensuring Workflow 2 never emails or re-processes the same candidate twice, even across multiple scheduled runs.

---

## Screenshots

> Screenshots below are illustrative placeholders — replace with your own exported images in the `/screenshots` folder.

**Workflow 1 — full canvas, successful execution:**
![Workflow 1 execution](./screenshots/workflow1-execution_hiring_application_intake.png)

**Workflow 2 — full canvas, successful execution:**
![Workflow 2 execution](./screenshots/workflow2-execution_hiring_review_routing.png)

**Supabase `candidates` table with live data:**
![Supabase table](./screenshots/supabase-table.png)

**Sample acknowledgment email received:**
![Acknowledgment email](./screenshots/acknowledgment-email.jpg)

**AI scoring output (Gemini response):**
![AI scoring output](./screenshots/ai-scoring-output.jpg)

---

## Setup & Reproduction

1. Install [Docker Desktop](https://www.docker.com/products/docker-desktop/).
2. Run n8n:
   ```bash
   docker volume create n8n_data
   docker run -d --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n n8nio/n8n
   ```
3. Open `http://localhost:5678` and create your local account.
4. Import `workflow1_application_intake.json` and `workflow2_review_routing.json` from the Workflows list page.
5. Create a [Supabase](https://supabase.com) project and run the SQL schema above.
6. Configure credentials in each workflow: Supabase (legacy `service_role` key), IMAP, Google Sheets OAuth, SMTP, and a Gemini API key (free tier available via [Google AI Studio](https://aistudio.google.com/apikey)).
7. Test each node individually, then publish both workflows to run automatically.

---

## Repository Contents

```
├── workflow1_application_intake.json   # Workflow 1 — importable n8n workflow
├── workflow2_review_routing.json       # Workflow 2 — importable n8n workflow
├── README.md                           # This file
└── screenshots/                        # Execution screenshots
```

---

## Production Readiness Notes

This project is a working demonstration built for an internship evaluation. Moving it to real production use would require:

- Dedicated email addresses (not a personal inbox) for intake, sending, and manager notifications
- A transactional email service (SendGrid/Postmark/SES) instead of Gmail SMTP
- Hosting n8n on a persistent server rather than a personal machine, for true 24/7 uptime
- Row Level Security re-enabled on the Supabase table with proper access policies
- A paid AI API tier to handle real applicant volume
- Proper secrets management instead of locally stored credentials
- Active failure alerting (e.g. Slack/email) rather than silent fallback-to-manual-review on AI errors

---

## Author

Built by **Rahmatullah Rahi** as part of the DevSynt AI Internship Program's practical evaluation project.

Connect on [LinkedIn](#) · Project repo: [GitHub](#)
