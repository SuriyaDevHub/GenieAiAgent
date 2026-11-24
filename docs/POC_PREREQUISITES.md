# Genie AI Agent – POC Prerequisite Questions

| | |
|---|---|
| **Purpose** | Questions that must be answered **before development starts**, covering the whole flow: Genie run → S3 log → agent → diagnosis → review → Jira / email → back to the Genie user |
| **Related** | [`POC_PLAN.md`](POC_PLAN.md) |
| **How to use** | Go through each section with the named owner, fill the **Answer** column, and mark the status. Every **P0** question must be answered (or its default accepted) before Phase 1 starts. |

**Priority**
- **P0 – Blocker**: development can't start, or the design changes, until this is answered.
- **P1 – Needed before that phase**: development can start; needed before the phase shown.
- **P2 – Good to know**: has a safe default.

**Default** = what we assume if no answer comes. Accepting a default counts as an answer.

**Status**: ☐ Open · ◐ In progress · ☑ Answered / default accepted

---

## Summary – the P0 blockers

| ID | Question | Owner |
|---|---|---|
| A1 | When does Genie upload the log to S3: at the end of a run, on error, or never if the bot hangs or is killed? | Genie team |
| A5 | Who owns the Genie code, and can it take small changes during the POC? | Genie team / Manager |
| B1 | What is the S3 bucket and file name layout (bot, date, machine, user, run)? | Genie / Cloud team |
| C1 | How do we identify the user and machine of a run, and map it to an email / Teams ID? | Genie team / IT |
| D1 | Which data types are sensitive and must be masked (sign-off on the classification list)? | InfoSec / Data Protection |
| D2 | May masked log data be sent to the internal AI platform? | InfoSec |
| E1 | Which LLM and embedding models are available, and at what price and limits? | AI platform team |
| E4 | What is the POC token / cost budget, and who approves spending? | Manager / AI platform team |
| F1 | Can we get a historical Jira export, and how many usable resolved tickets does it have? | Prod Support / Jira admin |
| G1 | Jira Cloud or Server/Data Center, and is there a test project we can write to? | Jira admin |
| K1 | Where will the POC run, and can that host reach S3, the AI platform, Jira and email? | Infra / Cloud team |
| L1 | Which bots and how many users are in the POC pilot? | Manager / Prod Support |

---

## A. Genie Bot and runtime behaviour — Owner: Genie team

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| A1 | When is the log uploaded to S3: only at the end of a run, during the run, or right after an error? What happens if the bot **hangs** or the user **kills** it in Task Manager – is the log ever uploaded? | If logs only arrive at a clean end, hung or killed runs are never seen by the agent. This decides whether Genie changes are required. | P0 | Logs upload only at the end → hung/killed runs are out of scope until A6 is done | | ☐ |
| A2 | Is a log file written once, or re-uploaded / appended under the same S3 key while the run continues? | Decides how the poller detects changes and avoids reading half-written logs. | P0 | Written once at the end | | ☐ |
| A3 | Does Genie run the automation on its **UI thread**? Is that why the window freezes after an error? | Explains the "stuck window" problem; decides the C# fix. | P1 (Phase 0) | Yes – to be confirmed by code review | | ☐ |
| A4 | Does every bot use the same log format (`M/D/YYYY h:mm:ss AM [LEVEL] : message;`)? Are there other levels (WARN, FATAL, DEBUG)? Any bots with a different format? | Parser design and test coverage. | P0 | Same format for all bots | | ☐ |
| A5 | Who owns the Genie code, how is it released to user machines, and can it take small changes during the POC (fail-fast, early log upload, heartbeat, background thread)? | Many improvements (stuck window, hung runs, in-app result display) need small C# changes. | P0 | No Genie changes in the POC – agent works from S3 only | | ☐ |
| A6 | If changes are possible: can Genie (a) stop after N identical errors in a row, (b) upload the log immediately on error / crash / exit, (c) write a heartbeat to S3, (d) show a message instead of freezing? | Fixes the stuck window and makes hung runs visible. | P1 (Phase 1) | Recommended for Genie backlog, not a POC dependency | | ☐ |
| A7 | How many bots / processes exist, which target applications do they drive (ICON, Excel, Office 365, web…), and which UI-automation libraries (TestStack.White, others)? | Rule table, frame classification (library vs own code), pilot choice. | P1 (Phase 1) | — | | ☐ |
| A8 | Is there a stable **run ID**, or can one be added to the log header (run ID, bot, user, machine, version)? | Today a run is detected from the `Bot triggered` line; a real run ID is more reliable. | P2 | Derive run ID from S3 key + "Bot triggered" line | | ☐ |
| A9 | What time zone are log timestamps in? Why do some lines appear out of order (2:16:20 after 2:17:26)? | Correct times in tickets; ordering logic. | P1 (Phase 1) | Local machine time; order by line number | | ☐ |
| A10 | Are screenshots or other artefacts (input workbooks, output files) produced on error and stored anywhere? | Extra diagnosis context; extra PII risk. | P2 | Not used in the POC | | ☐ |
| A11 | Does the bot run attended (user watching) or unattended (scheduled / overnight)? | Decides whether user notification is immediate or next-morning, and who reruns. | P1 (Phase 0) | Attended, on the user's machine | | ☐ |

