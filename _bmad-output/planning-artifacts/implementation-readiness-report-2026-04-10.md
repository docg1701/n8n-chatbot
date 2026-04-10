---
lastUpdated: 2026-04-10
stepsCompleted:
  - step-01-document-discovery
  - step-02-prd-analysis
  - step-03-epic-coverage-validation
  - step-04-ux-alignment
  - step-05-epic-quality-review
  - step-06-final-assessment
filesReviewed:
  - _bmad-output/planning-artifacts/prd.md
  - _bmad-output/planning-artifacts/product-brief-n8n-chatbot.md
  - _bmad-output/planning-artifacts/product-brief-n8n-chatbot-distillate.md
assessorRole: Expert Product Manager (BMad readiness validation)
assessmentDate: 2026-04-10
---

# Implementation Readiness Assessment Report

**Date:** 2026-04-10
**Project:** n8n-chatbot
**Assessor:** BMad `bmad-check-implementation-readiness` workflow

## Scope Note (read this first)

This readiness check was run at a point in the project lifecycle where **only the PRD
exists**. No Architecture document, no Epics & Stories document, and no UX document
have been produced yet — by design, since the project workflow explicitly sequences
Architecture and Epics as the next phase after PRD completion.

The six-step BMad readiness workflow was nevertheless executed end-to-end. Steps that
compare Architecture/Epics/UX against the PRD are reported as "not applicable — artifact
does not exist yet" instead of being skipped. The useful, actionable output of this run
is therefore concentrated in:

1. **PRD completeness assessment** — does the PRD provide enough structured input to
   drive Architecture and Epics creation without gaps?
2. **Forward-looking coverage scaffolding** — a traceability list of every FR and NFR
   that the future Epics document must cover, so the validation can be re-run later
   with the matrix already seeded.

## Step 1 — Document Discovery

### PRD Files Found

**Whole Documents:**
- `prd.md` (118 KB, 1063 lines, 2026-04-10)

**Sharded Documents:** none

### Architecture Files Found

None. Expected — Architecture is the next planned artifact.

### Epics & Stories Files Found

None. Expected — Epics follow Architecture.

### UX Design Files Found

None. The product has no GUI — the end-user surface is Telegram/WhatsApp conversations
(text + voice) and the attorney interface is a Telegram inline keyboard inside the
firm's private group. UX in the conventional "screens and components" sense does not
apply; conversational UX is instead captured inside the PRD User Journeys section and
the response-library requirements.

### Supporting Documents

- `product-brief-n8n-chatbot.md` (11 KB)
- `product-brief-n8n-chatbot-distillate.md` (12 KB)

### Issues Identified

- **No duplicates.** Clean single-source PRD.
- **Missing Architecture/Epics/UX documents** — expected at current project phase
  (planning), not a gap to flag against the user.
- **Proceeding with partial workflow.** The remaining steps are adapted to "PRD-only
  readiness" mode and explicitly document what cannot be validated today.

## Step 2 — PRD Analysis

### PRD Structural Completeness

The PRD contains all 11 top-level BMad PRD sections and a Table of Contents. Frontmatter
shows all 12 creation steps completed (`step-01-init` through `step-12-complete`). Input
documents are listed. Document counts are populated. Classification fields (`projectType`,
`domain`, `complexity`, `projectContext`, `progressionModel`, `platform`) are all set.

### Functional Requirements Extracted (70 total)

**Conversational Intake (FR1–FR10):**

- FR1: Text messages from client via Telegram.
- FR2: Text messages from client via WhatsApp.
- FR3: Voice messages from client via Telegram and WhatsApp.
- FR4: Voice transcription preserving semantic content for triage.
- FR5: Modality mirroring (text→text, voice→text+audio).
- FR6: Audio responses in natural pt-BR with neural male voice.
- FR7: Conversational context maintained across consecutive messages in a session.
- FR8: Context windowing/summarization for long conversations.
- FR9: Returning-client recognition with stored profile retrieval.
- FR10: Detect image/document attachments and flag for human review.

**Classification & Data Extraction (FR11–FR15):**

