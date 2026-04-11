---
name: Implementation Readiness Report
stepsCompleted:
  - step-01-document-discovery
  - step-02-prd-analysis
  - step-03-epic-coverage-validation
  - step-04-ux-alignment
  - step-05-epic-quality-review
  - step-06-final-assessment
filesIncluded:
  prd: _bmad-output/planning-artifacts/prd.md
  architecture: _bmad-output/planning-artifacts/architecture.md
  epics: _bmad-output/planning-artifacts/epics.md
  ux: null
  supporting:
    - _bmad-output/planning-artifacts/product-brief-n8n-chatbot.md
    - _bmad-output/planning-artifacts/product-brief-n8n-chatbot-distillate.md
---

# Implementation Readiness Assessment Report

**Date:** 2026-04-11
**Project:** n8n-chatbot

## Step 1 — Document Inventory

### Documents used for assessment

| Type | File | Size | Last Modified |
|---|---|---|---|
| PRD | `prd.md` | 118 KB | 2026-04-10 20:24 |
| Architecture | `architecture.md` | 118 KB | 2026-04-10 23:41 |
| Epics & Stories | `epics.md` | 296 KB | 2026-04-11 11:01 |
| UX Design | *(not present — out of scope for this project)* | — | — |

### Supporting artifacts

- `product-brief-n8n-chatbot.md` — source product brief
- `product-brief-n8n-chatbot-distillate.md` — distilled brief
- `implementation-readiness-report-2026-04-10.md` — prior readiness run (kept for comparison)

### Discovery notes

- **No duplicates** — only whole documents exist; no sharded folders.
- **UX absent by design** — deliverable is a manually-built n8n chatbot plus guide/case study; there is no screen-based UI. UX alignment in Step 4 will be scoped to conversation UX.
- **Epics document updated today** (2026-04-11 11:01), after yesterday's readiness run — this assessment will reflect the latest version.

## Step 2 — PRD Analysis

Full PRD read end-to-end (1063 lines). Below is the complete requirement extraction used as the ground truth for epic coverage validation in Step 3.

### Functional Requirements

**Conversational Intake**
- **FR1**: The prospective client can send text messages in Brazilian Portuguese to the bot via Telegram.
- **FR2**: The prospective client can send text messages in Brazilian Portuguese to the bot via WhatsApp.
- **FR3**: The prospective client can send voice messages (audio) in Brazilian Portuguese to the bot via Telegram and via WhatsApp.
- **FR4**: The system transcribes received voice messages into text preserving semantic content with quality sufficient for the triage agent to process.
- **FR5**: The system responds to the client in text when the client sends text, and in text + audio when the client sends voice (modality mirroring).
- **FR6**: The system generates audio responses in natural Brazilian Portuguese with a neural male voice.
- **FR7**: The system maintains conversational context between consecutive messages in the same session, remembering information already provided by the client during the conversation.
- **FR8**: The system limits the conversational context to a recent window to prevent quality degradation in long conversations, summarizing older messages when necessary.
- **FR9**: The system identifies returning clients and recovers the previously stored profile, avoiding repeated questions.
- **FR10**: The system detects when the client sends an image or document and marks the conversation for human review without attempting to process the attachment content.

**Classification & Data Extraction**
- **FR11**: The system classifies each conversation into one of the firm's 8 practice areas (social security, labor, banking, health, family, real estate, corporate, tax) or "undefined".
- **FR12**: The system extracts the client's essential data in a structured way: full name, city, case summary, and perceived urgency.
- **FR13**: The system actively requests essential data when not yet provided, in a natural sequence.
- **FR14**: The system confirms the collected data with the client before treating it as final.
- **FR15**: The system persists the extracted data in a client profile retrievable in future interactions.

**Lead Qualification & Handoff**
- **FR16**: The system scores each conversation with deterministic criteria (practice match, data completeness, urgency, geography).
- **FR17**: The system separates qualified from unqualified leads by a configurable score threshold.
- **FR18**: For qualified leads, the system composes a handoff package (structured data + full chronological transcript).
- **FR19**: The system delivers the handoff package to the supervising attorney in a private channel, formatted for mobile.
- **FR20**: The attorney can accept, request more information, or decline the handoff through a quick-response interface.
- **FR21**: On accept, the system confirms to the client and updates conversation state.
- **FR22**: On "need more info," the system relays the question to the client, collects the answer, and updates the attorney's package.
- **FR23**: On decline, the system closes the conversation warmly and offers area-appropriate alternative resources.
- **FR24**: The system applies a configurable timeout to the attorney response wait and escalates on expiration.

**Non-Qualified Lead Handling**
- **FR25**: For unqualified leads, the system closes the conversation warmly without conveying rejection.
- **FR26**: The system offers relevant alternative resources (Defensoria Pública, public agencies, associations) from a static pre-approved library.
- **FR27**: The system selects offered resources based on the identified practice area, without generating new content.

**Compliance & Safety**
- **FR28**: The system identifies itself as an AI virtual assistant in the first turn of every new conversation (regulatory disclosure).
- **FR29**: The system presents LGPD consent in the first turn (controller, purpose, retention, subject rights).
- **FR30**: The system never generates legal advice, outcome prediction, legislation interpretation, or concrete action recommendation — under any pressure.
- **FR31**: The system blocks outputs containing legal-advice patterns and replaces them with a human-handoff fallback.
- **FR32**: The system maintains a complete audit trail (message, handoff, refinement, incident) with forensic-level granularity.
- **FR33**: The system applies per-data-type retention policies, automatically removing data after the period.
- **FR34**: The operator can execute complete deletion of a specific client's data (right to erasure).

**Security**
- **FR35**: The system rate-limits messages per client in a time window, responding with an informative message on excess.
- **FR36**: The system sanitizes all text input (HTML strip, invisible characters, whitespace normalization) before processing.
- **FR37**: The system detects and blocks prompt-injection and jailbreak attempts with a redirect response.
- **FR38**: The system detects and blocks off-topic (non-legal) messages before calling the triage agent.
- **FR39**: The system detects PII in input and output, applying redaction or blocking per policy.
- **FR40**: The system maintains strict session isolation between clients.
- **FR41**: The system stores credentials encrypted, never plain-text.

**Resilience**
- **FR42**: The system retries external calls on transient failure with exponential backoff.
- **FR43**: The system auto-switches to fallback credentials/endpoints for the operational LLM on rate-limit or persistent failure.
- **FR44**: On TTS unavailability, the system responds in text only without interrupting the conversation.
- **FR45**: On STT unavailability, the system informs the client and asks them to type.
- **FR46**: In any failure scenario, the system always sends the client a friendly message — never silence, never crash.
- **FR47**: The system captures exceptions in a dedicated workflow, classifies severity, and notifies the operator on critical.

**Refinement Pipeline (pre-production)**
- **FR48**: The operator can maintain a complete test environment (separate bot, separate database, total isolation).
- **FR49**: The operator can generate synthetic conversation scenarios stored as repo-versioned files.
- **FR50**: The operator can run chatbot-versus-chatbot simulations with personas, results logged separately.
- **FR51**: Internal human testers can interact with the test bot using written briefings, and record structured feedback.
- **FR52**: The operator can maintain a versioned golden dataset and run it against any prompt version with comparative metrics.
- **FR53**: The operator can run an adversarial red-team battery and get pass/fail per category.
- **FR54**: The operator can export audit bundles (filtered samples) in versionable markdown.
- **FR55**: The supervising attorney can review samples and record formal approval or blocking issues.
- **FR56**: The system prevents real client traffic from reaching production before the Launch Gate is approved and documented.