## B. Logs and AWS S3 — Owner: Genie team / Cloud team

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| B1 | Bucket name(s), region, and the **key layout** – e.g. `logs/<bot>/<yyyy-mm-dd>/<machine or user>/<run>.log`? One file per run or per day? | Poller listing strategy, run detection, user identification. | P0 | — (must be answered) | | ☐ |
| B2 | Typical and maximum log size; number of log files per day; failed runs per day? | Poller sizing, cost, token budget. | P1 (Phase 0) | < 5 MB per file, < 1,000 files/day | | ☐ |
| B3 | Log retention period in S3? | Back-fill window, links in Jira that must stay valid, golden-corpus collection. | P1 (Phase 0) | ≥ 30 days | | ☐ |
| B4 | Can we get a **read-only IAM role** (`s3:ListBucket`, `s3:GetObject` on the log prefix) for the POC host, instead of access keys? | Security; current config has access keys. | P0 | — (must be answered) | | ☐ |
| B5 | Is the bucket encrypted with KMS? Do we need `kms:Decrypt` too? | Access setup. | P1 (Phase 0) | SSE-S3, no extra permission | | ☐ |
| B6 | Are S3 Event Notifications / SQS allowed later (instead of polling)? | Phase 2 design; not needed for the POC. | P2 | Polling only | | ☐ |
| B7 | Is the "working directory" from the diagram (on the bot machine) also uploaded anywhere? Is it needed for diagnosis? | Scope of data the agent can see. | P2 | Out of scope for POC | | ☐ |
| B8 | May the agent **write** to a separate S3 prefix (e.g. `status/`, `heartbeat/`) for results shown back in Genie later? | Phase 2 in-app notification. | P2 | Read-only in POC | | ☐ |

## C. Users, identity and notifications — Owner: Genie team / IT / Prod Support

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| C1 | How do we know **which user and machine** ran a bot (S3 key, log line, Windows user path)? How do we map that to the user's email / Teams ID (AD lookup, a mapping file)? | Needed to notify the user and for the "multi-user → higher priority" rule. | P0 | Mapping file maintained by Prod Support | | ☐ |
| C2 | Channel for user notifications: **email**, **Teams** (incoming webhook / Power Automate / Graph), or both? What is allowed? | Integration design. | P1 (Phase 5) | Email only | | ☐ |
| C3 | What should the user be told, and when? (a) "error detected, being analysed", (b) advice such as "rerun", (c) ticket created, (d) ticket resolved | Notification design and volume. | P1 (Phase 5) | (b), (c), (d) | | ☐ |
| C4 | May users get advice **directly** when it comes from an approved rule or an earlier approved answer, or must Prod Support approve every message? | Speed vs control. | P1 (Phase 5) | Direct for approved rules/answers; LLM-generated advice only after review | | ☐ |
| C5 | After "rerun recommended", **who reruns** – the user or Prod Support? | Message wording and process. | P1 (Phase 4) | The user | | ☐ |
| C6 | Should users still be able to report an issue themselves (e.g. a "Report issue" button or email), in addition to automatic detection? | Covers failures without error lines; extra intake path. | P2 | Existing email route stays as is | | ☐ |
| C7 | For a known widespread issue (e.g. ICON down), may we send one **broadcast** to all affected users? | Reduces duplicate reports. | P2 | Yes, to affected users only | | ☐ |