- FR11: Classification into 8 practice areas (or "undefined").
- FR12: Structured extraction of name, city, case summary, urgency.
- FR13: Active request of missing essential data in a natural sequence.
- FR14: Confirmation of collected data with the client.
- FR15: Profile persistence for future interactions.

**Lead Qualification & Handoff (FR16–FR24):**

- FR16: Deterministic scoring on area match, completeness, urgency, geography.
- FR17: Qualified/unqualified separation on a configurable threshold.
- FR18: Handoff package = structured data + full chronological transcript.
- FR19: Handoff delivery to attorney via private-channel notification, mobile-optimized.
- FR20: Quick-response interface (Accept / Need More Info / Decline).
- FR21: On Accept — client confirmation + state update.
- FR22: On Need More Info — follow-up relay flow.
- FR23: On Decline — warm closure with alternative resources.
- FR24: Configurable timeout with escalation branch.

**Non-Qualified Lead Handling (FR25–FR27):**

- FR25: Warm closure without rejection feel.
- FR26: Pre-approved alternative resources library.
- FR27: Area-based selection, never generating new content.

**Compliance & Safety (FR28–FR34):**

- FR28: AI disclosure in first turn.
- FR29: LGPD consent in first turn.
- FR30: Never generate legal advice / outcome prediction / legislation interpretation.
- FR31: Output block + fallback on legal-advice patterns.
- FR32: Complete audit trail per message / handoff / refinement / incident.
- FR33: Automatic retention policy per data type.
- FR34: Right-to-erasure execution by unique client identifier.

**Security (FR35–FR41):**

- FR35: Rate limiting per client.
- FR36: Input sanitization (HTML, invisible chars, whitespace).
- FR37: Jailbreak/prompt-injection detection and block.
- FR38: Off-topic detection before agent invocation.
- FR39: PII detection in input and output with redaction/block.
- FR40: Strict session isolation between clients.
- FR41: Credential encryption at rest.

**Resilience (FR42–FR47):**

- FR42: Retry with exponential backoff on transient failures.
- FR43: Automatic LLM failover (Groq free → Groq paid) on rate limits / persistent errors.
- FR44: Graceful degradation to text-only when TTS down.
- FR45: Graceful degradation (ask client to type) when STT down.
- FR46: Always send a friendly message — never silence, never crash.
- FR47: Exception capture in dedicated error workflow, severity-routed, operator notified.

**Refinement Pipeline Pre-Production (FR48–FR56):**

- FR48: Complete test environment (separate bot, separate database).
- FR49: Synthetic scenario generation in versioned repo files.
- FR50: Chatbot-vs-chatbot simulation, results logged separately.
- FR51: Internal human tests with persona briefings and structured feedback.
- FR52: Versioned golden dataset + regression runner.
- FR53: Adversarial red-team battery with pass/fail report per category.
- FR54: Audit bundle export to markdown.
- FR55: Attorney review + formal approval or blocking-issues list.
- FR56: Block any real-client traffic from production until formal Launch Gate passes.

**Operations & Monitoring (FR57–FR62):**

- FR57: Per-conversation metrics in analytics storage.
- FR58: Query interface for individual and aggregated metrics.
- FR59: Periodic operational report delivered to operator.
- FR60: Cost tracking for external services.
- FR61: Automatic notification on critical-metric threshold breach.
- FR62: Attorney can mark misclassified conversations to feed refinement.

**Template Export & Replicability (FR63–FR68):**

- FR63: Single self-contained JSON export of the full workflow.
- FR64: Import reproduces behavior after credentials + client-specific data configured.
- FR65: SQL initialization schema accompanies the template.
- FR66: Full credentials list + step-by-step configuration guide.
- FR67: Base prompt markdown files customizable per deployment.
- FR68: Template deployment dry run in a test environment before client production.

**Case Study Documentation (FR69–FR70):**

- FR69: Structured log of every refinement cycle (date, motivation, change, metrics, time).
- FR70: Repository documents the full journey for an external developer.

**Total FRs: 70.**

### Non-Functional Requirements Extracted (47 total)