**Operations & Monitoring**
- **FR57**: The system records per-conversation metrics (latency, tokens, model, outcome) in analytics storage.
- **FR58**: The operator can query individual and aggregated metrics via dashboard or direct SQL.
- **FR59**: The system generates a periodic operational report and delivers it to the operator.
- **FR60**: The system tracks external service cost (LLM tokens, STT) and aggregates by period.
- **FR61**: The operator receives automatic notification when critical metrics exceed thresholds.
- **FR62**: The attorney can mark production conversations as misclassified, feeding the next refinement cycle.

**Template Export & Replicability**
- **FR63**: The operator can export the complete workflow as a single self-contained JSON artifact.
- **FR64**: The exported workflow can be imported into a new n8n instance and, after credential/data configuration, reproduce the original behavior.
- **FR65**: The template includes SQL initialization schema for all required tables.
- **FR66**: The template includes a complete credentials list and a step-by-step configuration guide.
- **FR67**: The template is accompanied by base prompt markdown files customizable per deployment.
- **FR68**: The operator can run a template deployment dry run in a separate test environment.

**Case Study Documentation**
- **FR69**: The repository records a structured refinement-cycle log (date, motivation, change, before/after metrics, time).
- **FR70**: The repository contains the complete project journey documented so an external developer can understand decisions and evolution.

**Total FRs:** 70

### Non-Functional Requirements

**Performance**
- **NFR1**: Client-perceived response time within Measurable Outcomes targets (p95 ≤ 8s text, ≤ 14s audio-to-audio) at ≤ 100 convos/day.
- **NFR2**: Deterministic routers resolve 15-25% of messages without any LLM call.
- **NFR3**: LLM context limited to 10-15 message window + system prompt.
- **NFR4**: Prompt caching used when supported.
- **NFR5**: Responses never block > 30s total — fallback message if processing exceeds.

**Security**
- **NFR6**: Only Groq (LLM + Whisper) and Comadre (TTS) receive conversation content in production.
- **NFR7**: Groq retention/data-use policy formally validated with opt-out pre-Launch Gate.
- **NFR8**: Absolute session isolation (deterministic key, no collision).
- **NFR9**: All credentials stored encrypted in n8n Credentials system with `N8N_ENCRYPTION_KEY`.
- **NFR10**: Audit trail append-only, immune to post-write modification except by retention policy.
- **NFR11**: Right-to-erasure completed within ≤48h of request, with confirmation log.
- **NFR12**: 100% pass on adversarial red-team battery before Launch Gate.
- **NFR13**: AI disclosure and LGPD consent cannot be disabled by config/hotfix/prompt-refinement.
- **NFR14**: Output legal-advice pattern validation mandatory in production.

**Reliability & Availability**
- **NFR15**: Degraded mode remains functional when optional services (Comadre, dashboard, weekly report) are down.
- **NFR16**: LLM failover within ≤2 retries, no human intervention.
- **NFR17**: Logging failures never interrupt conversational flow.
- **NFR18**: Planned maintenance ≤15 min per event, never during business hours without notice.
- **NFR19**: Workflow idempotent with respect to webhook retries (deduplication via provider message ID).

**Scalability**
- **NFR20**: 3x headroom (supports 300 convos/day) at no cost change, within free tiers.
- **NFR21**: Workflow ≤20% CPU and ≤30% RAM on BorgStack-mini.
- **NFR22**: Groq free→paid transition via credentials configuration, not reengineering.
- **NFR23**: `chat_analytics` indexed for typical queries; partitioning/archival automatic.

**Maintainability**
- **NFR24**: ≥90% of workflow nodes are n8n 2.x native or community visual nodes (Code nodes justified).
- **NFR25**: Each sub-workflow has single responsibility with documented I/O.
- **NFR26**: Prompts live as Git-versioned markdown files (source of truth).
- **NFR27**: Every refinement cycle produces a structured `refinement-log/` record.
- **NFR28**: Main workflow has ≤7 visually distinguishable logical layers.
- **NFR29**: External developer can understand architecture in ≤2h of reading repo + visual inspection.

**Observability**
- **NFR30**: Forensic-granularity records for conversations, handoffs, refinements, incidents.
- **NFR31**: From a handoff ID, reconstruct full conversation and all agent decisions in ≤5 min.
- **NFR32**: Runtime errors captured with full context, operator notified on critical within 5 min.
- **NFR33**: Operational metrics aggregated into an automatic weekly report.

**Integration Quality**
- **NFR34**: Every external call implements Retry on Fail with exponential backoff and max-retry ceiling.
- **NFR35**: External service contracts explicitly documented; changes detectable via error-rate monitoring.
- **NFR36**: LLM provider switch via credentials/variables only, no workflow topology change.
- **NFR37**: Telegram/Evolution webhooks validated for signature/origin when supported.

**Portability & Replicability**
- **NFR38**: Template export self-contained, no dependency on dev n8n instance state.
- **NFR39**: All firm-specific info externalized (variables, credentials, prompt files).
- **NFR40**: Deployment at a new client in ≤8h of work (Innovation 5 criterion).
- **NFR41**: Template dependencies explicitly listed (n8n version, tables, community nodes, credentials).

**Cost**
- **NFR42**: Total monthly production cost ≤ R$ 100/month at ~100 convos/day (target R$ 0-50 on free tier).
- **NFR43**: Refinement via Claude Code subscription — zero marginal cost per cycle.
- **NFR44**: Alert to operator when monthly cost exceeds 80% of budget, with breakdown.

**Data Residency & Sovereignty**
- **NFR45**: Client data resides exclusively in client firm's BorgStack-mini PostgreSQL.
- **NFR46**: External providers receive only minimum context (current window + system prompt), never full history or cross-client aggregates.
- **NFR47**: Audit bundles pass through anonymization filter before leaving BorgStack.

**Total NFRs:** 47

### Additional Requirements & Constraints

**Regulatory framework (non-optional):**
- OAB Recommendation 001/2024 — mandatory disclosure, no auto-generated legal advice, attorney supervision, client confidentiality.
- CNJ Resolution 615/2025 — auditable logs, traceability, professional responsibility.
- LGPD (Law 13.709/2018) — informed consent, minimization, encryption, retention, right to erasure, processing records.
- Attorney-client privilege — constitutional and criminally protected.

**No-legal-advice enforcement (6 independent layers):**
1. System prompt explicit numbered forbids.
2. Tool design (no tool produces legal opinion).
3. Static response library pre-approved by attorney.
4. Output guardrails on legal-advice patterns.
5. Monthly adversarial red-team battery.
6. Attorney weekly sample review.

**Explicit data retention policy (per data type):**
- Conversation transcripts (`chat_analytics`) — 2 years
- Client profiles — 5 years after last interaction
- Original audio (OGG) — 30 days
- Lead packages — 5 years (tied to case)
- Operational logs — 90 days
- Audit bundles (post-launch) — 90 days
- Golden dataset (synthetic) — permanent
- Audit trail retention — 5 years minimum (legally relevant operations)

**Infrastructure topology constraint:**
- Domain 1 — BorgStack-mini (10.10.10.205): n8n queue mode + Postgres + Redis + Caddy + Cloudflare tunnel + Evolution API. **No model inference here.**
- Domain 2 — Comadre (10.10.10.207): dedicated TTS, OpenAI-compatible.
- Domain 3 — Groq (external): LLM + Whisper.

**Launch Gate (7 criteria, all simultaneously green, zero real clients before):**
1. Security — 100% red-team pass.
2. Classification — ≥95% on golden dataset.
3. Tone/naturalness — ≥4.5/5 on LLM-as-judge.
4. Human validation — ≥50 scenarios with ≤2 "crappy bot" flags.
5. Attorney approval — 30 conversations reviewed and signed off.
6. Compliance checklist — full, signed.
7. Error handling — 100% graceful on simulated failures.