## D. Data protection, InfoSec and compliance — Owner: InfoSec / Data Protection / Compliance

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| D1 | Sign-off on the **data classification**: which data must be masked (employee IDs, login IDs, ISIN, SEDOL, CUSIP, trade IDs, account numbers, names, emails, hosts, IPs, file names, secrets)? Any others (client names, fund codes, amounts)? | Masking scope (plan §5.2). | P0 | Plan §5.2 list | | ☐ |
| D2 | May **masked** log data be sent to the internal AI platform? Is the platform approved for this data class? Is data kept or used for training by the platform? | Whether the LLM step is allowed at all. | P0 | — (must be answered) | | ☐ |
| D3 | May Jira tickets contain **real** identifiers (L2 may need the actual ISIN to fix data), or masked only? | Jira content; `JIRA_UNMASK_TYPES`. | P1 (Phase 5) | Masked only + S3 key to raw log | | ☐ |
| D4 | May reviewers use a **Reveal** option (re-read raw values from S3 on screen, audited, nothing stored)? | Review UI design. | P1 (Phase 5) | Not allowed | | ☐ |
| D5 | Is a local NER model (Microsoft Presidio + spaCy) allowed for detecting names? | Masking layer L4. | P1 (Phase 2) | Not used; regex + context only | | ☐ |
| D6 | Where may the **golden corpus** (real failed logs, for masking tests) be stored, and who may access it? | Masking tests must use real data, but not in Git. | P0 | Secure share, dev team only | | ☐ |
| D7 | Data residency / retention rules for agent data (SQLite incidents, FAISS index, audit) – how long may masked data be kept? | Retention jobs. | P1 (Phase 1) | 90 days | | ☐ |
| D8 | Is a security review / DPIA / architecture approval needed before the pilot touches production logs? How long does it take? | Timeline – can block the pilot. | P0 | Required – start in week 1 | | ☐ |
| D9 | Where must secrets live (OpenAM credentials, Jira token, SMTP, HMAC key): vault, Windows Credential Manager, env vars? | Config design. | P1 (Phase 0) | Env vars on the POC host, not in Git | | ☐ |
| D10 | Is a human approval step mandatory for any AI output that reaches users or Jira (AI governance policy)? | Confirms human-in-the-loop scope. | P1 (Phase 0) | Yes, as in the plan | | ☐ |

## E. Internal AI platform and cost — Owner: AI platform team / Manager

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| E1 | Which **LLM** and **embedding** models are available? Context size, embedding dimension (config says 1536)? | Model choice, FAISS index setup. | P0 | — (must be answered) | | ☐ |
| E2 | **Price** per 1k input/output tokens per model, and how is it charged (per team, per call, monthly)? | Cost estimate and budget. | P0 | — | | ☐ |
| E3 | Rate limits / quotas (requests per minute, tokens per day)? | Batching for index build, throttling. | P1 (Phase 3) | 60 RPM | | ☐ |
| E4 | POC **token budget** (daily / total) and who approves spending above it? | `DAILY_TOKEN_BUDGET`, kill switch. | P0 | — | | ☐ |
| E5 | API details: auth flow (OpenAM `AUTH_URL` → token), token lifetime, endpoint format (OpenAI-compatible?), JSON / structured output support, TLS certificates. | `llm_client.py` design. | P1 (Phase 0) | OpenAI-compatible, bearer token | | ☐ |
| E6 | Does the platform support **prompt caching** or batch (cheaper, slower) calls? | Extra cost savings. | P2 | No | | ☐ |
| E7 | Does the platform log prompts/responses? Who can see them? Retention? | Data protection (links to D2). | P1 (Phase 0) | Logged, platform team access only | | ☐ |
| E8 | Is there a non-production endpoint for development and testing? | Avoid spending production quota during development. | P1 (Phase 0) | Same endpoint, low budget | | ☐ |
| E9 | Service account for the POC (instead of a personal login in `OPENAM_USERNAME`)? | Security; continuity. | P1 (Phase 0) | Service account | | ☐ |