**Performance (NFR1–NFR5):** p95 latency budgets (text ≤ 8s, audio ≤ 14s @ ≤100 convs/day),
deterministic-router offload 15–25%, 10–15 message context window, prompt caching when
supported, 30s hard ceiling with "one moment" fallback.

**Security (NFR6–NFR14):** Minimum-necessary external surface (Groq + Comadre only),
pre-Launch-Gate Groq retention/opt-out paperwork, deterministic session-key derivation,
credentials via `N8N_ENCRYPTION_KEY`, append-only audit trail, 48h right-to-erasure SLA,
100% red-team pass required for Launch Gate, disclosure/consent architecturally enforced
and not togglable, output legal-advice guardrail mandatory in production.

**Reliability & Availability (NFR15–NFR19):** Degraded-mode continuity, LLM failover in
≤ 2 retries without human intervention, logging failures do not interrupt conversations,
planned downtime ≤ 15 min outside business hours, webhook idempotency via provider
message IDs.

**Scalability (NFR20–NFR23):** 3× headroom (300 convs/day) at current cost, ≤ 20% CPU /
≤ 30% RAM on borgstack-mini, credentials-only scale to Groq paid, chat_analytics
indexing and partitioning per retention.

**Maintainability (NFR24–NFR29):** ≥ 90% visual-node ratio, single-responsibility
sub-workflows with documented I/O, prompts as Git-versioned markdown as source of truth,
structured refinement-log entries, ≤ 7 logical layers in main workflow, new-operator
onboarding ≤ 2 hours.

**Observability (NFR30–NFR33):** Forensic-grade granularity in chat_analytics (message,
response, prompt version, model, tokens, latency, guardrail flags), 5-minute
handoff-to-full-context reconstruction SLA, Error Trigger + 5-minute critical alert,
self-contained weekly report independent of external dashboards.

**Integration Quality (NFR34–NFR37):** Retry with exponential backoff per endpoint
tuned to sensitivity, external contracts documented, provider swap via credentials only
(topology unchanged), webhook signature/origin validation where supported.

**Portability & Replicability (NFR38–NFR41):** Self-contained export artifact, firm data
fully externalized, ≤ 8-hour new-client deployment, minimum-version and dependency list.

**Cost (NFR42–NFR44):** ≤ R$ 100/month operating cost (target R$ 0–50), frontier model
exclusively via Claude Code subscription (zero marginal cost), 80%-budget alerting.

**Data Residency & Sovereignty (NFR45–NFR47):** Client data exclusively in firm's
borgstack PostgreSQL, external providers receive only current-window context,
anonymization filter on every post-launch audit bundle before leaving borgstack.

**Total NFRs: 47.**

### Additional Requirements and Constraints (non-FR/NFR)

The PRD also codifies the following cross-cutting constraints that a future Architecture
and Epics document must honor, even though they are not numbered as FR/NFR:

- **Launch Gate (7 binary criteria)** in Product Maturity Milestones § Milestone 1:
  adversarial red-team 100%, golden-dataset classification ≥ 95%, LLM-as-judge tone
  ≥ 4.5/5, ≥ 50 human-tested scenarios with ≤ 2 "crappy bot" flags, formal attorney
  sign-off on ≥ 30 conversations, complete compliance checklist, validated error
  handling for all simulated failure modes.
- **6-layer no-legal-advice enforcement** in Domain-Specific Requirements: system
  prompt, tool design, static response library, output guardrails, red-team battery,
  attorney review. A single-point-of-failure architecture for this constraint is
  incompatible with the PRD.
- **12 Measurable Outcomes table** with per-metric source and cadence — implementation
  must wire every metric to a logging/instrumentation point.
- **34-step progression sequence** in Project Scope § Progression Sequence. This is the
  PRD's own seed for Epics/Stories — Architecture and Epics work should start from it.
- **Hard constraint: native n8n nodes first, JS Code nodes only when absolutely
  necessary** (NFR24 quantifies it at ≥ 90% visual nodes).
- **Hard constraint: BorgStack never runs model inference.** All inference is cloud
  (Groq) or Comadre.
- **Hard constraint: frontier intelligence (Claude Opus via Claude Code subscription)
  is operator-paced, never an n8n API call.**