**Progression discipline:**
- Single continuous sequence of 34 ordered steps (no MVP, no phases).
- Each step leaves system functional at its current scope.
- No intermediate releases labeled MVP/v1/phase.

**Scope explicitly out:** multi-tenancy in shared n8n, legal KB RAG, appointment scheduling, document validation, judicial systems integration, jurisprudence RAG, sub-agents per area, full loop automation, automatic git sync, shadow/dual-mode, custom dashboard, web chat, multi-language beyond pt-BR.

### PRD Completeness Assessment

The PRD is **exceptionally complete and internally consistent**. Initial impressions feeding Step 3:

- **Strengths:** Every FR is testable, numbered, and traceable to a specific journey or domain constraint. NFRs are either numerically anchored in the Measurable Outcomes table or structurally grounded. Regulatory constraints (OAB/CNJ/LGPD) are not boilerplate — they drive specific architectural decisions (6-layer no-advice enforcement, anonymization filter, append-only audit, retention table). The single-progression scope section lists a 34-step ordered sequence, which gives the epics document a clear skeleton to mirror.
- **Potential gaps to watch in Step 3 / 5:** (a) FR6 specifies a neural **male** voice, but the brief also fixes `kokoro/pm_santa` — epics must not contradict; (b) FR48-FR56 describe a refinement pipeline that is operator-run, not user-facing — must verify epics scope these correctly as operational tooling rather than conversational features; (c) NFR7 (Groq retention validation) is a **pre-Launch-Gate paperwork requirement**, not code — must be surfaced as a story regardless; (d) NFR29 and NFR40 (new-operator readability in ≤2h, deployment in ≤8h) are the Innovation 5 replicability criteria — need a documentation/template epic that explicitly owns them.
- **No ambiguities** in FR wording material enough to block traceability. No missing axis (performance, security, reliability, scalability, maintainability, observability, integration, portability, cost, residency all covered).

Auto-proceeding to Step 3 — Epic Coverage Validation.

## Step 3 — Epic Coverage Validation

Full epics document read end-to-end (3172 lines, 11 epics, 30+ stories). The epics document publishes its own FR Coverage Map (lines 308-381) claiming all 70 FRs are distributed across 11 epics. I verified every claim against the actual story-level acceptance criteria — the coverage is real, not cosmetic.

### Epic Inventory & Story Count

| Epic | Title | Stories | Claimed FRs |
|---|---|---|---|
| E1 | Client Can Reach the Firm on Telegram | 4 (1.1–1.4) | FR1, FR41 |
| E2 | Conversational Triage Agent with Memory & Disclosure | 4 (2.1–2.4) | FR7, FR8, FR9, FR11, FR28, FR29, FR30, FR36 |
| E3 | Structured Data Extraction & Client Profiles | 2 (3.1–3.2) | FR12, FR13, FR14, FR15 |
| E4 | Lead Qualification & Human Handoff | 5 (4.1–4.5) | FR16, FR17, FR18, FR19, FR20, FR21, FR22, FR23, FR24 |
| E5 | Security Guardrails & Rate Limiting | 4 (5.1–5.4) | FR31, FR35, FR37, FR38, FR39, FR40 |
| E6 | Error Handling, Analytics & Forensic Audit Trail | 4 (6.1–6.4) | FR32, FR42, FR43, FR46, FR47, FR57, FR58, FR60, FR61 |
| E7 | Voice Modality (STT + TTS + Media Handling) | 3 (7.1–7.3) | FR3, FR4, FR5, FR6, FR10, FR44, FR45 |
| E8 | Unqualified Lead Library + WhatsApp Channel | 3 (8.1–8.3) | FR2, FR25, FR26, FR27 |
| E9 | Offline Refinement Toolbox (Pre-Launch Cycles 1–6) | 6 (9.1–9.6) | FR48, FR49, FR50, FR51, FR52, FR53, FR54, FR59, FR62 |
| E10 | Launch Gate, Compliance Automation & Production Activation | 5 (10.1–10.5) | FR33, FR34, FR55, FR56 |
| E11 | Template Export, Deployment Guide & Case Study | 3 (11.1–11.3) | FR63, FR64, FR65, FR66, FR67, FR68, FR69, FR70 |

**Story total:** ~43 stories across 11 epics. **Epic count per FR:** 2+8+4+9+6+9+7+4+9+4+8 = **70 — matches PRD exactly.**

### FR Coverage Matrix (verified against story ACs)