## F. Historical data and knowledge base — Owner: Prod Support / Jira admin

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| F1 | Can we get a **Jira export** of past Genie support tickets? How many tickets, which years? | Knowledge base for "similar past tickets". | P0 | — (must be answered) | | ☐ |
| F2 | Which fields are reliable – summary, description, root cause, resolution, component, bot name? Is the resolution actually written down, or just "fixed"? | Decides how useful FAISS search will be. | P0 | Summary + resolution only | | ☐ |
| F3 | Are past tickets linked to the S3 log or error text, so we can match them to error signatures? | Builds the signature cache and evaluation set. | P1 (Phase 3) | No link – match by text | | ☐ |
| F4 | Is there an existing known-errors list / runbook / FAQ for Genie? | Seeds the rule table → 0-token diagnoses from day one. | P1 (Phase 4) | None – build with L2 SME | | ☐ |
| F5 | Current **support template**: fields, which are mandatory, sample filled templates (`TEMPLATE_CSV_PATH`)? | Template Generator output. | P1 (Phase 5) | Use fields from the current email template | | ☐ |
| F6 | Can an L2 SME label 50–100 past failures (correct root cause / fix) for evaluation? How many hours can they give? | Accuracy measurement; go/no-go. | P0 | — | | ☐ |

## G. Jira and L2 process — Owner: Jira admin / L2 lead

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| G1 | Jira **Cloud** or **Server/Data Center**? Auth method (API token / Personal Access Token / service account)? | Integration code; config has a password field today. | P0 | Server/DC with PAT | | ☐ |
| G2 | Is there a **test project** the POC may write to? | Phases 6–7 testing without touching the real queue. | P0 | — | | ☐ |
| G3 | Target project key, issue type, **mandatory fields**, components, labels (`genie-…`), priority scheme for the L2 queue? | Ticket payload. | P1 (Phase 6) | — | | ☐ |
| G4 | Attachment size limits and allowed file types? | Masked log excerpt attachment. | P1 (Phase 6) | 10 MB, `.txt` | | ☐ |
| G5 | Rules for duplicates: comment on the existing ticket, or link a new one? When is priority raised (number of users / runs)? | Dedup logic (`multi_user_priority_threshold` = 3). | P1 (Phase 6) | Comment on existing; raise at 3 users | | ☐ |
| G6 | How do we read ticket **resolution** back (status names, resolution field) for the feedback loop and the "resolved" notification to the user? | Nightly sync. | P1 (Phase 6) | Status "Resolved"/"Done" + resolution field | | ☐ |
| G7 | L2 response SLAs today? Current average time from failure to ticket? | Baseline for success metrics. | P1 (Phase 0) | — | | ☐ |

## H. Email — Owner: IT / Messaging team

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| H1 | Send via **SMTP relay** or **Microsoft Graph**? Host, port, auth, sender address / shared mailbox? | Mailer code. | P1 (Phase 6) | SMTP relay, no auth, internal only | | ☐ |
| H2 | Genie Support Mailbox address; distribution lists for L2? | Recipients. | P1 (Phase 6) | — | | ☐ |
| H3 | Email format rules (HTML allowed, classification banner / footer required, external links)? | Template design. | P2 | HTML with classification footer | | ☐ |

## I. Support process and human review — Owner: Prod Support lead

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| I1 | Who **reviews** drafts in the review screen (L1 Prod Support?), how many people, which hours / time zones? | Review UI access, notification of pending reviews. | P1 (Phase 5) | Prod Support L1, business hours | | ☐ |
| I2 | Target review time per incident, and what happens to drafts outside working hours? | Queue design; unattended / overnight runs. | P1 (Phase 5) | Reviewed next business morning | | ☐ |
| I3 | How should reviewers be alerted to new drafts (email, Teams, just the queue)? | Notification design. | P2 | Email digest | | ☐ |
| I4 | Login for the review screen: Windows / SSO, or simple local accounts for the POC? | Review UI security. | P1 (Phase 5) | Restricted host + local accounts | | ☐ |
| I5 | Which outcomes count as success for the pilot (time to ticket, effort saved, accuracy, tickets avoided)? Targets? | Go/no-go criteria (plan §14). | P0 | Plan §14 targets | | ☐ |

## J. Rerun and business safety — Owner: Business / Process owners

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| J1 | Which processes are **safe to rerun** (read-only or idempotent, e.g. duplicate check) and which are not (creating records in ICON, sending files)? | "Rerun recommended" advice must never cause duplicate records. | P1 (Phase 4) | Rerun advice only for rules marked safe by L2 | | ☐ |
| J2 | If a run fails half-way, can a rerun cause duplicates (e.g. some securities already created)? Is there a clean-up step? | Advice wording; possible manual check. | P1 (Phase 4) | Advice includes "check partial results first" | | ☐ |
| J3 | Is automatic rerun wanted later (Phase 2)? Who must approve it? | Long-term scope. | P2 | Not in POC | | ☐ |

