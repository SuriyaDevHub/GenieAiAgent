# Genie AI Agent – POC Implementation Plan

> Goal: turn the "Proposed Agentic AI Workflow (To-Be)" into a working POC for Genie Bot
> (C#/.NET automation) production support, using **logs in AWS S3**, a **local FAISS index**,
> and **LLM + embedding models from the internal AI platform**. Token use costs money, so
> the design is **deterministic first, LLM last**.

---

## 1. Guiding principles (cost-aware)

| Principle | What it means in practice |
|---|---|
| **No LLM where code works** | Log parsing, stack-trace extraction, masking, Jira/email calls, template filling and dedup are plain Python. Only *diagnosis wording* needs the LLM. |
| **At most 1 LLM call per incident** | The Diagnostic and Template Generator agents share **one** structured-JSON call (root cause + fix + template fields). No chained agent-to-agent LLM chatter. |
| **Cache before you call** | Same error signature seen before → reuse the stored answer (0 tokens). Close match in history (≥ 0.85) → reuse the historical resolution (0 LLM tokens, 1 cheap embedding). |
| **Send less** | Strip framework frames, dedupe repeated lines, cap the trace and log tail, send the 3 best historical matches as short summaries, not whole tickets. |
| **Hard budgets** | Per-incident token cap, daily token cap, kill switch. Every call logged with tokens used. |
| **Mask before anything leaves the box** | Masking runs before embedding, before the LLM, and before Jira/email. |

Expected effect: most repeat incidents cost **0 LLM tokens**; new ones cost roughly
**2–3.5k input + ≤ 800 output tokens**.

---

## 2. Target architecture

```
 Genie Bot (C#) ──run fails──► POST /incidents (FastAPI, Python)      ◄── fallback: S3 poller
        │                          │
        └─ execution logs ─► S3    ▼
                         ┌──────────────────────────────────────────────────────────┐
                         │ Orchestrator (simple state machine / LangGraph optional) │
                         └──────────────────────────────────────────────────────────┘
   1. Log Parser Agent      (no LLM)  fetch S3 log → extract .NET exception + stack trace
                                      → normalise → mask → error signature (hash)
   2. Diagnostic Agent      (L1)      a) signature cache hit?          → reuse   (0 tokens)
                                      b) FAISS top-k over history
                                         score ≥ 0.85                  → reuse   (0 LLM tokens)
                                         0.75–0.85 or new              → 1 LLM call (JSON)
                                      c) classify: TRANSIENT / DATA / CONFIG / CODE / UNKNOWN
         ├── TRANSIENT & allow-listed & cooldown ok ──► Controlled Rerun (self-healing) via Genie API
         └── needs escalation ▼
   3. Template Generator    (no extra LLM call) fill support template from the
                                      diagnostic JSON + metadata + historical refs
   4. Human-in-the-Loop     Review UI: edit, add comments, Approve / Reject
   5. Integration Agent     (no LLM)  dedup check in Jira → create ticket + attach masked log
                                      (or comment on existing one) → email Genie Support Mailbox
```

### Components and where cost happens

| Component | Tech | LLM tokens | Embedding tokens |
|---|---|---|---|
| Intake API | FastAPI | – | – |
| Log Parser Agent | boto3 + regex | – | – |
| Masking | regex from `cfg.masking_patterns` | – | – |
| Signature cache | SQLite | – | – |
| Knowledge base | FAISS (local, `IndexFlatIP`, normalised vectors) | – | one-time build + 1 per new incident |
| Diagnostic Agent | internal LLM, JSON output | only on cache/history miss | – |
| Template Generator | Jinja2 + template CSV | – | – |
| HITL review | Streamlit (POC) | – | – |
| Integration Agent | Jira REST + SMTP | – | – |
| Self-healing | Genie Bot rerun API | – | – |

---

## 3. Integration with Genie Bot (C#)

Small change on the .NET side, everything else stays in Python.

1. **Failure hook (preferred):** in Genie's global exception handler / run-finalizer, after the log
   is uploaded to S3, call `POST {AGENT_URL}/incidents` with:
   ```json
   { "run_id": "...", "bot_name": "...", "process": "...", "user_id": "...",
     "s3_key": "logs/<bot>/<run_id>.log", "working_dir": "...", "failed_at": "ISO-8601" }
   ```
   Fire-and-forget with a 2–3 s timeout so the bot never waits on the agent.
2. **Fallback (no .NET change needed to start):** S3 poller every N minutes on `s3_log_prefix`,
   picks up new log objects that contain `Exception`/`ERROR`. Swap to S3 Event → SQS later.
3. **Rerun endpoint (needed for self-healing):** Genie exposes `POST /runs/{run_id}/rerun`
   (or a queue/command table it already reads). Without this, Phase 6 is skipped and the
   agent only *recommends* a rerun.
4. **Optional:** a "Report issue" button in Genie for the end user → the same `/incidents` call,
   with the user's free-text added.

---

## 4. Agent details

### 4.1 Log Parser Agent (deterministic)
- Download the log from S3 (`s3_bucket_name` / `s3_log_prefix`); use a byte-range for large logs.
- Extract .NET exception blocks:
  - Header: `^(?<type>[\w\.]+(Exception|Error))(: (?<msg>.*))?`
  - Frames: `^\s*at (?<method>[\w\.\<\>`\[\],]+)\((?<args>.*)\)( in (?<file>.*):line (?<line>\d+))?`
  - Inner exceptions: `---> ` and `--- End of inner exception stack trace ---`.
- Normalise: drop `System.*`, `Microsoft.*`, async state-machine frames; keep the top 10
  Genie/own-namespace frames; replace GUIDs, timestamps, numbers, paths with placeholders.
- **Error signature** = `sha1(exception_type + top 5 normalised own frames)` – this is the
  key for caching, dedup and multi-user counting.
- Capture the last 30–50 log lines before the exception (deduped) as context.
- Apply masking (§6) and produce a `ParsedIncident` object.

### 4.2 Diagnostic Agent (L1 triage)
1. **Signature cache** (SQLite `signatures` table): hit and last answer was approved by a human
   → reuse it directly.
2. **Retrieve**: embed the *masked, normalised* `exception + top frames + message`
   (≈ 150–300 tokens) → FAISS search `retrieval_top_k = 5` over historical tickets.
3. **Decide:**
   - top score ≥ `high_confidence_threshold` (0.85) → take the historical root cause / resolution,
     no LLM call.
   - `similarity_threshold` (0.75) ≤ score < 0.85 → LLM call with the 3 best matches as context.
   - score < 0.75 → LLM call with trace only (flag "no historical match").
4. **Single LLM call**, temperature 0.1–0.2, `max_tokens` 800, JSON-only output:
   ```json
   { "category": "TRANSIENT|DATA|CONFIG|CODE|UNKNOWN",
     "root_cause": "...", "suggested_fix": "...",
     "rerun_recommended": true, "confidence": 0.0,
     "summary": "...", "steps_to_reproduce": "...", "impact": "...",
     "related_tickets": ["GEN-123"] }
   ```
   Validate with Pydantic; one cheap retry only if the JSON is invalid.
5. **Rule override:** a small rule table maps known exceptions to categories without the LLM,
   e.g. `TimeoutException`, `SqlException` deadlock (1205), `HttpRequestException`,
   `IOException: file in use` → TRANSIENT.

### 4.3 Controlled Rerun (self-healing)
- Only when **all** are true: category TRANSIENT, exception type in a configured allow-list,
  no rerun for this run in the last `rerun_cooldown_minutes` (30), max **1** auto-rerun per run.
- Call the Genie rerun API and watch the result; if the rerun fails → escalate with both
  run logs referenced.
- Start in **"recommend only"** mode in the pilot; switch on auto-rerun per process later.

### 4.4 Template Generator Agent (no extra LLM call)
- Load the support template from `TEMPLATE_CSV_PATH`; fill it with Jinja2 from:
  incident metadata (bot, process, run id, user, time, S3 link, working dir) +
  diagnostic JSON + linked historical tickets.
- This replaces the manual template the end user fills today.

### 4.5 Human-in-the-Loop
- Streamlit page (POC): queue of pending incidents → view pre-filled template, masked trace,
  similar tickets and confidence → edit / add comments → **Submit** or **Reject**.
- Approved (and edited) answers go back into the signature cache and, after resolution, into
  FAISS → the system gets cheaper over time.

### 4.6 Integration Agent (deterministic)
- **Dedup before create:** same signature, or FAISS similarity ≥ `dedup_similarity_threshold`
  (0.85) against *open* tickets → add a comment + "affected users" count instead of a new ticket.
  When affected users ≥ `multi_user_priority_threshold` (3) → raise priority.
- **Jira:** `POST /rest/api/2/issue` (project `jira_project_key`, label `jira_l2_queue_label`),
  then `POST /rest/api/2/issue/{key}/attachments` with the **masked** log excerpt.
- **Email:** SMTP (or Graph API) to Genie Support Mailbox with the template + Jira link.
- Both calls are idempotent (keyed by incident id) and retried with back-off.

---

## 5. Knowledge base (local FAISS)

- **Source:** historical Jira export (`JIRA_CSV_PATH`) + support templates (`CSV_PATH`).
- **Document per ticket:** `summary + exception type + key frames + root cause + resolution`,
  masked, trimmed to ~300 tokens. Don't embed comments or full descriptions.
- **Chunking:** one ticket = one vector in most cases; only split long ones
  (`chunk_size` from config).
- **Store:** `faiss.IndexFlatIP` with L2-normalised vectors (cosine) at `INDEX_PATH`;
  metadata (ticket key, status, resolution, signature) in JSON/SQLite at `META_PATH`.
- `embedding_dimensions` (1536) must match the embedding model actually used – check at load
  time and fail fast if it doesn't.
- **Cost:** build once, batch the embedding calls (e.g. 50–100 texts per request), cache
  vectors by text hash so a rebuild doesn't re-embed. Afterwards only add newly resolved
  tickets (nightly/weekly incremental job).
  Rough estimate: 5,000 tickets × 300 tokens ≈ 1.5M embedding tokens, one time.
- POC scale (thousands to tens of thousands of vectors) is fine with a flat index; move to a
  shared vector DB (e.g. OpenSearch/pgvector) only if this goes to production on several hosts.

---

## 6. Security and masking

- Mask before **embedding, LLM, Jira, email and logs written by the agent**.
- Current patterns (ISIN, Trade IDs, Account numbers, generic 8–12 digit numbers) plus:
  emails, IPs, hostnames/FQDN, Windows user paths (`C:\Users\<id>\`), connection strings,
  passwords/tokens in URLs or `key=value` pairs, JWTs.
- Replace with typed placeholders (`<ISIN_1>`, `<ACC_2>`) so the LLM still understands the
  structure; keep the mapping only in memory.
- Note: `\b\d{8,12}\b` will also mask dates/timestamps like `20261124` – run timestamp
  normalisation *before* masking, or narrow this pattern.
- Secrets: no AWS keys in code. Use an IAM role / AWS profile; keep OpenAM, Jira and SMTP
  credentials in env vars / a vault, never in the repo. `.env` in `.gitignore`.
- Internal AI platform auth: get the token from `AUTH_URL` with OpenAM credentials, cache it
  until expiry, refresh on 401.

---

## 7. Token and cost controls (POC)

| Control | Setting |
|---|---|
| LLM calls per incident | max 1 (+1 retry only for invalid JSON) |
| Input size | trace ≤ 10 frames, log tail ≤ 40 lines, 3 historical summaries ≤ 150 tokens each → target ≤ 3.5k input tokens |
| Output | `llm_max_tokens` 800, JSON only, no explanations outside JSON |
| Temperature | 0.1–0.2 for diagnosis (config shows 0.5 – lower it) |
| Caching | signature cache (answers), embedding cache (vectors), auth token cache |
| Skip rules | high-confidence history match, known-rule category, duplicate of open ticket → no LLM call |
| Budgets | `MAX_TOKENS_PER_INCIDENT`, `DAILY_TOKEN_BUDGET`; over budget → fall back to "history-only" diagnosis and flag for human |
| Kill switch | `LLM_ENABLED=false` → pipeline still runs (parse, match, template, Jira) without LLM |
| Metering | `token_usage` table: incident id, model, input/output tokens, cache hit/miss, latency → daily report |
| Model choice | smallest model on the platform that passes the eval set for diagnosis; bigger model only if confidence is low (optional) |

---

## 8. Project structure

```
GenieAiAgent/
├── config/
│   ├── settings.py           # Config dataclass (fixed version of the ideation config)
│   └── rules.yaml            # exception → category rules, rerun allow-list
├── genie_agent/
│   ├── api.py                # FastAPI: POST /incidents, GET /health
│   ├── orchestrator.py       # state machine wiring the agents
│   ├── agents/
│   │   ├── log_parser.py
│   │   ├── diagnostic.py
│   │   ├── template_generator.py
│   │   ├── integration.py
│   │   └── self_healing.py
│   ├── services/
│   │   ├── s3_client.py
│   │   ├── llm_client.py     # internal platform: auth, budget, metering, retries
│   │   ├── embeddings.py     # batching + cache
│   │   ├── vector_store.py   # FAISS load/search/add
│   │   ├── jira_client.py
│   │   ├── mailer.py
│   │   └── genie_client.py   # rerun API
│   ├── masking.py
│   ├── models.py             # Pydantic: ParsedIncident, Diagnosis, TicketDraft
│   └── store.py              # SQLite: incidents, signatures, token_usage, audit
├── scripts/
│   ├── build_index.py        # one-time / incremental FAISS build from Jira CSV
│   └── eval_diagnosis.py     # replay historical incidents, score accuracy + tokens
├── ui/review_app.py          # Streamlit HITL
├── tests/                    # parser, masking, dedup, budget, sample .NET logs
├── .env.example
└── requirements.txt          # fastapi, boto3, faiss-cpu, numpy, pydantic, jinja2,
                              # requests, streamlit, python-dotenv, pytest