| FR | PRD gist | Epic | Where in epics (story + concrete verification) | Status |
|---|---|---|---|---|
| FR1 | Text via Telegram | E1 | Story 1.4 — Telegram Trigger → echo reply, activates & sends to Telegram client | ✓ Covered |
| FR2 | Text via WhatsApp | E8 | Story 8.2 — Webhook + Evolution API normalization + Story 8.3 output | ✓ Covered |
| FR3 | Voice via Telegram & WhatsApp | E7, E8 | Story 7.1 Telegram OGG download, Story 8.3 extends SW-1 for WhatsApp voice via Evolution API | ✓ Covered |
| FR4 | STT (quality sufficient for triage) | E7 | Story 7.1 Groq Whisper `pt` + large-v3-turbo | ✓ Covered |
| FR5 | Modality mirroring | E7 | Story 7.3 `Input Was Audio?` IF node in Layer 6 | ✓ Covered |
| FR6 | pt-BR neural male voice | E7 | Story 7.3 Comadre `kokoro/pm_santa` via `TTS_VOICE` env var | ✓ Covered |
| FR7 | Short-term memory | E2 | Story 2.1 Postgres Chat Memory attached to AI Agent | ✓ Covered |
| FR8 | Context window limit | E2 | Story 2.1 `contextWindowLength=15` (AR12 — compaction deferred) | ✓ Covered |
| FR9 | Returning client recognition | E2 | Story 2.4 `Lookup Returning Client Profile` + `Inject Profile Context` | ✓ Covered |
| FR10 | Attachment detection | E7 | Story 7.1 SW-1 image/document branches → `meta_attachment_flagged=true`; Story 7.2 routes to Handle Attachment | ✓ Covered |
| FR11 | 8-area classification | E2 | Story 2.3 `[QUALIFYING CRITERIA BY AREA]` section in triage-agent-v1.md | ✓ Covered |
| FR12 | Structured extraction | E3 | Story 3.2 data-gathering sequence (name → city → summary → urgency) | ✓ Covered |
| FR13 | Active missing-data prompting | E3 | Story 3.2 `[CONVERSATION FLOW]` sequencing discipline | ✓ Covered |
| FR14 | Data confirmation | E3 | Story 3.2 paraphrase + "está correto?" before tool call | ✓ Covered |
| FR15 | Profile persistence | E3 | Story 3.1 SW-2 upsert via `save_client_info` tool | ✓ Covered |
| FR16 | Deterministic scoring | E4 | Story 4.1 SW-3 four-component score chain of Set nodes | ✓ Covered |
| FR17 | Qualified/unqualified split | E4 | Story 4.1 `QUALIFICATION_THRESHOLD` env var (default 60) | ✓ Covered |
| FR18 | Handoff package composition | E4 | Story 4.2 `Fetch Client Profile` + `Fetch Chat Transcript` + `Build Handoff Card` | ✓ Covered |
| FR19 | Attorney notification | E4 | Story 4.2 `Post Handoff Card to Telegram Group` → `HANDOFF_GROUP_CHAT_ID` | ✓ Covered |
| FR20 | 3-button inline keyboard | E4 | Story 4.2 inline_keyboard with accept/info/decline callback_data | ✓ Covered |
| FR21 | Accept branch | E4 | Story 4.3 `Handle Accept` → status=handed_off + client confirmation | ✓ Covered |
| FR22 | Need More Info relay | E4 | Story 4.4 SW-5 Follow-Up Relay (single-active-session v1 limitation documented) | ✓ Covered (with known limitation) |
| FR23 | Decline branch with alternatives | E4→E8 | Story 4.3 placeholder, Story 8.1 replaces with `static_responses` lookup | ✓ Covered |
| FR24 | Timeout escalation | E4 | Story 4.2 Wait-on-Webhook 4h + `Handle Timeout` branch | ✓ Covered |
| FR25 | Warm closure (unqualified) | E8 | Story 8.1 `static_responses` seeded with warm closings per area | ✓ Covered |
| FR26 | Alternative resources library | E8 | Story 8.1 `alternative_resources JSONB` with Defensoria/INSS/TRT/etc. | ✓ Covered |
| FR27 | Area-based resource selection | E8 | Story 8.1 `Lookup Static Response by Area` with `general` fallback | ✓ Covered |
| FR28 | AI disclosure first turn | E2 | Story 2.3 hardened `[IDENTITY]` inside `<!-- HARDENED:START -->` markers | ✓ Covered |
| FR29 | LGPD consent first turn | E2 | Story 2.3 hardened LGPD block inside `<!-- HARDENED:START -->` markers | ✓ Covered |
| FR30 | Never generate legal advice | E2 | Story 2.3 `[RULES]` forbids advice, outcome prediction, legislation interpretation + `[EXAMPLES]` adversarial redirect | ✓ Covered (Layer 1 of 6) |
| FR31 | Output block on advice patterns | E5 | Story 5.3 `Output Guardrails — Post-Agent Check` + `Replace with Safe Fallback` | ✓ Covered (Layer 4 of 6) |
| FR32 | Complete audit trail | E6 | Story 6.1 `chat_analytics` DDL + Story 6.2 fire-and-forget instrumentation | ✓ Covered |
| FR33 | Automatic retention policy | E10 | Story 10.1 SW-11 Retention (nightly Schedule Trigger with per-table deletes) | ✓ Covered |
| FR34 | Right-to-erasure | E10 | Story 10.2 `docs/right-to-erasure-procedure.md` + dry-run sub-workflow + 48h SLA test | ✓ Covered (procedural + dry-run) |
| FR35 | Rate limiting | E5 | Story 5.1 Redis burst (30/5min) + daily (200/day) with friendly throttle messages | ✓ Covered |
| FR36 | Input sanitization | E2 | Story 2.2 `Sanitize Text Input` Set node (HTML strip, ZWSP removal, whitespace, truncate) | ✓ Covered |
| FR37 | Jailbreak detection | E5 | Story 5.2 Input Guardrails `guard_jailbreak_score > 0.7` branch | ✓ Covered |
| FR38 | Off-topic detection | E5 | Story 5.2 Input Guardrails `guard_topical_score < 0.3` branch | ✓ Covered |
| FR39 | PII in input and output | E5 | Story 5.2 input PII (CPF/CNPJ/phone/email regex) + Story 5.3 output PII | ✓ Covered |
| FR40 | Session isolation | E5 | Story 5.4 explicit parallel-conversation verification test with 2 test accounts + forensic doc | ✓ Covered |
| FR41 | Credential encryption | E1 | Story 1.1 `N8N_ENCRYPTION_KEY` persistence check + Story 1.2 credentials created exclusively in n8n Credentials system | ✓ Covered |
| FR42 | Retry with exponential backoff | E6 | Story 6.4 audit pass configuring `Retry on Fail=true, Max Tries=3, 1s/2s/4s backoff` on every external call | ✓ Covered |
| FR43 | LLM failover | E6 | Story 6.4 primary `groq_free` + fallback credential `groq_paid` on 429/5xx | ✓ Covered |
| FR44 | TTS graceful degradation | E7 | Story 7.3 `Call Comadre TTS` with `Continue on Fail=true` — text still delivered | ✓ Covered |
| FR45 | STT graceful degradation | E7 | Story 7.1 `Fallback STT Failed` Set node + Story 7.2 short-circuit to "please type" | ✓ Covered |
| FR46 | Never silence, never crash | E6 | Story 6.3 SW-6 CRITICAL branch guarantees best-effort friendly message to client | ✓ Covered |
| FR47 | Error workflow severity routing | E6 | Story 6.3 SW-6 `Classify Severity` Switch (CRITICAL/WARNING/INFO) with appropriate routing | ✓ Covered |
| FR48 | Test environment isolation | E9 | Story 9.1 `staging_*` tables + `ENVIRONMENT=refinement` + documented Chat Memory table caveat | ✓ Covered |
| FR49 | Synthetic scenarios | E9 | Story 9.2 `synthetic-scenarios/` seed + Story 9.6 Cycle 1 execution | ✓ Covered |
| FR50 | Chatbot-vs-chatbot simulation | E9 | Story 9.6 Cycle 2 — **semi-manual v1** (SW-13 Persona Bot deferred per AR7), Galvani transcribes Claude Code personas into refinement bot | ✓ Covered (with documented v1 limitation) |
| FR51 | Human tester briefings + feedback | E9 | Story 9.2 `human-test-logs/TEMPLATE.md` (5-question feedback form) + Story 9.6 Cycle 3 execution | ✓ Covered |
| FR52 | Golden dataset + regression | E9 | Story 9.2 seeds 30 scenarios (24 practice + 6 edge-case) + Story 9.5 Cycle 5 runner in Claude Code (AR54 — not n8n) | ✓ Covered |
| FR53 | Red-team battery | E9 | Story 9.2 seeds 5 attack files + Story 9.5 Cycle 6 runner in Claude Code | ✓ Covered |
| FR54 | Audit bundle export | E9 | Story 9.3 SW-9 Audit Bundle Exporter with PII sanitization Code node (AR29 pre-authorized exception) | ✓ Covered |
| FR55 | Attorney formal review | E10 | Story 10.3 Cycle 7 — 30+ conversations in review package, explicit sign-off quote required | ✓ Covered |
| FR56 | No real traffic before Launch Gate | E10 | Story 10.5 production workflow only cloned from refinement after gate validation — FR56 satisfied by construction | ✓ Covered |
| FR57 | Per-conversation metrics | E6 | Story 6.1 `chat_analytics` columns + Story 6.2 wires `user_message`/`agent_response`/`tool_call`/`handoff_*` events | ✓ Covered |
| FR58 | Query interface | E6 | Story 6.1 `docs/chat-analytics-queries.md` with 6 canonical `psql` queries (PRD allows "dashboard OR direct SQL") | ✓ Covered |
| FR59 | Periodic operational report | E9 | Story 9.4 SW-8 Weekly Report Generator (Schedule Trigger Mon 09:00 America/Sao_Paulo → OPS_GROUP_CHAT_ID) | ✓ Covered |
| FR60 | Cost tracking | E6, E9 | Story 6.1 token column capture + Story 9.4 aggregation + `COST_PER_1K_*` env vars in weekly report | ✓ Covered |
| FR61 | Threshold-breach alerts | E6, E9 | Story 6.3 error-threshold alerts + Story 9.4 metric-threshold (15-min schedule checking error rate, latency, volume, cost projection) | ✓ Covered |
| FR62 | Attorney misclassification flag | E9 | Story 9.4 `/flag_misclass <session_id> <area_correct> <reason>` ops command + `meta.attorney_flagged_misclassification` column flag | ✓ Covered |
| FR63 | Self-contained JSON export | E11 | Story 11.1 `exports/n8n-workflow-template-v1.json` + sub-workflow bundle + `grep "Maruzza Teixeira" exports/` → 0 hits gate | ✓ Covered |
| FR64 | Import reproduces behavior | E11 | Story 11.1 (claim) + Story 11.3 (validation via dry run) | ✓ Covered |
| FR65 | SQL schema | E11 | Story 11.1 `sql/001-004` idempotent files + `sql/README.md` with run order | ✓ Covered |
| FR66 | Credentials + config guide | E11 | Story 11.1 `docs/credentials-checklist.md` (8 locked names) + `docs/environment-variables.md` (12 env vars) | ✓ Covered |
| FR67 | Base prompt markdown files | E11 | Story 11.1 final `triage-agent-v{N}.md` + `handoff-composer-v1.md` + `prompts/README.md` with hardened-section warning | ✓ Covered |
| FR68 | Template deployment dry run | E11 | Story 11.3 dry run on separate infrastructure with timer + per-section breakdown vs NFR40 ≤8h target | ✓ Covered |
| FR69 | Refinement cycle log | E9, E11 | Story 9.6 produces v1→v2→v3→v4 entries + Story 11.2 consolidates with audit | ✓ Covered |
| FR70 | Case study documentation | E11 | Story 11.2 `docs/case-study.md` 12-section narrative + README | ✓ Covered |