## K. Hosting, network and access — Owner: Infra / Cloud / Network team

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| K1 | Where does the POC run (VM, container, AWS EC2, on-prem server)? Windows or Linux? | Setup, scheduler, paths. | P0 | — (must be answered) | | ☐ |
| K2 | Can that host reach: S3, the AI platform, Jira, SMTP/Graph, and (if needed) Teams? Proxy settings, internal CA certificates (`VERIFY_TLS`)? | Every integration depends on it. | P0 | — | | ☐ |
| K3 | Python version allowed; access to PyPI or an internal mirror (`faiss-cpu`, `boto3`, `streamlit`, …)? | Dependencies. | P0 | Python 3.11, internal mirror | | ☐ |
| K4 | Can the review screen (Streamlit, port 8501) be opened by reviewers from their machines? | HITL access. | P1 (Phase 5) | Internal network only | | ☐ |
| K5 | Git repository, CI, and where the POC code may live (this repo / internal Git)? | Collaboration, secret scanning. | P1 (Phase 0) | This repo for planning; code in internal Git | | ☐ |
| K6 | Backup of the FAISS index and SQLite file? Monitoring / alerting for the service? | Reliability during the pilot. | P2 | Daily file copy; log-based alerts | | ☐ |

## L. Scope, governance and timeline — Owner: Manager / Sponsor

| ID | Question | Why it matters | Pri | Default if no answer | Answer | Status |
|---|---|---|---|---|---|---|
| L1 | Which **bots and users** are in the pilot (suggest 1–2 bots with frequent failures, e.g. IM PROD)? | Focus, rule table, data collection. | P0 | IM PROD | | ☐ |
| L2 | Team capacity: developers, % time of Genie dev, L2 SME, InfoSec reviewer? | 7-week plan is realistic only with these people. | P0 | 1–2 devs full-time + part-time SMEs | | ☐ |
| L3 | Target dates: POC start, pilot, go/no-go review? Any freeze periods? | Plan dates. | P1 (Phase 0) | 7 weeks from access granted | | ☐ |
| L4 | Who signs off at the end of each phase and at go/no-go? | Governance. | P1 (Phase 0) | Sponsor + Prod Support lead | | ☐ |
| L5 | Is the pilot **shadow mode** first (drafts only, no real tickets or user messages), then live? | Risk control. | P1 (Phase 7) | Yes | | ☐ |
| L6 | What happens after a successful POC (production budget, owning team, support model)? | Shapes design choices (e.g. shared vector DB later). | P2 | Decide at go/no-go | | ☐ |

---

## Items to collect before development

| # | Item | From | Status |
|---|---|---|---|
| 1 | 50–100 real failed-run logs from S3 (stored per D6) + 20 successful runs | Genie team / Prod Support | ☐ |
| 2 | Historical Jira export (CSV) of Genie tickets | Jira admin | ☐ |
| 3 | Current support template + 10 filled examples | Prod Support | ☐ |
| 4 | Known-errors list / runbook, if any | L2 | ☐ |
| 5 | S3 read-only IAM role for the POC host | Cloud team | ☐ |
| 6 | AI platform service account + model list + price sheet | AI platform team | ☐ |
| 7 | Jira test project + service account / PAT | Jira admin | ☐ |
| 8 | SMTP / Graph details + sender mailbox | IT | ☐ |
| 9 | User → email mapping (or AD lookup access) | IT / Prod Support | ☐ |
| 10 | Signed data classification + approval to send masked data to the AI platform | InfoSec | ☐ |
| 11 | POC host with network access to all of the above | Infra | ☐ |

---

## Suggested clarification sessions (week 0)

| Session | Attendees | Sections | Length |
|---|---|---|---|
| 1. Genie runtime and logs | Genie dev, Cloud team | A, B | 60 min |
| 2. Data protection | InfoSec, Data Protection | D (+ E7) | 60 min |
| 3. AI platform | AI platform team | E | 45 min |
| 4. Support process and Jira | Prod Support lead, L2 SME, Jira admin | C, F, G, I, J | 90 min |
| 5. Infra and email | Infra, Network, IT messaging | H, K | 45 min |
| 6. Scope and sign-off | Sponsor, all leads | L + review of open P0s | 30 min |

**Ready to start development when:** all P0 questions are ☑, items 1, 2, 5, 6, 10, 11 are collected,
and the sponsor has signed off on scope (L1–L2).