- **Hard constraint: zero real clients until Launch Gate passes** (FR56 + the Innovation 2
  validation criterion).

### PRD Completeness Assessment

Against the BMad PRD template the document scores as follows:

| Section | Present | Depth | Comments |
|---|---|---|---|
| Executive Summary | Yes | High | Names the product, clients, channels, stack, constraints. |
| Project Classification | Yes | High | All 6 classification fields set with justification. |
| Success Criteria | Yes | High | User + Business + Technical + 12-row measurable outcomes table. |
| Product Maturity Milestones | Yes | High | Launch Gate has 7 explicit binary criteria. |
| User Journeys | Yes | High | 5 personas (3 end-users + 2 operational), each with capabilities reveal. |
| Domain-Specific Requirements | Yes | Very high | Regulatory (OAB/CNJ/LGPD), privilege, retention, 6-layer advice enforcement, audit trail, integrations, 7 risks with mitigations. |
| Innovation & Novel Patterns | Yes | High | 5 innovations with explicit validation criteria and counter-proofs. |
| Technical Architecture | Yes | High | 3-domain infra topology, 7-layer workflow blueprint, 9 sub-workflows, external integration table, replicability considerations. |
| Project Scope | Yes | Very high | In-scope, 13 explicit out-of-scope items with reasons, 34-step progression sequence, 5 scoping risks. |
| Functional Requirements | Yes | High | 70 FRs across 11 capability groups, all implementation-agnostic and testable. |
| Non-Functional Requirements | Yes | High | 47 NFRs across 10 quality attributes, linked back to Measurable Outcomes instead of duplicating. |

**Verdict on PRD itself: READY to feed Architecture and Epics creation.** No missing
section, no hand-wavy requirements, and the Technical Architecture section is unusually
detailed for a PRD (arguably over-detailed, but that is because the subsequent
Architecture document will refine node-by-node specs, not re-invent the topology).

### Observations the Architecture document should address

These are not gaps in the PRD — they are decisions the PRD explicitly **delegates** to
Architecture, and should therefore appear early in the Architecture document:

1. **Exact Groq model ID** (the PRD says "7-12B multilingual" but does not pin a
   specific model such as `llama-3.1-8b-instant`).
2. **Exact schema DDL** for `chat_analytics`, `client_profiles`, `test_scenarios`,
   `static_responses`, `n8n_chat_histories` (auto-created by the memory node) — PRD
   names the tables, Architecture should define columns, indexes, partitioning.
3. **Deterministic scoring formula** for `qualify_lead` (PRD describes inputs; the
   exact weights and threshold belong in Architecture).
4. **Guardrails node configuration** (which Guardrails community node, which patterns,
   which fallback text).
5. **Chat memory sizing** — "10–15 messages" is a range; Architecture should pick one.
6. **Redis rate-limit keys and TTLs** — PRD mandates rate limiting; exact window not set.
7. **Retry policies per endpoint** (NFR34 requires them, exact backoff curves not set).
8. **Evolution API payload normalization schema** — the PRD shows it exists; the actual
   unified schema is still to be defined.
9. **Anonymization filter implementation** (NFR47) — PRD mandates it, Architecture must
   specify how (regex? LLM? which PII categories?).

These are appropriate delegations, not PRD defects.

## Step 3 — Epic Coverage Validation

### Status: NOT APPLICABLE — Epics document does not exist yet

No `*epic*.md` file was found in `_bmad-output/planning-artifacts/`. Epics are the next
BMad artifact to produce, after Architecture. A full coverage matrix (70 FRs × epics)
cannot be constructed against a document that does not exist.

However, the readiness check can still perform **forward-looking coverage scaffolding**:
produce the list of FRs that the future Epics document will have to cover, with a
recommended epic grouping inferred from the PRD's own capability clusters and
Progression Sequence. This gives the future `bmad-create-epics-and-stories` run a
starting point and lets this readiness check be re-run later with the matrix seeded.

### Recommended Epic Grouping (hypothesis, not a mandate)