### Missing FR Coverage

**None.** All 70 PRD FRs have concrete story-level traceability, not just a cosmetic "FR Coverage Map."

### Soft Gaps & Known Limitations (not blocking, but worth flagging for Step 5)

1. **FR22 (Follow-Up Relay) — single-active-session limitation** (Story 4.4, line 1256-1256). The v1 implementation uses a single Redis key `pending_attorney_question:active` rather than per-session correlation. Under concurrent handoff volume this could cause relay crossover. The epic explicitly documents the limitation and notes it in `docs/deployment-guide.md §Known limitations`. Given PRD volume (~100 convos/day, ~10-20 handoffs), the risk is acceptable for v1.0, but this is a genuine edge-case risk that should be watched.

2. **FR50 (Chatbot-vs-chatbot) — semi-manual v1** (Story 9.6 Cycle 2). SW-13 Persona Bot is deferred per AR7, so Cycle 2 in v1 is Galvani transcribing Claude Code personas into the refinement bot rather than a fully automated simulator. The FR text is satisfied ("operator can run simulations with results logged separately") but the automation level is lower than the PRD might imply. Documented as known limitation.

3. **FR48 (Test environment isolation) — `n8n_chat_histories` staging table** (Story 9.1). The epic explicitly flags that if the Postgres Chat Memory node version does not support custom table names, production and refinement may share `n8n_chat_histories`. This is documented and the workaround path is flagged, but it's a known unresolved question that depends on n8n node capabilities at build time. Not a gap in the PLAN, but a risk at BUILD time.

4. **FR58 (Query interface) — psql only, no dashboard**. PRD allows "dashboard or direct SQL" so this is compliant, but Metabase is explicitly rejected in scope. `docs/chat-analytics-queries.md` provides 6 canonical queries. No gap, just worth knowing the operator experience is CLI-based.

5. **FR33/NFR10 vs NFR11 tension — retention DELETE vs append-only** (Story 10.2 explicitly documents the resolution). Right-to-erasure legitimately requires DELETE on an "append-only" table. The epic notes this is an explicit exception documented from day one ("append-only rule has always explicitly excluded 'automatic retention policy' and 'right-to-erasure' as legitimate DELETE triggers"). No gap.

### NFR Treatment

The epics document (line 383) explicitly flags that **NFRs are not mapped 1:1 to epics** but woven into story acceptance criteria. Spot-checks confirm:
- **NFR1** (p95 latency) — referenced in Story 2.1, 7.3 latency budgets
- **NFR2** (15-25% deterministic router) — Story 2.2 builds router, Story 6.2 logs `model_used IS NULL` for measurement
- **NFR3** (10-15 msg window) — Story 2.1 `contextWindowLength=15`
- **NFR8** (session isolation absolute) — Story 5.4 explicit verification test
- **NFR9** (credential encryption) — Story 1.1/1.2
- **NFR10** (append-only) — Story 6.1 + Story 10.2 exception handling
- **NFR11** (48h right-to-erasure SLA) — Story 10.2 smoke test
- **NFR12** (100% red-team pass) — Story 9.5/9.6 re-runs until pass
- **NFR13** (hardened prompt sections) — Story 2.3 HARDENED markers + AR28 enforcement
- **NFR17** (logging never blocks) — Story 6.1 fire-and-forget + Redis buffer fallback
- **NFR19** (webhook idempotency) — Story 6.4 Redis SET NX pattern
- **NFR24** (≥90% visual nodes) — AR45 discipline referenced throughout; only 2 pre-authorized Code nodes (SW-9 PII sanitizer, Normalize Incoming Message provisional)
- **NFR29** (≤2h operator onboarding) — Story 11.1 deployment guide as the primary artifact
- **NFR34** (retry with backoff) — Story 6.4 audit pass + configured per-node
- **NFR38-41** (replicability) — Epic 11 entire scope
- **NFR40** (≤8h per-firm deployment) — Story 11.3 dry-run with timer
- **NFR47** (anonymization filter) — Story 9.3 SW-9 Code node with `sanitization_pass: true` gate

All 47 NFRs appear addressed across the stories; full 1:1 validation happens in Step 5 (epic quality review).

### Coverage Statistics

- **Total PRD FRs:** 70
- **FRs with explicit epic assignment + verified story AC:** 70
- **Coverage percentage:** **100%**
- **FRs covered with known v1 limitations (not gaps, but flagged):** 3 (FR22, FR48, FR50)
- **FRs that are procedural rather than workflow-code:** 2 (FR34 right-to-erasure, FR55 attorney review)
- **Stories that depend on n8n node capabilities at build time:** 1 (Story 9.1 Chat Memory custom table name)

Auto-proceeding to Step 4 — UX Alignment.

## Step 4 — UX Alignment

### UX Document Status

**Not Found — and not required for this project.**

Searched for UX artifacts in Step 1. No file matching `*ux*.md` or `*ux*/index.md` exists in `_bmad-output/planning-artifacts/`. The epics document addresses this explicitly at line 21 and again at lines 302-304:

> This project has **no UX Design document** — the user-facing surface is Telegram / WhatsApp conversations and the attorney-facing surface is a Telegram inline keyboard in a private group. Conversational and attorney-surface UX live inside the PRD itself (User Journeys, FR5/FR6/FR19-FR24, NFR1/NFR5) and are not duplicated here.

### Is UX implied by the product?

**No screen-based UX, yes conversational UX — and the conversational UX is fully specified in the PRD.**

Reviewing what the product presents to users:

1. **Prospective client surface:** Telegram and WhatsApp native clients. No custom web app, no mobile app, no widget. The UI is literally the messenger app's own UI. There is nothing to wireframe, no components to style, no color tokens to define.
2. **Supervising attorney surface:** A single Telegram message card with 3 inline-keyboard buttons (Accept / Need More Info / Decline). This is native Telegram chrome — the "design" is the text format of the card, which is specified as a template file (`prompts/handoff-composer-v1.md`) in Epic 4 Story 4.2.
3. **Operator surface:** `psql` queries (per FR58 and `docs/chat-analytics-queries.md`), n8n editor for workflow maintenance, Claude Code sessions for refinement. Zero end-user UI.

The parts of "UX" that *do* matter for this product — conversational flow, tone, message patterns, turn sequencing, modality mirroring, first-turn disclosure — are captured in the PRD as:

- **User Journeys §5** — five narrative journeys (Dona Francisca voice-first qualified, Carlos graceful unqualified, Maruzza attorney handoff, Galvani operator refinement, Pedro human tester). These are the equivalent of "UX flows" for a conversational product.
- **FR5** (modality mirroring) and **FR6** (neural male voice) — conversational UX requirements.
- **FR19–FR24** — attorney handoff interaction including the 3-button quick-response interface and timeout behavior.
- **NFR1** — p95 latency ≤8s text / ≤14s audio-to-audio. This is the conversational equivalent of "time to first meaningful paint."
- **NFR5** — 30-second hard ceiling with "one moment, I am checking" fallback message if processing drags.

