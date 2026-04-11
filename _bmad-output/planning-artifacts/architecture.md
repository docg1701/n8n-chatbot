---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
lastStep: 8
status: 'complete'
completedAt: '2026-04-10'
inputDocuments:
  - _bmad-output/planning-artifacts/prd.md
  - _bmad-output/planning-artifacts/product-brief-n8n-chatbot.md
  - _bmad-output/planning-artifacts/product-brief-n8n-chatbot-distillate.md
  - _bmad-output/planning-artifacts/implementation-readiness-report-2026-04-10.md
  - research/01-agent-patterns-and-optimization.md
  - research/01b-agent-patterns-supplementary.md
  - research/02-memory-rag-and-context.md
  - research/03-media-integrations-and-security.md
  - research/04-production-operations.md
  - docs/plan.md
  - CLAUDE.md
workflowType: 'architecture'
project_name: 'n8n-chatbot'
user_name: 'Galvani'
date: '2026-04-10'
supersededInputs:
  - docs/plan.md — phase-based framing (Phase 1-4) and GPT-4o-mini/Haiku model choices; treat as historical context only
  - product-brief-n8n-chatbot-distillate.md — GPT-4o-mini/Haiku model choices; superseded by PRD and standing constraints
authoritativeInput: _bmad-output/planning-artifacts/prd.md
---

# Architecture Decision Document — n8n-chatbot (Legal Intake AI)

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements:** 70 FRs across 11 categories.

- **Conversational intake (FR1-10):** Bidirectional text and voice on Telegram
  and WhatsApp in Brazilian Portuguese, modality mirroring (voice-in → text +
  voice-out), short-term memory with windowing, returning-client profile
  recognition, attachment detection without content processing.
- **Classification & data extraction (FR11-15):** Deterministic triage across 8
  Maruzza practice areas, structured extraction of name/city/summary/urgency,
  confirm-before-persist, persistent client profile.
- **Lead qualification & handoff (FR16-24):** Deterministic scoring,
  handoff-package composition with full transcript, private-channel notification
  with 3-way inline keyboard (Accept / Need More Info / Decline) + 4h timeout,
  follow-up relay branch, graceful decline with alternative resources.
- **Non-qualified handling (FR25-27):** Static pre-approved per-area content
  library — selection, never generation.
- **Compliance & safety (FR28-34):** AI disclosure + LGPD consent in turn 1,
  zero legal-advice generation enforced in 6 layers, forensic audit trail,
  retention policy per data type, right-to-erasure procedure.
- **Security (FR35-41):** Rate limiting, sanitization, input + output guardrails
  (jailbreak / topical / PII), strict session isolation, encrypted credentials.
- **Resilience (FR42-47):** Retry with exponential backoff, LLM provider
  failover, graceful degradation when Comadre/Whisper fail, friendly message in
  every failure path, dedicated Error Trigger workflow with severity routing.
- **Refinement pipeline (FR48-56):** Separate test environment, synthetic and
  chatbot-vs-chatbot simulation, internal human testers, versioned golden
  dataset, adversarial red team, attorney review, audit bundle exporter, formal
  Launch Gate as production gate.
- **Operations & monitoring (FR57-62):** Analytics logging, query interface,
  weekly report workflow, cost tracking, critical-metric alerting, attorney
  feedback loop feeding refinement.
- **Template export & replicability (FR63-68):** Self-contained JSON export,
  SQL init schema, credentials checklist, markdown prompt files, dry-run target
  for new-firm deployment.
- **Case study documentation (FR69-70):** Structured refinement log + complete
  project-journey documentation in the repo.

**Non-Functional Requirements:** 47 NFRs across 10 categories.

- **Performance:** p95 ≤ 8s text / ≤ 14s audio-to-audio, 15-25% LLM-free
  resolution via deterministic router, 10-15-message context window, prompt
  caching where supported, 30s hard ceiling with async fallback.
- **Security:** Minimum-external-provider posture (Groq + Comadre only),
  pre-Launch Groq zero-retention validation with written contingency,
  deterministic session key per channel, N8N_ENCRYPTION_KEY mandatory,
  append-only audit trail, 48h right-to-erasure SLA, 100% red-team pass at
  Launch Gate, undefeatable AI disclosure + LGPD consent, mandatory output
  legal-advice validation.
- **Reliability:** Degraded-mode operation when optional services fail,
  ≤2-retry LLM failover, logging-never-blocks-conversation, webhook
  idempotency via provider message ID, ≤15 min planned downtime windows.
- **Scalability:** 3× headroom on free tiers at 100 conv/day,
  ≤20% CPU / ≤30% RAM budget on BorgStack-mini, provider scaling by
  credentials change only, indexed + partitioned chat_analytics table.
- **Maintainability:** ≥90% visual nodes in final workflow, single-responsibility
  sub-workflows, prompts as Git-versioned markdown (source of truth, manual
  copy-paste only), structured refinement log, main workflow ≤7 logical layers,
  ≤2h onboarding for a new operator.
- **Observability:** Per-message/per-handoff/per-refinement/per-incident
  granularity, ≤5 min forensic reconstruction from handoff ID, ≤5 min alert
  latency for critical errors, weekly report independent of external dashboards.
- **Integration quality:** Retry on Fail + backoff on every external call,
  documented external contracts, provider swap via credentials only, webhook
  signature validation where platform supports it.
- **Portability & replicability:** Self-contained JSON export, firm-specific
  data fully externalized, ≤8h per-firm deployment target, explicit dependency
  list (min n8n version, community nodes, SQL schema).
- **Cost:** ≤ R$100/month at 100 conv/day, normal-regime R$0-50 using free
  tiers, frontier auditor on fixed subscription (zero marginal cost), 80%
  budget alert with provider breakdown.
- **Data residency:** Client data exclusively on firm's BorgStack Postgres,
  minimum-payload policy to Groq (current window only, never full history),
  anonymization filter on every audit bundle leaving BorgStack.

**Scale & Complexity**

- **Primary domain:** n8n workflow automation / conversational AI agent /
  self-hosted legaltech. The deliverable is not an application server or a web
  frontend — it is an n8n workflow (exported as JSON), Git-versioned markdown
  prompts, SQL schema, and operational playbooks.
- **Complexity level:** **high** (as classified in the PRD). Conjunction of
  regulated domain (OAB 001/2024 + CNJ 615/2025 + LGPD + constitutional
  attorney-client privilege), queue-mode memory constraint, two-tier intelligence
  architecture with an operator-paced frontier auditor, multimodal input/output
  across two messaging platforms, zero-tolerance pre-production validation
  pipeline as the dominant development phase, and simultaneous live-instance +
  exported-template + case-study deliverables.
- **Estimated architectural components (high-level):** 6 main-workflow layers
  + 9 sub-workflows (media processor, save client info, lead qualification,
  handoff flow, follow-up relay, error handler, analytics logger, weekly report
  generator, audit bundle exporter) + 5 Postgres tables (`chat_analytics`,
  `client_profiles`, `n8n_chat_histories`, `static_responses`, `test_scenarios`
  or prefixed staging) + Redis (rate limit counters + static response cache)
  + 5 external integration surfaces (Telegram, WhatsApp via Evolution API,
  Groq LLM, Groq Whisper, Comadre TTS) + offline refinement loop operated in
  Claude Code locally under the operator's subscription (outside n8n).

### Technical Constraints & Dependencies

**Hard constraints that pre-determine architectural choices:**

- **n8n 2.x visual-first, queue mode:** Native and community visual nodes
  always come first. JavaScript Code nodes are a documented last resort. Queue
  mode forbids Simple Memory — Postgres Chat Memory is mandatory for every
  stateful agent. No node can fall back to in-process memory.
- **BorgStack-mini never runs local inference:** No Ollama, no local Whisper,
  no local Kokoro. All model inference is either cloud API (Groq) or on a
  dedicated external server (Comadre at 10.10.10.207). LLM / STT / TTS are all
  HTTP Request integrations with external endpoints.
- **Operational LLM posture:** Groq free tier (7-12B multilingual model) is the
  primary, Groq paid credits are the automatic failover, and flat-rate monthly
  OpenAI-compatible providers are the long-term evolution path. Pay-per-token
  premium models (Claude Haiku, GPT-4o-mini, Claude Sonnet, GPT-4o) are
  explicitly off the table for production operational calls.
- **TTS is exclusively Comadre:** Default voice `kokoro/pm_santa`, OpenAI-
  compatible `/v1/audio/speech` endpoint. No other TTS engine (Edge TTS, Piper,
  Kokoro Web, OpenEdAI, OpenAI TTS) is acceptable.
- **Frontier auditor runs in Claude Code locally, not via n8n:** The
  meta-prompting cycle (Claude Opus 8.6 via the author's fixed monthly
  subscription) is explicitly outside the n8n workflow. The architecture may
  not propose an n8n sub-workflow that calls a frontier API — zero per-token
  spend on frontier reasoning is a non-negotiable constraint.
- **Prompts as Git-versioned markdown, manually copy-pasted into n8n:** No
  auto-sync between the repo and the n8n editor. Every prompt change is an
  explicit, audited, manually-applied decision.
- **Single continuous progression, no phases:** The architecture describes the
  full v1.0 topology. There is no "MVP architecture" versus a "v2 architecture"
  — there is one architecture, constructed through a linear ordered progression
  of 34 steps (PRD §Progression Sequence), each of which leaves the system
  functional at its current scope.
- **n8n security floor:** Minimum version 1.122.0+ to avoid CVE-2025-68613.
  `N8N_ENCRYPTION_KEY` is mandatory and durable.

**Soft constraints driven by the operational model:**

- Production and refinement environments must be fully isolated (separate bot
  tokens, separate or prefixed database schemas, separate workflows), but
  co-located on the same BorgStack-mini infrastructure.
- The launch switch from refinement bot to production bot happens by
  configuration / credentials only, not by redeployment.
- Firm-specific data (firm name, practice areas, voice, prompts, qualifying
  criteria) must be externalized from day one — retrofitting externalization
  later would compromise the template replicability Innovation 5 criterion.

### Cross-Cutting Concerns Identified

1. **Zero-legal-advice enforcement** — layered across system prompt, tool
   design, static response library, output guardrails, adversarial red team,
   and attorney review. Never a single control. Must be architecturally visible
   in multiple components.
2. **Session isolation & attorney-client privilege preservation** — session
   key derivation, memory isolation, minimum-payload external calls, audit
   bundle anonymization, backup encryption, right-to-erasure SLA.
3. **Forensic audit trail** — append-only, granular, never blocks the
   conversation on logging failure (NFR17 vs NFR10 tension must be resolved
   by the architecture, e.g., via best-effort insert with separate incident
   reconciliation).
4. **Resilience as user-facing contract** — FR46: never silence, never crash.
   Every failure path terminates in a friendly client message. Constrains how
   the Error Trigger workflow connects back to the conversation channel.
5. **Cost control through disciplined engineering** — deterministic router
   (15-25% LLM-free), context window cap (10-15 messages), prompt caching,
   single operational provider, free-tier-first model posture, compaction of
   older context. Architectural enforcement, not operational best effort.
6. **Template replicability** — every firm-specific value externalized via
   environment variables, credentials, or prompt files. No hardcoded firm
   name, practice areas, voice, or prompt text anywhere in the workflow.
7. **LGPD & regulatory compliance as architectural enforcement** — AI
   disclosure + LGPD consent cannot be disabled by refinement or hotfix,
   retention policy executes automatically, right-to-erasure is a first-class
   operator procedure.