The 70 FRs cluster naturally into 10 implementation epics. Grouping honors the PRD's
**single-progression, no-MVP** principle — each epic leaves the system functional at its
current scope — and maps onto the PRD's own 34-step sequence. Architecture/Epics work
can accept, reject, or reshape this grouping, but it is a defensible starting point.

| Proposed Epic | Covers FRs | Progression steps (PRD) |
|---|---|---|
| **E1 — Foundations & Echo Bot** | FR1 (text/Telegram only, partial) | 1–2 |
| **E2 — Conversational Triage Agent with Memory** | FR1, FR7, FR8, FR9, FR11, FR28, FR29, FR30, FR36 | 3–7 |
| **E3 — Structured Data Extraction & Profile Persistence** | FR12, FR13, FR14, FR15, FR27 (partial) | 8–9 |
| **E4 — Lead Scoring & Human Handoff** | FR16, FR17, FR18, FR19, FR20, FR21, FR22, FR23, FR24 | 10–11 |
| **E5 — Input & Output Guardrails** | FR31, FR35, FR37, FR38, FR39, FR40, FR41 | 12–14 |
| **E6 — Error Handling, Analytics & Audit Trail** | FR32, FR42, FR43, FR46, FR47, FR57, FR58, FR60, FR61 | 15–17 |
| **E7 — Voice Modality (STT + TTS)** | FR3, FR4, FR5, FR6, FR10, FR44, FR45 | 18–19 |
| **E8 — Non-Qualified Lead Library + WhatsApp Channel** | FR2, FR25, FR26, FR27 | 20–21 |
| **E9 — Refinement Pipeline, Compliance Automation & Launch Gate** | FR33, FR34, FR48, FR49, FR50, FR51, FR52, FR53, FR54, FR55, FR56, FR59, FR62 | 22–30 |
| **E10 — Template Export, Deployment Guide & Case Study** | FR63, FR64, FR65, FR66, FR67, FR68, FR69, FR70 | 31–34 |

### Forward Coverage Matrix (seeded)