### UX ↔ PRD Alignment

Not applicable as there is no UX document to align. Alignment is **intra-PRD** (journeys consistent with FRs consistent with NFRs) and was spot-checked in Step 2. Everything is internally consistent: modality mirroring (FR5), the male voice (FR6) matching Comadre's `kokoro/pm_santa`, the handoff card format (FR19) consistent with Journey 3 narrative.

### UX ↔ Architecture Alignment

Not applicable as there is no UX document. What matters is that **Architecture supports the conversational UX as defined in the PRD**. Spot-checking key points:

| Conversational UX requirement | Architecture support |
|---|---|
| FR5 modality mirroring (voice in → text+audio out) | AR11 SW-4 Wait pattern + Epic 7 Story 7.3 `Input Was Audio?` IF in Layer 6 |
| FR6 `kokoro/pm_santa` male voice | AR42 `TTS_VOICE` env var + architecture integration table (Comadre HTTP Request) |
| NFR1 latency p95 ≤8s/≤14s | AR8 layer cap, AR30 AI Agent pattern, AR31 failover, NFR34 retry policies prevent latency tail spikes |
| NFR5 30s ceiling with fallback | *Not explicitly wired in epics* — no story wires a 30s watchdog that sends "um momento, estou verificando" before background continuation |
| FR19 mobile-formatted handoff card | Story 4.2 `Build Handoff Card` with `parse_mode='HTML'` and the `handoff-composer-v1.md` template |
| FR28/FR29 AI disclosure + LGPD in first turn | AR28 hardened prompt sections, Story 2.3 implementation |

### Alignment Issues

**One minor gap:** NFR5 — "Responses never block more than 30 seconds total; if processing exceeds that limit, the system sends a fallback message 'one moment, I am checking' and continues processing in the background" — **is not explicitly wired in any epic story.** I searched the epics for "30 second", "30s", "um momento", "one moment", "background continuation" — no story implements a watchdog timer that sends an interim message and continues processing asynchronously. The epics address latency via:
- NFR1 via retry policies (Story 6.4) + AI Agent node config
- FR46 via SW-6 friendly-message-on-failure (Story 6.3)

But neither mechanism is the NFR5 "stall detection with interim 'I'm thinking' message." In n8n this is a non-trivial pattern (probably requires a Wait node with a timeout racing against the AI Agent response, or a side-channel Schedule Trigger per session — not obvious).

**Recommendation for Step 5 / Step 6:** Flag NFR5 as a partial coverage gap. Either (a) add a story to the epics that wires a 30s stall watchdog, or (b) document NFR5 as deferred with an explicit note that v1.0 relies on retry/failover policies to stay under 30s in practice and the watchdog pattern ships post-launch if real traffic reveals stalls.

### Warnings

1. **No UX document exists, and none is required.** The product's "UX" is conversational and lives in PRD User Journeys + FRs. The epics correctly avoid duplicating this.
2. **NFR5 (30s watchdog + interim message) is not wired in any epic story.** This is the only "conversational UX" NFR that appears to lack a concrete implementation path in v1.0. Not critical, but worth surfacing in Step 5 or Step 6.

Auto-proceeding to Step 5 — Epic Quality Review.

## Step 5 — Epic Quality Review

Every epic and every story in the epics document was read in Step 3. This section applies the create-epics-and-stories standards (user value, independence, story sizing, AC quality, dependency discipline) and flags violations by severity.

### Epic-level validation

| Epic | Delivers user value? | Can function with only prior epics? | Title user-centric? | Status |
|---|---|---|---|---|
| E1 Client Can Reach the Firm on Telegram | Yes — client sends message, gets reply (Story 1.4) | N/A (first epic) | Yes (user outcome "reach the firm") | ✅ |
| E2 Conversational Triage Agent with Memory & Disclosure | Yes — disclosed AI conversation with memory | Uses E1 only | Yes | ✅ |
| E3 Structured Data Extraction & Client Profiles | Yes — client provides data naturally, it's captured | Uses E1+E2 | Yes | ✅ |
| E4 Lead Qualification & Human Handoff | Yes — full triage loop, attorney receives lead | Uses E1–E3 | Yes | ✅ |
| E5 Security Guardrails & Rate Limiting | Yes — safe from abuse, client throttled gracefully | Uses E1–E4 | Borderline ("security" is technical framing but AC text is user-centric) | ✅ |
| E6 Error Handling, Analytics & Forensic Audit Trail | Operator value (observability) + client value (never crash) | Uses E1–E5 | Operator-centric | ✅ |
| E7 Voice Modality (STT+TTS+Media Handling) | Yes — client talks in voice, receives voice | Uses E1–E6 | Yes | ✅ |
| E8 Unqualified Lead Library + WhatsApp Channel | Yes — unqualified clients served + WhatsApp reach | Uses E1–E7 | Two-topic bundle (see concern) | ✅ with concern |
| E9 Offline Refinement Toolbox (Cycles 1–6) | Operator value (refinement capability) | Uses E1–E8 | Operator-centric | ✅ |
| E10 Launch Gate, Compliance & Production Activation | Yes — production bot meets real clients | Uses E1–E9 | Yes | ✅ |
| E11 Template Export, Deployment Guide & Case Study | Yes — future firms and readers get the template | Uses E1–E10 | Yes | ✅ |