8. **Refinement cycle as first-class architectural citizen** — production and
   refinement environments are parallel topologies (not "test as an
   afterthought"), linked only by the audit bundle exporter and the manual
   copy-paste-prompt-back-into-editor ritual.
9. **Multimodality mirroring** — `inputWasAudio` flag flows from the media
   processor through the agent to the response delivery layer, deciding
   whether Comadre TTS is invoked. Single source of truth, no ambiguity.
10. **Cross-platform normalization** — Telegram trigger and Evolution API
    webhook both land in a unified internal schema before any downstream
    logic. Adding Instagram or Facebook later is a new ingestion source,
    never a workflow rewrite.
11. **Launch Gate as architectural switch** — the transition from refinement
    to production is flipped by credentials / configuration only (NFR36),
    never by workflow edits.

## Starter Template & Foundation

### Reframing the step for an n8n workflow project

The conventional "starter template" question (Next.js, NestJS, T3, etc.) does
not apply — this project delivers an n8n visual workflow, not a conventional
software scaffold. The architecturally meaningful equivalents are:

1.  **What foundation already exists?** (infrastructure + platform pinning)
2.  **Do we seed the workflow from a community template, or start from an
    empty canvas?** (the actual architectural fork)

### Foundation already in place (no scaffolding required)

All orchestration and persistence infrastructure is already running on
BorgStack-mini at `10.10.10.205`:

- n8n (queue mode: editor + webhook + worker)
- PostgreSQL + pgvector
- Redis
- Caddy (reverse proxy with TLS termination)
- Cloudflare Tunnel (secure inbound ingress, no public IP exposure)
- Evolution API (WhatsApp bridge, ready for the production bot)
- Authelia + lldap + Homer (not used by this workflow but share the node)

TTS is provided by **Comadre** on a dedicated server at `10.10.10.207`,
exposing an OpenAI-compatible `/v1/audio/speech` endpoint, default voice
`kokoro/pm_santa`. Operational LLM and STT are provided by **Groq** at
`api.groq.com/openai/v1` (free tier first, paid fallback, flat-rate monthly
alternatives as long-term evolution).

The architecture document will treat BorgStack as a black-box platform and
specify only the n8n-, database-, and credentials-level contracts this
workflow depends on.

### Platform version pinning

- **n8n minimum version:** 2.15.1 (current stable as of 2026-04-10). The
  project is clearly on n8n 2.x and comfortably past the CVE-2025-68613 fix
  floor (1.120.4+).
- **Version tracking:** every workflow JSON exported to
  `exports/workflow-snapshots/` will be tagged with the n8n version it was
  exported from (satisfies NFR41 — explicit dependency list).
- **`N8N_ENCRYPTION_KEY`:** mandatory, persistent, and backed up out-of-band.
  Without this, the entire credentials system is unrecoverable after restart.

### Seed decision — empty canvas, not a community template

**Decision: build the main workflow and all sub-workflows from an empty n8n
canvas, using PRD §8 (Technical Architecture — n8n Workflow Blueprint) as
the structural guide and PRD §21 (Progression Sequence) as the construction
order.** Community workflow templates are consulted as *reference
configurations for specific node settings only*, never as structural
starting points.

**Rationale:**

- Every candidate community template (n8n.io #11754, #3586, #6077, #10037,
  #11141) is opinionated toward GPT-4o or Claude, which violates the
  Groq-first operational-LLM constraint.
- None implements the 6-layer no-legal-advice enforcement, the 9 sub-workflows
  defined in PRD §8, the refinement pipeline + audit bundle exporter, the
  Launch Gate credential switch, the client profile long-term memory, or the
  static pre-approved response library. The legaltech surface is entirely
  absent from community templates.
- Community templates use Code nodes liberally where visual-first discipline
  would solve the problem with Switch + IF + Set nodes. Importing carries
  that cruft in and forces refactoring work that contradicts NFR24 (≥90%
  visual nodes).
- **The v1.0 exported workflow JSON is itself the product** (FR63-68 — the
  replicable template). Starting from a community template means the exported
  artifact is a descendant of someone else's opinionated choices, which
  future importing firms would have to sift through — a direct conflict with
  the Innovation 5 criterion (≤8h per-firm deployment, reproducible quality).
- The refactoring tax of an imported template outweighs its initial velocity
  advantage because the PRD §8 blueprint is already a layer-by-layer
  construction guide that removes the blank-canvas friction.

**Reference configurations — community templates consulted as node-level
recipes only:**

| Template | Consulted for |
|---|---|
| [#11754] WhatsApp Assistant + Evolution API | Evolution API webhook payload shape, `sendWhatsAppAudio` endpoint, media download endpoint |
| [#3586] WhatsApp Chatbot + Voice + Memory | Postgres Chat Memory wiring, binary data handling for voice |
| [#6077] WhatsApp Audio Transcription via Groq | Groq Whisper multipart HTTP Request configuration |
| [#10037] Telegram + Groq Whisper | Telegram voice download + OGG → Whisper flow |
| [#11141] 9 Guardrail Layers with Groq | Guardrails node configuration patterns, jailbreak/topical/PII examples |

### Evolution API integration — HTTP Request node, not community node

**Decision: use the generic HTTP Request node for all Evolution API
endpoints. Do not install the `n8n-nodes-evolution-api` community node.**

**Rationale:**

- Evolution API is a pure HTTP REST API; the community node is syntactic
  sugar, not new capability.
- Multiple community reports (2025-2026) of node-type registration breaking
  after n8n upgrades — a real stability risk for a production legaltech
  deployment.
- Consistent with how we already call Groq LLM, Groq Whisper, and Comadre TTS
  (all HTTP Request). One integration pattern, not two. Reduces cognitive
  load for any future operator reading the workflow.
- Reduces the template's dependency footprint (NFR41) — one fewer npm package
  a new firm has to install and audit.
- If Evolution API ships new endpoints before the community node is updated,
  the workflow is not blocked.

### Bootstrap checklist ("Step 1 — Infrastructure Setup")

This is the n8n-project equivalent of the "first implementation story / run
the create command" that the step's conventional framing expects. Must be
executed once before any workflow construction begins:

1. Verify BorgStack-mini is healthy at 10.10.10.205 — n8n reachable in queue
   mode, Postgres + pgvector accepting connections, Redis reachable, Caddy
   + Cloudflare Tunnel serving inbound webhooks over HTTPS, Evolution API
   instance exists and is WhatsApp-connected.
2. Verify n8n version ≥ 2.15.1; upgrade BorgStack n8n container if needed.
   Pin the version in deployment notes.
3. Verify `N8N_ENCRYPTION_KEY` is set, persistent, and backed up out-of-band.
4. Verify Comadre is reachable at 10.10.10.207 from BorgStack's internal
   network and `/v1/audio/speech` with `voice=kokoro/pm_santa` returns audio.
5. Create the Telegram **refinement** bot via @BotFather; store as n8n
   credential `telegram_bot_refinement`.
6. (Deferred) Create the Telegram **production** bot as
   `telegram_bot_production` — not activated yet.
7. Create Groq API credentials `groq_free` and `groq_paid` in n8n.
8. Create Comadre credential `comadre_tts` (if behind an auth layer).
9. Create Postgres credential `postgres_main` (database `chatbot` or
   equivalent per BorgStack conventions).
10. Create Redis credential `redis_main`.
11. Create Evolution API credential `evolution_api` (base URL + apikey header).
12. Run `sql/001-init-tables.sql` against `postgres_main` to create
    `client_profiles`, `chat_analytics`, `static_responses`, and
    staging-prefixed refinement tables. Do NOT pre-create `n8n_chat_histories`
    — the Postgres Chat Memory node auto-creates it on first run.
13. Create a private Telegram group for handoff notifications; add the
    refinement bot as a member with permission to post; store the group's
    `chat_id` as n8n environment variable `HANDOFF_GROUP_CHAT_ID`.
14. Create the main workflow (empty) and all 9 sub-workflows (empty) in n8n
    with placeholder Manual Trigger nodes, using the naming convention defined
    later in this architecture document.

**Completion gate:** when all 14 items are green, PRD §21 progression steps
2-34 (workflow construction) can begin.

**Note:** because this project has no `npx create-*` equivalent, the bootstrap
checklist above serves the role that a "project initialization story" would
in a conventional software project.

## Core Architectural Decisions

### Decision Priority Analysis

**Critical decisions (block construction):**

- Unified internal message schema (Ingestion → all downstream layers)
- Audit trail resilience model (NFR10 vs NFR17 resolution)
- Refinement vs production isolation (two workflow files on same instance)
- Sub-workflow invocation pattern (Call Tool for agent tools, Execute for pipeline)
- Analytics logger fire-and-forget
- Idempotency key source and TTL
- Session key derivation
- Conversation state column ownership
- Error Workflow routing model (severity-based switch)
- SW-4 Wait-on-Webhook handoff pattern

**Important decisions (shape architecture significantly):**

- Schema versioning via idempotent numbered SQL files
- Refinement schema isolation via prefixed tables on same database
- Static response library stored in Postgres, not markdown
- Prompt loading: paste-into-AI-Agent-node-field only
- Retention automation via n8n Schedule Trigger sub-workflow (not pg_cron)
- Rate limit window pairing (5-min burst + daily cap)
- Webhook origin validation per platform
- PII sanitization in SW-9 as a documented Code node exception
- Two Telegram groups (handoff and ops) with distinct credentials
- `ENVIRONMENT` env var signaling in every workflow

**Deferred decisions:**

- Compaction sub-workflow (deferred until production data justifies, see below)
- Cross-channel profile merging (deferred, requires dedup UI)
- Metabase dashboard (BorgStack-mini does not include it)
- Append-only Postgres trigger hardening (added as a later progression step)
- External secrets vault (out of scope)
- RAG over legal knowledge base (PRD §Out of Scope)

### Data Architecture

**Database & schema**

- **Single PostgreSQL instance with pgvector**, shared with other BorgStack
  services on the mini node. Database name: `chatbot` (or per BorgStack
  conventions during the bootstrap).
- **Schema versioning via idempotent numbered SQL files** in `sql/`:
  `001-init-client-profiles.sql`, `002-init-chat-analytics.sql`,
  `003-init-static-responses.sql`, `004-init-staging-tables.sql`. Every
  statement uses `CREATE ... IF NOT EXISTS` so re-running is safe. Galvani
  runs them manually via `psql` against the BorgStack Postgres container.
  **No migration framework** — one less dependency for the template.
- **Schema isolation for refinement vs production:** same database, prefixed
  tables. Production: `client_profiles`, `chat_analytics`, `static_responses`,
  `n8n_chat_histories`. Refinement: `staging_client_profiles`,
  `staging_chat_analytics`, `staging_n8n_chat_histories`. `static_responses`
  is shared read-only reference data. Rejected alternatives: separate database
  (doubles backup/admin overhead), separate schema (complicates n8n credential
  wiring).

**Table ownership by component**

| Table | Owner | Written by | Read by |
|---|---|---|---|
| `client_profiles` | SW-2 Save Client Info | SW-2 (upsert) | Triage Agent (returning-client lookup), SW-4 Handoff Flow, SW-9 Audit Bundle Exporter (for name dictionary), retention sub-workflow |
| `chat_analytics` | SW-7 Analytics Logger | SW-7 (append-only) | SW-8 Weekly Report, SW-9 Audit Bundle Exporter, operator ad-hoc SQL, retention sub-workflow |
| `static_responses` | Manual (attorney via SQL) | SQL init + manual updates | Triage Agent (via `lookup_practice_areas` tool or Layer 4 deterministic router) |
| `n8n_chat_histories` | Postgres Chat Memory node | Auto-managed by node | Auto-managed by node |

**Conversation state:** stored in `client_profiles.status` column, enumerated
as `new`, `qualifying`, `qualified`, `handed_off`, `active_human`, `closed`.
Transitions logged to `chat_analytics` with `event_type='state_transition'`
for forensic reconstruction. No separate state table.

**Returning-client session key:**

- Telegram: `chat.id` (integer, stringified)
- WhatsApp: normalized E.164 phone number without `+` prefix
- No cross-channel profile merging in v1.0. One person contacting on both
  channels creates two profiles. Deferred.

**Audit trail — resolving NFR10 (append-only) vs NFR17 (never block
conversation):**

- `chat_analytics` is **append-only by policy**: no UPDATE statements anywhere
  in the workflow; no DELETE except by the automatic retention sub-workflow.
  A Postgres trigger enforcing this at the database level is added as a later
  progression step (post-v1.0 hardening).
- `chat_analytics` writes are **best-effort via async fire-and-forget** from
  SW-7 Analytics Logger, invoked via `Execute Sub-Workflow` without wiring its
  output back into the main flow. SW-7's Postgres Insert node has
  `Continue on Fail = true`. On failure, SW-7 writes a minimal incident
  record to Redis under `analytics_failure:{session_id}:{timestamp}` with 24h
  TTL and fires a low-severity Error Trigger event.
- A reconciliation sub-workflow (`SW-Reconcile`, added later in progression)
  runs every 10 minutes and drains Redis failure records back into
  `chat_analytics`.
- **Resolution:** successful writes are append-only; logging failure never
  blocks the conversation; lost logs are recoverable from Redis during the
  TTL window.

**Retention automation:**

- Dedicated n8n Schedule Trigger sub-workflow (`SW-Retention`, added as a
  later progression step), running nightly, executing the retention rules
  from PRD §"Data Retention Policy":

  | Table / data | Retention | Action |
  |---|---|---|
  | `chat_analytics` | 2 years | DELETE rows older than 2 years |
  | `client_profiles` (inactive) | 5 years since last interaction | DELETE |
  | Audio binary (temp filesystem) | 30 days | Filesystem cleanup via Execute Command |
  | Operational log rows (latency/error subset) | 90 days | DELETE |
  | `audit-bundles/` repo files | 90 days | Execute Command `find -mtime +90 -delete` |

- Retention runs emit a log event to `chat_analytics` so the operation is
  itself auditable.

**Static response library seeding:**

- `003-init-static-responses.sql` seeds the initial per-area content: 8
  entries (one per practice area: previdenciário, trabalhista, bancário,
  saúde, família, imobiliário, empresarial, tributário), each containing a
  warm-closure message + curated alternative resources (Defensoria Pública,
  INSS, TRT, etc.). Content is attorney-approved before Launch Gate.
- Schema: `id SERIAL`, `practice_area VARCHAR`, `content TEXT`,
  `alternative_resources JSONB`, `version INT`, `approved_by VARCHAR`,
  `approved_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`.

**Unified internal message schema** (produced by the Ingestion layer,
consumed by every downstream layer). Field naming: **snake_case** to match
Postgres columns.

```json
{
  "platform": "telegram" | "whatsapp",
  "user_id": "string — chat.id for Telegram, E.164 phone for WhatsApp",
  "user_name": "string — best-effort from Telegram first_name+last_name or WhatsApp pushName",
  "session_key": "string — same as user_id, explicit for memory node wiring",
  "message_id": "string — platform-specific, used for idempotency dedup",
  "message_type": "text" | "voice" | "image" | "document" | "other",
  "text": "string — original text or STT-transcribed text",
  "media_binary_key": "string — n8n binary data key if applicable, else null",
  "media_mime_type": "string — e.g., audio/ogg",
  "input_was_audio": "boolean — true if STT was used on this turn",
  "timestamp": "ISO 8601 UTC",
  "environment": "refinement" | "production",
  "raw_payload": "object — original provider payload for forensic/debug, not exposed to LLM"
}
```

All sub-workflows accept and emit objects conforming to this shape.
`raw_payload` is preserved for audit but explicitly stripped before any
external LLM call to minimize payload size (NFR46).

**Idempotency:** Telegram `update_id` and Evolution API `key.id` stored in
Redis under `idempotency:{platform}:{message_id}` with 24h TTL, checked at
the entry of the ingestion layer via Redis `SET ... NX` pattern. Duplicate
webhook delivery produces a silent skip + info-level log. Satisfies NFR19.

### Authentication & Security

**Rate limiting:**

- Redis counters, two independent windows per `user_id`:
  - **Burst window:** 30 messages per 5 minutes → friendly "please slow down"
    message, skip LLM, increment abuse counter.
  - **Daily cap:** 200 messages per day → friendly "you've sent a lot of
    messages today, an attorney will reach out tomorrow" + status set to
    `closed`.
- Thresholds are n8n environment variables (`RATE_LIMIT_BURST`,
  `RATE_LIMIT_BURST_WINDOW_SEC`, `RATE_LIMIT_DAILY`) for per-firm tuning
  during template replication.
- Implementation: Redis `INCR` + `EXPIRE` nodes followed by an `IF` node.
  Zero Code nodes required.

**Webhook origin validation:**

- **Telegram:** `Telegram Trigger` node uses bot token authentication by
  design — only Telegram can reach the auto-registered webhook URL. No
  additional check needed.
- **Evolution API:** `apikey` header is required and compared against a
  stored credential value in an early `IF` node before any downstream
  processing. The `apikey` lives in a credential, never in the workflow field.
- Cloudflare Tunnel + Caddy already prevent direct IP-bypass attempts.

**PII sanitization in audit bundles (SW-9):**

- Implemented as a **Code node** in SW-9 Audit Bundle Exporter. This is one
  of the documented Code node exceptions, justified because the Guardrails
  node operates on n8n items per execution, not on post-assembled markdown
  bundles.
- Regex patterns for: CPF, CNPJ, BR phone numbers, email addresses, first
  names (using a dictionary loaded from `client_profiles.name` for the
  exported batch, not arbitrary capitalized words).
- Replacement tokens: `[CPF]`, `[CNPJ]`, `[PHONE]`, `[EMAIL]`, `[NAME]`.
- Justification documented on the node itself and in
  `refinement-log/code-node-exceptions.md`.

**Refinement vs production isolation:**

- **Two separate workflow files** on the same n8n instance:
  - `main-chatbot-refinement` — Telegram credential
    `telegram_bot_refinement`, Postgres nodes bound to `staging_*` tables,
    SW-7 writes to `staging_chat_analytics`, `ENVIRONMENT=refinement`.
  - `main-chatbot-production` — Telegram credential `telegram_bot_production`,
    Postgres bound to production tables, SW-7 writes to `chat_analytics`,
    `ENVIRONMENT=production`.
- Identical topology (the production file is a clone of the refinement file).
  Differences are purely credential / variable bindings.
- The exported template ships both workflow files + a deployment guide
  explaining that a new firm activates only the refinement workflow until
  Launch Gate passes, then activates the production workflow.

**Secrets and credentials:**

- All credentials live exclusively in n8n Credentials system. No external
  vault (HashiCorp Vault, AWS SM) — out of scope.
- `N8N_ENCRYPTION_KEY` backed up out-of-band to the firm's encrypted backup
  location.
- BorgStack-managed Postgres backup includes n8n credentials (encrypted with
  `N8N_ENCRYPTION_KEY`, unreadable without it).

### API & Communication Patterns

**LLM call pattern:**

- **AI Agent node** with its **Chat Model sub-node connected to an HTTP
  Request Chat Model** pointing to Groq's OpenAI-compatible endpoint
  (`https://api.groq.com/openai/v1/chat/completions`).
- Rationale: the AI Agent node provides native tool calling, memory
  integration, and structured output parsing. A raw HTTP Request Basic LLM
  Chain loses those features and would force a custom orchestration loop.

**LLM provider failover:**

- Primary credential: `groq_free` (free tier 7-12B multilingual model).
- Fallback credential: `groq_paid` (same provider, paid credits, same model).
- Failover mechanism: AI Agent node's built-in Retry on Fail + fallback
  credential. When `groq_free` returns 429 or 5xx, retry with exponential
  backoff up to 2 attempts, then switch to `groq_paid` automatically.
- Provider-swap case (Groq policy change forcing exit): change the HTTP
  Request Chat Model's base URL + credential, no topology change required
  (satisfies NFR36).

**Sub-workflow invocation pattern — mixed by purpose:**

- **`Call n8n Workflow Tool`** (LLM decides when to invoke) for agent tools:
  - SW-2 `save_client_info` — persist extracted data to `client_profiles`
  - `lookup_practice_areas` — return static data for the 8 areas
  - SW-3 `qualify_lead` — deterministic scoring
  - SW-4 `request_human_handoff` — trigger HITL notification
- **`Execute Sub-Workflow`** (deterministic, fires unconditionally at a fixed
  workflow position) for pipeline sub-workflows:
  - SW-1 Media Processor (voice download + Whisper)
  - SW-5 Follow-Up Relay (triggered from inside SW-4, not by the main flow)
  - SW-7 Analytics Logger (fire-and-forget, no output wiring)
- **Dedicated trigger nodes** for the rest:
  - SW-6 Error Handler — Error Trigger node
  - SW-8 Weekly Report Generator — Schedule Trigger node
  - SW-9 Audit Bundle Exporter — Manual Trigger (operator-invoked)

**Error propagation:**

- Every main workflow and sub-workflow assigns **SW-6 Error Handler** as its
  Error Workflow (Workflow Settings → Error Workflow field).
- SW-6 receives the Error Trigger payload and classifies severity via a
  Switch node:
  - **Critical** (agent completely failed, Postgres unreachable, Groq
    `groq_free` AND `groq_paid` both failing) → immediate Telegram alert to
    `OPS_GROUP_CHAT_ID` + best-effort friendly-message-to-client via direct
    Telegram API call, preserving NFR46 (never silence).
  - **Warning** (TTS failed and fell back to text-only, Whisper timed out,
    one retry before success, output guardrail fired) → log to
    `chat_analytics` with `error_occurred=true`, no alert.
  - **Info** (analytics insert failed and went to Redis reconciliation
    buffer, idempotency skip) → increment Redis counter; alert only if the
    counter crosses a 24h threshold.

**Attorney follow-up wait (SW-4):**

- **Wait node in "Resume on Webhook Call" mode**, not "Time Interval" mode.
- The attorney's Telegram inline keyboard button click POSTs a callback
  query to Telegram, which the `Telegram Trigger` node receives; a dedicated
  callback-handler sub-workflow matches the callback to the paused Wait node
  by session_id and resumes the SW-4 execution.
- Timeout fallback: Wait node's `Resume After Interval` property set to 4h.
  If no webhook arrives in 4h, the Wait node auto-resumes via the timeout
  branch, which escalates to the backup-attorney flow.
- Rationale: Resume-on-Webhook releases the execution loop during the wait
  period, reducing n8n worker memory pressure (helps NFR21 resource budget).

**Prompt loading:**

- System prompts are **pasted directly into the AI Agent node's System
  Prompt field**. The Git file `prompts/triage-agent-v{N}.md` is the source
  of truth.
- Rejected alternatives:
  - Postgres table (adds infrastructure, contradicts the "manual copy-paste
    ritual" principle)
  - Environment variable (n8n env vars have length limits, multi-KB legal
    prompts would hit them)
  - Code node reading a file (violates visual-first discipline)
  - HTTP Request to GitHub raw (silent drift on network failure)

### Messaging Client Surface (the closest thing to Frontend Architecture)

There is no conventional frontend. The Telegram and WhatsApp apps are the
user-facing UI. The attorney-facing surface is a private Telegram group with
a structured notification format and inline keyboard.

**Handoff notification format** (posted to `HANDOFF_GROUP_CHAT_ID`):

```text
🆕 NEW QUALIFIED LEAD — {practice_area}
{name}, {city}
Urgency: {urgency} | Score: {score}/100

📋 Case summary:
{case_summary}

🕐 First contact: {first_contact_ts}
📎 Full transcript: /transcript_{session_id}
```

Plus a 3-button inline keyboard emitting callback queries
`accept:{session_id}`, `info:{session_id}`, `decline:{session_id}`.

**Full transcript retrieval:** the `/transcript_{session_id}` command is
handled by a lightweight Telegram command handler sub-workflow
(`SW-TranscriptViewer`, added as a later progression step). It queries
`chat_analytics` filtered by `session_id`, formats chronologically with
`[voice]` markers where `was_audio=true`, and sends the formatted transcript
back to the same Telegram group.

**Operations group:** separate Telegram group `OPS_GROUP_CHAT_ID`, distinct
from the handoff group. Receives critical alerts only (ERR level), cost
budget warnings, weekly report delivery, workflow-activation notifications.
Plain text format, no inline keyboards.

### Infrastructure & Deployment

**Workflow export pipeline:**

- **Manual export from n8n editor at significant milestones**, committed to
  `exports/workflow-snapshots/YYYY-MM-DD-description.json`.
- No n8n API automation. Consistent with the "manual ritual" principle and
  the decision that every workflow change is an explicit, audited operation.
- The final v1.0 export that becomes the replicable template lives at
  `exports/n8n-workflow-template-v1.json`, tagged with the n8n version it
  was exported from (NFR41).

**Monitoring and alerting delivery:**

- Two Telegram groups (`HANDOFF_GROUP_CHAT_ID`, `OPS_GROUP_CHAT_ID`).
- No Slack, no email, no PagerDuty. All out of scope.
- All observability is either `chat_analytics` queries (ad-hoc via `psql`)
  or Telegram messages (automatic via SW-6 and SW-8).

**Dashboard:**

- **None in v1.0.** Metabase is available in BorgStack cluster mode but
  **not in BorgStack-mini single-node**. Replacement:
  - Ad-hoc `psql` queries for investigation
  - SW-8 Weekly Report Generator delivering aggregates to the ops group
- If BorgStack-mini adds Metabase in the future, the dashboard enters as a
  later progression step, not v1.0 scope.

**Environment signaling:**

- Each workflow file has an early `Set` node that reads the `ENVIRONMENT`
  env var and tags every downstream item with it.
- SW-7 Analytics Logger writes the tag to `chat_analytics.environment`.
- SW-9 Audit Bundle Exporter filters by environment — refinement bundles
  can be exported on demand during refinement; production bundles can only
  be exported post-Launch-Gate with attorney approval recorded.

**Backup dependency:**

- BorgStack-managed daily Postgres backup, 30-day local retention, weekly
  off-site. The architecture depends on this but does not implement it.
- The deployment guide flags this dependency for any firm replicating the
  template; without equivalent Postgres backup, the compliance posture is
  incomplete.

**Resource budget enforcement (NFR21):**

- Architecture commits:
  - Context window capped at 15 messages (prevents token-driven memory
    blow-up on n8n workers)
  - Sub-workflow isolation via `Execute Sub-Workflow` (each runs in its own
    execution)
  - Redis for rate-limit counters instead of n8n workflow state (prevents
    worker RAM growth)
  - Analytics logging fire-and-forget (prevents Postgres latency from
    blocking a worker)
- If real measurements show budget overrun, add a concurrency cap via n8n's
  `Execution Settings → Max Concurrent Executions` as a later progression
  step.

### Memory Compaction — Fork A Resolution

**Decision: Option 1 (skip compaction in v1.0). Option 2 (synchronous
per-turn compaction) is the chosen shape when compaction is added later —
but only if production data justifies it, and the threshold is data-driven,
not pre-committed.**

**Rationale:**

- The Postgres Chat Memory node's `contextWindowLength=15` already truncates
  the agent's visible window automatically.
- The architecture already solves the "remember facts from earlier" problem
  via structured extraction: `save_client_info` persists name, city, case
  summary, urgency to `client_profiles` as soon as they're gathered. At
  handoff time, the agent does not need to remember turn-3 facts — it reads
  them back from the table.
- Legal intake conversations are short (Journey 1 Dona Francisca concludes
  in ~20 turns including confirmations); real sessions comfortably fit the
  15-message window.
- Pre-committing a compaction threshold now would be speculative. The number
  I cited earlier (20 messages) was not well-sourced — the research cites
  "20 interactions = 40 messages" as the sweet-spot boundary, not a
  compaction trigger.

**When compaction is added (future progression step):**

- **Trigger criteria** (both must be met):
  1. 95th percentile session length in `chat_analytics` exceeds ~40 messages.
  2. Attorney weekly review flags "agent re-asked already-answered questions"
     above a tolerance threshold to be defined in SW-8 reporting.
- **Chosen shape:** synchronous per-turn Chat Memory Manager check +
  Basic LLM Chain summarization on the cheap Groq model + Chat Memory Manager
  Override All Messages with the summary + recent 5 messages verbatim.
- **Threshold value:** set from production data at the time the step is
  added, not speculated now.

### Decision Impact Analysis

**Implementation sequence** (which decisions must be committed in which
order, mapped onto PRD §21 progression steps):

| PRD §21 step | Architectural decisions needed |
|---|---|
| 1. Infrastructure setup | All Foundation + Bootstrap decisions |
| 2-3. Echo bot → basic LLM | LLM call pattern, Groq credentials, AI Agent node |
| 4. Short-term memory | Postgres Chat Memory table, session key derivation |
| 5. Normalization & sanitization | Unified internal schema, text preprocessing |
| 6. Deterministic router | Static responses table seeding (SQL 003), router regex |
| 7. System prompt | Prompt loading model (paste-into-field), prompts/ repo layout |
| 8-11. Tools (SW-2, lookup, SW-3, SW-4) | Call n8n Workflow Tool pattern, `client_profiles` schema (SQL 001), handoff notification format, Wait-on-Webhook SW-4 |
| 12-14. Guardrails & rate limiting | Guardrails node config, Redis rate limiting, webhook origin validation, output guardrails placement |
| 15. Error workflow | SW-6 severity routing, Error Workflow assignment pattern |
| 16. Analytics logging | SW-7 fire-and-forget, `chat_analytics` schema (SQL 002), append-only policy, Redis failure buffer |
| 17. Context compaction | **Deferred** — skipped per Fork A |
| 18-19. Voice STT + TTS | SW-1 Media Processor, `input_was_audio` flag flow, Comadre HTTP Request config |
| 20. Static response library | `static_responses` table lookup in unqualified-lead flow |
| 21. WhatsApp adaptation | Unified internal schema extended to WhatsApp, Evolution API HTTP Request, webhook origin validation via `apikey` |
| 22-27. Refinement pipeline | Refinement workflow file clone, `staging_*` tables (SQL 004), SW-9 Audit Bundle Exporter + Code node PII sanitization, prompts/ versioning, golden dataset structure |
| 28. Weekly report | SW-8 aggregation queries, ops group delivery |
| 29-30. Launch gate | Environment variable switchover, both workflow files activation |
| 31. Production switch | `telegram_bot_production` credential activation, production workflow file activation |
| 32-34. Steady state + export + case study | Template export, deployment guide, case study documentation |

**Cross-component dependencies** (decisions that constrain each other):

- **Unified internal schema** is the contract every sub-workflow depends on.
  Changing it is a breaking change across SW-1, SW-2, SW-3, SW-4, SW-5, SW-7.
  Committed now; frozen until Launch Gate.
- **`ENVIRONMENT` env var** is referenced by SW-7 (analytics destination),
  SW-9 (bundle filter), and every workflow's Telegram credential binding.
  Adding a new environment value requires updating all three.
- **SW-6 Error Handler** is assigned as Error Workflow to every other
  workflow and sub-workflow. Adding a new severity level or routing change
  requires SW-6 republish but no changes to the other workflows.
- **`static_responses` table schema** is frozen because SW-4 Handoff Flow's
  decline branch and the deterministic router's unqualified-lead path both
  read from it.
- **`client_profiles.status` enum values** are referenced by SW-4, SW-5,
  SW-8, the retention sub-workflow, and the transcript viewer. Adding a
  state requires coordinated updates.
- **Postgres Chat Memory session key format** (plain `user_id`) is referenced
  by the AI Agent node, SW-9 audit bundle filtering, and right-to-erasure
  operations. Must remain deterministic and collision-free.

## Implementation Patterns & Consistency Rules

### Pattern Scope Reframing

Conventional pattern rules (test file location, JSON response wrappers,
component-by-feature vs component-by-type) do not apply here — the project
produces an n8n visual workflow, Git-versioned markdown prompts, SQL init
files, and operational playbooks, not conventionally-generated code. The
patterns below target the real conflict surface:

1. **Galvani building manually in the n8n visual editor** — needs stable
   conventions to navigate a large workflow months later.
2. **Claude Code operating the refinement loop** — needs structured,
   parseable inputs (audit bundles, prompt files, refinement logs).
3. **Future operators importing the template** — needs predictable names,
   predictable locations, predictable SQL init order (NFR29 ≤ 2h onboarding,
   NFR40 ≤ 8h per-firm deployment).
4. **Attorneys reviewing static responses and prompt changes** — needs
   human-legible formats, version trails, attribution.

### Language Policy

- **User-facing text** (bot messages to clients, handoff card labels, TTS
  audio): **Brazilian Portuguese** (`pt-BR`).
- **Technical artifacts** (workflow node names, table/column names,
  environment variables, credential names, Git commit messages, markdown
  documentation, Code node comments, SQL): **English**.
- **Bilingual surfaces that must be in both:**
  - `static_responses.content` is Portuguese (user-facing) but the schema
    columns (`practice_area`, `content`, `approved_by`) are English.
  - `prompts/*.md` system prompt content is Portuguese (the agent's persona
    speaks Portuguese) but the file's frontmatter and the
    [identity]/[rules]/[tools] section headers are English for consistency
    with the rest of the repo.
- Matches the repo-level rule in `CLAUDE.md`: *all code, comments, commits,
  docs, and file contents in this repository must be in native English.*

### n8n Workflow Naming

**Main workflows:**

- `main-chatbot-refinement` (the refinement-environment workflow)
- `main-chatbot-production` (the production-environment workflow)

Not `Chatbot v1`, `Legal Bot`, `Test Bot`, `Copy of ...`. Case is lowercase
with hyphens. No version number in the workflow name — version belongs in
the exported JSON filename.

**Sub-workflows:** `SW-N <PascalCase Name>` where `N` is the reference
number from PRD §8 (stable across versions):

- `SW-1 Media Processor`
- `SW-2 Save Client Info`
- `SW-3 Lead Qualification`
- `SW-4 Handoff Flow`
- `SW-5 Follow-Up Relay`
- `SW-6 Error Handler`
- `SW-7 Analytics Logger`
- `SW-8 Weekly Report Generator`
- `SW-9 Audit Bundle Exporter`

Sub-workflows added later in the progression receive sequential numbers
beyond 9: `SW-10 Reconciliation`, `SW-11 Retention`, etc. Numbers are
never reused.

**Node naming inside workflows:**

- **Every node gets a descriptive name** — never leave n8n's default
  (`HTTP Request`, `Set`, `IF1`, `Code`).
- **Naming form:** imperative verb + object, title case. Examples:
  `Normalize Text Input`, `Check Rate Limit`, `Route by Intent`,
  `Build Handoff Card`, `Call Groq Chat Completion`,
  `Wait for Attorney Response`.
- **Exception:** trigger nodes keep their default name with a descriptive
  suffix: `Telegram Trigger — Client Messages`, `Webhook — Evolution API`,
  `Schedule Trigger — Weekly Report`, `Error Trigger — All Workflows`.
- Why it matters: NFR29 — a new operator must understand the workflow in
  ≤ 2 hours of visual inspection. Generic node names destroy this.

### Credential Naming

- **Format:** `snake_case`, `<service>_<purpose_or_environment>`.
- **Committed names** (locked as the template contract):
  - `telegram_bot_refinement`
  - `telegram_bot_production`
  - `groq_free` (primary operational LLM)
  - `groq_paid` (failover credential)
  - `comadre_tts`
  - `postgres_main`
  - `redis_main`
  - `evolution_api`
- When a new firm imports the template, they recreate these credentials
  with their own values — but the names stay identical so the workflow
  topology does not need to change.

### Environment Variable Naming

- **Format:** `SCREAMING_SNAKE_CASE`, grouped by prefix:
  - `HANDOFF_GROUP_CHAT_ID` — private Telegram group for lead notifications
  - `OPS_GROUP_CHAT_ID` — ops group for critical alerts and weekly reports
  - `ENVIRONMENT` — `refinement` or `production`
  - `RATE_LIMIT_BURST` — integer, messages in burst window
  - `RATE_LIMIT_BURST_WINDOW_SEC` — integer, window size
  - `RATE_LIMIT_DAILY` — integer, per-day cap
  - `MODEL_PRIMARY` — Groq model identifier (e.g., `llama-3.3-70b-versatile`
    or whichever 7-12B multilingual model is in use)
  - `MODEL_WHISPER` — Groq Whisper model identifier
  - `TTS_VOICE` — Comadre voice preset (`kokoro/pm_santa` for Maruzza)
  - `FIRM_NAME` — e.g., `Maruzza Teixeira Advocacia`
  - `FIRM_CITY` — e.g., `São Luís/MA`
- Every firm-specific value lives here or in credentials, never hardcoded
  in a workflow node (FR67, NFR39).

### Database Naming Conventions

- **Table names:** `snake_case`, plural for collections (`client_profiles`,
  `chat_analytics`, `static_responses`), singular only when the table holds
  a true singleton (none in v1.0).
- **Column names:** `snake_case`. `created_at` / `updated_at` / `deleted_at`
  are the conventional timestamp names.
- **Primary keys:** `id SERIAL PRIMARY KEY` (or `BIGSERIAL` when expected
  volume justifies it). No UUIDs in v1.0 — not needed.
- **Foreign keys:** `{referenced_table_singular}_id`. E.g., `client_id`
  references `client_profiles.id`.
- **Indexes:** `idx_{table}_{column(s)}`. E.g.,
  `idx_chat_analytics_session_id`, `idx_chat_analytics_created_at`.
- **Enums:** stored as `VARCHAR(N)` with a `CHECK` constraint listing
  allowed values (Postgres native enums would require migration steps we
  don't have a framework for — plain VARCHAR + CHECK is template-friendly).
- **JSONB** for semi-structured data (e.g., `client_profiles.meta`,
  `static_responses.alternative_resources`).
- **Timestamps:** `TIMESTAMPTZ` always, never `TIMESTAMP` without timezone.
  Stored in UTC, displayed in operator local time via Telegram's native
  rendering or `psql` `SET TIMEZONE`.

### n8n Expression Conventions

n8n supports several expression syntaxes for referencing data across nodes:

- **Primary style — use this by default:** `{{ $json.field }}` for the
  current item, and `{{ $('Node Name').item.json.field }}` for cross-node
  references. Always use the parenthesized form `$('Node Name')` over the
  deprecated `$node["Node Name"]` bracket form.
- **When referencing nested fields:** use the dot form when the field is a
  valid identifier (`$json.user_name`), and bracket form only for keys with
  special characters (`$json['key-with-dash']`).
- **Null-safety:** use optional chaining `$json.field?.subfield` to avoid
  runtime errors on missing fields. Never rely on `|| null` patterns that
  hide bugs.
- **Date/time:** use n8n's built-in Luxon helpers where possible
  (`{{ $now.toISO() }}`, `{{ $now.toFormat('yyyy-MM-dd') }}`), never
  bespoke `Date` manipulation in expressions.
- **Avoid multi-line expressions:** if an expression spans more than one
  line or contains more than one ternary, extract the logic into an
  upstream `Set` node with named fields. This is the visual-first
  discipline applied to expression readability.

### Code Node Acceptance Criteria

A Code node is **admissible only** when all four conditions are met:

1. **No native n8n node covers the need.** Checked against: Set, IF, Switch,
   Merge, SplitInBatches, ExtractFromFile, ConvertToFile, EditFields,
   DateTime, Filter, AggregationLoop, HTTP Request, and the AI Agent +
   Guardrails stack.
2. **No combination of 3-4 visual nodes with expressions solves it** with
   comparable clarity. Visual cascades with Set/IF/Switch are preferred
   even when slightly more verbose.
3. **The language is JavaScript.** Python is never used in Code nodes
   (memory rule: JS-only in Code nodes, even though n8n supports Python).
4. **The node has a documented justification** — a header comment at the
   top of the Code node explaining *why* no visual combination works, and
   a line-item entry in `refinement-log/code-node-exceptions.md` with the
   date, sub-workflow, and rationale.

**Documented Code node exceptions in v1.0** (pre-authorized at this step):

- **SW-9 Audit Bundle Exporter — PII sanitization.** Post-assembly regex
  sanitization of a concatenated markdown bundle exceeds what the
  Guardrails node handles per-item. Documented.
- **`Normalize Incoming Message` node in the Ingestion layer** *if*
  Telegram and WhatsApp payload normalization exceeds what a Set + Switch
  combination can express cleanly. Provisional — committed as an exception
  only if the visual approach proves unwieldy during construction.

**Anti-patterns explicitly forbidden:**

- Code node that duplicates a Postgres INSERT via `pg` library calls
  (use the Postgres node).
- Code node that builds HTTP request payloads via `fetch` (use HTTP Request
  node with expressions).
- Code node that shells out via `execSync` (use Execute Command node if
  genuinely needed, but prefer n8n built-ins).
- Code node whose justification is "it was faster to write" — speed is
  not a justification.

### Message Schema Consistency

The unified internal message schema (committed in Core Decisions §Data
Architecture) is a **frozen contract** across every sub-workflow. Rules:

1. **Every sub-workflow that accepts a message object must accept the full
   schema** even if it uses only a subset. No partial-object handoffs.
2. **Field names never change in flight.** A sub-workflow may *add*
   enrichment fields but may not rename or delete existing fields. A
   sub-workflow's output must be a superset of its input.
3. **Reserved extension prefixes:**
   - `meta_` — metadata computed during processing (e.g., `meta_lang`,
     `meta_has_url`, `meta_word_count`)
   - `guard_` — guardrail decisions (e.g., `guard_jailbreak_score`,
     `guard_topical_score`)
   - `route_` — routing outcomes (e.g., `route_intent`, `route_path`)
4. **`raw_payload` is never sent to external services.** It is stripped
   before every Groq / Comadre call. Enforced by the LLM call nodes
   explicitly constructing their request body from the normalized fields.
5. **Schema changes require a versioned entry** in
   `docs/message-schema-changelog.md` and a rationale. Retroactive schema
   changes in a running workflow require a migration story in the
   progression sequence.

### Error Classification Taxonomy

SW-6 Error Handler classifies every Error Trigger event into exactly one
of three levels. Each level has a deterministic routing rule:

| Severity | Criteria | Routing |
|---|---|---|
| **CRITICAL** | Agent completely failed; Postgres unreachable; both `groq_free` AND `groq_paid` failing; Evolution API disconnected; Telegram API persistently down | Immediate Telegram message to `OPS_GROUP_CHAT_ID` with full error context + best-effort friendly-message-to-client via direct Telegram/Evolution API call + `chat_analytics` row with `error_occurred=true` and `severity='critical'` |
| **WARNING** | TTS failed and fell back to text-only; Whisper timed out and fallback message sent; output guardrail fired and response rewritten; one retry before success; 429 from primary before fallback succeeded | `chat_analytics` row with `error_occurred=true` and `severity='warning'`; no operator alert unless rate exceeds threshold |
| **INFO** | Analytics insert failed and went to Redis reconciliation buffer; idempotency skip; deterministic router resolved without LLM | Redis counter increment; alert only if counter exceeds 24h threshold; no direct operator message |

**Classification is deterministic in SW-6** — a Switch node matches on the
error source, node name, and error message pattern. No LLM involvement in
classification.

### Logging and Audit Conventions

- **All logs go to `chat_analytics`** via SW-7, never to stdout, never to
  n8n's native execution history (which is subject to default 168h
  retention and not suitable as a forensic audit trail).
- **One `chat_analytics` row per event**, never aggregated batches. Row
  types (stored in an `event_type` column):
  - `user_message` — incoming user turn
  - `agent_response` — outgoing agent turn
  - `tool_call` — agent invoked a tool
  - `guardrail_fire` — input or output guardrail triggered
  - `state_transition` — conversation state changed
  - `handoff_sent` — lead package posted to ops group
  - `handoff_response` — attorney clicked an inline button
  - `error` — captured by SW-6 at any severity
  - `rate_limit_hit` — user hit burst or daily cap
  - `idempotency_skip` — duplicate webhook silenced
- **Required columns on every row:** `session_id`, `telegram_id` (or
  WhatsApp phone), `event_type`, `created_at`, `environment`, `severity`
  (nullable except on `error`), `model_used` (nullable except on
  `agent_response` and `tool_call`), `tokens_input`, `tokens_output`,
  `response_time_ms`, `message_content` (nullable for structural events).
- **No PII masking at write time.** `chat_analytics` stores raw content for
  forensic reconstruction (NFR31). PII masking happens at *export* time via
  SW-9's sanitization Code node.

### SQL File Conventions

- **File naming:** `NNN-<action>-<scope>.sql` — numeric prefix zero-padded
  to 3 digits, lowercase kebab-case. Examples:
  - `001-init-client-profiles.sql`
  - `002-init-chat-analytics.sql`
  - `003-init-static-responses.sql`
  - `004-init-staging-tables.sql`
- **Every statement is idempotent** — `CREATE TABLE IF NOT EXISTS`,
  `CREATE INDEX IF NOT EXISTS`, `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`,
  `INSERT ... ON CONFLICT DO NOTHING`. Re-running a file is always safe.
- **Every file ends with a `SELECT 1 AS ok;` sanity check** so the operator
  can confirm execution.
- **No `GRANT` / `REVOKE` / `CREATE ROLE` statements** — BorgStack manages
  roles and permissions at the infrastructure level.
- **Comments in English** at the top of each file explaining: purpose,
  tables created, indexes created, constraints added, dependencies on
  prior files.

### Prompt File Conventions

- **Location:** `prompts/<agent-name>-v<N>.md`
- **Versioning:** monotonically increasing integer in the filename. Never
  edit a published version in place — create a new version file.
- **Filename examples:**
  - `prompts/triage-agent-v1.md`
  - `prompts/triage-agent-v2.md`
  - `prompts/handoff-composer-v1.md`
- **Required frontmatter:**
  ```yaml
  ---
  agent: triage-agent
  version: 2
  supersedes: 1
  author: Galvani
  attorney_approved_by: Maruzza Teixeira
  attorney_approved_at: 2026-04-26
  motivation: Disambiguate "chefe" between labor and banking contexts
  golden_dataset_pass_rate_before: 0.92
  golden_dataset_pass_rate_after: 1.00
  related_audit_bundle: audit-bundles/2026-04-26-cycle-3.md
  related_refinement_log: refinement-log/2026-04-26-triage-v1-to-v2.md
  ---
  ```
- **Body structure** (enforced so Claude Code can diff prompts meaningfully
  and attorneys can review consistent sections):
  ```text
  ## [IDENTITY]
  ## [GOAL]
  ## [CONVERSATION FLOW]
  ## [QUALIFYING CRITERIA BY AREA]
  ## [TOOLS]
  ## [RULES]
  ## [OUTPUT FORMAT]
  ## [EXAMPLES]
  ```
- Section order is fixed. Adding a new section requires a refinement log
  entry justifying it.

### Refinement Log Conventions

- **Location:** `refinement-log/YYYY-MM-DD-<scope>-v<from>-to-v<to>.md`
- **Filename examples:**
  - `refinement-log/2026-04-26-triage-v1-to-v2.md`
  - `refinement-log/2026-05-03-router-static-responses-v3-to-v4.md`
- **Required frontmatter:**
  ```yaml
  ---
  date: 2026-04-26
  scope: triage-agent
  from_version: 1
  to_version: 2
  author: Galvani
  frontier_auditor: Claude Opus 8.6 via Claude Code
  cycle_type: model-and-human
  trigger: attorney weekly review flagged 2 misclassifications
  motivation: Disambiguate "chefe" between labor and banking contexts
  golden_dataset_before: 0.92
  golden_dataset_after: 1.00
  attorney_approval: Maruzza Teixeira (inline review, 2026-04-26)
  ---
  ```
- **Body structure:**
  - **Context:** what triggered this cycle, what logs or reviews informed it
  - **Analysis:** the frontier auditor's diagnosis
  - **Proposed change:** the exact diff (prompt before/after)
  - **Golden dataset delta:** pass rate, specific cases now passing or
    failing
  - **Attorney review notes:** what the attorney asked to change in the
    proposal before approving
  - **Production deployment:** when the prompt was copy-pasted into n8n
- **Cycle types enumerated:**
  - `synthetic-scenarios` — Cycle 1
  - `chatbot-vs-chatbot` — Cycle 2
  - `internal-human-tests` — Cycle 3
  - `model-and-human` — Cycle 4 (the operator + frontier auditor loop)
  - `golden-regression` — Cycle 5
  - `adversarial-red-team` — Cycle 6
  - `attorney-review` — Cycle 7

### Audit Bundle Conventions

- **Location:** `audit-bundles/YYYY-MM-DD-cycle-N.md`
- **Cycle number:** monotonically increasing integer, never reused. Cycle 1
  is the first bundle ever exported.
- **Required frontmatter:**
  ```yaml
  ---
  exported_at: 2026-04-26T14:22:00-03:00
  exported_by: SW-9 Audit Bundle Exporter
  environment: production
  filter_criteria: "flagged_by_attorney_review OR lead_score < 60"
  conversation_count: 12
  sanitization_pass: true
  sanitizer_version: 1
  ---
  ```
- **Body:** each conversation is a `## Conversation {session_id}` section
  with a header block (metadata), the chronological message thread with
  `[voice]` markers, the agent's tool calls, and the final outcome. All PII
  is replaced with typed placeholders.
- **Anonymization is a hard requirement.** SW-9 must not export a bundle
  with `sanitization_pass: false`.

### Git Repository Layout and Commit Conventions

- **Directory layout** (committed in full in the next step — Project
  Structure — this is a preview for context):
  ```text
  n8n-chatbot/
  ├── CLAUDE.md
  ├── _bmad-output/planning-artifacts/
  │   ├── prd.md
  │   ├── architecture.md
  │   └── ...
  ├── docs/
  ├── research/
  ├── prompts/
  │   ├── triage-agent-v1.md
  │   └── handoff-composer-v1.md
  ├── sql/
  │   ├── 001-init-client-profiles.sql
  │   ├── 002-init-chat-analytics.sql
  │   ├── 003-init-static-responses.sql
  │   └── 004-init-staging-tables.sql
  ├── exports/
  │   ├── workflow-snapshots/
  │   │   └── 2026-04-26-first-agent-with-memory.json
  │   └── n8n-workflow-template-v1.json
  ├── audit-bundles/
  │   ├── 2026-04-26-cycle-1.md
  │   └── ...
  ├── refinement-log/
  │   ├── 2026-04-26-triage-v1-to-v2.md
  │   └── code-node-exceptions.md
  ├── golden-dataset/
  │   ├── previdenciario-01.md
  │   └── ...
  ├── synthetic-scenarios/
  ├── red-team-battery/
  └── human-test-logs/
  ```
- **Commit messages in English**, imperative mood, under 72 chars for the
  subject. Body (if any) explains *why*, not *what*. Examples:
  - `Add SW-4 Handoff Flow with Wait-on-Webhook attorney resume`
  - `Refine triage-agent to disambiguate "chefe" labor vs banking`
  - `Export workflow snapshot at progression step 11`
- **Never squash refinement log entries** — the case study depends on the
  per-cycle granularity.

### Timestamp Handling

- **Storage:** UTC, `TIMESTAMPTZ` in Postgres, ISO 8601 in JSON.
- **Display:** Brazilian time (`America/Sao_Paulo`, UTC-3) rendered at the
  messaging client. n8n expressions convert via Luxon
  `{{ $now.setZone('America/Sao_Paulo').toFormat('dd/LL/yyyy HH:mm') }}`.
- **Audit bundles and refinement logs:** ISO 8601 with timezone offset
  `-03:00` for unambiguous operator-time reading.
- **No bare `Date.now()` or `new Date()` in Code nodes** — use
  `DateTime.now()` via Luxon for timezone-aware operations.

### Enforcement Guidelines

**All implementors (Galvani, Claude Code, future operators) MUST:**

- Follow node naming convention — imperative verb + object, descriptive
- Use snake_case in Postgres and in the internal message schema
- Use SCREAMING_SNAKE_CASE for environment variables
- Use the committed credential names verbatim
- Never hardcode firm-specific data (use env vars or credentials)
- Document every Code node with a justification header comment and an
  entry in `refinement-log/code-node-exceptions.md`
- Preserve the full message schema across sub-workflow boundaries
- Log every legally-relevant event to `chat_analytics` via SW-7
- Keep prompt content in Portuguese but structural markers in English
- Write commit messages in English
- Export workflow JSON snapshots at significant progression steps

**Pattern verification (reviewed during Cycle 7 — attorney review, and
during each Claude Code refinement session):**

- `grep -r` the repo for forbidden patterns: hardcoded firm name, "Haiku",
  "GPT-4o-mini", "GPT-4o" in prompts or comments, Python in Code nodes,
  unjustified Code nodes, `new Date()` in Code nodes.
- Visual inspection of the n8n editor for generic node names (`HTTP Request`,
  `Set`, `IF1`) — any found are renamed before the next snapshot export.
- SQL file idempotency check: run each `sql/*.sql` file twice in sequence
  against a fresh database — the second run must produce zero errors and
  zero changes.

### Pattern Examples

**Good example — descriptive node naming:**

```text
[Telegram Trigger — Client Messages]
        ↓
[Normalize Telegram Payload]
        ↓
[Check Idempotency (Redis SET NX)]
        ↓
[Route by Message Type]
 ├─ text ──→ [Sanitize Text Input]
 ├─ voice ──→ [Execute SW-1 Media Processor]
 └─ other ──→ [Flag for Human Review]
```

**Anti-pattern — what to avoid:**

```text
[Telegram Trigger]
        ↓
[Code]              ← no name, no justification
        ↓
[Set2]              ← numeric suffix, no description
        ↓
[Switch1]           ← generic, unreadable in 3 weeks
```

**Good example — credential binding:**

```text
AI Agent → Chat Model sub-node → HTTP Request Chat Model
    credential: groq_free
    fallback credential: groq_paid
    model: {{ $env.MODEL_PRIMARY }}
```

**Anti-pattern — hardcoded values:**

```text
AI Agent → Chat Model sub-node
    api_key: sk_groq_abc123...    ← NEVER in a workflow field
    model: "llama-3.3-70b"        ← NEVER hardcoded, use env var
```

## Project Structure & Boundaries

### Four Surfaces of This Project

Unlike conventional software projects, this project lives on **four distinct
surfaces**, and the "project structure" must describe all four:

1. **Git repository (`~/dev/n8n-chatbot/`)** — source of truth for prompts,
   SQL, exports, audit bundles, refinement logs, documentation, case study
2. **n8n workflows** (runtime objects inside the n8n editor on
   BorgStack-mini)
3. **PostgreSQL schema** (tables inside the `chatbot` database on
   BorgStack-mini)
4. **Redis keyspace** (ephemeral state inside the BorgStack-mini Redis)

External services (Groq, Comadre, Telegram, Evolution API) are not part of
the project structure per se — they are integration boundaries.

### Surface 1 — Git Repository Structure

```text
n8n-chatbot/
├── README.md                             # Case study entry point (authored later in progression)
├── CLAUDE.md                             # Project conventions and permissions (EXISTS)
├── .gitignore                            # Node modules, OS junk, secrets-safe
│
├── _bmad-output/                         # BMad planning artifacts (EXISTS)
│   └── planning-artifacts/
│       ├── product-brief-n8n-chatbot.md              # (EXISTS)
│       ├── product-brief-n8n-chatbot-distillate.md   # (EXISTS)
│       ├── prd.md                                     # (EXISTS, authoritative)
│       ├── implementation-readiness-report-2026-04-10.md  # (EXISTS, premature pass)
│       └── architecture.md                            # (THIS DOCUMENT)
│
├── research/                             # Deep research (EXISTS)
│   ├── 01-agent-patterns-and-optimization.md
│   ├── 01b-agent-patterns-supplementary.md
│   ├── 02-memory-rag-and-context.md
│   ├── 03-media-integrations-and-security.md
│   └── 04-production-operations.md
│
├── docs/                                 # Project documentation
│   ├── plan.md                           # (EXISTS, historical — pre-PRD planning)
│   ├── claude-code-instruction-quality-guide.md  # (EXISTS, unrelated meta-guide)
│   ├── deployment-guide.md               # Template deployment steps for new firms (progression step 33)
│   ├── credentials-checklist.md          # List of required credentials with types + purpose (progression step 33)
│   ├── environment-variables.md          # All env vars, defaults, per-firm tuning (progression step 33)
│   ├── telegram-bot-setup.md             # @BotFather walk-through (progression step 1)
│   ├── evolution-api-setup.md            # Evolution API webhook + apikey config (progression step 21)
│   ├── message-schema-changelog.md       # Unified internal message schema versions
│   └── case-study.md                     # Journey narrative for external readers (progression step 34)
│
├── prompts/                              # Git-versioned system prompts (source of truth)
│   ├── triage-agent-v1.md                # (progression step 7)
│   ├── triage-agent-v2.md                # (added by refinement cycles)
│   ├── triage-agent-v3.md
│   └── handoff-composer-v1.md            # Prompt for lead package composition (progression step 11)
│
├── sql/                                  # Idempotent database init files
│   ├── 001-init-client-profiles.sql      # (progression step 8)
│   ├── 002-init-chat-analytics.sql       # (progression step 16)
│   ├── 003-init-static-responses.sql     # (progression step 20)
│   └── 004-init-staging-tables.sql       # (progression step 24)
│
├── exports/                              # n8n workflow JSON exports
│   ├── workflow-snapshots/               # Snapshots at significant progression milestones
│   │   ├── 2026-04-NN-infra-verified.json
│   │   ├── 2026-04-NN-first-agent-with-memory.json
│   │   ├── 2026-04-NN-all-tools-wired.json
│   │   ├── 2026-04-NN-guardrails-complete.json
│   │   ├── 2026-05-NN-voice-bidirectional.json
│   │   ├── 2026-05-NN-whatsapp-added.json
│   │   └── 2026-MM-NN-launch-gate-passed.json
│   └── n8n-workflow-template-v1.json     # Final v1.0 template (progression step 33)
│
├── audit-bundles/                        # Exported conversation bundles for refinement cycles
│   ├── 2026-MM-NN-cycle-1.md             # First cycle (synthetic scenarios)
│   ├── 2026-MM-NN-cycle-2.md
│   └── ...
│
├── refinement-log/                       # Per-cycle refinement records
│   ├── 2026-MM-NN-triage-v1-to-v2.md
│   ├── 2026-MM-NN-static-responses-v1-to-v2.md
│   ├── ...
│   └── code-node-exceptions.md           # Running log of every Code node + justification
│
├── golden-dataset/                       # Versioned canonical test conversations
│   ├── previdenciario-01-denied-bpc.md
│   ├── previdenciario-02-retirement-denial.md
│   ├── previdenciario-03-disability-appeal.md
│   ├── trabalhista-01-dismissal-without-cause.md
│   ├── trabalhista-02-overtime-violation.md
│   ├── trabalhista-03-workplace-accident.md
│   ├── bancario-01-abusive-interest.md
│   ├── bancario-02-debt-renegotiation.md
│   ├── bancario-03-unauthorized-charge.md
│   ├── saude-01-denial-of-coverage.md
│   ├── saude-02-treatment-access.md
│   ├── saude-03-malpractice.md
│   ├── familia-01-divorce-contested.md
│   ├── familia-02-child-custody.md
│   ├── familia-03-inheritance.md
│   ├── imobiliario-01-title-dispute.md
│   ├── imobiliario-02-eviction.md
│   ├── imobiliario-03-buyer-default.md
│   ├── empresarial-01-recovery.md
│   ├── empresarial-02-contract-dispute.md
│   ├── empresarial-03-partnership-dissolution.md
│   ├── tributario-01-assessment-dispute.md
│   ├── tributario-02-compliance-support.md
│   ├── tributario-03-penalty-reduction.md
│   ├── edge-ambiguous-area-01.md         # Gray-zone scenarios
│   ├── edge-unqualified-gracefully-01.md
│   ├── edge-aggressive-client-01.md
│   ├── edge-out-of-scope-01.md
│   ├── edge-returning-client-01.md
│   └── edge-voice-transcription-artifacts-01.md
│
├── synthetic-scenarios/                  # Generated test scenarios (Cycle 1)
│   ├── batch-2026-04-NN-previdenciario.md
│   ├── batch-2026-04-NN-trabalhista.md
│   └── ...
│
├── red-team-battery/                     # Adversarial inputs (Cycle 6)
│   ├── jailbreak-attempts.md             # DAN, system-prompt extraction, obfuscation
│   ├── legal-advice-extraction.md        # "What should I do?" variants
│   ├── pii-leak-probes.md                # Cross-session data leak attempts
│   ├── topical-breakouts.md              # Off-topic probe messages
│   ├── prompt-injection-indirect.md      # Injection via pasted content
│   └── results-2026-MM-NN.md             # Per-run pass/fail log
│
├── human-test-logs/                      # Internal human tester feedback (Cycle 3)
│   ├── 2026-MM-NN-pedro-truck-driver-labor.md
│   ├── 2026-MM-NN-maria-homemaker-social-security.md
│   └── ...
│
└── (no package.json, no node_modules, no dist/, no CI/CD workflows)
```

**Notable absences** (deliberate):

- **No `package.json`** — this is not a Node project. The `prompts/` files
  are markdown, not a build target.
- **No `node_modules/`** — nothing to install.
- **No `.github/workflows/`** — no CI. n8n workflows cannot run in a
  GitHub Actions environment without bespoke infrastructure, and the
  refinement cycle is operator-paced anyway. Repo hygiene (link checking,
  markdown lint) can be added later if needed.
- **No `tests/`** — the testing story is the offline refinement pipeline
  (`golden-dataset/`, `synthetic-scenarios/`, `red-team-battery/`,
  `human-test-logs/`), not traditional unit tests.
- **No `src/`** — no source code.

**BMad `_bmad/` directory** (already in repo) is BMad's own config/working
directory, distinct from `_bmad-output/` (the artifacts). Not documented in
the tree above because it's tooling infrastructure rather than project
content.

### Surface 2 — n8n Workflows (runtime objects)

**Main workflows** (two, identical topology, differing in credentials):

- **`main-chatbot-refinement`** — activated first, bound to the refinement
  bot and staging tables
- **`main-chatbot-production`** — cloned from refinement, activated at
  Launch Gate

**Sub-workflows** (inventory with IDs, purpose, invocation pattern):

| ID | Name | Purpose | Invocation | FRs satisfied |
|---|---|---|---|---|
| SW-1 | Media Processor | Voice download → Groq Whisper STT → text; detect images/docs | `Execute Sub-Workflow` from main, inside ingestion layer | FR3, FR4, FR10 |
| SW-2 | Save Client Info | Upsert `client_profiles` from extracted data | `Call n8n Workflow Tool` (agent tool) | FR12, FR13, FR14, FR15 |
| SW-3 | Lead Qualification | Deterministic scoring via Set cascade | `Call n8n Workflow Tool` (agent tool) | FR16, FR17 |
| SW-4 | Handoff Flow | Compose package, notify ops group, Wait-on-Webhook, route decision | `Call n8n Workflow Tool` (agent tool) | FR18, FR19, FR20, FR21, FR22, FR23, FR24 |
| SW-5 | Follow-Up Relay | Relay attorney "Need More Info" question to client, collect answer | `Execute Sub-Workflow` from SW-4 | FR22 |
| SW-6 | Error Handler | Severity classification + routing to ops group / friendly client message | `Error Trigger` (assigned as Error Workflow on all other workflows) | FR46, FR47, NFR32 |
| SW-7 | Analytics Logger | Append-only insert to `chat_analytics`, fire-and-forget | `Execute Sub-Workflow` (without awaiting) | FR32, FR57, NFR10, NFR17, NFR30 |
| SW-8 | Weekly Report Generator | Aggregate metrics, deliver to ops group | `Schedule Trigger` | FR59, FR61 |
| SW-9 | Audit Bundle Exporter | Query flagged conversations, sanitize PII, write markdown bundle | `Manual Trigger` (operator-invoked) | FR54, NFR47 |
| SW-10 | Reconciliation (deferred) | Drain Redis failure records into `chat_analytics` | `Schedule Trigger` every 10 min | NFR17 tension resolution |
| SW-11 | Retention (deferred) | Execute per-table retention rules nightly | `Schedule Trigger` nightly | FR33 |
| SW-12 | Transcript Viewer (deferred) | Handle `/transcript_{session_id}` command from ops group | `Telegram Trigger` on ops group | FR19 (expanded) |

**Main workflow layer topology** (matches PRD §8):

```text
Layer 0 — Messaging Client (Telegram / WhatsApp)
              ↓
Layer 1 — Ingestion
  ├─ Telegram Trigger — Client Messages         (telegram_bot_* credential)
  ├─ Webhook — Evolution API                    (apikey validation IF)
  ├─ Check Idempotency (Redis SET NX)
  └─ Normalize to Unified Schema                (emits the canonical message object)
              ↓
Layer 2 — Media Processing (Execute SW-1 if message_type=voice)
              ↓
Layer 3 — Security Pre-Check
  ├─ Check Rate Limit Burst (Redis INCR + TTL)
  ├─ Check Rate Limit Daily (Redis INCR + TTL)
  ├─ Sanitize Text Input (Set node with inline regex)
  └─ Input Guardrails Node (jailbreak + topical + PII)
              ↓
Layer 4 — Deterministic Router (Switch + Set cascade)
  ├─ /start, /help, /menu               → Static response
  ├─ greetings                          → Welcome message
  ├─ FAQ patterns                       → static_responses lookup or Set response
  ├─ thanks / bye                       → Polite closure
  └─ fall-through                       → Layer 5
              ↓
Layer 5 — Triage Agent (AI Agent node)
  ├─ Chat Model: HTTP Request → Groq (groq_free / groq_paid fallback)
  ├─ Memory: Postgres Chat Memory (session_key = user_id, window=15)
  ├─ System Prompt: pasted from prompts/triage-agent-v{N}.md
  ├─ Tools:
  │    ├─ SW-2 save_client_info (Call n8n Workflow Tool)
  │    ├─ lookup_practice_areas (Call n8n Workflow Tool, static data)
  │    ├─ SW-3 qualify_lead (Call n8n Workflow Tool)
  │    └─ SW-4 request_human_handoff (Call n8n Workflow Tool)
  └─ Output Guardrails Node (PII + no-legal-advice patterns)
              ↓
    ┌─────────┴─────────┐
    ▼                   ▼
Normal Reply    Qualified Lead → SW-4 Handoff Flow
    │
    ▼
Layer 6 — Response Delivery
  ├─ Send Text (Telegram Action / Evolution API HTTP Request)
  └─ IF input_was_audio:
       ├─ Call Comadre TTS (HTTP Request, Continue on Fail = true)
       └─ Send Voice (Telegram Action / Evolution API HTTP Request)
              ↓
Layer 7 — Post-Response (fire-and-forget)
  └─ Execute SW-7 Analytics Logger (no output wiring)
```

### Surface 3 — PostgreSQL Schema

**Database:** `chatbot` (or per BorgStack conventions during bootstrap)
**Schema:** `public`

**Tables (production):**

```sql
-- Table: client_profiles
-- SQL file: sql/001-init-client-profiles.sql
-- Purpose: long-term memory of every client who has interacted
CREATE TABLE IF NOT EXISTS client_profiles (
    id              BIGSERIAL PRIMARY KEY,
    platform        VARCHAR(20)  NOT NULL CHECK (platform IN ('telegram','whatsapp')),
    user_id         VARCHAR(64)  NOT NULL,          -- chat.id or E.164 phone
    user_name       VARCHAR(255),
    phone           VARCHAR(32),
    city            VARCHAR(100),
    practice_area   VARCHAR(30)  CHECK (practice_area IN
        ('previdenciario','trabalhista','bancario','saude','familia',
         'imobiliario','empresarial','tributario','undefined')),
    case_summary    TEXT,
    urgency         VARCHAR(16)  DEFAULT 'normal'
                      CHECK (urgency IN ('low','normal','high','urgent')),
    lead_score      INTEGER      DEFAULT 0,
    status          VARCHAR(20)  DEFAULT 'new'
                      CHECK (status IN ('new','qualifying','qualified',
                                        'handed_off','active_human','closed')),
    assigned_attorney VARCHAR(100),
    handed_off_at   TIMESTAMPTZ,
    meta            JSONB        DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ  DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  DEFAULT NOW(),
    UNIQUE (platform, user_id)
);
CREATE INDEX IF NOT EXISTS idx_client_profiles_user_id   ON client_profiles(user_id);
CREATE INDEX IF NOT EXISTS idx_client_profiles_status    ON client_profiles(status);
CREATE INDEX IF NOT EXISTS idx_client_profiles_updated   ON client_profiles(updated_at);
```

```sql
-- Table: chat_analytics
-- SQL file: sql/002-init-chat-analytics.sql
-- Purpose: append-only forensic audit trail
CREATE TABLE IF NOT EXISTS chat_analytics (
    id              BIGSERIAL PRIMARY KEY,
    session_id      VARCHAR(128) NOT NULL,          -- derived from user_id
    telegram_id     VARCHAR(64),                    -- or WhatsApp phone
    environment     VARCHAR(20)  NOT NULL
                      CHECK (environment IN ('refinement','production')),
    event_type      VARCHAR(32)  NOT NULL
                      CHECK (event_type IN (
                        'user_message','agent_response','tool_call',
                        'guardrail_fire','state_transition','handoff_sent',
                        'handoff_response','error','rate_limit_hit',
                        'idempotency_skip')),
    severity        VARCHAR(16)  CHECK (severity IN
                                        ('critical','warning','info')),
    message_role    VARCHAR(20),
    message_content TEXT,
    model_used      VARCHAR(100),
    tokens_input    INTEGER,
    tokens_output   INTEGER,
    response_time_ms INTEGER,
    was_audio       BOOLEAN      DEFAULT FALSE,
    tool_name       VARCHAR(64),
    guardrail_name  VARCHAR(64),
    error_occurred  BOOLEAN      DEFAULT FALSE,
    error_context   JSONB,
    meta            JSONB        DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ  DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_chat_analytics_session_id  ON chat_analytics(session_id);
CREATE INDEX IF NOT EXISTS idx_chat_analytics_created_at  ON chat_analytics(created_at);
CREATE INDEX IF NOT EXISTS idx_chat_analytics_environment ON chat_analytics(environment);
CREATE INDEX IF NOT EXISTS idx_chat_analytics_event_type  ON chat_analytics(event_type);
CREATE INDEX IF NOT EXISTS idx_chat_analytics_severity    ON chat_analytics(severity) WHERE severity IS NOT NULL;
```

```sql
-- Table: static_responses
-- SQL file: sql/003-init-static-responses.sql
-- Purpose: pre-approved per-area content library for unqualified leads
CREATE TABLE IF NOT EXISTS static_responses (
    id                     SERIAL PRIMARY KEY,
    practice_area          VARCHAR(30) NOT NULL
                             CHECK (practice_area IN
                               ('previdenciario','trabalhista','bancario',
                                'saude','familia','imobiliario','empresarial',
                                'tributario','general')),
    content                TEXT        NOT NULL,  -- Brazilian Portuguese
    alternative_resources  JSONB       DEFAULT '[]'::jsonb,
    version                INTEGER     NOT NULL DEFAULT 1,
    approved_by            VARCHAR(100),
    approved_at            TIMESTAMPTZ,
    created_at             TIMESTAMPTZ DEFAULT NOW(),
    updated_at             TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (practice_area, version)
);
CREATE INDEX IF NOT EXISTS idx_static_responses_area ON static_responses(practice_area);
-- Seed data inserted via ON CONFLICT DO NOTHING pattern
```

```sql
-- Table: n8n_chat_histories
-- Auto-created by n8n Postgres Chat Memory node on first run
-- DO NOT pre-create. Schema is owned by n8n.
```

**Staging tables** (refinement environment, same schema, prefixed names):
`staging_client_profiles`, `staging_chat_analytics`,
`staging_n8n_chat_histories`. Created by `sql/004-init-staging-tables.sql`.
`static_responses` is shared read-only between environments.

### Surface 4 — Redis Keyspace

**Namespace conventions** (all keys prefixed):

| Key pattern | Purpose | TTL | Written by | Read by |
|---|---|---|---|---|
| `idempotency:telegram:{update_id}` | Dedupe Telegram webhook retries | 24h | Ingestion layer | Ingestion layer |
| `idempotency:whatsapp:{key_id}` | Dedupe Evolution API webhook retries | 24h | Ingestion layer | Ingestion layer |
| `rate_limit:burst:{user_id}` | Burst window counter (5 min) | `RATE_LIMIT_BURST_WINDOW_SEC` | Layer 3 Security | Layer 3 Security |
| `rate_limit:daily:{user_id}` | Daily cap counter | 86400s | Layer 3 Security | Layer 3 Security |
| `analytics_failure:{session_id}:{timestamp}` | Buffered failed insert | 24h | SW-7 on failure | SW-10 Reconciliation (deferred) |
| `error_count:{severity}:{hour}` | SW-6 severity counter for threshold alerting | 86400s | SW-6 | SW-6 |
| `abuse_counter:{user_id}` | Rate-limit-hit accumulator | 604800s (7d) | Layer 3 Security | SW-8 Weekly Report |

**Keyspace rules:**

- No unprefixed keys — every key starts with a namespace from the table above.
- TTLs are mandatory — no persistent Redis keys.
- Values are small (counters, timestamps, single booleans) — Redis is not
  used for conversation state or memory.

### External Integration Boundaries

| # | Service | Endpoint | Protocol | Credential | Retry | Fallback |
|---|---|---|---|---|---|---|
| 1 | Telegram (client bot) | Auto-registered webhook URL | Telegram Bot API | `telegram_bot_refinement` / `telegram_bot_production` | Telegram retries automatically; n8n `Retry on Fail` on send (exponential, max 3) | None — Telegram outage means platform-level degradation |
| 2 | Telegram (ops group) | Same Telegram API | Telegram Bot API | Same bot credential | Same | Same |
| 3 | WhatsApp via Evolution API | `POST /message/*` + webhook `MESSAGES_UPSERT` | HTTP/REST | `evolution_api` (base URL + apikey header) | n8n `Retry on Fail` (exponential, max 3); Evolution API auto-reconnects | CONNECTION_UPDATE event triggers ops-group alert |
| 4 | Groq LLM (chat) | `https://api.groq.com/openai/v1/chat/completions` | HTTPS OpenAI-compatible | `groq_free` primary, `groq_paid` fallback | n8n `Retry on Fail` (exponential, max 2 per credential) | Automatic credential failover from `groq_free` → `groq_paid`; if both fail, SW-6 CRITICAL |
| 5 | Groq Whisper (STT) | `https://api.groq.com/openai/v1/audio/transcriptions` | HTTPS multipart upload | `groq_free` primary, `groq_paid` fallback | Same as LLM | Same; if Whisper fails, friendly "please type your message" response (FR45) |
| 6 | Comadre TTS | `http://10.10.10.207:8000/v1/audio/speech` | HTTP (internal network) OpenAI-compatible | `comadre_tts` | `Retry on Fail` (exponential, max 2) + `Continue on Fail = true` | Text-only response if all retries fail (FR44) |

**Trust boundaries:**

- **Trusted, on-premise:** BorgStack-mini (n8n, Postgres, Redis, Caddy,
  Cloudflare Tunnel, Evolution API), Comadre (separate dedicated server,
  internal network).
- **External, conditionally trusted:** Groq (requires zero-retention
  contract validation pre-Launch-Gate — NFR7).
- **External, platform-trusted:** Telegram (signed requests, bot token),
  WhatsApp via Evolution API (Baileys protocol over Meta's WhatsApp Web).
- **Operator-local, out-of-band:** Claude Code subscription (running on
  Galvani's workstation, reads the Git repo, writes to `prompts/*.md`,
  never directly touches the running n8n instance).
- **Remote, repo only:** Git remote (source control, no runtime effect).

**What crosses each boundary:**

- **n8n → Groq:** minimum payload — current conversation window (10-15
  messages) + system prompt + current user turn. Never the full history,
  never aggregated multi-client data. Enforced by the Postgres Chat Memory
  node's window configuration.
- **n8n → Comadre:** response text only. No conversation history, no
  metadata. Simple TTS request.
- **n8n → Telegram:** messages, inline keyboards, file uploads. Bot token
  in header.
- **n8n ↔ Evolution API:** messages, media, webhook events. apikey header.
- **SW-9 → Git repo:** anonymized audit bundle markdown written to
  `audit-bundles/`. PII sanitized by Code node before write.
- **Claude Code → Git repo:** prompt file edits in `prompts/`, refinement
  log entries in `refinement-log/`. No direct n8n access.
- **Galvani → n8n:** manual copy-paste of prompt content from
  `prompts/triage-agent-v{N}.md` into the AI Agent node's System Prompt
  field. Manual ritual.
- **Galvani → Postgres:** manual SQL execution via `psql` for `sql/*.sql`
  files and ad-hoc investigation.

### Requirements-to-Structure Mapping

**Functional Requirements (70 FRs):**

| FR | Component(s) | Location |
|---|---|---|
| FR1-2 | Telegram Trigger + Evolution API Webhook | Layer 1 Ingestion (main workflow) |
| FR3 | SW-1 Media Processor | Sub-workflow, invoked from Layer 2 |
| FR4 | SW-1 → Groq Whisper HTTP Request | Inside SW-1 |
| FR5, FR6 | Layer 6 Response Delivery + Comadre HTTP Request | Main workflow |
| FR7 | Postgres Chat Memory sub-node on AI Agent | Layer 5 Triage Agent |
| FR8 | Postgres Chat Memory `contextWindowLength=15` (compaction deferred) | Layer 5 |
| FR9 | Returning-client lookup via Postgres node, injected into system prompt | Layer 5 pre-agent Set node |
| FR10 | Layer 1 Normalize node detects non-text `message_type`, routes to SW-4 with "image/document received" flag | Layer 1 → SW-4 |
| FR11 | Triage Agent system prompt + `lookup_practice_areas` tool | Layer 5 / prompts/triage-agent-v1.md |
| FR12-15 | SW-2 Save Client Info + `client_profiles` table | Sub-workflow + Postgres |
| FR16-17 | SW-3 Lead Qualification (Set cascade scoring) | Sub-workflow |
| FR18-24 | SW-4 Handoff Flow (compose, notify, Wait-on-Webhook, branches) | Sub-workflow |
| FR25-27 | `static_responses` table + Layer 5 unqualified branch | Postgres + main workflow |
| FR28 | System prompt [IDENTITY] + first-turn disclosure node | prompts/triage-agent-v1.md + Layer 5 |
| FR29 | System prompt [LGPD CONSENT] + first-turn consent node | prompts/triage-agent-v1.md + Layer 5 |
| FR30-31 | 6-layer no-legal-advice (prompt rules + tool design + static lib + output guardrails + red team + attorney review) | All surfaces |
| FR32 | SW-7 Analytics Logger + `chat_analytics` table | Sub-workflow + Postgres |
| FR33 | SW-11 Retention (deferred) | Sub-workflow |
| FR34 | Right-to-erasure ad-hoc SQL + operator procedure in `docs/deployment-guide.md` | SQL + doc |
| FR35 | Layer 3 Rate Limit (Redis INCR + IF) | Main workflow |
| FR36 | Layer 3 Sanitize Text Input (Set with inline regex) | Main workflow |
| FR37-39 | Layer 3 Input Guardrails Node (jailbreak + topical + PII) | Main workflow |
| FR40 | Postgres Chat Memory session key + UNIQUE constraint on `client_profiles(platform, user_id)` | n8n node + Postgres schema |
| FR41 | n8n Credentials system + `N8N_ENCRYPTION_KEY` | Platform |
| FR42 | Retry on Fail on every external HTTP Request node | Main workflow + sub-workflows |
| FR43 | AI Agent node fallback credential (`groq_free` → `groq_paid`) | Layer 5 |
| FR44 | Comadre HTTP Request `Continue on Fail = true` + text-only branch | Layer 6 |
| FR45 | SW-1 Whisper HTTP Request `Continue on Fail = true` + "please type" fallback | SW-1 |
| FR46 | SW-6 Error Handler friendly-message-to-client routing | Sub-workflow |
| FR47 | SW-6 Error Handler severity classification | Sub-workflow |
| FR48 | `main-chatbot-refinement` workflow + `staging_*` tables | n8n + Postgres |
| FR49 | `synthetic-scenarios/` directory + Claude Code generation sessions | Git repo |
| FR50 | `main-chatbot-refinement` driven by a secondary "persona bot" workflow (progression step 25) | n8n |
| FR51 | `telegram_bot_refinement` + `human-test-logs/` briefings and feedback | n8n + Git repo |
| FR52 | `golden-dataset/` + Claude Code regression session or dedicated n8n Evaluation workflow | Git repo + n8n |
| FR53 | `red-team-battery/` + Claude Code or dedicated test workflow | Git repo + n8n |
| FR54 | SW-9 Audit Bundle Exporter + `audit-bundles/` | Sub-workflow + Git repo |
| FR55 | `refinement-log/` attorney review entries | Git repo |
| FR56 | Launch Gate gate = manual activation of `main-chatbot-production` (no real bot token until gate passes) | Operational |
| FR57 | SW-7 + `chat_analytics` | Sub-workflow + Postgres |
| FR58 | Ad-hoc `psql` + SW-8 Weekly Report | Operational + sub-workflow |
| FR59 | SW-8 Weekly Report Generator | Sub-workflow |
| FR60 | Cost calculation inside SW-8 based on `tokens_input + tokens_output` aggregation | Sub-workflow |
| FR61 | SW-6 threshold alerting (Redis counter + IF on every event) | Sub-workflow |
| FR62 | SW-8 includes "attorney flagged" section + `chat_analytics` `meta` field for flag marking | Sub-workflow + Postgres |
| FR63 | Manual export to `exports/n8n-workflow-template-v1.json` | Git repo |
| FR64 | Template import + credential recreation per `docs/credentials-checklist.md` | Operational |
| FR65 | `sql/001-*.sql` through `sql/004-*.sql` + staging variant | Git repo |
| FR66 | `docs/credentials-checklist.md` + `docs/deployment-guide.md` | Git repo |
| FR67 | `prompts/triage-agent-v1.md` + `prompts/handoff-composer-v1.md` | Git repo |
| FR68 | Deployment dry-run procedure in `docs/deployment-guide.md` | Operational |
| FR69 | `refinement-log/YYYY-MM-DD-*.md` entries | Git repo |
| FR70 | `docs/case-study.md` + the repo as a whole | Git repo |

**Non-Functional Requirements (47 NFRs):**

| NFR | Architectural enforcement |
|---|---|
| NFR1 | Workflow topology + p95 budget respected by Layer 4 deterministic router + Layer 5 fast model |
| NFR2 | Layer 4 Deterministic Router + `route_` meta fields for counting |
| NFR3 | Postgres Chat Memory `contextWindowLength=15`, compaction deferred |
| NFR4 | Groq prompt caching enabled on LLM HTTP Request (when provider supports it) |
| NFR5 | Main workflow 30s execution ceiling + fallback message path |
| NFR6 | External providers restricted to Groq + Comadre only by architectural commitment |
| NFR7 | Manual Groq policy validation step in `docs/deployment-guide.md` — Launch Gate requirement |
| NFR8 | `client_profiles.UNIQUE(platform, user_id)` + Postgres Chat Memory session key |
| NFR9 | n8n Credentials system + `N8N_ENCRYPTION_KEY` documented in bootstrap checklist |
| NFR10 | Append-only discipline on `chat_analytics` (trigger hardening as a later progression step) |
| NFR11 | Right-to-erasure operator procedure + 48h SLA in `docs/deployment-guide.md` |
| NFR12 | `red-team-battery/` + Launch Gate criterion 1 |
| NFR13 | AI disclosure + LGPD consent baked into `prompts/triage-agent-v1.md` hardened sections (non-removable by refinement) |
| NFR14 | Output Guardrails Node (Layer 5 post-agent) |
| NFR15 | Degraded-mode branches in Layer 6 (text-only when Comadre fails) + SW-7 fire-and-forget |
| NFR16 | AI Agent fallback credential mechanism |
| NFR17 | SW-7 fire-and-forget pattern |
| NFR18 | `docs/deployment-guide.md` maintenance window policy |
| NFR19 | Redis idempotency keys |
| NFR20 | 3× headroom measured against Groq free tier quotas (documented in `docs/environment-variables.md`) |
| NFR21 | Concurrency caps set in n8n Execution Settings per `docs/deployment-guide.md` |
| NFR22 | Provider failover by credential only — no topology change |
| NFR23 | `chat_analytics` indexes defined in `sql/002-init-chat-analytics.sql` |
| NFR24 | ≥ 90% visual nodes discipline + Code node exceptions tracked in `refinement-log/code-node-exceptions.md` |
| NFR25 | Each sub-workflow has a Description field documenting inputs and outputs |
| NFR26 | `prompts/` as Git-versioned source of truth |
| NFR27 | `refinement-log/` structured entries |
| NFR28 | 6-layer main workflow limit enforced by architecture |
| NFR29 | `docs/case-study.md` + descriptive node naming + `docs/deployment-guide.md` |
| NFR30 | `chat_analytics` schema + SW-7 event types |
| NFR31 | Forensic reconstruction procedure documented in `docs/deployment-guide.md` — depends on `chat_analytics` indexes |
| NFR32 | SW-6 Error Handler + ops group Telegram delivery |
| NFR33 | SW-8 Weekly Report Generator — independent of external dashboards |
| NFR34 | Retry on Fail policy on every external call (documented in `docs/deployment-guide.md`) |
| NFR35 | Integration contracts documented in `docs/evolution-api-setup.md`, `docs/deployment-guide.md` |
| NFR36 | Credential-only LLM provider swap (HTTP Request Chat Model with env-var model identifier) |
| NFR37 | Evolution API apikey IF check + Telegram bot-token platform trust |
| NFR38 | `exports/n8n-workflow-template-v1.json` self-contained |
| NFR39 | All firm-specific values externalized via `MODEL_*`, `FIRM_*`, `TTS_VOICE`, credentials |
| NFR40 | `docs/deployment-guide.md` ≤ 8h target + dry-run validation |
| NFR41 | `docs/credentials-checklist.md` + n8n version pin in exported JSON filename |
| NFR42 | Cost tracking in SW-7 + SW-8 |
| NFR43 | Claude Code subscription is operator-owned, out of workflow scope |
| NFR44 | SW-6 threshold alerting at 80% of budget (added by SW-8 monthly aggregation) |
| NFR45 | `chat_analytics`, `client_profiles`, `n8n_chat_histories` all on firm's Postgres |
| NFR46 | Minimum payload policy enforced by explicit Groq HTTP Request body construction |
| NFR47 | SW-9 PII sanitization Code node + `sanitization_pass` frontmatter flag |

### Data Flow — Happy Path (Journey 1, Dona Francisca, voice)

```text
1. Francisca records voice → sends to Telegram bot
2. Telegram Trigger node (main-chatbot-production) fires
3. Normalize Telegram Payload → emits unified schema with message_type=voice
4. Check Idempotency (Redis SET NX on telegram:update_id:{N}) → new, proceeds
5. Route by Message Type → voice → Execute SW-1 Media Processor
6. SW-1 downloads OGG via Telegram Get File → sends to Groq Whisper
   → returns text + input_was_audio=true
7. Back in main: Check Rate Limit Burst (Redis INCR) → under limit
8. Check Rate Limit Daily (Redis INCR) → under limit
9. Sanitize Text Input (Set node) → clean text
10. Input Guardrails Node (jailbreak + topical + PII) → pass
11. Route by Intent (Layer 4 Deterministic Router) → fall-through
    (not a greeting/FAQ)
12. Triage Agent (Layer 5):
    - Query client_profiles by user_id → not found (new client)
    - Postgres Chat Memory returns empty window
    - System prompt + AI disclosure + LGPD consent in turn 1
    - Agent responds warmly in Portuguese, asks Francisca to tell her story
13. Output Guardrails Node (PII + no-legal-advice patterns) → pass
14. Send Text (Telegram Action) → Francisca sees the text
15. IF input_was_audio: Call Comadre TTS → Send Voice → Francisca hears
    the audio
16. Execute SW-7 Analytics Logger (fire-and-forget) → one user_message row
    + one agent_response row
17. Workflow execution ends

[Francisca sends another voice message describing her husband's BPC
 denial...]
[Workflow fires again, similar pipeline]
[After several turns, agent has enough info]
18. Agent decides to call save_client_info tool → SW-2 upserts
    client_profiles
19. Agent decides to call qualify_lead tool → SW-3 returns score=85,
    qualified=true
20. Agent confirms data with Francisca → Francisca confirms
21. Agent calls request_human_handoff tool → SW-4 Handoff Flow:
    - Composes lead package (name, city, practice_area, case_summary,
      urgency, transcript)
    - Posts notification to HANDOFF_GROUP_CHAT_ID with inline keyboard
    - Wait node enters Resume-on-Webhook mode, 4h timeout
    - Main workflow pauses for this session
22. Maruzza taps "Accept" in the ops group
23. Telegram callback webhook fires → wakes up the paused SW-4 execution
24. SW-4 sends confirmation message to Francisca via Telegram
25. SW-4 updates client_profiles.status='handed_off', handed_off_at=NOW()
26. SW-7 logs handoff_sent + handoff_response events
```

### Data Flow — Refinement Cycle 4 (model + human, Journey 4)

```text
1. SW-8 Weekly Report delivered to ops group, Galvani sees misclassification
   flags
2. Galvani triggers SW-9 Audit Bundle Exporter via Manual Trigger with
   filter: {flagged_by_attorney=true AND created_at > 2026-04-19}
3. SW-9 queries staging_chat_analytics + staging_client_profiles
4. SW-9 assembles markdown bundle, runs PII sanitization Code node
5. SW-9 writes audit-bundles/2026-04-26-cycle-3.md to the Git repo
   (via local file system write)
6. Galvani opens Claude Code in ~/dev/n8n-chatbot/
7. Galvani: "analyze audit-bundles/2026-04-26-cycle-3.md against
   prompts/triage-agent-v1.md"
8. Claude Code reads both files, identifies pattern ("chefe" context
   disambiguation)
9. Claude Code proposes diff, Galvani reviews, Claude writes
   prompts/triage-agent-v2.md
10. Galvani runs golden dataset regression locally (Claude Code +
    golden-dataset/*.md)
11. All scenarios pass, Galvani commits: "Refine triage-agent to
    disambiguate 'chefe' labor vs banking"
12. Galvani writes refinement-log/2026-04-26-triage-v1-to-v2.md with
    attorney approval note
13. Galvani opens n8n editor at 10.10.10.205, navigates to
    main-chatbot-refinement
14. Galvani opens AI Agent node, copies contents of
    prompts/triage-agent-v2.md, pastes into System Prompt field
15. Galvani saves the workflow (new execution uses v2 from here on)
16. Next weekly report will show impact
```

### Development Workflow Integration

**Construction workflow** (follows PRD §21 progression steps 2-34):

1. Build in `main-chatbot-refinement` on BorgStack-mini
2. At each significant step, export workflow JSON to
   `exports/workflow-snapshots/`
3. Run the latest prompt against `golden-dataset/` via Claude Code
4. Commit artifacts (prompts, SQL, snapshots, refinement logs)
5. Push to Git remote at milestones

**Refinement workflow** (Cycles 1-7):

- **Cycle 1 (synthetic-scenarios):** generate scenarios in Claude Code →
  feed to refinement bot → log to `staging_chat_analytics` → export bundle
  → review
- **Cycle 2 (chatbot-vs-chatbot):** dedicated "persona bot" workflow drives
  the refinement bot → log → review
- **Cycle 3 (internal-human-tests):** distribute briefings, collect
  feedback in `human-test-logs/`
- **Cycle 4 (model-and-human):** the Claude Code flow described above
- **Cycle 5 (golden-regression):** Claude Code session or n8n Evaluation
  workflow against `golden-dataset/`
- **Cycle 6 (adversarial-red-team):** `red-team-battery/` executed against
  refinement bot, 100% pass required
- **Cycle 7 (attorney-review):** Maruzza reads 30+ conversations, signs
  formal approval documented in `refinement-log/`

**Launch Gate passage:**

- All 7 Launch Gate criteria documented as green in
  `refinement-log/YYYY-MM-DD-launch-gate-passed.md`
- Maruzza's formal approval attached as a document or markdown file
- Switch from refinement to production = activate `main-chatbot-production`
  workflow in n8n + attach the production Telegram bot token

**Template export (Milestone 3):**

- Export `main-chatbot-production` as
  `exports/n8n-workflow-template-v1.json`
- Finalize `docs/deployment-guide.md`, `docs/credentials-checklist.md`,
  `docs/environment-variables.md`
- Perform dry-run import on a fresh BorgStack-mini equivalent (different
  VPS or VM)

## Architecture Validation Results

### Coherence Validation — PASSED

- **Decision compatibility:** All technology choices work together without
  conflicts. n8n 2.15.1 + PostgreSQL + pgvector + Redis + Groq +
  Comadre is a fully compatible stack; no version conflicts; all
  integrations are either native n8n nodes or HTTP-over-OpenAI-compatible.
- **Pattern consistency:** snake_case in Postgres → internal message
  schema → analytics columns. Sub-workflow `SW-N` naming → inventory →
  structure. Credential names referenced identically everywhere.
- **Structure alignment:** repo tree, n8n sub-workflow inventory, Postgres
  schema, and Redis keyspace are internally consistent and aligned to the
  PRD §8 blueprint.
- **Trust boundaries:** Groq never receives full history (enforced by
  Postgres Chat Memory window). Comadre never receives conversation data.
  SW-9 sanitizes before any PII leaves BorgStack.

### Requirements Coverage Validation — PASSED WITH PATCHES

All 70 FRs and 47 NFRs are mapped to architectural components in §Project
Structure & Boundaries "Requirements-to-Structure Mapping". Validation
found 2 mapping gaps, both patched inline in this step:

- **FR50 (chatbot-vs-chatbot simulation):** referenced a persona bot
  component not in the sub-workflow inventory → added **SW-13 Persona Bot**
  (deferred, progression step 25).
- **NFR13 (AI disclosure + LGPD consent undefeatable):** was convention-only
  → patched with **hardened prompt section markers**
  (`<!-- HARDENED:START -->` / `<!-- HARDENED:END -->`) + refinement-log
  `hardened_sections_unchanged: true` checkbox + git-diff self-check in
  every refinement cycle.

All other FRs and NFRs are architecturally supported as documented in the
requirements mapping tables in §Project Structure & Boundaries.

### Implementation Readiness Validation — PASSED WITH PATCHES

- **Decision completeness:** every critical architectural decision is
  committed with rationale. Fork A (memory compaction) resolved: v1.0 skips
  compaction, Option 2 deferred with data-driven threshold.
- **Structure completeness:** four surfaces (Git repo, n8n workflows,
  Postgres schema, Redis keyspace) mapped concretely, no placeholders.
- **Pattern completeness:** 14 pattern categories cover naming, structure,
  format, communication, and process conflict points relevant to an n8n
  workflow project.

Validation found 3 readiness gaps, all patched inline:

- **Cycle 5 and Cycle 6 execution venue was stated as "Claude Code or
  dedicated n8n Evaluation workflow"** — patched to commit Claude Code for
  both, leveraging the operator's subscription and avoiding template bloat.
- **Pattern verification had no concrete home** — patched to a
  human-executed checklist in `docs/deployment-guide.md` §"Pre-Launch Gate
  verification" (not a runnable script, since we are not introducing a
  `scripts/` directory or CI infrastructure).
- **Telegram dual-entry-path routing** (client messages vs ops group
  callback queries vs ops commands) was implicit in the topology diagram —
  patched with an explicit `Route Telegram Update` Switch node
  specification documenting Branches A/B/C/D.

### Gap Analysis — Patches Applied Inline

**Critical gaps:** none.

**Important gaps patched during validation:**

1. **SW-13 Persona Bot added to sub-workflow inventory.** Deferred,
   progression step 25. Schedule Trigger-driven, reads persona definitions
   from `synthetic-scenarios/personas/*.md`, drives simulated messages into
   the refinement bot via a direct HTTP call to the Telegram Bot API, logs
   both sides to `staging_chat_analytics` with `event_type='simulated_user'`
   / `'simulated_agent'`. Exits on handoff, decline, or configured turn cap.

2. **Hardened prompt section enforcement.** `prompts/triage-agent-v*.md`
   wraps the AI disclosure block and LGPD consent block in
   `<!-- HARDENED:START -->` and `<!-- HARDENED:END -->` markers.
   Every refinement log frontmatter requires `hardened_sections_unchanged:
   true`. A `git diff` check in each refinement cycle verifies that no
   line between the hardened markers has changed. Any modification to a
   hardened section requires a PRD amendment, not just attorney approval —
   reflecting NFR13's "cannot be disabled without explicit documented
   change to this PRD" language.

3. **Cycle 5 (golden regression) and Cycle 6 (adversarial red team)
   committed to Claude Code.** Rationale: operator-paced, interactive,
   matches refinement discipline, zero per-token frontier cost under the
   subscription, avoids n8n template bloat from Evaluation workflows.

4. **Pattern verification formalized as a checklist in
   `docs/deployment-guide.md` §"Pre-Launch Gate verification".**
   Human-executed, not a runnable script. Items include:
   - `grep -rn "Haiku\|GPT-4o-mini\|GPT-4o" prompts/ docs/` → zero hits
   - `grep -rn "Maruzza Teixeira" exports/` → zero hits in exported JSON
     (firm name externalized to `FIRM_NAME` env var)
   - `grep -rn "new Date()" refinement-log/code-node-exceptions.md`
     → zero hits
   - Visual inspection of n8n editor for generic node names
     (`HTTP Request`, `Set`, `IF1`) — renamed before snapshot export
   - `psql -f sql/001-init-client-profiles.sql` twice in sequence on a
     fresh DB → second run produces zero errors and zero changes

5. **Telegram dual-entry-path routing.** `main-chatbot-production` uses a
   single `Telegram Trigger — Client Messages` node followed by a
   `Route Telegram Update` Switch node with four branches:
   - **Branch A — Client message:** `chat.id != HANDOFF_GROUP_CHAT_ID` AND
     `message != undefined` → full ingestion pipeline
   - **Branch B — Ops callback query:** `callback_query != undefined` AND
     `callback_query.message.chat.id == HANDOFF_GROUP_CHAT_ID` → routed to
     SW-4's callback resume path, matching `callback_query.data` against
     `accept:{session_id}` / `info:{session_id}` / `decline:{session_id}`
   - **Branch C — Ops command:** `chat.id == HANDOFF_GROUP_CHAT_ID` AND
     `message.text` starts with `/transcript_` → SW-12 Transcript Viewer
     (deferred)
   - **Branch D — Ignored:** anything else, logged as INFO with
     `event_type='unhandled_update'`
   - Same pattern in `main-chatbot-refinement` bound to a distinct
     refinement group chat via `HANDOFF_GROUP_CHAT_ID` env var.

**Nice-to-have gaps (deferred):**

- A lightweight `scripts/` directory with pattern-lint and SQL-idempotency
  helpers would be nice but adds a dependency footprint that conflicts
  with the "zero infrastructure for the template" discipline. Deferred.
- Mermaid diagrams for the dual-entry-path routing and the refinement
  cycle flows would aid onboarding; added to `docs/deployment-guide.md`
  scope but not committed as a v1.0 requirement.

### Architecture Completeness Checklist

**Requirements Analysis**

- [x] Project context thoroughly analyzed (§Project Context Analysis)
- [x] Scale and complexity assessed — classified as high
- [x] Technical constraints identified (PRD + CLAUDE.md + standing memory)
- [x] Cross-cutting concerns mapped (11 concerns identified)

**Architectural Decisions**

- [x] Critical decisions documented with rationale
- [x] Technology stack fully specified with version pinning
- [x] Integration patterns defined (6 external surfaces)
- [x] Performance considerations addressed (NFR1-NFR5 enforcement)
- [x] Fork A (memory compaction) resolved with data-driven future threshold

**Implementation Patterns**

- [x] Language policy committed (pt-BR user text, English technical)
- [x] n8n workflow and node naming conventions established
- [x] Credential and env var naming locked as template contract
- [x] Database naming conventions committed
- [x] n8n expression conventions defined
- [x] Code node acceptance criteria enumerated
- [x] Message schema consistency rules frozen
- [x] Error classification taxonomy defined
- [x] Logging and audit conventions specified
- [x] SQL file conventions committed
- [x] Prompt file conventions with frontmatter schema
- [x] Refinement log conventions with cycle type enum
- [x] Audit bundle conventions with anonymization requirement
- [x] Git repo layout and commit conventions
- [x] Timestamp handling policy

**Project Structure**

- [x] Complete Git repository tree (no placeholders)
- [x] n8n workflow and sub-workflow inventory (12 entries with FR mapping,
      plus SW-13 added during validation)
- [x] PostgreSQL schema with full DDL for all tables
- [x] Redis keyspace with TTL policy
- [x] External integration contracts (6 surfaces)
- [x] Trust boundaries and cross-boundary payload rules
- [x] 70 FRs mapped to components
- [x] 47 NFRs mapped to enforcement mechanisms
- [x] Two concrete data flow walkthroughs (Journey 1 happy path, Cycle 4
      refinement loop)

**Validation**

- [x] Coherence validated
- [x] Requirements coverage validated with 2 patches applied
- [x] Implementation readiness validated with 3 patches applied
- [x] Gap analysis completed — no critical gaps, all important gaps
      addressed inline

### Architecture Readiness Assessment

**Overall Status:** READY FOR CONSTRUCTION

**Confidence Level:** high. The architecture is buildable as documented.
Every FR and NFR has a named architectural owner. No unresolved forks.
No deferred decisions that block early progression steps. The 5 gaps
found during validation were all patched inline rather than deferred.

**Key Strengths:**

- Four-surface structure (Git repo + n8n workflows + Postgres + Redis)
  matches the real shape of an n8n workflow project — no conventional
  software framing forced onto it.
- Zero-per-token frontier cost architecture — the Claude Code subscription
  model is committed as the refinement loop, not as an n8n sub-workflow.
- Template replicability is designed in from day one via externalized
  firm-specific data, stable credential names, idempotent SQL, and
  explicit dependency contracts.
- Append-only audit trail with fire-and-forget logging resolves the
  NFR10-vs-NFR17 tension cleanly.
- Hardened prompt sections give NFR13 (undefeatable AI disclosure + LGPD
  consent) architectural teeth rather than relying on convention.
- Single continuous progression matches the PRD's explicit anti-MVP stance.

**Areas for future enhancement (post-v1.0):**

- SW-10 Reconciliation (Redis → `chat_analytics` drain)
- SW-11 Retention (nightly retention policy execution)
- SW-12 Transcript Viewer (on-demand `/transcript_{session_id}` handler)
- SW-13 Persona Bot (Cycle 2 automation)
- Memory compaction (Fork A Option 2) when production data justifies it
- Append-only Postgres trigger hardening on `chat_analytics`
- Metabase dashboard if BorgStack-mini gains it
- RAG over legal knowledge base if a concrete use case emerges
- Cross-channel profile merging if operator requests it

### Implementation Handoff

**Construction guidelines for Galvani (primary implementer):**

- Follow PRD §21 Progression Sequence steps 2-34 as the construction order,
  using this architecture document as the structural guide.
- Before starting, execute the 14-step Bootstrap Checklist from
  §Starter Template & Foundation.
- For each progression step, build in `main-chatbot-refinement`, export a
  workflow snapshot at significant milestones, commit artifacts to Git.
- All decisions made in this document are binding. Deviation requires an
  architecture amendment and a refinement log entry.
- When a Code node feels necessary, re-check the four acceptance criteria
  before adding it. When justified, document in
  `refinement-log/code-node-exceptions.md`.

**First implementation priority:**

**Bootstrap Checklist (14 items)** from §Starter Template & Foundation.
When all 14 are green, PRD §21 progression steps 2-34 (workflow
construction) can begin.