| FR | PRD Requirement (abbreviated) | Proposed Epic | Status |
|---|---|---|---|
| FR1 | Text via Telegram | E1, E2 | Pending epics |
| FR2 | Text via WhatsApp | E8 | Pending epics |
| FR3 | Voice via Telegram and WhatsApp | E7 | Pending epics |
| FR4 | Voice transcription | E7 | Pending epics |
| FR5 | Modality mirroring | E7 | Pending epics |
| FR6 | Audio responses pt-BR neural voice | E7 | Pending epics |
| FR7 | Conversational context maintained | E2 | Pending epics |
| FR8 | Context windowing/summarization | E2 | Pending epics |
| FR9 | Returning-client recognition | E2 | Pending epics |
| FR10 | Attachment detection | E7 | Pending epics |
| FR11 | Practice-area classification | E2 | Pending epics |
| FR12 | Essential-data extraction | E3 | Pending epics |
| FR13 | Active missing-data prompting | E3 | Pending epics |
| FR14 | Data confirmation with client | E3 | Pending epics |
| FR15 | Profile persistence | E3 | Pending epics |
| FR16 | Deterministic scoring | E4 | Pending epics |
| FR17 | Qualified/unqualified split | E4 | Pending epics |
| FR18 | Handoff package composition | E4 | Pending epics |
| FR19 | Attorney notification | E4 | Pending epics |
| FR20 | Accept/More info/Decline interface | E4 | Pending epics |
| FR21 | Accept branch | E4 | Pending epics |
| FR22 | Need-More-Info relay | E4 | Pending epics |
| FR23 | Decline branch with alternatives | E4 | Pending epics |
| FR24 | Timeout escalation | E4 | Pending epics |
| FR25 | Warm closure for unqualified | E8 | Pending epics |
| FR26 | Alternative-resources library | E8 | Pending epics |
| FR27 | Area-based resource selection | E8 | Pending epics |
| FR28 | AI disclosure first turn | E2 | Pending epics |
| FR29 | LGPD consent first turn | E2 | Pending epics |
| FR30 | No legal advice (prompt layer) | E2 | Pending epics |
| FR31 | Legal-advice output block | E5 | Pending epics |
| FR32 | Complete audit trail | E6 | Pending epics |
| FR33 | Automatic retention policy | E9 | Pending epics |
| FR34 | Right-to-erasure workflow | E9 | Pending epics |
| FR35 | Rate limiting | E5 | Pending epics |
| FR36 | Input sanitization | E2 | Pending epics |
| FR37 | Jailbreak/prompt-injection block | E5 | Pending epics |
| FR38 | Off-topic block | E5 | Pending epics |
| FR39 | PII detection | E5 | Pending epics |
| FR40 | Session isolation | E5 | Pending epics |
| FR41 | Credential encryption | E5 | Pending epics |
| FR42 | Retry with exponential backoff | E6 | Pending epics |
| FR43 | LLM failover | E6 | Pending epics |
| FR44 | TTS graceful degradation | E7 | Pending epics |
| FR45 | STT graceful degradation | E7 | Pending epics |
| FR46 | Always-friendly fallback | E6 | Pending epics |
| FR47 | Error-trigger dedicated workflow | E6 | Pending epics |
| FR48 | Test environment isolation | E9 | Pending epics |
| FR49 | Synthetic scenario generation | E9 | Pending epics |
| FR50 | Chatbot-vs-chatbot simulation | E9 | Pending epics |
| FR51 | Internal human tests | E9 | Pending epics |
| FR52 | Golden dataset + regression | E9 | Pending epics |
| FR53 | Red-team battery | E9 | Pending epics |
| FR54 | Audit bundle exporter | E9 | Pending epics |
| FR55 | Attorney formal review | E9 | Pending epics |
| FR56 | Launch Gate enforcement | E9 | Pending epics |
| FR57 | Per-conversation metrics | E6 | Pending epics |
| FR58 | Query interface | E6 | Pending epics |
| FR59 | Periodic operational report | E9 | Pending epics |
| FR60 | Cost tracking | E6 | Pending epics |
| FR61 | Threshold-breach alerting | E6 | Pending epics |
| FR62 | Attorney misclassification marking | E9 | Pending epics |
| FR63 | Single JSON export | E10 | Pending epics |
| FR64 | Import reproduces behavior | E10 | Pending epics |
| FR65 | SQL schema | E10 | Pending epics |
| FR66 | Credentials list + config guide | E10 | Pending epics |
| FR67 | Prompt markdown files | E10 | Pending epics |
| FR68 | Template deployment dry run | E10 | Pending epics |
| FR69 | Refinement-cycle log | E10 | Pending epics |
| FR70 | Full case-study documentation | E10 | Pending epics |

### Missing Requirements

None missing from the PRD — the 70 FRs together already describe the full v1.0 template.
The "missing coverage" check will become meaningful only once the Epics document exists.

### Coverage Statistics

- Total PRD FRs: **70**
- FRs covered in epics today: **0** (epics do not exist)
- Coverage percentage today: **n/a**
- FRs with a proposed epic landing spot in the forward matrix: **70** (100%)

## Step 4 — UX Alignment

### UX Document Status

**Not Found** — and **not required** for this project.

### Assessment

This product has no graphical user interface. The human-facing surfaces are:

1. **Client-facing conversational UI** — Telegram and WhatsApp, where UX is carried
   entirely by the bot's natural-language behavior. The PRD captures this in:
   - User Journeys (5 personas, full narrative flows)
   - FR5 modality mirroring
   - FR6 Brazilian Portuguese neural voice
   - FR7–FR9 memory and returning-client recognition
   - FR25–FR27 warm closure for unqualified leads
   - FR28–FR29 disclosure + consent in first turn
   - NFR1, NFR5 latency budgets
2. **Attorney-facing triage UI** — a Telegram inline keyboard with 3 buttons inside the
   firm's private Telegram group. The PRD captures this in:
   - Journey 3 (Dr. Maruzza receiving the handoff)
   - FR19–FR24 notification, quick-response, follow-up relay, timeout