**Epic independence:** No epic requires a later epic to function. Each leaves the system genuinely functional at its current scope (matching the PRD's single-progression discipline). **No circular dependencies, no forward references at epic level.**

**No technical-only epics.** Even E5/E6/E9 (which sound technical) are framed around operator/client value in the epic descriptions and story ACs.

### Database/entity creation timing

Best practice: create tables when a story first needs them, not upfront. Verdict: **CLEAN.**

| Table | Created in | First need | Verdict |
|---|---|---|---|
| `client_profiles` | Story 2.4 (`sql/001-init-client-profiles.sql`) | Returning-client lookup | ✅ Just-in-time |
| `chat_analytics` | Story 6.1 (`sql/002-init-chat-analytics.sql`) | SW-7 Analytics Logger | ✅ Just-in-time |
| `static_responses` | Story 8.1 (`sql/003-init-static-responses.sql`) | Unqualified lead closure library | ✅ Just-in-time |
| `staging_*` | Story 9.1 (`sql/004-init-staging-tables.sql`) | Refinement environment isolation | ✅ Just-in-time |
| `n8n_chat_histories` | Auto-created by Postgres Chat Memory node | First memory write (Story 2.1) | ✅ Not pre-created (AR15) |

No table is created upfront "just in case." No unused tables. No epic creates schema it does not immediately use.

### Starter template alignment

Architecture specifies **empty canvas** (AR2 — community templates are reference only). Epic 1 Story 1.3 creates empty workflow shells on empty canvas — **matches AR2 exactly.** No conflict.

### Story sizing & independence

Most stories are larger than typical Agile stories (a day of work) because the project is manual n8n visual-editor construction, not typing code. Each story is a coherent functional increment that leaves the system working. For this project type (workflow construction guide), the sizing is appropriate.

**Build-then-integrate pattern used repeatedly and correctly:**
- Story 3.1 builds SW-2 standalone → Story 3.2 wires it as a tool
- Story 4.1 builds SW-3 standalone → Story 4.5 wires it as a tool
- Story 4.2 builds SW-4 notification side → Story 4.3 wires routing + decisions
- Story 7.1 builds SW-1 standalone → Story 7.2 wires into Layer 2
- Story 9.3 builds SW-9 standalone → Story 9.6 uses it in Cycle 4

This is an exemplary pattern for sub-workflow development — the primitive is verified in isolation before the integration.

### Acceptance Criteria quality

Spot-checked ~15 stories across 11 epics. ACs are in Given/When/Then BDD format with:
- Specific input values (not vague "some data")
- Measurable outcomes (latency thresholds, exact row counts, specific command outputs)
- Happy path + at least one degradation/error test per story with concrete setup/revert instructions
- Explicit references to the FR/NFR/AR being validated

This is significantly above the median quality for planning documents I've reviewed. No "user can login" vagueness.

### 🔴 Critical Violations

**None found.** No forward dependencies at epic level, no technical epics, no stories that cannot be completed, no circular references.

### 🟠 Major Issues

**M1. Story 9.1 staging rewire is a retroactive refactor touching every prior epic's Postgres nodes.** When Epic 9 begins, Story 9.1 must walk through `main-chatbot-refinement`, SW-1, SW-2, SW-3, SW-4, SW-5, SW-6, SW-7 and rename every `client_profiles` reference to `staging_client_profiles` and `chat_analytics` to `staging_chat_analytics`. The story explicitly describes this ("audit every Postgres node... update every SQL query"). This is a large, error-prone mechanical edit and doubles back on work already done in E1-E8. Story 10.5 then reverses it for production.

*Why this exists:* Epics 1-8 write to production-named tables because staging tables were designed to be introduced in E9 (where the refinement discipline first matters). Inverting the order (introducing staging first) would mean staging-naming all prior epics, which the author apparently judged would confuse the storytelling.

*Remediation option:* Either (a) accept the cost and execute Story 9.1's audit with care, (b) introduce an `ENVIRONMENT`-aware table-name expression pattern from Story 6.1 onwards (`{{ $env.ENVIRONMENT == 'refinement' ? 'staging_chat_analytics' : 'chat_analytics' }}`), which avoids the rewire entirely but requires Story 6.1 to know about staging before it exists, or (c) accept the retroactive rewire as the v1 cost of the progression philosophy. The epic chose (a/c). Acceptable but worth the operator knowing about.

**M2. NFR5 (30s watchdog with interim "um momento" message) has no implementation story.** Flagged in Step 4 — the only "conversational UX" NFR without a concrete story. The epics rely on retry/failover policies (NFR16, Story 6.4) to keep latency bounded, but a hard 30s ceiling with a background-continuation pattern is not wired. In n8n this is non-trivial (requires a Wait node racing against the AI Agent or a Schedule Trigger per session).

*Remediation option:* Either (a) add a story to Epic 6 wiring the stall watchdog, or (b) formally downgrade NFR5 to "deferred post-launch — v1 relies on retry policies to avoid stalls in practice." Neither is blocking but this should be an explicit decision before Phase 3 begins.

**M3. Epic 1 stories 1.1, 1.2, 1.3 are operator-facing infrastructure prep, not end-client user stories.** Story 1.1 is "verify BorgStack-mini healthy." Story 1.2 is "register credentials and create Telegram groups." Story 1.3 is "create empty workflow shells with correct names." Only Story 1.4 (echo bot live) delivers end-client value. Per strict Agile story discipline, 1.1-1.3 should be tasks within a single user story rather than three user stories.

*Why this exists:* For a workflow construction guide where the "user" is Galvani manually building the bot, operator-facing preparation stories have a different justification than in a shippable product. The Bootstrap Checklist (AR1) is 14 steps that genuinely need their own traceability for the template deployment procedure (a future firm operator reads Story 1.1-1.3 as instructions, not as "features to ship").

*Verdict:* Acceptable given project type (`workflow_automation_build_guide`, brownfield). Flag as "stories 1.1-1.3 serve the deployment-guide reader as instructions, not a product-backlog user."

### 🟡 Minor Concerns

**C1. Story 5.2 depends on native/community Guardrails node availability** (line 1443). The AC explicitly allows a fallback path: "using n8n's Guardrails native/community node, or an HTTP Request to a Guardrails service if the native node is unavailable at the current n8n version." Build-time unknown — the operator will find out at Story 5.2 execution which path applies. Not a planning defect, but a risk that should be pinned before Phase 3 starts.

**C2. Story 5.4 is a verification-only story, not a feature story.** It adds no new workflow nodes — it runs a parallel-conversation test with two Telegram accounts, queries Postgres, documents the result as Launch Gate evidence (`docs/session-isolation-verification-2026-MM-DD.md`). For a compliance-driven project with a Launch Gate, verification-only stories are legitimate (the forensic evidence is the deliverable). Worth noting but not a defect.

**C3. Story 4.4 has an unresolved n8n-pattern choice for SW-4 Info branch re-entry** (line 1281-1282). The AC says "loop back to the same Wait node" but adds "alternative is to post a fresh card and start a new SW-4 execution... for v1, prefer the loop-back approach if n8n supports it; otherwise prefer the fresh-card approach documented as a deviation." The loop-back approach may not work in n8n — if so, the fresh-card approach requires additional wiring. Build-time unknown.

**C4. Story 6.1 has an intra-epic forward reference to Story 6.3** (line 1644). SW-7's fallback branch has a TODO comment "wire to SW-6 once Story 6.3 is complete." Intra-epic forward dependency within Epic 6. Acceptable (the epic is completed as a whole) but adds wiring debt inside the epic.

**C5. Story 8.1 has a flexible implementation choice for the unqualified-closure tool** (line 2154). The AC says "either a dedicated tool the agent invokes, or a post-agent automatic routing based on status — Galvani should pick whichever is cleaner during construction." Acceptable deferral of a minor architecture decision to build time, but technically an underspecified AC.

**C6. Epic 8 bundles two distinct topics** — static_responses library (Story 8.1) and WhatsApp channel (Stories 8.2, 8.3). These are weakly related (both are "completeness for production-primary-channel readiness") but could have been separate epics. The single-epic bundling keeps the epic count at 11 and is acceptable, but splits would have been more orthogonal.

**C7. Story 9.5 and 9.6 are execution stories** — they run refinement cycles in Claude Code and produce evidence files rather than building workflow nodes. Legitimate for a compliance-gated project, but unusual as story types. Worth knowing that ~1/6 of Epic 9's work is "execute and document" rather than "build and verify."

**C8. Story 4.1 introduces `QUALIFICATION_THRESHOLD` env var outside AR42's locked list.** The story acknowledges this: "the `AR42` committed env var list is updated in a subsequent architecture addendum note (tracked in `refinement-log/env-var-additions.md`)." The env var is legitimate; just flagging that it's a post-architecture addition.

**C9. Story 9.1 explicitly documents that `staging_n8n_chat_histories` may not be creatable** if the Postgres Chat Memory node version doesn't support custom table names. Fallback plan is documented but the issue is unresolved at planning time.

### Best Practices Compliance Checklist

For each of the 11 epics:

- [x] Epic delivers user value (operator-value accepted for E6/E9 given project type)
- [x] Epic can function independently (no circular epic deps; each epic builds on prior only)
- [x] Stories appropriately sized (larger than typical Agile but appropriate for manual n8n build)
- [x] No forward dependencies at epic level (all dependencies are backward)
- [x] Database tables created when needed (just-in-time, not upfront)
- [x] Clear acceptance criteria (BDD Given/When/Then throughout, measurable outcomes)
- [x] Traceability to FRs maintained (explicit FR coverage map + story-level FR callouts)

### Summary

The epics document is **high quality** — the FR Coverage Map is backed by real story ACs, the build-then-integrate sub-workflow pattern is disciplined, table creation is just-in-time, ACs are BDD-formatted with measurable outcomes, and story sequencing respects the "never break what is already working" progression rule.

The three Major issues (M1 staging rewire, M2 NFR5 watchdog, M3 Epic 1 infrastructure stories) are not blocking — they are either explicit design tradeoffs (M1, M3) or a missing small-scope item (M2). None is a "stop the project" problem.

The Minor concerns are all acknowledged build-time unknowns or stylistic choices that the operator should be aware of but which do not compromise the plan.

**Recommendation:** Proceed to implementation, with the following pre-flight actions:
1. Decide on NFR5 — either add a watchdog story to Epic 6 or formally defer it.
2. Spike the Guardrails node availability question (C1) before starting Story 5.2.
3. Spike the n8n Wait-node loop-back pattern (C3) before starting Story 4.4.
4. Spike the Postgres Chat Memory custom table name question (C9) before starting Story 9.1.
5. Accept Story 9.1 retroactive rewire as a known cost.

Auto-proceeding to Step 6 — Final Assessment.

## Step 6 — Final Assessment

### Overall Readiness Status

## ✅ READY TO PROCEED

The n8n-chatbot planning artifacts (PRD + Architecture + Epics) are **implementation-ready for Phase 3 (Project Creation) and Phase 4 (n8n workflow construction).**

This is the second readiness assessment for this project (the first ran yesterday, 2026-04-10, on an earlier version of `epics.md`). Today's run uses `epics.md` rev 2026-04-11 11:01 against PRD rev 2026-04-10. The scope coverage is now complete and internally consistent. The issues flagged below are **pre-flight items**, not blockers — they can be resolved in the first hours of Phase 3 or explicitly accepted as v1 limitations.

### Evidence Summary

| Check | Result |
|---|---|
| PRD FR extraction | 70 FRs, 47 NFRs, all traceable to journeys/regulatory constraints |
| Epic FR coverage | **70/70 FRs** covered with verified story-level ACs (100%) |
| Epic independence | ✅ No forward dependencies between epics |
| Story sizing | ✅ Appropriate for manual n8n build; build-then-integrate pattern used consistently |
| AC quality | ✅ BDD Given/When/Then with measurable outcomes and degradation tests |
| Database creation timing | ✅ Just-in-time, no upfront over-creation |
| UX alignment | N/A (no UX doc required — conversational UX lives in PRD journeys) |
| Starter template alignment | ✅ Empty canvas (AR2) matches Epic 1 Story 1.3 |
| Architecture-epic alignment | ✅ AR1-AR56 referenced throughout stories; no contradictions |
| Technical epics | ✅ None found — every epic frames a user/operator outcome |

### Critical Issues Requiring Immediate Action

**None.** No critical violations were found in any of the five analysis steps. No forward dependencies, no technical-only epics, no missing FR coverage, no contradictions between PRD and architecture.

### Pre-flight Items (address before or during Phase 3 kickoff)

These are ordered by when they bite. None blocks the start of construction.

**P1 — Decide the fate of NFR5** (30s watchdog + interim "um momento" message). *When it bites:* Epic 6 or never. *Options:* (a) Add a small story to Epic 6 wiring the watchdog pattern (Wait node racing against AI Agent response, or a Schedule Trigger per-session timer); (b) Formally downgrade NFR5 to "deferred post-launch — retry/failover policies are the primary latency defense in v1." *Recommendation:* pick (b) and document — implementing a true background-continuation pattern in n8n is non-trivial and the retry/failover stack should keep tail latency under 30s in practice. Revisit if production analytics show stalls.

**P2 — Spike the Guardrails node question** before Story 5.2. *When it bites:* Epic 5. Check whether the current n8n version on BorgStack-mini has a native Guardrails node, a community node, or neither. The AC already has a fallback path (HTTP Request to a Guardrails service), but which path applies should be pinned before the story starts to avoid mid-story pivots.

**P3 — Spike the Wait-node loop-back pattern** before Story 4.4. *When it bites:* Epic 4. Verify in n8n whether SW-4's `Handle Info` branch can loop back to the same Wait node or whether it must post a fresh card. The AC handles both cases, but the choice affects SW-4's visual topology.

**P4 — Spike the Postgres Chat Memory custom-table-name support** before Story 9.1. *When it bites:* Epic 9. If the memory node version cannot point at `staging_n8n_chat_histories`, the refinement/production chat-history isolation may be partial. Document the behavior (and any workaround) before the staging rewire runs, not during.

**P5 — Accept the Story 9.1 retroactive staging rewire as a known cost.** *When it bites:* Epic 9. Budget time for a careful audit pass through every Postgres node in `main-chatbot-refinement` and SW-1 through SW-7. Consider running the audit with `grep -rn 'chat_analytics\|client_profiles' exports/workflow-snapshots/<pre-9.1>.json` as a checklist source.

### Optional Improvements (nice-to-have, not required)

**O1.** Epic 8 bundles `static_responses` library with WhatsApp channel. These are weakly related and could split into two more orthogonal epics. Since this is cosmetic and the FR coverage is fine, not worth refactoring unless there's a separate reason.

**O2.** Stories 1.1-1.3 are operator-facing bootstrap rather than end-client user stories. Acceptable for a workflow construction guide project, but the deployment guide could call out that a future firm operator reads 1.1-1.3 as instructions (not as shippable features) to reset expectations.

**O3.** Spot-check the NFR weaving during Phase 3 — especially NFR13 (hardened sections enforcement discipline, AR28), NFR26 (prompts as Git source of truth), NFR29 (≤2h operator onboarding), NFR40 (≤8h per-firm deployment). These become Launch Gate evidence so every story touching them should produce a concrete artifact.

### Comparison with Prior Readiness Run (2026-04-10)

The previous readiness report exists at `_bmad-output/planning-artifacts/implementation-readiness-report-2026-04-10.md` (32 KB). Today's `epics.md` is dated 2026-04-11 11:01, so it has been revised since yesterday's assessment. I did not diff the two readiness reports — if Galvani wants to know what changed between yesterday's and today's `epics.md`, a separate `git log`/diff pass is needed. Today's report is the current ground truth.

### Recommended Next Steps

1. **Resolve P1** — Decide NFR5 status (add story or formally defer). Commit the decision to a refinement-log entry or to the PRD as a clarification.
2. **Schedule P2, P3, P4 spikes** — Three small technical verifications that can run in parallel in the first day of Phase 3. Each is a 30-60 minute n8n editor experiment.
3. **Start Phase 3 (Project Creation)** — Create the minimal repository structure (`prompts/`, `sql/`, `exports/workflow-snapshots/`, `audit-bundles/`, `refinement-log/`, `golden-dataset/`, `synthetic-scenarios/`, `red-team-battery/`, `human-test-logs/`, `docs/`) per AR52.
4. **Begin Epic 1 Story 1.1** — Infrastructure verification on BorgStack-mini at 10.10.10.205. This is the first concrete "work happens" moment after planning.
5. **Parallel track — start authoring `prompts/triage-agent-v1.md`** — Galvani can begin drafting the system prompt content (hardened sections, 8-area criteria, rules, examples) while Epic 1 stories 1.1-1.3 run. This smooths the Story 2.3 timeline.

### Final Note

This assessment identified **0 critical issues**, **3 major issues** (all acknowledged design tradeoffs or missing small-scope items), **9 minor concerns** (mostly build-time unknowns with documented fallback paths), and **0 missing FR coverage items**. Against the scale and regulatory sensitivity of the product, this is an exceptionally complete planning package. The PRD, Architecture, and Epics are aligned, traceable, and disciplined. Phase 3 can begin.

Address P1-P4 during the first day of Phase 3. P5 is an accepted cost. Proceed with construction.

---

**Assessment completed:** 2026-04-11
**Assessor:** Implementation Readiness workflow (bmad-check-implementation-readiness)
**Artifacts reviewed:** `prd.md`, `architecture.md`, `epics.md`
**Total findings:** 12 (0 critical, 3 major, 9 minor) | **Coverage:** 70/70 FRs (100%)