```

---

## 9. Fixes to the ideation-stage config

Seen in the shared `Config` screenshots:

1. `validate()` lists `"OPENAM_USERNAME"` twice – the second should be `"OPENAM_PASSWORD"`.
2. `validate()` checks `self.jira_api_token`, but the class defines `jira_user_password` →
   `AttributeError`. Pick one: Jira Cloud = email + API token, Jira Server/DC = Personal Access Token.
3. `VERIFY_TLS: bool = os.getenv(...)` returns a **string**, so `"false"` is truthy. Parse it:
   `os.getenv("VERIFY_TLS", "true").lower() in ("1", "true", "yes")`.
4. `os.getenv(...)` as a dataclass default is evaluated once at import time; use
   `field(default_factory=lambda: os.getenv(...))` or pydantic-settings if config needs reloading/testing.
5. `JIRA_CSV_PATH` is a hard-coded personal Windows path (contains a user ID) – move it to an env var.
6. Error text "Copy .env to .env" → "Copy .env.example to .env".
7. `validate()` doesn't check AWS / S3 / FAISS / template paths – add them, and drop AWS keys
   in favour of IAM role/profile.
8. `llm_temperature` 0.5 → 0.1–0.2 for diagnosis; `llm_max_tokens` 1000 → 800.
9. Add new keys: `AGENT_URL`, `GENIE_RERUN_URL`, `SMTP_*`/mailbox, `LLM_ENABLED`,
   `MAX_TOKENS_PER_INCIDENT`, `DAILY_TOKEN_BUDGET`, `AUTO_RERUN_ENABLED`, `RERUN_ALLOWLIST`.

---

## 10. Phased delivery (≈ 8 weeks, 1–2 developers)

| Phase | Weeks | Scope | Exit criteria |
|---|---|---|---|
| **0. Setup & data** | 1 | Repo, config fix, `.env.example`, access to S3 / AI platform / Jira test project / SMTP. Collect 50–100 real failed-run logs + historical Jira CSV. Agree template fields. | All connections tested from the POC host. Sample dataset ready. |
| **1. Log Parser + masking** | 1–2 | S3 fetch, .NET trace extraction, normalisation, signature, masking, unit tests on real logs. | ≥ 95 % of sample logs parsed correctly; no unmasked sensitive data in output. |
| **2. Knowledge base** | 2–3 | `build_index.py`, embedding batching + cache, FAISS search, metadata. | Top-5 retrieval returns the right historical ticket for ≥ 70 % of a labelled sample. One-time embedding cost recorded. |
| **3. Diagnostic Agent** | 3–4 | Cache → retrieval → threshold logic → single JSON LLM call, rule table, budget/metering in `llm_client`. `eval_diagnosis.py`. | Root cause judged correct/useful by L2 for ≥ 60–70 % of the eval set; avg tokens/incident and % zero-token incidents reported. |
| **4. Template + HITL** | 5 | Jinja template fill, Streamlit review app, feedback into signature cache. | Reviewer can approve/edit in < 2 min per incident. |
| **5. Integration Agent + intake** | 6 | Jira create/attach/dedup/priority, email, FastAPI `/incidents`, S3 poller fallback; Genie C# hook. | End-to-end: failed Genie run → ticket + email in Jira **test** project. |
| **6. Self-healing (gated)** | 7 | Rule allow-list, cooldown, max 1 retry, Genie rerun API, "recommend only" by default. | Reruns only for allow-listed transient errors; full audit trail. |
| **7. Pilot & evaluation** | 8 | Shadow mode on 1–2 bots (agent output compared with what L2 actually did), then live with HITL. | Metrics report (§11) and go/no-go for production. |

**Shadow mode first:** in the pilot, the agent drafts but does not create real tickets until
quality is accepted.

---

## 11. Success metrics

- Time from failure to ticket in L2 queue (today vs POC).
- Manual effort saved per incident (template fill, log search).
- Diagnosis accuracy (L2 rating: correct / partially / wrong).
- % incidents resolved by rerun without L2.
- Duplicate tickets avoided.
- **Tokens and cost per incident**, % zero-LLM incidents, daily spend vs budget.

---

## 12. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Token cost creeps up | One call per incident, caches, budgets, kill switch, daily report. |
| Sensitive data sent to LLM / Jira | Masking at every boundary, masking unit tests, review samples in Phase 1. |
| Wrong diagnosis / hallucination | Grounded on retrieved tickets, confidence shown, human approval before submit. |
| Unsafe automatic rerun (e.g. duplicate trades/payments) | Allow-list only idempotent processes, max 1 retry, recommend-only by default, business sign-off per process. |
| Unstructured / inconsistent Genie logs | Parser tests on real logs; ask Genie team to log a structured header (run id, process, step) – cheap and very helpful. |
| Poor historical data quality | Clean / filter the Jira CSV (resolved tickets with a real resolution only). |
| Local FAISS is single-host | Fine for POC; move to a shared vector store for production. |

---

## 13. Open questions to close in Phase 0

1. Can Genie Bot call an HTTP endpoint on failure, and can it expose a rerun API?
2. Log format and layout in S3 (one file per run? key naming? size?).
3. Which LLM and embedding models does the internal platform offer, their price per 1k
   tokens, rate limits, and embedding dimension?
4. Jira Cloud or Server/Data Center (API token vs PAT)? Test project available?
5. Email: SMTP relay or Microsoft Graph? Mailbox address?
6. Which processes are safe to auto-rerun (idempotent)?
7. How many historical tickets, and which fields are reliable?
8. Where does the POC run (VM / container) and can it reach S3, the AI platform and Jira?