3. **Operator-facing refinement UI** — the n8n editor, Claude Code inside the repo, and
   direct SQL queries. The PRD captures this in:
   - Journey 4 (Galvani's refinement cycle)
   - FR48–FR62 refinement pipeline and operations

None of these warrants a traditional UX document with wireframes, color tokens, or
component specifications. **Conversational UX lives inside the PRD itself**, which is
the correct place for it.

### Alignment Issues

None.

### Warnings

None. The absence of a UX document is not a readiness gap for this project.

## Step 5 — Epic Quality Review

### Status: NOT APPLICABLE — Epics document does not exist yet

A rigorous epic-quality review against the `bmad-create-epics-and-stories` standards
(user-value focus, epic independence, story sizing, forward-dependency forbidden,
table-creation timing, etc.) requires the Epics document as input. It does not exist.

### Forward-looking guidance for the future Epics work

These notes are intended to arm the future `bmad-create-epics-and-stories` run so that
the epics it produces pass this quality review without rework:

1. **Do not create "Infrastructure Setup" or "Database Setup" as standalone epics with no
   user value.** The PRD's Progression Sequence step 1 is "infra + credentials"; in the
   epics document that work belongs inside E1 (Echo Bot) — tables are created when a
   story first needs them (e.g., `client_profiles` is created inside an E3 story, not
   upfront in E1).
2. **Honor the single-progression principle in epic ordering.** Each of the 10 proposed
   epics above leaves the system genuinely functional at its current scope. E2 (Agent
   with memory) does not depend on E3 (extraction/persistence) — it simply answers with
   less structure. This is correct; resist the temptation to merge E2 and E3.
3. **Prompt files are source-of-truth Git artifacts, not n8n internal state.** When
   writing stories, be explicit: the story delivers a markdown prompt file in the repo
   AND the instruction that Galvani copy-pastes it into the AI Agent node — never a
   story that "edits the prompt in n8n" without the Git counterpart.
4. **No story should instruct Claude to write or edit n8n workflow JSON.** The hard rule
   in the project memory is: Galvani builds workflows manually in the visual editor.
   Stories should produce: (a) prompt markdown files, (b) SQL DDL, (c) JavaScript
   snippets for Code nodes only when truly needed, (d) build instructions for the
   human operator. A story like "generate the workflow JSON for Epic X" violates this.
5. **FR56 (block real traffic until Launch Gate) is an architectural gate, not a story.**
   E9 should contain a story that builds the separation (test bot token, staging
   database), and the Launch Gate itself should be an explicit story with acceptance
   criteria tied to each of the 7 binary gate criteria.
6. **Innovation 3 compliance:** the Epics document and every story must avoid "MVP,"
   "Phase 1," "v1 (initial)," "Phase 2" phrasing. Use "progression step" or "epic" or
   "step N" instead.
7. **6-layer no-legal-advice enforcement** must not be collapsed into a single story.
   Layer 1 (system prompt), Layer 3 (static response library), Layer 4 (output
   guardrail) live in different epics (E2, E8, E5 respectively). This is correct: the
   PRD explicitly says "never trust a single layer." The Epics document should preserve
   the distribution.
8. **Voice is a full epic (E7) not a pair of stories.** The PRD's voice requirements are
   cross-cutting (ingestion, transcription, modality flag, TTS, degradation) and
   warrant their own epic so the work is not fragmented across agent and security epics.

### Critical Violations found

None — no epics to violate anything yet.

### Major Issues found

None.

### Minor Concerns found

None.

## Step 6 — Final Assessment

### Overall Readiness Status

**PRD is READY** to drive the next BMad phase (Architecture, then Epics).

**Full implementation readiness (Architecture + Epics + PRD aligned) is NOT YET
APPLICABLE** — those artifacts do not exist, which is correct for the project's
current planning phase.

### Critical Issues Requiring Immediate Action

**None.** No blocking gaps were found in the PRD.

### Non-Critical Suggestions (Improvements, Not Blockers)

1. **Pin the Groq model identifier in Architecture** — PRD says "7–12B multilingual";
   Architecture should name the specific model ID (e.g., `llama-3.1-8b-instant`) with
   a fallback list.
2. **Define the `qualify_lead` scoring formula in Architecture** — the PRD names the
   inputs; weights and thresholds need to be made concrete.
3. **Specify the unified Telegram/WhatsApp payload schema in Architecture** — the PRD
   declares it exists; Architecture should publish its JSON shape.
4. **Define `chat_analytics` and `client_profiles` DDL in Architecture** — including
   indexes, partitioning key, and the retention-driven archival trigger called for by
   NFR23.
5. **Call out deduplication key source (webhook message ID) in Architecture** — NFR19
   requires idempotency; the exact dedup store (Redis SET with TTL? Postgres unique
   index?) is delegated to Architecture.
6. **Make anonymization filter specification explicit in Architecture** — NFR47 mandates
   it; the PRD does not fix its implementation.
7. **Add a small "Prompts Inventory" section to Architecture or the first Epic** listing
   every prompt markdown file the project expects (triage agent system prompt, fallback
   messages, disclosure, consent text, static response library entries, red-team battery
   prompts, synthetic scenario personas). This will be the canonical list that Epics and
   story acceptance criteria reference.

### Constraint Conflict Check (the 8 project hard rules)

| # | Hard rule | PRD-level conflict? | Notes |
|---|---|---|---|
| 1 | Native n8n nodes first, JS in Code nodes only when absolutely necessary (never Python) | NO | Enforced by NFR24 (≥ 90% visual nodes) and by Technical Architecture § Node Selection Principles. |
| 2 | Claude never writes n8n workflow JSON; Galvani builds workflows manually | NO | PRD says "Construction is manual in the n8n visual editor" (Executive Summary) and "The workflow is built manually in the visual editor" (Technical Architecture introduction). |
| 3 | Single-phase gradual progression, never MVP/Phase1/Phase2 | NO | Innovation 3 is explicitly this. Scope Philosophy reinforces it. Progression Sequence is 34 linear steps, no phase labels. |
| 4 | No real clients until Launch Gate passes | NO | Enforced by FR56, Innovation 2, and Milestone 1 (7 binary criteria). |
| 5 | TTS is always Comadre at 10.10.10.207, no alternatives | NO | Technical Architecture names Comadre as the only TTS, and NFR6 limits external services to Groq + Comadre. |
| 6 | BorgStack never runs local model inference | NO | Technical Architecture § Domain 1 says verbatim: "BorgStack does not run model inference." |
| 7 | Frontier auditor is Galvani running Claude Code with his subscription, not an n8n API call | NO | Innovation 1 is this. NFR43 reinforces zero marginal cost. Journey 4 illustrates the manual loop. |
| 8 | Groq free → Groq paid; Groq Whisper for STT; no Haiku, no GPT-4o-mini | NO | Executive Summary, Technical Architecture, FR43, NFR16, NFR22 all name Groq free + Groq paid fallback and Groq Whisper. No other operational model is referenced. |

**Conclusion:** this readiness check did not produce any recommendation that violates
the 8 hard rules. Nothing to override.

### Recommended Next Steps

1. **Run `bmad-create-architecture`** against `prd.md` to produce the Architecture
   document. Use the 9 "observations the Architecture document should address" from
   Step 2 as the first sanity-check list for that document.
2. **Run `bmad-create-epics-and-stories`** against `prd.md` + the new Architecture
   document. Use the proposed 10-epic grouping in Step 3 as a starting hypothesis —
   accept, reject, or reshape it based on what Architecture produces.
3. **Re-run `bmad-check-implementation-readiness`** after Epics exist. At that point
   the forward coverage matrix in Step 3 becomes a real coverage matrix and
   Step 5's epic-quality review will produce actionable findings.
4. **Do not add new scope to the PRD just because it is easy to.** The PRD is tight
   and the 13 explicit out-of-scope items in Project Scope are well-justified; any
   new capability request should enter as a later progression step, not a v1.0 FR.

### Final Note

This assessment identified **0 critical issues, 0 major issues, 7 non-critical
improvement suggestions, and 0 conflicts with project hard rules** across
the 6 review steps. The PRD is ready to feed the next planning phase.

The "readiness" verdict is therefore partitioned:

- **PRD-level readiness:** READY.
- **Full implementation readiness (PRD + Architecture + Epics aligned):** cannot be
  asserted today because Architecture and Epics do not yet exist. Re-run this skill
  once they do.
