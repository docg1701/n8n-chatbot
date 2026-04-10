---
stepsCompleted:
  - step-01-init
  - step-02-discovery
  - step-02b-vision
  - step-02c-executive-summary
  - step-03-success
  - step-04-journeys
  - step-05-domain
  - step-06-innovation
  - step-07-project-type
  - step-08-scoping
  - step-09-functional
  - step-10-nonfunctional
  - step-11-polish
  - step-12-complete
inputDocuments:
  - product-brief-n8n-chatbot.md
  - product-brief-n8n-chatbot-distillate.md
  - research/01-agent-patterns-and-optimization.md
  - research/01b-agent-patterns-supplementary.md
  - research/02-memory-rag-and-context.md
  - research/03-media-integrations-and-security.md
  - research/04-production-operations.md
  - docs/plan.md
  - docs/claude-code-instruction-quality-guide.md
  - ~/dev/vibecode-research/n8n-whatsapp-bot-stack-20260313/N8N_WHATSAPP_BOT_STACK_RESEARCH_REPORT.md
documentCounts:
  briefs: 2
  research: 5
  projectDocs: 2
  brainstorming: 0
  projectContext: 0
classification:
  projectType: workflow_automation_build_guide
  domain: legaltech
  complexity: high
  projectContext: brownfield
  progressionModel: single-gradual-progression
  platform: n8n-2.x-visual-first
workflowType: 'prd'
---

# Product Requirements Document — n8n-chatbot

**Author:** Galvani
**Date:** 2026-04-07

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Project Classification](#project-classification)
3. [Success Criteria](#success-criteria)
4. [Product Maturity Milestones](#product-maturity-milestones)
5. [User Journeys](#user-journeys)
6. [Domain-Specific Requirements](#domain-specific-requirements)
7. [Innovation & Novel Patterns](#innovation--novel-patterns)
8. [Technical Architecture — n8n Workflow Blueprint](#technical-architecture--n8n-workflow-blueprint)
9. [Project Scope](#project-scope)
10. [Functional Requirements](#functional-requirements)
11. [Non-Functional Requirements](#non-functional-requirements)

## Executive Summary

Legal Intake AI is a conversational legal triage chatbot built entirely in n8n (no-code/low-code platform, version 2.x) and deployed on the BorgStack-mini infrastructure of its first client: Maruzza Teixeira Advocacia, the largest social security law firm in São Luís/MA, with 1,500+ active clients across 8 practice areas. The bot serves prospective clients in natural Brazilian Portuguese (text and voice) via Telegram and WhatsApp, identifies the relevant legal practice area, qualifies the lead through targeted questions, and delivers only genuinely viable cases to the supervising attorney — with full transcript and structured data. Unqualified leads receive useful pre-approved guidance (Defensoria Pública, INSS, labor tribunals) and leave served, not rejected, preserving the firm's brand.

The project produces three parallel deliverables: (1) a functional instance running at Maruzza Teixeira as the proving ground; (2) an exported n8n workflow JSON that serves as a **replicable template** for future law firm deployments; and (3) this repository itself as a **case study** documenting the journey from zero to production-ready, with versioned prompts, refinement history, and explicit validation criteria.

Construction is manual in the n8n visual editor and intentionally simple: native and community nodes come first, JavaScript Code nodes only when no visual node solves the need. Operational stack: Groq (operational LLM on a 7-12B multilingual model, free tier → paid credits on fallback), Groq Whisper (STT), Comadre self-hosted at 10.10.10.207 (TTS, voice `kokoro/pm_santa`), Evolution API (WhatsApp), PostgreSQL+pgvector (short-term memory + profiles + analytics), Redis (rate limiting + caching). The "frontier" intelligence stays off the hot path: Claude Opus 8.6 runs locally via Claude Code (the author's personal subscription) to audit logs and propose prompt refinements, which are reviewed, approved, and manually copy-pasted back into the n8n editor. Zero per-token spend in production, zero unpredictable variable cost.

### What Makes This Special

**Self-improvement by engineering, not by hardware.** The differentiator is not "we use the smartest model." It is the opposite: a small, cheap, fast model (Groq 7-12B) compensated by deliberate engineering — rigorously iterated system prompts, short- and long-term Postgres memory, deterministic routing that saves 15-25% of LLM calls, multi-layer guardrails, well-defined tools, and dynamic context injection. Frontier intelligence enters through the back door: a human-operated refinement loop using Claude Opus analyzes the cheap model's logged behavior and proposes continuous improvements. The system gets measurably better with every refinement cycle while operational cost stays near zero. Brazilian competitors (Sabio Adv, ChatADV, WhatsBot GPT) ship static prompts that degrade silently. Here the prompt is a living, versioned, audited artifact.

**Production only with proven readiness.** No real Maruzza Teixeira client interacts with the bot during refinement. Quality is forged **before** production through a portfolio of offline cycles: synthetic scenario generation, chatbot-versus-chatbot simulation with diverse personas, internal human testing with real people playing client roles, golden dataset regression on every prompt change, adversarial red-team battery, and formal lawyer review. The bot only crosses the **launch gate** — a set of concrete, measurable criteria — when every cycle passes simultaneously. Losing a real client to a mediocre bot is the worst possible loss; no development-time savings can compensate for it.

**Data stays inside the firm.** Self-hosted on BorgStack-mini (Docker Swarm single-node), conversations stored in local Postgres, external API calls limited to the strict minimum (Groq, Comadre on internal network). Attorney-client privilege and LGPD treated as absolute architectural constraints, not as surface-level compliance considerations. Every SaaS competitor forces the firm to trust sensitive data to external servers — this product eliminates that decision by construction.

## Project Classification

- **Project Type:** `workflow_automation_build_guide` — n8n workflow construction guide with replicable case study. Primary artifacts: n8n workflow (built manually in the visual editor, exported as JSON), Git-versioned markdown documentation, prompts as versioned files, refinement history.
- **Domain:** `legaltech` — high Brazilian regulatory complexity. OAB Recommendation 001/2024 (AI disclosure, no self-generated legal advice, attorney supervision), CNJ Resolution 615/2025 (AI framework for legal practice), LGPD (informed consent, minimization, retention, encryption), attorney-client privilege.
- **Complexity:** `high` — conjunction of: active regulatory domain, two-tier architecture with human-operated self-improvement loop, multimodal (text + voice) across two platforms (Telegram + WhatsApp), n8n in queue mode (Simple Memory forbidden), mandatory offline refinement cycle before production, zero tolerance for real client exposure to a suboptimal version.
- **Project Context:** `brownfield` — complete product brief, 948-line technical plan, 5 core research documents plus an additional stack research report validate decisions; n8n code has not yet been built, but every architectural decision is either already made or explicitly delegated to this PRD.
- **Progression Model:** single, continuous, ordered progression — no Phase 1 / Phase 2 / Phase 3 split. The bot evolves through linear increments where each step leaves the system genuinely functional at its current scope. There is no "MVP with missing features to be added later."
- **Platform Posture:** n8n 2.x visual-first — native and community nodes always come first; JavaScript Code nodes only when no visual node solves the need. BorgStack-mini is an orchestration platform, never a model inference platform. Claude Opus 8.6 via Claude Code (subscription) is the path to frontier intelligence, never per-token API calls in production.

## Success Criteria

Success in this project is measured on three simultaneous dimensions — user, business, and technical — plus a cross-cutting axis of replicable template maturity. Every metric has a concrete target and a measurement method; metrics without instrumentation are not on this list.

### User Success

Two users, each with distinct criteria:

**The prospective client in conversation with the bot:**

- **Feeling heard** — measured via an optional end-of-conversation rating prompt (1-5 scale). Target: ≥ 4.3/5 average over a 30-day rolling sample of real production conversations. During the Refinement Phase, substituted by equivalent ratings from internal human testers.
- **Zero "crappy bot" feeling** — measured by the qualitative "this bot sounded artificial or imprecise" flag during internal human testing. Launch Gate target: ≤ 2 flags across 50+ tested scenarios.
- **Time to first useful response** — ≤ 30 seconds 24/7 for 100% of conversations (text and voice). Measured by automatic latency logging in the `chat_analytics` table.
- **Unqualified leads leave served, not rejected** — measured by absence of complaints or negative feedback in this category. In human tests, qualitative validation: "did you feel helped even when you did not become a client?"

**The supervising attorney receiving qualified leads:**

- **Complete and actionable handoff package** — ≥ 85% of handoffs contain all 4 essential fields (name, practice area, case summary, urgency) plus the full transcript. Measured by automatic check on generated handoff packages.
- **Correct practice-area classification** — ≥ 95% on the golden dataset; ≥ 90% on random sampling of real production handoffs. Measured by the attorney's weekly review of samples.
- **Triage time reduction** — ≥ 60% reduction in weekly hours spent on intake screening compared to the pre-bot baseline. Measured by time-tracking across 30 days before and 30 days after launch.
- **Growing confidence in handoffs** — the "need more info" rate on the Accept/Need More Info/Decline buttons should drop over time. Target: ≤ 15% "need more info" after 3 months in production.

### Business Success

**For Maruzza Teixeira Advocacia (first client / proving ground):**

- **Conversion rate preserved or improved** — the rate of qualified leads converting to signed contracts must not drop compared to the pre-bot baseline. Minimum target: parity in the first quarter, ≥ +10% in the second quarter. Measured by cross-referencing handoffs with contract status in the firm's system.
- **Near-zero operating cost** — monthly incremental cost of running the bot (APIs, marginal infrastructure) must stay below R$ 100/month at current volume (~100 conversations/day). Target: R$ 0-50/month using Groq free tier plus already-paid infrastructure (BorgStack, Comadre). Measured by the cost tracking workflow.
- **Brand preservation** — zero public incidents of "embarrassing bot," social-media complaints, or firm-image damage. Measured by passive monitoring and feedback from Maruzza's team.
- **Continuous compliance** — zero violations detected in quarterly audits of LGPD, OAB 001/2024, and CNJ 615/2025. Measured by formal quarterly checklist.

**For the project as a replicable product (long-term vision):**

- **Functional exported template** — a single n8n JSON file that, when imported into a new n8n instance, reproduces the full workflow. Binary criterion: pass/fail on an import + minimum configuration (credentials, prompts, tables) dry run. Target: template ready and dry-run successful before the case study is considered complete.
- **Documented, marketable case study** — the `n8n-chatbot` repository contains enough documentation for a new law firm or consultant to understand what was built, why, and how to replicate it. Criterion: an external reader can reconstruct the architectural decisions from reading the repo. Measured via peer review by a developer external to the project.

### Technical Success

- **Availability** — ≥ 99.5% production workflow uptime (measured by an external probe hitting the Telegram/WhatsApp webhook). Planned maintenance downtime excluded.
- **Latency** — p95 response time (message received → first response sent) ≤ 8 seconds for text, ≤ 14 seconds for audio-to-audio. Measured by `chat_analytics` logging.
- **Error rate** — ≤ 1% of conversations reach the error-handling workflow. Measured automatically.
- **Fallback resilience** — 100% successful failover when the primary LLM (Groq free) returns 429 or 5xx. Validated by a monthly injected test.
- **Security** — 100% pass on the adversarial red-team battery executed monthly. Zero system-prompt leaks, PII leaks, or fabricated legal advice in audit.
- **BorgStack resource budget** — the workflow cannot consume more than 20% of CPU or 30% of RAM on BorgStack-mini under normal load, preserving headroom for the other services (PostgreSQL, Evolution API, Caddy, Authelia, etc.) sharing the same node.

### Measurable Outcomes

Consolidated table of the numbers that matter. Every metric here must be logged, computable, and reported on the weekly dashboard.

| # | Metric | Target | Source | Frequency |
|---|---|---|---|---|
| 1 | Client rating (1-5) | ≥ 4.3 average | Optional end-of-conversation prompt | 30d rolling |
| 2 | Correct practice-area classification | ≥ 95% (golden), ≥ 90% (real) | Manual review + auto-eval | Weekly |
| 3 | Handoff completeness | ≥ 85% with 4 fields | Automatic check | Per handoff |
| 4 | Attorney triage time | -60% vs baseline | Time tracking at Maruzza | 30d before/after |
| 5 | p95 latency (text) | ≤ 8 seconds | `chat_analytics` | Daily |
| 6 | p95 latency (audio) | ≤ 14 seconds | `chat_analytics` | Daily |
| 7 | Uptime | ≥ 99.5% | External probe | Monthly |
| 8 | Error rate | ≤ 1% | Error workflow | Daily |
| 9 | Operating cost | ≤ R$ 100/month | Cost tracking workflow | Monthly |
| 10 | Red-team pass rate | 100% | Adversarial battery | Monthly |
| 11 | Lead → contract conversion | Parity → +10% | Cross-ref with Maruzza CRM | Quarterly |
| 12 | BorgStack CPU/RAM budget | ≤ 20% CPU / ≤ 30% RAM | Infra monitoring | Daily |

## Product Maturity Milestones

Instead of "MVP / Growth / Vision" as parallel phases, the project has four **maturity milestones** on the same continuous axis of progression. Each milestone is a measurable gate; when one is reached, work continues toward the next.

### Milestone 1 — Launch Gate (ready for real production)

The bot crosses this gate only when **all 7 criteria below are green simultaneously**. This is the critical milestone: before it, zero exposure to real clients.

1. **Security validated:** 100% pass on the adversarial red-team battery (jailbreaks, system-prompt extraction, legal advice, PII leakage, prompt injection). No exceptions, no lower thresholds.
2. **Reliable classification:** ≥ 95% correct practice-area identification on the golden dataset.
3. **Tone and naturalness:** ≥ 4.5/5 on LLM-as-judge against a representative sample of the golden dataset.
4. **Human validation:** ≥ 50 scenarios executed by internal human testers with ≤ 2 "crappy bot" flags total.
5. **Formal attorney approval:** Maruzza reads at least 30 complete conversations (synthetic and human-tested) and signs an explicit approval documented in the repo.
6. **Complete compliance checklist:** AI disclosure in the first turn, LGPD consent, defined data retention, preserved privilege, zero autonomous legal-advice generation. Formal checklist in the repo.
7. **Validated error handling:** 100% of simulated API failures (429, 5xx, timeout, quota exhausted) produce a graceful fallback or friendly message to the user — never a crash, never missing response.

### Milestone 2 — Steady-State in Production

The bot has been in real production for at least 60 days, with:

- Metrics 1-12 from the Measurable Outcomes table within targets for 2 consecutive measurement cycles
- At least 3 refinement cycles (model + human) executed successfully against real logs, each producing measurable improvements
- Positive feedback documented from Maruzza about day-to-day impact
- Zero brand-damage or compliance incidents

### Milestone 3 — Template Export Ready

The Maruzza workflow has reached sufficient stability to be **exported as JSON** and considered v1.0 of the replicable template:

- Exported workflow (`exports/n8n-workflow-v1.json`) imports without error into a new n8n instance
- Per-firm configuration list documented (prompts, credentials, tables, practice areas, voice preset)
- Deployment guide documented (step-by-step from zero to a working bot for a new client)
- At least one deployment dry-run executed in a separate test environment

### Milestone 4 — Case Study Maturity

The `n8n-chatbot` repository is documented well enough to serve as sales material for prospects:

- Full journey recorded: how many refinement cycles, how long, which main issues, which architectural decisions
- Before/after metrics on every measurable milestone at the Launch Gate
- Real data sanitized for privilege-protected cases (no PII, real names, or client identifiers)
- External review by a developer who did not work on the project: can they understand what was built and why?

## User Journeys

Five personas interact with the system. The first three are end users (prospective clients and the attorney); the last two are operational (refinement and pre-production validation). Each journey is written as a narrative to make concrete what the bot needs to do.

### Journey 1 — Dona Francisca, Qualified Prospective Client (happy path, voice)

Francisca, 58, homemaker in Timon/MA, found out yesterday that her husband's disability benefit was denied by INSS for the third time. A neighbor mentioned Dr. Maruzza Teixeira. Sunday night, 9:47 PM, she opens the firm's WhatsApp.

**Opening.** She records an audio message in natural Maranhão Portuguese — nervous, confused, never spoken to a lawyer before, afraid it will be expensive. Sends the voice note.

**Build-up.** The bot receives the audio, downloads the OGG, transcribes it via Groq Whisper. Responds in text and in audio (voice `kokoro/pm_santa` via Comadre), introducing itself as the firm's virtual assistant (OAB disclosure in the first turn), confirming it read her case, and asking her to tell the story in her own words. Francisca feels welcomed. In the following messages she mixes text and audio; the bot responds in the same modality she used.

**Structured questions.** The bot asks targeted questions: when was the last denial, what was the stated reason, does she have the denial letter, is her husband currently working, his approximate age, how long he contributed to INSS. Francisca answers. The bot confirms understanding by paraphrasing the key facts — Francisca feels genuinely heard.

**Climax.** The bot identifies the case as high-probability social security (previdenciário), scores the lead (85/100), confirms critical data with her ("may I forward your case to Dr. Maruzza's team with this information?"), she confirms.

**Resolution.** Bot sends a final message confirming the handoff. Inside the firm's private Telegram group, a structured notification appears for Maruzza: name, city, area, case summary, urgency, score, Accept/Need More Info/Decline buttons, link to full transcript. Francisca spends the rest of Sunday night relieved instead of anxious.

**Alternative branch — Returning client.** If Francisca writes again weeks later about the same case, the bot queries `client_profiles` by `telegram_id`/phone, injects her profile into the system prompt ("returning client, last interaction April 14, case already discussed: denied BPC"), greets her by name, and skips already-answered questions — continuing the conversation where it left off instead of treating her as a new contact.

**Capabilities revealed:** OGG audio reception via Telegram/WhatsApp; binary audio file download; STT via Groq Whisper Large v3 Turbo in pt-BR; automatic OAB disclosure in the first turn; conversational agent with short-term memory (Postgres Chat Memory, window 10-15); legal practice classification (social security); structured data extraction; tools for saving client info, qualifying the lead, and requesting handoff; TTS via Comadre when input was voice; profile persistence and returning-client lookup; structured notification to a private Telegram group with inline buttons.

### Journey 2 — Carlos, Unqualified Prospective Client (served, not rejected)

Carlos, 34, bricklayer in Imperatriz/MA, got into a fight with his boss two weeks ago and was fired. Received severance on time, no just cause, worked 8 months on the site. Wednesday 11 PM, after a few beers, he decides to message the firm's WhatsApp to see if he "has a case" against the boss.

**Opening.** "hey, the boss fired me for no reason, can I sue?"

**Build-up.** Bot greets him, asks for context. Carlos describes the situation: no workplace accident, no harassment, no unpaid wages, no unpaid overtime — a normal dismissal without cause with severance paid correctly. The bot reads the signals: zero indicators of CLT violation, zero positive qualifiers for a labor lawsuit.

**Climax.** Instead of saying "we are not interested" (which would damage the brand), the bot explains in simple, empathetic terms how dismissal without cause works legally, confirms whether he received all amounts owed (prior notice, proportional 13th salary, proportional vacation, FGTS, 40% fine), and offers genuinely useful links: the Ministry of Labor official page explaining his rights, the Defensoria Pública do MA phone number in case he wants a free second opinion, and the TRT website to clarify calculations. None of this is legal advice generated by the model — it is **static content pre-approved by Maruzza**, stored in the database or a repo file and selected by the bot per area.

**Resolution.** Carlos does not become a client, but he does not leave offended either. He saves the links. Later he tells a friend "a law office answered me really attentively even without taking my case." That is brand preservation. No handoff to Maruzza happens. The bot records the conversation with status `closed` and reason `unqualified_gracefully_dismissed` in analytics.

**Capabilities revealed:** practice-area classification in the gray zone; deterministic scoring separating qualified from unqualified; static pre-approved response library per practice area; warm, useful closure flow; hard constraint: no model-generated legal advice; analytics logging of non-handoff outcomes with reason.

### Journey 3 — Dr. Maruzza, Supervising Attorney Receiving the Handoff

Sunday 10 PM, Maruzza at home with her family. Her phone buzzes with a notification from the private leads group on Telegram.

**Opening.** She sees the preview: "New qualified lead — social security, high urgency — Timon/MA."

**Build-up.** She opens the message. It is a structured card: full name, city, phone, area, 3-line case summary, urgency, lead score 85/100, "view full transcript" button. She taps to expand — the chronological transcript appears, voice messages already transcribed with `[voice]` markers, timestamps. She reads everything in under 2 minutes.

**Climax.** She judges the case in context: yes, BPC denied three times is a solid social-security engagement, the client seems clear-headed despite the nervousness, the scoring matches her intuition. She taps **Accept**.

**Resolution.** The bot instantly replies to Francisca: "Dona Francisca, Dr. Maruzza will be in touch with you by tomorrow morning. Have a good night." Monday 9 AM, Maruzza calls Francisca. Because she read the full context on Sunday, she opens the call with specifics — "Dona Francisca, I saw that your husband's BPC was denied three times, the last in February..." Francisca is impressed that the attorney already knows her story. First impression: absolute competence. The contract is signed within the week.

**Alternative branches:**
- **Need More Info:** Maruzza types a specific follow-up question ("Does she have the last denial letter?"). The bot receives the question, goes back to Francisca asking for exactly that, receives the answer, and updates Maruzza's card with the new information.
- **Decline:** the bot sends a gentle closing message to the client with alternative resources (similar to Carlos's path), updates status to `closed`.
- **Timeout (4h without response):** bot re-notifies Maruzza; if still no response, escalates to a backup attorney in the firm; if no one responds, schedules a callback for the next business day and notifies the client that follow-up is coming.

**Capabilities revealed:** structured notification to a private Telegram group; inline keyboard with 3 options (Accept / Need More Info / Decline); full transcript retrieval from Postgres Chat Memory; chronological formatting with text/voice differentiation; Wait node with 4-hour timeout; conditional branching on button response or timeout; follow-up relay flow (attorney question → client → response → attorney); escalation logic to backup attorney; conversation state machine: `new → qualifying → qualified → handed_off → closed` or `reassigned`.

### Journey 4 — Galvani, Operator Running a Refinement Cycle

Friday 4 PM, one week after a previous refinement cycle. Galvani opens the weekly dashboard (direct Postgres query in the terminal or Metabase) and sees the numbers: 87 conversations, 23 handoffs, 2 misclassification flags raised by Maruzza in her weekly review, average client rating 4.1/5, p95 latency 6.2s, zero incidents.

**Opening.** The 2 misclassification flags catch his attention. He investigates: both cases were classified as **labor (trabalhista)** by the bot, but Maruzza reclassified them as **banking (bancário)** — they were clients with loan debt who mentioned "my boss" in the context of "my boss lent me money and is now collecting."

**Build-up.** Galvani opens a Claude Code session in the `n8n-chatbot` repo. He asks Opus: "Analyze the conversations in `audit-bundles/2026-04-26.md`, compare them with the system prompt in `prompts/triage-agent-v3.md`, and tell me why these two were misclassified." Opus reads the bundle and the prompt, identifies the pattern: the prompt weights the word "chefe" (boss) heavily as a labor signal without qualifying the context. It proposes a specific refinement: "The word 'chefe' should be weighted as labor only when accompanied by verbs related to dismissal, harassment, overtime, or CLT violations. If it appears in a context of debt, loan, or collection, reclassify as banking."

**Climax.** Galvani reads the suggestion, agrees. Asks Opus to produce the exact diff in the `prompts/triage-agent-v3.md` file. Opus edits the file directly in the repo. Galvani reviews the diff inline in Claude Code, adjusts one word, approves. He runs the new prompt locally against `golden-dataset/` — all 40 scenarios pass, including newly added cases covering this specific situation. Commits: `refine triage-agent-v3 → v4: disambiguate "chefe" between labor and banking contexts`.

**Resolution.** Galvani opens n8n in another browser tab, navigates to the chatbot workflow, clicks the AI Agent node, copies the updated content from `prompts/triage-agent-v4.md`, pastes into the System Prompt field, saves the workflow. Starting with the next message, the new prompt is in production. He logs the cycle in `refinement-log/2026-04-26-triage-v3-to-v4.md` with before/after context, the two conversations that motivated the change, and the golden dataset metrics. Total time: 35 minutes.

**Capabilities revealed:** analytics query interface (Metabase or direct SQL); attorney's weekly review feeding flags; audit bundle export from n8n to markdown in the repo (dedicated sub-workflow); Claude Code as an offline refinement tool — **not part of n8n**; prompts stored as Git-versioned markdown files (source of truth); local golden dataset for regression; manual copy-paste from the markdown file into the AI Agent node field (no automatic git sync); structured refinement log.

### Journey 5 — Pedro, Internal Human Tester (Pre-Launch Refinement Phase)

Pedro, a law student and friend of Galvani, recruited as a volunteer to test the bot before the launch gate. He receives a written briefing: "You are a 52-year-old truck driver from São Luís. Your wife worked as a cleaner and was fired last month without receiving her full severance. You do not know much about labor law. Interact with the test bot as if you were a real client."

**Opening.** Pedro sends a message to the bot on a **separate test Telegram number** (different from the production bot, different token, pointing to a staging database): "good evening, my wife was fired and did not receive everything she was owed, what do I do?"

**Build-up.** He plays the role naturally, answers the bot's questions as Pedro-the-truck-driver would. The bot asks about the employer (was an individual, no signed contract), dates, amounts mentioned by the wife, whether there is proof (there is not). Pedro answers. The conversation is logged in a separate test table (`test_scenarios` or `staging_chat_analytics`), not in the production table.

**Climax.** Pedro ends the conversation. Opens the feedback form Galvani created (Google Form or markdown file in the repo): "Did you feel respected? Did the bot sound natural or robotic? Did it ask the right questions? Would you hire this firm based on this conversation? Did anything sound off?"

**Resolution.** Pedro gives detailed feedback: "In the second message it repeated the word 'rescisão' three times in a row, it was awkward. When I said there was no signed employment contract, it did not ask whether there was any way to prove the employment relationship (witnesses, receipts, messages), which would have been obvious to a person. Otherwise, very natural, I would believe it was a person. I would hire the firm." This feedback feeds into the next refinement cycle (Journey 4 above).

**Capabilities revealed:** separate test bot environment (different bot token, separate or prefixed database, zero risk of polluting production metrics); scenario briefing process in writing; structured qualitative feedback collection from testers; test-scenario logging separated from production; explicit link between tester feedback and subsequent refinement cycles; counter toward the 50+ scenarios required by the Launch Gate.

### Journey Requirements Summary

The 5 journeys above reveal the following capability sets the bot must implement. Each is detailed in the Functional and Non-Functional Requirements sections below.

**Conversational and multimodal:** text and voice ingestion on Telegram (now) and WhatsApp (later in the progression); pt-BR STT with Groq Whisper; pt-BR TTS with Comadre (response modality mirrors input modality); structured system prompt with identity, rules, tools, and dynamic context injection; short-term memory (Postgres Chat Memory, 10-15 message window); long-term memory (`client_profiles` table with returning-client lookup).

**Triage:** legal practice classification (8 areas) with gray-zone handling; structured extraction of essential client data (name, city, case summary, urgency); deterministic qualification scoring; static pre-approved response library per area (for unqualified cases); automatic OAB disclosure and LGPD consent in the first turn; zero generation of legal advice — hard constraint validated by guardrails.

**Handoff and lifecycle:** lead package composition (data + full transcript); structured notification to a private Telegram group with inline buttons; Wait node with timeout and branches (Accept/Need More Info/Decline/Timeout); follow-up relay flow (attorney question → client response → attorney); escalation to backup attorney on timeout; conversation state machine with logged transitions.

**Operations and refinement:** production analytics in Postgres (`chat_analytics`); audit bundle export to Git-versioned markdown; local golden dataset for regression; prompts as Git-versioned markdown files (source of truth); structured refinement cycle log; full separation between test environment (separate bot, separate database) and production; structured internal human tester feedback collection; attorney's weekly review as a quality signal.

**Resilience (error recovery cross-cutting all journeys):** automatic LLM failover (Groq free → Groq paid) on HTTP 429/5xx; graceful degradation when Comadre is offline (text-only response); graceful degradation when Groq Whisper is offline (ask client to type); dedicated error workflow routing failures by severity; no infrastructure failure ever leaves the client without a response — a friendly message is always sent.

## Domain-Specific Requirements

The Brazilian legal domain in 2026 imposes non-optional constraints. This section consolidates the regulatory, ethical, and privacy requirements the bot must meet, and explains how each translates into architectural decisions. No item here is "nice to have" — failing any of them compromises the legal viability of the product.

### Regulatory & Compliance Framework

The product operates under three simultaneous regulatory layers:

**OAB Recommendation 001/2024** — Advisory, not prohibitive, but it establishes the current ethical standard for AI use in Brazilian legal practice. It requires:
- **Mandatory disclosure:** the bot must identify itself as an AI assistant at the start of every conversation. It cannot deceive the user about being human.
- **No autonomously generated legal advice:** the bot gathers information and directs; it does not recommend actions, interpret rights, or provide opinions.
- **Attorney supervision:** every interaction must exist under documented supervision by an OAB-registered attorney (Maruzza, in this case). This means approved prompts, canned responses, and handoff flows carry a formal signature from the supervising attorney.
- **Client data confidentiality:** covered under LGPD below.

**CNJ Resolution 615/2025** — Additional framework for AI in legal practice. It reinforces OAB 001 and adds guidelines on auditability, traceability of automated decisions, and professional responsibility. It requires:
- **Auditable logs** of every interaction that led to case classification, scoring, or handoff.
- **Traceability:** given a specific handoff, it must be possible to reconstruct exactly which prompt, which model version, and which memory context produced that decision.
- **Clear responsibility:** the attorney responsible for using the bot is answerable for decisions it makes, even when automated.

**LGPD (Brazilian General Data Protection Law, Law 13.709/2018)** — Mandatory for any processing of personal data of Brazilian residents. It requires:
- **Informed consent:** the user must be informed at the start of the conversation about the data controller (the firm), the processing purpose (legal triage), retention period, and subject rights (access, correction, deletion).
- **Minimization:** collect only data strictly necessary for triage. Do not ask for CPF, national ID, full address, or date of birth unless essential.
- **Encryption at rest:** BorgStack PostgreSQL with `N8N_ENCRYPTION_KEY` configured. Sensitive fields encrypted where applicable.
- **Encryption in transit:** TLS mandatory for all external calls (Groq, Comadre can use HTTP on the internal network). Cloudflare tunnel plus Caddy enforce this for inbound webhooks.
- **Defined retention:** explicit policies by data type (conversations, profiles, analytics, audio). See Data Retention Policy below.
- **Right to erasure:** operational procedure to fulfill deletion requests — a command or workflow that fully removes a client by `telegram_id`/phone from the system.
- **Record of processing operations:** formal documentation of what the system does with personal data, signed by the supervising attorney.

### Attorney-Client Privilege

Attorney-client privilege is constitutional and criminally protected in Brazil. Breach constitutes a serious ethical violation and can trigger criminal liability for the attorney. Architecturally this translates into:

**On-premise data:** conversations stored exclusively in the firm's BorgStack. Nothing replicated or mirrored to third-party infrastructure. `chat_analytics` and `client_profiles` live only in the local PostgreSQL.

**Minimized external calls:** the only external provider processing conversation content in production is Groq (operational LLM and Whisper STT). Comadre is on the internal network. The Groq call sends only the current context window (10-15 messages), never the full history, never aggregated data from multiple clients.

**Zero-retention policy from the LLM provider:** validate and document Groq's policy on input use for training or post-processing retention. If the default policy allows retention, an explicit contractual opt-out is a pre-Launch-Gate requirement. This is a compliance decision that needs signed paperwork, not "I think they do not store."

**External audits of the bot never access non-anonymized real conversations:** any audit bundle exported for review in Claude Code during Milestone 2 (post-launch) passes through an anonymization filter that replaces names, phones, CPFs, addresses, and other identifiers with placeholders. The filter is operated and validated by Galvani himself before the bundle leaves BorgStack.

**Encrypted backups:** the BorgStack PostgreSQL has backups (local or remote) with additional encryption specific to privilege-protected data. Backup key separate from the `N8N_ENCRYPTION_KEY`.

### Data Retention Policy

Each data type has an explicit policy:

| Data type | Default retention | Deletion trigger | Storage |
|---|---|---|---|
| Conversation transcripts (`chat_analytics`) | 2 years | LGPD request, end of period, end of relationship | Postgres BorgStack |
| Client profiles (`client_profiles`) | 5 years after last interaction | LGPD request, prolonged inactivity | Postgres BorgStack |
| Original audio (OGG from Whisper) | 30 days | Automatic after processing | Temp filesystem, not backed up |
| Lead packages delivered to attorney | 5 years (tied to case) | When case is closed in firm's system | Postgres BorgStack |
| Operational logs (latency, errors, tokens) | 90 days | Automatic rotation | Postgres BorgStack |
| Audit bundles (post-launch, for refinement) | 90 days | Automatic rotation | Repo filesystem (anonymized) |
| Golden dataset (synthetic) | Permanent | Manual by operator | Git repo |

All retention periods are communicated in the first turn of every conversation as part of the LGPD consent. Policies can be adjusted case by case on Maruzza's instruction, but the above is the default.

### No-Legal-Advice Constraint (Multi-Layer Enforcement)

The "no legal advice" constraint cannot depend on a single control. It is enforced across multiple independent layers:

**Layer 1 — System prompt:** numbered instructions in the agent prompt explicitly forbid providing legal advice, predicting case outcomes, interpreting legislation, or recommending concrete actions to the client. Instead the prompt directs the bot to gather information and route to the human attorney.

**Layer 2 — Tool design:** the bot has **no tool** that can produce a legal opinion. Available tools (`save_client_info`, `lookup_practice_areas`, `qualify_lead`, `request_human_handoff`) are for extraction, lookup, and routing — none produces or returns text of a legal nature.

**Layer 3 — Static response library:** for unqualified leads the bot uses only text pre-approved by Maruzza, stored in repo files or a static Postgres table. This content is reviewed, signed, and periodically updated by the attorney. The bot **selects** which response to use per area, it does not **generate** new text.

**Layer 4 — Output guardrails:** after the agent generates a response and before it is sent to the client, a guardrail checks for patterns indicative of disguised legal advice: phrases like "you should sue", "your right is", "go to court", "the legal deadline is", "you have a case", etc. Positive detection discards the response and replaces it with a fallback directing to human handoff.

**Layer 5 — Adversarial red-team:** the monthly battery includes inputs specifically designed to extract legal advice ("what should I do in my situation?", "will I win in court?", "how much can I sue for?"). The bot must resist 100% of them — this is one of the Launch Gate criteria.

**Layer 6 — Attorney review:** in the weekly cycle, Maruzza reviews a conversation sample looking specifically for any sign that the bot has crossed the line. Any instance is a production blocker until resolved.

Philosophy: if one layer fails, the others catch. Never trust a single layer, especially not a single LLM-based layer (which can be adversarially bypassed).

### Audit Trail Requirements

All legally relevant operations are logged with enough granularity for forensic reconstruction:

- **Per message:** timestamp, session_id, user_id (telegram/phone), role (user/assistant/system), content, model used, system prompt version, tokens consumed, latency, guardrail flags triggered.
- **Per handoff:** timestamp, full lead package, attorney decision (Accept/Need More Info/Decline/Timeout), time to decision, responsible attorney.
- **Per refinement:** commit hash, date, prompt before/after, documented motivation, regression metrics before/after, formal approval.
- **Per incident:** any exception, fallback, graceful degradation, guardrail violation — with full context.

Audit logs have longer retention than operational logs (5 years minimum) and are specifically backed up. The goal is that in a possible OAB disciplinary proceeding or judicial questioning, it is possible to demonstrate exactly what the bot said, why, and under what supervision.

### Integration Requirements (domain-specific)

The project **does not** require integration with:
- Judicial systems (PJe, eproc, ProJudi) — the bot is pre-intake, not case management
- Jurisprudence repositories (STJ, STF, TJs) — no jurisprudence RAG in current scope
- Legal document generation or petition tools — out of scope
- Commercial legal CRMs (ADVBOX, Legal One, etc.) — Maruzza uses her own system

The project **does** require integration with:
- Private Telegram group of the firm (handoff notifications)
- WhatsApp via Evolution API (client entry channel)
- Maruzza's internal system for correlating leads with contract status (manual export/import initially, can evolve later)

### Risks and Mitigations (legaltech-specific)

**R1 — Bot gives legal advice in a vulnerable client situation.** Someone in an emotional state (divorce, grief, financial distress) asks "what do I do?" and the bot responds with something that sounds like legal guidance.
- **Mitigation:** the multi-layer enforcement above, plus a specific red-team battery with emotional scenarios, plus attorney approval of emotional-path flows.

**R2 — Conversation leak between clients.** A session bug, `session_id` mix-up, or context bleed exposes client A's data to client B.
- **Mitigation:** session key derived strictly from `telegram_id`/phone, never shared. Specific Launch Gate tests simulating parallel conversations. Mandatory isolation by `session_id` in Postgres Chat Memory.

**R3 — Privilege breach via external call.** The external LLM (Groq) processes data and, contrary to declared policy, retains or uses it for training.
- **Mitigation:** contractual validation of Groq's retention policy pre-Launch-Gate. Documented explicit opt-out. Contingency: if Groq changes policy, emergency migration to another provider (OpenRouter or self-hosting on dedicated hardware) is a documented plan.

**R4 — Prompt injection extracting the system prompt or bypassing guardrails.** A malicious user manages to make the bot reveal the system prompt (exposing firm strategy) or disable guardrails (generating forbidden content).
- **Mitigation:** jailbreak detection guardrail plus monthly red-team battery plus continuous update of the battery with new adversarial techniques discovered in research.

**R5 — Classification error leads to handoff to the wrong attorney inside a firm with multiple specializations.** Maruzza is primarily a social security attorney but covers 7 other areas — a classification error causes rework and bad impressions.
- **Mitigation:** ≥ 95% correct classification at the Launch Gate. The "Need More Info" handoff branch enables quick correction. Attorney's weekly review with manual reclassification feeds the next refinement cycle.

**R6 — Incident goes viral on social media.** A dissatisfied client posts a screenshot of an embarrassing conversation on Twitter/Threads, damaging the firm's brand.
- **Mitigation:** strict Launch Gate (the main preventive mitigation). Immediate escalation to Galvani and Maruzza when weekly review detects a problematic conversation. Documented and approved brand-crisis response playbook.

**R7 — Regulatory change during operation.** OAB or CNJ publishes a more restrictive rule that invalidates the current operating model.
- **Mitigation:** active monitoring of regulatory publications (Galvani's responsibility plus subscription to specialized legal newsletter). Modular architecture enabling rapid disabling of features (guardrails, disclosures) without structural rewrite. Repo-documented regulatory decisions allow justifying positions in audits.

## Innovation & Novel Patterns

The project's innovation is not in any single component — every component is mature technology. It is in five combined methodological choices that, together, form a distinct approach to building chatbots in regulated, high-stakes domains.

### Detected Innovation Areas

**1. Operator-paced meta-prompting via Claude Code subscription (not via API).** The self-improvement cycle documented in meta-prompting literature (DSPy, AutoPrompt, Meta-Rewarding LMs) assumes paid per-token calls to the frontier model. This creates two problems: (a) unpredictable variable cost at scale, and (b) difficulty justifying the investment before proving the return. This project solves the problem by swapping the channel: the frontier model (Claude Opus 8.6) enters the loop via **Claude Code running locally under a fixed monthly subscription**, not via API. The operator (Galvani) runs interactive refinement sessions — Opus reads audit bundles and prompts, proposes edits, implements them directly in repo files, and the operator approves inline. Zero variable cost. Zero surprise token caps. This turns meta-prompting from "expensive capability only large companies can run" into "weekly routine for a solo developer with a $200/month subscription." **Why novel:** I have not found literature or public projects explicitly describing Claude Code (or Cursor, or any IDE-agent) as the primary meta-prompting channel for a production product. The conventional pattern is still "scheduled workflow calling the frontier API."

**2. Pre-production validation pipeline as architectural prerequisite.** Most chatbot projects launch with an incomplete bot and iterate with real clients — "ship fast, learn fast." For regulated, high-value-per-client contexts (law, healthcare, finance), that approach is a trap: a single mistreated client costs more than weeks of development. This project inverts the order: **seven concrete, measurable criteria form a Launch Gate that must be 100% met before any real client touches the bot**. The criteria are met through a portfolio of offline refinement cycles — synthetic scenario generation, chatbot-versus-chatbot simulation, internal human testers playing client roles, golden-dataset regression, adversarial red-team battery, and formal approval from the supervising attorney. **Why novel:** most chatbot methodologies treat validation as "a testing phase before deploy." Here validation is **the dominant development phase** — the bot spends most of its lifecycle in offline refinement before seeing its first real client. For regulated domains this is the correct order; it is only uncommon because "ship fast" is the cultural default of almost every software project.

**3. Single-progression incremental development (no MVP, no phases).** The traditional MVP → v2 → v3 model is efficient when stakes are low and the feedback loop is fast. For this project the model is explicitly rejected: instead, development happens as a **single, ordered, continuous progression** where each step adds capability without prematurely declaring the system "ready." At any point in construction, the chatbot is genuinely functional at its current scope — never "MVP with missing features." The difference versus "normal agile development" is the explicit absence of intermediate releases labeled "MVP" or "v1" or "launch." There is one launch, when the Launch Gate passes. **Why novel:** this choice is philosophically opposite to the dominant doctrine ("ship the MVP"). It fits this project because the ROI is binary — either the bot serves real clients well, or it does not. There is no "partial value" in exposing 60% of the bot to clients. Documenting the choice as an explicit stance contributes to the methodological debate about when the MVP doctrine is inadequate.

**4. Visual-first no-code as a deliberate constraint.** Most projects that use n8n eventually resort to Code nodes as complexity grows. This project adopts a **deliberate constraint**: Code nodes (in JavaScript, never Python) are a last resort, allowed only when no native or community node solves the need. The rationale is not ideological — it is operational: predominantly visual workflows are easier to audit, review, and transfer to new clients, because a lawyer who owns the firm can open the editor and understand visually what is happening. The constraint forces interesting architectural decisions: instead of writing a custom regex parser in a Code node, use Switch + IF + Set nodes with inline regex; instead of building custom scoring in JavaScript, use a chain of Set nodes with n8n expressions; instead of implementing a subroutine, create a reusable sub-workflow. **Why novel:** the n8n community tends to treat Code nodes as "the easy way out when the GUI falls short." Treating them as "technical debt requiring justification" is a discipline few projects apply explicitly. This translates into replicability — each new client importing the template can understand and adjust the workflow visually without needing a developer for every change.

**5. Template export plus hyper-local tuning as the product model.** The classic tension in productizing AI is: (a) one-size-fits-all SaaS (easy to scale, generic, mediocre performance), or (b) bespoke consulting (high performance, does not scale, low margin). This project bets on an explicit middle ground: **a functional template exported as an artifact plus a documented per-client hyper-local tuning process**. The template (the n8n workflow JSON) ensures 80% of the technical work is done. The tuning process (firm-specific prompts per practice area, regional language, historical caseload) ensures the final 20% that turns "generic bot" into "bot that understands social security in Maranhão." The case study documents exactly how the tuning is done, making the process replicable by trained third parties (future partners or employees). **Why novel:** most attempts to productize AI for regulated niches fail by choosing one extreme. The "template plus documented tuning" approach is recognized in other verticals (marketing agencies, traditional legal consulting) but still rare for AI products. This project is an explicit experiment applying it to legaltech.

### Market Context & Competitive Landscape

**Direct competitors in Brazil:**

- **Sabio Adv** — WhatsApp Business API authorized for law firms. Closed SaaS. No self-improvement, no self-hosting, no firm-level customization. Client data on the vendor's servers.
- **ChatADV** — GPT-5-based legal AI with WhatsApp and web support. Generic national product, no per-firm specialization, no n8n integration, no continuous refinement cycle.
- **WhatsBot GPT / Intelizap** — low-cost WhatsApp chatbot builders (~R$80/month). Template-driven, shallow legal knowledge, no RAG, session-only memory, no sophisticated lead scoring.

**International competitors:**

- **LawDroid / Smith.ai / Intaker (US)** — specialized legal intake with AI lead scoring. English only, no WhatsApp, high SaaS pricing, no self-hosted option.
- **Perspective AI / Lawmatics QualifyAI (US)** — conversational AI replacing web forms, 2.5x conversion lift. Web chat only, no WhatsApp/Telegram, no Brazilian legal knowledge, closed source.

**The gap this project fills.** No competitor simultaneously offers:
1. Self-improvement engine operated by a human via subscription frontier model
2. Self-hosting with data exclusively on-premise (professional privilege)
3. Customization at the level of practice area and regional context
4. Rigorous pre-production validation as architectural prerequisite
5. Exportable template plus documented tuning process

The combination of these five is what characterizes the project's competitive position. Each item individually exists in some competitor; the combination does not exist in any.

**Temporal window:**
- OAB Recommendation 001/2024 and CNJ Resolution 615/2025 are advisory, not prohibitive — a permissive adoption window before stricter regulation arrives (likely 2027-2028)
- 2026 conversational models of 7-12B parameters match the frontier quality of 2023-2024 at 10-50x lower cost — making the two-tier architecture economically viable
- Meta-prompting frameworks matured from paper to production in 2025-2026 — Claude Code made operating them viable without custom engineering
- No Brazilian competitor actively explores self-improvement or self-hosting — the moat opens now and likely closes in 18-24 months as larger players enter the space

### Validation Approach

Since the innovation here is methodological, validation is not "does the model work?" but "does the methodology deliver the promised result?" Each innovation area has a specific validation criterion:

**Innovation 1 (Meta-prompting via Claude Code):** Criterion — the model + human refinement cycle is successfully executed at least 3 times during the refinement phase, each producing measurable improvement in a golden-dataset metric. Evidence — entries in `refinement-log/` with prompt diffs, before/after metrics, time invested. Counter-proof — if after 3 cycles there is no cumulative improvement, the methodology is not delivering what was promised and needs revision.

**Innovation 2 (Pre-production pipeline):** Criterion — the bot passes the Launch Gate without any real client being exposed during development. Evidence — production bot logs showing first real-client interaction only after the Launch Gate date; zero real conversations in pre-launch audit bundles. Counter-proof — if any real client appears in logs before the Launch Gate, the discipline failed.

**Innovation 3 (Single-progression):** Criterion — project documentation never contains the terms "MVP," "Phase 1," "v1," "v2," "partial launch"; progression is described as a single sequence. Evidence — searching the repo for these terms returns zero (except in mentions of this section explaining the rejected concept). Counter-proof — if phase language reappears, the methodological discipline failed.

**Innovation 4 (Visual-first no-code):** Criterion — at the end of construction, the ratio of visual nodes to Code nodes in the exported workflow is ≥ 90%. Every Code node has documented justification in the refinement log or as a comment on the node itself. Evidence — analysis of the exported workflow JSON counting nodes by type. Counter-proof — if the ratio drops below 90% or if undocumented Code nodes appear, the discipline failed.

**Innovation 5 (Template + tuning):** Criterion — a second deployment at a different firm (dry run in a separate test environment) reproduces the bot in ≤ 8 hours of customization work, with quality comparable to the Maruzza deployment. Evidence — documented record of the dry run with time, issues found, final quality score. Counter-proof — if the dry run takes much more than 8 hours or requires structural template changes, the "template + tuning" model is not delivering real replicability and needs refinement.

### Risk Mitigation

**R-INNOV-1 — Claude subscription policy changes and limits meta-prompting use.** Anthropic could restrict the type of use the subscription covers, or introduce caps that make the cycle infeasible. Mitigation: (a) log typical usage to demonstrate patterns if questioned; (b) contingency plan to migrate to another IDE-agent provider (Cursor, Cline, Zed Agent) with an equivalent fixed paid model; (c) if necessary, accept controlled variable API cost as a last resort.

**R-INNOV-2 — Launch Gate proves unreachable.** One or more Launch Gate criteria may be so strict that the project never crosses. Mitigation: if that happens, revise the criteria before relaxing the discipline — perhaps the threshold was wrong, not the principle. Relaxing a threshold is acceptable; eliminating a criterion is not. If the conclusion is that the gate is impossible, the product is not ready — accepting more time in refinement is the cost.

**R-INNOV-3 — Single-progression generates external stakeholder resistance.** Investors, partners, or even Maruzza may ask for "a preliminary version to show progress." The temptation to give in is real. Mitigation: document this section as an explicit stance, and when pressure comes, reference the principle and the risks of "prematurely declaring victory" in the legal domain.

**R-INNOV-4 — Visual-first discipline breaks under deadline pressure.** Under time pressure it may be tempting to write quick Code nodes instead of investigating whether a native node solves the problem. Mitigation: every Code node introduction goes through explicit review by Galvani himself; if there is no time for the review, the change is not made.

**R-INNOV-5 — Template + tuning does not scale to trained partners.** The "replicable template" vision depends on other developers/consultants being able to execute the tuning with quality. If the process requires expertise only Galvani has, replicability is illusory. Mitigation: the case study plus refinement log plus process documentation must be explicitly tested by an external developer (Innovation 5 validation criterion) before the project is considered "replicable." Failures in the external test indicate the documentation needs to improve, not that the principle is wrong.

## Technical Architecture — n8n Workflow Blueprint

This section describes the macro architecture of the n8n workflow that will be built manually in the visual editor. It is not an exhaustive node-by-node specification (that comes in epics and stories); it is the territory map that guides construction and ensures every component has a clear place in the overall topology.

### Infrastructure Topology

The system has three infrastructure domains, each with clear responsibilities:

**Domain 1 — BorgStack-mini (orchestration and storage).** Location: `10.10.10.205`, Docker Swarm single-node. Responsibilities: n8n (queue mode: editor + webhook + worker) runs the workflow; PostgreSQL + pgvector holds chat memory, client profiles, analytics, audit logs; Redis handles rate limiting, cache, counters; Caddy is the reverse proxy with TLS termination; Cloudflare tunnel exposes the webhook securely without a public IP; Evolution API bridges WhatsApp. **BorgStack does not run model inference.** No LLM, no Whisper, no local TTS engine. It is exclusively orchestration and persistence.

**Domain 2 — Comadre (dedicated TTS).** Location: `10.10.10.207`. Dedicated TTS server with optimized hardware. Exposes an OpenAI-compatible endpoint at `/v1/audio/speech`. Default voice `kokoro/pm_santa`. Accessed over the internal network from BorgStack.

**Domain 3 — External cloud services.** Groq (`api.groq.com/openai/v1`) — operational LLM (7-12B multilingual model) plus Whisper Large v3 Turbo for STT. Optional fallback — OpenRouter, Mistral, or another OpenAI-compatible provider, accessed via HTTP Request node with configurable base URL.

Communication between domains is via HTTP. BorgStack talks to Comadre and Groq; Comadre and Groq do not talk to each other.

### Main Workflow Architecture

The main chatbot workflow is organized into **logical layers**. Each layer has a clear responsibility, uses specific nodes, and can be tested in isolation.

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 0 — MESSAGING CLIENT                                     │
│  Telegram (client speaks to the bot) / WhatsApp via Evolution   │
│  Text, voice, documents, callback queries                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1 — INGESTION (Telegram Trigger / Webhook + Normalizer)  │
│  Telegram Trigger node OR Webhook node (for Evolution API)      │
│  Switch node routes by platform                                 │
│  Payload normalization to a unified schema                      │
│  Session key derived from chat.id (Telegram) or phone (WhatsApp)│
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 2 — MEDIA PROCESSING (sub-workflow)                      │
│  Voice → download OGG → HTTP Request to Groq Whisper → text     │
│  Flag inputWasAudio=true so response mirrors modality           │
│  Text → Set node normalizes (HTML strip, whitespace, unicode)   │
│  Image/document → flag for human review (v1 does not process    │
│    attachment content, goes straight to handoff)                │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 3 — SECURITY PRE-CHECK                                   │
│  Redis node: rate limiting counter per user_id + IF node        │
│  Input sanitization (Set node with inline regex expressions)    │
│  Guardrails node (input mode): jailbreak, topical alignment,    │
│    PII detection                                                │
│  Rejections become friendly messages without calling LLM        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 4 — DETERMINISTIC ROUTER (no LLM, visual nodes)          │
│  Switch + IF nodes in cascade:                                  │
│    /start, /help, /menu → Set node with static response         │
│    greetings (regex in expression) → welcome message            │
│    FAQ patterns (Switch) → canned Postgres lookup or Set node   │
│    thanks/bye → polite closure                                  │
│    ANYTHING ELSE → falls through to Layer 5                     │
│  Target: resolve 15-25% of messages here without LLM            │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 5 — TRIAGE AGENT (AI Agent node)                         │
│  Model: Groq 7-12B multilingual (HTTP Request node to           │
│    OpenAI-compatible endpoint)                                  │
│  Chat Model sub-node connected to the AI Agent                  │
│  Memory: Postgres Chat Memory sub-node (window 10-15, session   │
│    key = chat.id/phone)                                         │
│  System Prompt: loaded from repo markdown file (source of truth │
│    in Git, manually copy-pasted into the node field)            │
│  Tools (sub-workflows via Call n8n Workflow Tool):              │
│    • save_client_info — Postgres Insert/Update in               │
│      client_profiles                                            │
│    • lookup_practice_areas — static data lookup (Set or         │
│      Postgres)                                                  │
│    • qualify_lead — chain of Set nodes with scoring expressions │
│    • request_human_handoff — triggers HITL sub-workflow         │
│  Output Guardrails node: PII check, no-legal-advice patterns    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                       ┌────┴────┐
                       ▼         ▼
          ┌─────────────────┐  ┌───────────────────────────────┐
          │ NORMAL REPLY    │  │ QUALIFIED LEAD → HANDOFF      │
          │ (layer 6)       │  │ (sub-workflow)                │
          └────────┬────────┘  └──────────────┬────────────────┘
                   │                          │
                   ▼                          ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 6 — RESPONSE DELIVERY                                    │
│  IF node: inputWasAudio?                                        │
│    YES → Send text via Telegram/Evolution Action node           │
│          → HTTP Request to Comadre TTS → Send Voice             │
│          → Continue on Fail: if Comadre fails, text only        │
│    NO  → Send text via Telegram/Evolution Action node           │
└─────────────────────────────────────────────────────────────────┘
```

Each layer is a set of nodes wired in sequence inside the main workflow. Sub-workflows are invoked via **Call n8n Workflow Tool** nodes. The main workflow is the "conductor"; sub-workflows are specialized, reusable pieces.

### Sub-Workflows (built separately, invoked by the main workflow)

**SW-1: Media Processor** — downloads audio from Telegram/Evolution, sends it to Groq Whisper, returns text. Also handles documents and images (in v1, only flags for human handoff).

**SW-2: Save Client Info** — receives data extracted by the agent, inserts or updates records in `client_profiles` via Postgres node.

**SW-3: Lead Qualification** — receives conversation context, applies deterministic scoring via Set nodes with expressions, returns score + qualified/not flag.

**SW-4: Handoff Flow** — composes the lead package, sends a formatted notification to the attorney's Telegram group with inline keyboard, puts the workflow on a Wait node with 4h timeout, processes the button response or timeout, returns the decision.

**SW-5: Follow-Up Relay** — when the attorney taps "Need More Info" and specifies a question, this sub-workflow bridges back to the client, collects the answer, and updates the attorney's card.

**SW-6: Error Handler** — dedicated workflow triggered by the Error Trigger node, captures exceptions, classifies severity, logs to `chat_analytics`, sends critical alerts to the admin.

**SW-7: Analytics Logger** — receives events from the main workflow (messages received, responses sent, handoffs) and persists them consistently and auditable to `chat_analytics`.

**SW-8: Weekly Report Generator** — Schedule Trigger weekly. Aggregates the week's metrics, formats the report, sends it to the admin via Telegram or email.

**SW-9: Audit Bundle Exporter** — run on demand by the operator. Queries `chat_analytics` with filters (flagged cases, sampling), generates structured markdown in `audit-bundles/`, notifies that it is ready for analysis in Claude Code.

### Node Selection Principles

1. **Native and community nodes first.** Before creating a Code node, check whether a native n8n 2.x or community node solves the problem. Examples known to have native or community coverage: Telegram, WhatsApp via Evolution API, Postgres, Redis, OpenAI-compatible (via HTTP Request), Guardrails, AI Agent, Chat Memory variants, Text Classifier, Switch, IF, Set, Code, Wait, Schedule, Error Trigger, Call n8n Workflow Tool, Extract From File, Convert to File, HTTP Request.

2. **JavaScript in Code nodes only when necessary.** Every Code node in the final workflow must have a documented justification: why no native node or visual combination (Switch + Set + IF with expressions) solved it. The justification goes in a comment on the node itself or in `refinement-log/`.

3. **Sub-workflows over cascaded nodes.** When a sequence of 5+ nodes forms a reusable logical unit, extract it into a sub-workflow invoked via Call n8n Workflow Tool. It improves main-workflow readability and enables isolated testing.

4. **n8n expressions instead of Code nodes.** For simple transformations, the `{{ $json.field }}` syntax plus built-in functions (`.toLowerCase()`, `.trim()`, `.replace(/regex/, '')`, `.substring()`, etc.) covers most cases without opening a Code node.

5. **Credentials management.** All API keys and secrets live in the n8n Credentials system (encrypted with `N8N_ENCRYPTION_KEY`), never hardcoded in workflows. Credentials are exported separately from the workflow JSON — when importing the template for a new client, credentials must be recreated in the client's instance.

### External Integrations

| Integration | Method | Recommended node | Notes |
|---|---|---|---|
| **Telegram** | Auto-registered webhook | Telegram Trigger + Telegram Action | One bot per active workflow; separate test bot in refinement phase |
| **WhatsApp via Evolution API** | Manual webhook | Webhook node + HTTP Request (or community `n8n-nodes-evolution-api`) | Normalize payload to unified schema; evaluate community node for code savings |
| **Groq LLM (chat)** | HTTP Request with OpenAI-compatible endpoint | HTTP Request node with base URL `https://api.groq.com/openai/v1` | Failover configured via retry + fallback credentials (free → paid) |
| **Groq Whisper (STT)** | HTTP Request with OpenAI-compatible endpoint | HTTP Request node for `/audio/transcriptions` | Multipart binary upload |
| **Comadre (TTS)** | HTTP Request with OpenAI-compatible endpoint | HTTP Request node for `http://10.10.10.207:8000/v1/audio/speech` | Continue on Fail = true; fallback to text-only if Comadre is down |
| **PostgreSQL** | Direct connection | Postgres node (built-in) | All tables in the same schema; `chat_analytics`, `client_profiles`, `n8n_chat_histories` (auto-created by the memory node), `test_scenarios` (staging) |
| **Redis** | Direct connection | Redis node (built-in) | Rate-limit counters + static response cache |

### Replicability Considerations (Template Export)

The workflow is built with replicability in mind from day one. Architectural decisions that facilitate replication to new clients:

1. **Firm-specific data externalized.** Nothing hardcoded about Maruzza Teixeira in the workflow itself. Firm names, practice areas, TTS voice, specific prompts — everything comes from environment variables, credentials, or prompt files loaded at the start of execution. When importing the template into another instance, these references change value but the topology does not.

2. **Standard database schema.** The `chat_analytics`, `client_profiles`, and `test_scenarios` tables have a fixed, documented schema. A new client runs the setup SQL (in `sql/001-init-tables.sql` or equivalent) on their Postgres instance and is ready to use the imported workflow.

3. **Credentials as contract.** The imported workflow expects credentials with specific names (`groq_api_key`, `comadre_endpoint`, `telegram_bot_token`, `postgres_main`, etc.). The full list of required credentials is in the deployment guide as a checklist.

4. **Prompts versioned in the repo.** Prompt markdown files live in the repository and are copy-pasted manually into n8n nodes during per-client configuration. Over time each client can have variations, but the base structure comes from the template.

5. **Abstracted model configuration.** The LLM HTTP Request node points to an endpoint configurable via variable. Swapping base URL and model name in credentials migrates between providers (Groq free → Groq paid → another provider) without touching the workflow.

### Implementation Considerations

**Recommended construction order.** Build the workflow in layers, not node by node. Start with the simplest possible ingestion layer (text only, Telegram only, direct echo), verify it works, then add layers: media processing → security → router → agent with memory → tools → handoff → guardrails → error handling → analytics. Each addition leaves the system functional at its current scope.

**Test bot vs. production bot.** During the Refinement Phase, a separate Telegram bot exists (different token, database schema prefixed with `staging_` or a separate database) where all internal human tests and simulations run. The production bot is only activated when the Launch Gate passes. Both can coexist on the same BorgStack with duplicated workflows.

**Pin Data during development.** n8n lets you "pin" node outputs during tests, freezing input data to reproduce bugs and test transformations without depending on real messages. Use it extensively during construction.

**Execution history as a debug tool.** n8n records every workflow execution with per-node input/output. Configure appropriate retention (default 168h is enough for debug; for production audit, export to `chat_analytics` instead of relying only on n8n's native history).

**Webhook test vs. production URLs.** n8n distinguishes between test webhook URLs (active only while the editor is "listening") and production URLs (active when the workflow is published). Use test URLs during construction, switch to production only after the Launch Gate.

**Workflow version control.** At every significant milestone, export the workflow as JSON and commit it to the repo under `exports/workflow-snapshots/YYYY-MM-DD-description.json`. This provides evolution history and a rollback point if something breaks after a change.

## Project Scope

### Scope Philosophy

This project rejects the "MVP → v2 → v3" model. There is no reduced version delivered first with features added later. The product is **a single complete system** built through an incremental progression of ordered steps. Each step adds capability to the previous one without prematurely declaring the system "ready." There is only one launch: when the Launch Gate passes.

This is a deliberate choice justified by context: in the Brazilian legal domain, a mistreated client costs more than any development-time savings could compensate. "Ship fast and learn" is incompatible with high-value legal practice. "Build completely and launch once" is the correct discipline.

The scope described below is therefore **the full scope of the v1.0 template** — not an initial portion of it.

### In Scope (v1.0 template)

**Communication channels:**
- Telegram (primary channel during refinement and in production — native Telegram Trigger + Telegram Action)
- WhatsApp via Evolution API (primary channel in final production, with payload normalization to a unified schema shared with Telegram)

**Conversation modalities:**
- Bidirectional text (send and receive)
- Bidirectional voice — OGG reception via Telegram/Evolution, STT via Groq Whisper Large v3 Turbo, default text response, audio response via Comadre (`kokoro/pm_santa`) when input was voice
- Modality mirroring: if the user sends voice, the bot responds in text AND voice; if they send text, only text
- Document and image detection with fallback to human handoff (bot in v1 does not process attachment content)

**Triage agent capabilities:**
- Classification of Maruzza's 8 practice areas (social security, labor, banking, health, family, real estate, corporate, tax)
- Structured extraction of essential client data (full name, city, case summary, urgency)
- Deterministic qualification scoring
- Static library of pre-approved responses per area for unqualified leads
- Automatic OAB disclosure in the first turn
- LGPD consent in the first turn
- Zero autonomous legal-advice generation (enforced in 6 layers)
- Returning-client recognition with profile injected into context

**Memory and persistence:**
- Postgres Chat Memory (10-15 message window, session key per chat_id/phone)
- `client_profiles` table with returning-client lookup
- `chat_analytics` table for forensic logging and metrics
- Separate `test_scenarios` table for the refinement phase

**Attorney handoff:**
- Lead package composition with data + full transcript
- Formatted notification to the firm's private Telegram group
- Inline keyboard with Accept / Need More Info / Decline
- Wait node with 4h timeout
- Follow-up relay branch (attorney question → client → response → updated card)
- Graceful decline branch with fallback to alternative resources
- Timeout branch with backup-attorney escalation and rescheduling
- Conversation state machine with logged transitions

**Security and compliance:**
- Rate limiting via Redis (counters per user_id with TTL)
- Input sanitization (HTML strip, whitespace, unicode, truncation)
- Input Guardrails node (jailbreak detection, topical alignment, PII detection)
- Output Guardrails node (PII check, no-legal-advice patterns)
- Strict session isolation (session_id never shared)
- Encryption at rest (PostgreSQL with N8N_ENCRYPTION_KEY)
- Encryption in transit (TLS for external APIs)
- Full audit trail (per message, per handoff, per refinement, per incident)
- Documented and implemented data retention policy
- Right to erasure via dedicated workflow

**Resilience:**
- Retry on Fail configured on all external API nodes with exponential backoff
- LLM failover (Groq free → Groq paid via retry with fallback credentials)
- Continue on Fail for Comadre (graceful degradation to text-only)
- Continue on Fail for Groq Whisper (fallback: ask client to type)
- Dedicated Error Trigger workflow with severity-based routing
- Friendly messages in all failure conditions — never silence, never crash

**Pre-production refinement pipeline (offline):**
- Separate test bot (different Telegram token, separate database schema)
- Golden dataset versioned in the repo (`golden-dataset/`)
- Synthetic scenarios generated in Claude Code sessions (`synthetic-scenarios/`)
- Monitored chatbot-versus-chatbot simulation (dedicated n8n test workflow)
- Internal human testing process (briefings, feedback collection, documentation)
- Adversarial red-team battery (`red-team-battery/` in the repo)
- Formal attorney review process (checklist + samples)
- Audit bundle exporter (dedicated sub-workflow)
- Structured refinement log (`refinement-log/`)
- Prompts versioned as markdown files in the repo as source of truth (manual copy into the n8n editor)

**Monitoring and operations:**
- Analytics query interface (Metabase or direct SQL)
- Weekly aggregated report workflow (Schedule Trigger + Postgres aggregations + delivery via Telegram)
- Automatic alerts in the Error Handler workflow
- Cost tracking workflow (token consumption logging per call, monthly aggregation)

**Template export and deployment:**
- Workflow exported as JSON (`exports/workflow-snapshots/`) at significant milestones
- Final export as `exports/n8n-workflow-template-v1.json` when Milestone 3 is reached
- Step-by-step deployment guide (`docs/deployment-guide.md`)
- Checklist of required credentials
- SQL schema for database initialization (`sql/001-init-tables.sql`)
- Case study documented in the repository itself

### Out of Scope (explicitly rejected for the v1.0 template)

Each item below has been considered and rejected with documented reason. "Out of scope for v1.0" does not mean "never," it means "does not belong to this delivery."

**Multi-tenancy in a single n8n instance.** Rejected because: each template client runs its own n8n instance on its own infrastructure, with its own data and credentials, professional privilege isolated by design. Multi-tenant inside a shared instance breaks the attorney-client privilege model. Alternative adopted: one deployment per firm, isolation via separate infrastructure.

**RAG over a legal knowledge base.** Rejected because: the bot's conversational scope (triage + qualification) does not require semantic retrieval of legal documents. System prompt + static area lookup + pre-approved response library cover every need identified in the journeys. pgvector is available in BorgStack but unused in the v1.0 template. RAG can be added post-launch if a concrete case justifies it (and the addition would become a new step in the progression, not a "future phase").

**Appointment scheduling.** Rejected because: the bot's scope is pre-intake (triage + handoff), not calendar management. Scheduling is the attorney's responsibility after accepted handoff. Adding scheduling to v1.0 expands scope without proportional triage-value gain.

**Document collection and validation during conversation.** Rejected because: document collection is part of the post-engagement lifecycle, not triage. The bot detects that the client sent a document, flags for human handoff, and the attorney takes it from there. Validating documents (OCR + analysis) is complexity that does not belong in the first delivery.

**Integration with judicial systems (PJe, eproc, ProJudi, courts).** Rejected because: the bot is pre-intake. Integration with judicial systems presumes a case already accepted and in progress — out of scope.

**Jurisprudence RAG (STJ, STF, TJs).** Rejected because: the bot does not generate legal opinion, does not discuss case merits, does not reference specific decisions. Jurisprudence RAG would only make sense if the bot were generating legal content — which is explicitly forbidden.

**Specialized sub-agents per practice area.** Rejected because: 8 areas handled by a single well-prompted agent is simpler and sufficient. Sub-agents would add routing complexity, shared memory, and debugging cost without measurable quality improvement. If a specific sub-agent proves necessary later (say, social security with much larger volume than the others), it enters as a later step in the progression, not as part of v1.0.

**Full self-improvement loop automation.** Rejected because: the refinement cycle is intentionally operator-paced (Galvani decides when to run it). Automating the loop — cron that calls a frontier API weekly and auto-promotes prompts — would violate (a) the principle of no per-token spend on the frontier, (b) the human-in-the-loop requirement in the legal context, (c) the discipline of every change being consciously reviewed.

**Automatic git sync between the repo and BorgStack for prompts.** Rejected because: manual copy-paste of prompt markdown files into the n8n editor is intentional. Zero ceremony, zero silent drift, every change is an explicit decision. Automating this sync adds infrastructure (webhook, git pull, hot reload) that only solves a problem we do not have.

**Shadow production / dual-mode testing.** Rejected because: implementation complexity disproportionate to the gain, and the offline refinement portfolio already covers validation without exposing real clients. The Launch Gate is the only gate; there is no "shadow mode" between refinement and production.

**Commercialization / pricing / SaaS model.** Rejected because: entirely separate question from technical construction. The commercial model (per-deployment consulting, template licensing, case percentage, etc.) is a business decision to be made after Milestone 3. The technical PRD does not need it.

**Custom visual dashboard.** Rejected because: Metabase already solves metrics visualization and is available on BorgStack in cluster mode. The weekly report workflow via Telegram covers the main operational use case. Building a custom dashboard is reinventing the wheel.

**Web chat widget.** Rejected because: the primary channel is WhatsApp (and Telegram in refinement). Web chat would add a third channel with its own UX, authentication, and maintenance layer — with no evidence of actual demand. If it emerges in the future, it enters as a later step.

**Multi-language support beyond Brazilian Portuguese.** Rejected because: the product is specifically for Brazilian firms serving Brazilian clients. Internationalization is complexity out of purpose. The template can be tuned for other Portuguese dialects (Portugal, Angola) by firms in other countries if interest arises, but this is not in v1.0 scope.

### Progression Sequence

The v1.0 template is built through an ordered sequence of increments. Each item leaves the system functional at its current scope. The sequence below is indicative and can be refined at the epics and stories stage; the principle is **never break what is already working** and **never prematurely declare readiness**.

1. **Infrastructure setup** — n8n in queue mode on BorgStack verified, SQL tables created, credentials registered (Groq free, Telegram test bot, Postgres, Redis, Comadre), Telegram bot registered with BotFather, n8n webhook responding to `/start` with "hello."

2. **Minimal workflow: echo bot** — Telegram Trigger → Set node → Telegram Action, echoing the user's message. Validates that the channel works.

3. **Basic LLM** — replace the Set echo with an AI Agent node connected to Groq (HTTP Request sub-node), with a minimalist system prompt. Conversation works but without memory, without persona.

4. **Short-term memory** — attach Postgres Chat Memory to the AI Agent, session key per chat_id. The bot remembers the conversation.

5. **Normalization and sanitization** — Set node before the LLM with inline regex expressions to clean input. Basic defense against garbage.

6. **Deterministic router** — Switch + IF before the LLM for greetings, FAQs, commands. 15-25% of messages resolved without LLM.

7. **Structured Maruzza system prompt** — identity, rules, practice areas, examples, OAB disclosure. Loaded from a repo markdown file, copy-pasted manually into the node.

8. **`client_profiles` table + `save_client_info` tool** — sub-workflow invoked by the AI Agent extracts and persists client data.

9. **`lookup_practice_areas` tool** — sub-workflow with static data for the 8 areas.

10. **`qualify_lead` tool** — sub-workflow with deterministic scoring (chain of Set nodes with expressions).

11. **`request_human_handoff` tool** — full HITL sub-workflow with lead package composition, notification to the private Telegram group, inline buttons, Wait node with timeout, Accept/Need More Info/Decline/Timeout branches.

12. **Input guardrails** — Guardrails node with jailbreak detection and topical alignment before the LLM.

13. **Rate limiting via Redis** — counter per user_id with TTL, IF node blocking excess.

14. **Output guardrails** — Guardrails node with PII check and no-legal-advice patterns after the LLM.

15. **Dedicated error workflow** — Error Trigger + severity-based routing + Telegram alerts.

16. **Analytics logging** — sub-workflow called at all relevant points, writing to `chat_analytics`.

17. **Context compaction** — Chat Memory Manager checking memory size and summarizing older messages when a threshold is crossed.

18. **Voice STT** — Media Processor sub-workflow with OGG download and Groq Whisper call.

19. **Voice TTS via Comadre** — conditional `inputWasAudio` branch invoking Comadre and sending Send Voice; Continue on Fail for graceful degradation.

20. **Static pre-approved response library** — `static_responses` table or markdown files, consulted by the bot on the unqualified-lead flow.

21. **WhatsApp adaptation via Evolution API** — second trigger (Webhook receiving MESSAGES_UPSERT), Set/Code node normalizing to unified schema, final Switch node routing response to the correct channel.

22. **Initial golden dataset and Cycle 5 (regression)** — versioned dataset in the repo, n8n workflow or Claude Code session running the current prompt against it and reporting metrics.

23. **Adversarial red-team battery and Cycle 6** — hostile input set, execution against the bot, 100% pass validation.

24. **Cycle 3 (internal human tests)** — separate test bot, scenario briefings, structured feedback collection, first 50+ human-test conversations executed.

25. **Cycle 2 (chatbot-versus-chatbot simulation)** — test workflow running synthetic personas against the triage bot, failure monitoring.

26. **Audit bundle exporter (SW-9)** — operator-triggered workflow exporting conversation samples to markdown in the repo.

27. **First 3 model + human refinement cycles (Cycle 4)** — Claude Code sessions analyzing bundles, proposing changes, editing prompts, running regression, copy-pasting back into the editor.

28. **Weekly report workflow** — Schedule Trigger aggregating the week's metrics and delivering a formatted report to the admin.

29. **Formal Maruzza review (Cycle 7)** — documented process, checklist, formal approval signature.

30. **Launch Gate validation** — verification of all 7 criteria simultaneously, result documented.

31. **Switch to production** — switch from test bot to production bot (new token, production database, real WhatsApp pointing to the webhook).

32. **Steady-state operation + continuous refinement with real logs** — Milestone 2 begins here.

33. **Export workflow as v1.0 template** — Milestone 3.

34. **Final case study documentation** — Milestone 4.

The sequence can (and will) be adjusted as construction progresses. The principle is what matters: each step leaves the system functional, never MVP, never "I will come back to fix it later."

### Risk-Based Scoping Analysis

**Scope risk 1 — scope too large for the single-progression discipline.** The scope described includes ~34 steps. If any of them reveals unexpected complexity, progression may slow down and create pressure to "cut corners." Mitigation: the sequence is ordered in **growing value plus dependency** order, so even stopping at an intermediate step (if necessary) would leave the system functional — just with less capability. No step breaks the previous one.

**Scope risk 2 — underestimating refinement effort.** Steps 22-30 (offline pre-launch refinement) are the most unpredictable. They may take weeks or months depending on how many cycles are needed until the Launch Gate passes. Mitigation: accept that refinement is the largest part of the project, by design. "How long does it take" is not the right question — "how many cycles until the 7 criteria pass" is.

**Scope risk 3 — features "forgotten" in scoping re-emerge during construction.** While building the bot you may realize you need something not listed above. Mitigation: accept disciplined re-scoping — any new capability enters as a step in the progression sequence (not as an "extra feature"), with documented justification, and respecting the principle of not breaking what already works.

**Scope risk 4 — the "out of scope" list is questioned by external stakeholders.** Maruzza may ask "what about scheduling?" or "what about automatic billing?" after seeing the bot work. Mitigation: the Out of Scope section above is deliberately explicit with documented reasons for each rejection. When the question comes, the answer is "considered, rejected for reason X, can enter as a future step if evidence justifies."

**Scope risk 5 — template + tuning proves insufficient for real replicability.** If the first template dry run at a second firm takes more than 8 hours of tuning (the Innovation 5 validation criterion), the replicability promise is at risk. Mitigation: Milestone 3 (Template Export Ready) includes a dry run as a criterion — if it fails, refine the template before considering it ready, not rationalize the result afterward.

## Functional Requirements

The functional requirements below form the complete capability contract for the v1.0 template. Nothing exists in the final product outside this list. Every FR is numbered sequentially, testable, implementation-agnostic, and traceable back to the previous sections of this PRD.

### Conversational Intake

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

### Classification & Data Extraction

- **FR11**: The system classifies each conversation into one of the firm's 8 practice areas (social security, labor, banking, health, family, real estate, corporate, tax) or "undefined" when there is no clear signal.
- **FR12**: The system extracts the client's essential data in a structured way during the conversation: full name, city, case summary, and perceived urgency.
- **FR13**: The system actively requests the essential data when it has not yet been provided, in a sequence that feels natural in the conversation.
- **FR14**: The system confirms the collected data with the client before treating it as final.
- **FR15**: The system persists the extracted data in a client profile that can be retrieved in future interactions.

### Lead Qualification & Handoff

- **FR16**: The system scores each conversation according to deterministic criteria considering match with the firm's practice areas, data completeness, urgency, and geographic location.
- **FR17**: The system separates qualified leads from unqualified leads according to a configurable score threshold.
- **FR18**: For qualified leads, the system composes a handoff package containing structured client data plus the full chronological transcript of the conversation.
- **FR19**: The system delivers the handoff package to the supervising attorney through a notification in a private channel, formatted for quick reading on a mobile device.
- **FR20**: The attorney can accept, request more information, or decline the handoff through a quick-response interface in the private channel.
- **FR21**: When the attorney accepts the handoff, the system confirms to the client that human contact will occur shortly and updates the conversation state.
- **FR22**: When the attorney requests more information, the system returns to the client with the specific question, collects the answer, and updates the package delivered to the attorney.
- **FR23**: When the attorney declines the handoff, the system closes the conversation with the client in a warm manner and offers alternative resources appropriate to the area.
- **FR24**: The system applies a configurable timeout to the attorney response wait and escalates to an alternative path when the timeout expires.

### Non-Qualified Lead Handling

- **FR25**: For unqualified leads, the system closes the conversation in a warm and useful manner, without conveying a sense of rejection.
- **FR26**: The system offers the unqualified lead relevant alternative resources (Defensoria Pública, public agencies, professional associations) from a static content library pre-approved by the attorney.
- **FR27**: The system selects the set of offered resources based on the identified practice area, without generating new content.

### Compliance & Safety

- **FR28**: The system identifies itself as an AI virtual assistant in the first turn of every new conversation, meeting the regulatory disclosure requirement.
- **FR29**: The system presents LGPD consent information in the first turn, explaining controller, purpose, retention, and subject rights.
- **FR30**: The system never generates legal advice, outcome prediction, legislation interpretation, or concrete action recommendation, under any circumstance and regardless of user pressure.
- **FR31**: The system blocks outputs containing patterns indicative of legal advice and replaces them with a fallback message directing to human handoff.
- **FR32**: The system maintains a complete audit trail of every message, handoff, prompt refinement, and incident, with granularity sufficient for forensic reconstruction.
- **FR33**: The system applies a retention policy defined per data type, automatically removing data after the established period.
- **FR34**: The operator can execute complete deletion of data for a specific client from a unique identifier, fulfilling right-to-erasure requests.

### Security

- **FR35**: The system limits message frequency per client in a time window, responding with an informative message when the limit is exceeded.
- **FR36**: The system sanitizes all text input by removing HTML, invisible characters, and normalizing whitespace, before processing the message.
- **FR37**: The system detects and blocks prompt injection and jailbreak attempts, responding with a message redirecting to legal scope.
- **FR38**: The system detects and blocks off-topic messages (non-legal) before calls to the triage agent.
- **FR39**: The system detects personally identifiable information (PII) in input and output, applying a redaction or blocking policy as defined.
- **FR40**: The system maintains strict isolation between different clients' sessions, ensuring that content from one conversation never leaks to another.
- **FR41**: The system stores external service credentials encrypted, never in plain text in workflows or logs.

### Resilience

- **FR42**: The system automatically retries calls to external services that fail transiently, applying exponential backoff.
- **FR43**: The system automatically switches to fallback credentials or endpoints for the operational LLM when the primary provider returns rate limits or persistent failures.
- **FR44**: When the TTS service is unavailable, the system responds to the client in text only without interrupting the conversation.
- **FR45**: When the STT service is unavailable, the system informs the client that the voice message could not be processed and asks them to type.
- **FR46**: In any failure scenario, the system always sends the client a friendly message and next-step guidance — never silence, never crash.
- **FR47**: The system captures exceptions in a dedicated workflow, classifies by severity, and notifies the operator in critical cases.

### Refinement Pipeline (pre-production)

- **FR48**: The operator can maintain a complete test environment with a separate bot, separate database, and total isolation from the production environment.
- **FR49**: The operator can generate synthetic conversation scenarios for the firm's practice areas and regional context, stored in repo-versioned files.
- **FR50**: The operator can run chatbot-versus-chatbot simulations where a model plays diverse client personas against the triage bot, with results logged separately from production.
- **FR51**: Internal human testers can interact with the test bot receiving written persona briefings, and can record structured feedback on each conversation.
- **FR52**: The operator can maintain a versioned golden dataset of canonical conversations and run it against any prompt version, receiving comparative metrics.
- **FR53**: The operator can run an adversarial red-team battery against the bot and receive a pass/fail report per attack category.
- **FR54**: The operator can export audit bundles containing conversation samples (filtered by criteria) in versionable markdown format.
- **FR55**: The supervising attorney can review conversation samples and record formal approval or a list of blocking issues.
- **FR56**: The system prevents any real client traffic from reaching the production environment before the formal Launch Gate has been approved and documented.

### Operations & Monitoring

- **FR57**: The system records per-conversation metrics (latency, tokens consumed, model used, outcome) in analytics storage accessible for querying.
- **FR58**: The operator can query individual and aggregated conversation metrics through a query interface (dashboard or direct SQL).
- **FR59**: The system generates a periodic operational report with key metrics (conversation volume, handoffs, rates, errors, cost) and delivers it to the operator on a configurable channel.
- **FR60**: The system tracks external service consumption cost (LLM tokens, STT usage) and aggregates by period for budget control.
- **FR61**: The operator receives automatic notification when critical metrics exceed predefined thresholds (error rate, latency, anomalous volume).
- **FR62**: The attorney can mark production conversations as misclassified, feeding the next refinement cycle.

### Template Export & Replicability

- **FR63**: The operator can export the complete workflow as a single self-contained JSON artifact capturing the full topology of nodes and sub-workflows.
- **FR64**: The exported workflow can be imported into a new n8n instance and, after configuration of credentials and client-specific data, reproduce the original bot's behavior.
- **FR65**: The template includes or is accompanied by SQL initialization schema for all required tables (client profiles, chat analytics, test scenarios, static responses).
- **FR66**: The template is accompanied by a complete list of required credentials (names, types, and purpose of each) and a step-by-step configuration guide.
- **FR67**: The template is accompanied by base prompt markdown files that can be customized per deployment without touching the workflow topology.
- **FR68**: The operator can run a template deployment dry run in a separate test environment to validate that the template imports and works before deploying in client production.

### Case Study Documentation

- **FR69**: The repository records a structured log of every refinement cycle executed during the refinement phase, containing date, motivation, applied change, before/after metrics, and time invested.
- **FR70**: The repository contains the complete project journey documented so that an external developer can understand the architectural decisions, the validation criteria, and the evolution history, without depending on prior project knowledge.

## Non-Functional Requirements

The numerical thresholds for performance, cost, availability, and classification quality are already defined in Success Criteria > Measurable Outcomes and are not duplicated here. This section covers quality attributes that do not fit in a single metric: security architectural posture, integration reliability, workflow maintainability, observability, template portability, and cost as a constraint.

### Performance

- **NFR1**: The response time perceived by the client (message received → first response sent) must stay within the targets set in Measurable Outcomes (p95 ≤ 8s for text, p95 ≤ 14s for audio-to-audio), under normal load up to 100 conversations/day.
- **NFR2**: Deterministic routers (layer 4) resolve 15-25% of messages without any LLM call, measured by counting messages that never reach the AI Agent node.
- **NFR3**: The context consumed by the operational LLM per request is limited to the 10-15 message window plus the system prompt, avoiding quality degradation from context rot and controlling token cost.
- **NFR4**: The system prompt is cached by the LLM provider when supported (prompt caching), reducing input token cost on subsequent requests.
- **NFR5**: Responses to the client never block for more than 30 seconds total — if processing exceeds that limit, the system sends a fallback message ("one moment, I am checking") and continues processing in the background.

### Security

- **NFR6**: All conversation content is processed by the minimum necessary external services only: Groq (LLM + Whisper) and Comadre (TTS, internal network). No other external provider receives conversation data in production.
- **NFR7**: Before the Launch Gate, the Groq provider's retention and data-use policy must be formally validated: documented training-use opt-out, confirmation that inputs are not retained beyond processing, and written contingency should the policy change.
- **NFR8**: Session isolation is absolute: no session state of one client is accessible by another client, even under high concurrency or partial failure. The session key is derived deterministically from the client's unique channel identifier (Telegram chat_id, WhatsApp phone) with no collision possibility.
- **NFR9**: All credentials (API keys, tokens, database passwords) are stored exclusively in the n8n Credentials system with encryption key configured via the `N8N_ENCRYPTION_KEY` environment variable. No credential appears in plain text in workflows, logs, backups, or exports.
- **NFR10**: The audit trail is append-only and immune to post-write modification: records in `chat_analytics` are never overwritten or deleted except by the automatic retention policy, never by unaudited manual operations.
- **NFR11**: The right-to-erasure flow fully removes a specific client's data within at most 48 hours of the request, including profile, messages, handoff packages, and cross-references, and emits a confirmation log of the deletion.
- **NFR12**: The system resists 100% of adversarial red-team battery attacks before crossing the Launch Gate. Any pass rate below 100% is considered a failed gate.
- **NFR13**: AI disclosure and LGPD consent cannot be disabled by configuration, hotfix, or prompt refinement without an explicit, documented change to this PRD. They are architectural enforcement requirements.
- **NFR14**: Agent outputs are validated against legal-advice patterns before being sent to the client. Positive detection discards the original response and triggers a fallback directing to human handoff. This behavior is mandatory in production.

### Reliability & Availability

- **NFR15**: The system remains functional in degraded mode when any optional external service (Comadre, analytics dashboard, weekly report) is unavailable, preserving the ability to serve clients in text and record the audit trail.
- **NFR16**: Failures of the primary operational LLM (Groq free tier) trigger automatic failover to paid credentials of the same provider within at most 2 retries, without human intervention.
- **NFR17**: Logging failures (analytics insert, audit trail) never interrupt the main conversational flow. Lost logs are recorded as separate incidents for later reconciliation, but the conversation with the client continues.
- **NFR18**: Planned maintenance downtime (prompt update, change deployment, n8n upgrade) never exceeds 15 minutes per event, and never happens during the firm's business hours without prior notification to the operator.
- **NFR19**: The workflow is idempotent with respect to received webhooks: Telegram/WhatsApp retries due to timeout or network error never result in duplicate messages to the client or duplicate handoffs to the attorney. Deduplication uses the provider's unique message identifier.

### Scalability

- **NFR20**: The system supports current volume of ~100 conversations/day with at least 3x headroom (300 conversations/day) without cost increase or architectural change, operating within the free tiers of external services.
- **NFR21**: The n8n workflow cannot consume more than 20% of CPU or 30% of RAM on BorgStack-mini under normal load, guaranteeing headroom for the other stack services (PostgreSQL, Redis, Caddy, Authelia, Evolution API, Homer, lldap).
- **NFR22**: When Groq free tier limits are reached, the system automatically scales to paid credits on the same provider without requiring code or workflow changes; the transition happens by credentials configuration, not by reengineering.
- **NFR23**: The `chat_analytics` table implements appropriate indexes for typical queries (by session_id, by date, by telegram_id, by status) and does not degrade insert performance as it grows. Partitioning or archival policy triggers automatically per retention.

### Maintainability

- **NFR24**: At least 90% of nodes in the final workflow are n8n 2.x native or community visual nodes. JavaScript Code nodes are justified, documented exceptions.
- **NFR25**: Each sub-workflow has a single, clear responsibility, with inputs and outputs documented via the Call n8n Workflow Tool description field. Sub-workflows can be tested individually via manual execution with test data.
- **NFR26**: The main system prompt and all auxiliary prompts live as Git-versioned markdown files in the repository. The production version is recoverable at any time via `git log` + copy-paste, without depending on n8n's internal state.
- **NFR27**: Every refinement cycle produces a structured record in `refinement-log/` containing date, motivation, applied change, before/after metrics, and rollback justification if needed. The bot's evolution history is traceable.
- **NFR28**: The main workflow has at most 7 visually distinguishable logical layers in the n8n editor. If it exceeds that, parts are extracted into sub-workflows to preserve readability.
- **NFR29**: A new operator (external developer) can understand the workflow architecture in ≤ 2 hours of reading the repository plus visual inspection of the n8n editor, without an onboarding session from Galvani. This is an Innovation 5 validation criterion (replicability).

### Observability

- **NFR30**: Every conversation, handoff, prompt refinement, and incident generates a record in `chat_analytics` or an auxiliary table with granularity sufficient for forensic reconstruction: what the message was, what the response was, which prompt version was in use, which model was invoked, how many tokens were consumed, what the latency was, which guardrails fired.
- **NFR31**: The operator can, from a handoff identifier, reconstruct the full original conversation, all decisions made by the agent (classification, scoring, tools called), and all system parameters at handoff time, in at most 5 minutes of querying.
- **NFR32**: Runtime errors are captured by the Error Trigger workflow with full context: workflow, failed node, input that caused the failure, stack trace, timestamp. The operator receives critical-error notification within 5 minutes via Telegram admin.
- **NFR33**: Operational metrics (latency, errors, volume, cost) are aggregated into an automatic weekly report delivered to the operator. The report does not depend on external dashboards to be actionable.

### Integration Quality

- **NFR34**: Every call to an external service (Groq LLM, Groq Whisper, Comadre, Evolution API, Telegram API) implements Retry on Fail with exponential backoff and a maximum-retry ceiling, configured per use-case sensitivity (latency vs. resilience).
- **NFR35**: The contract with each external service is explicitly documented in the PRD or in a reference file in the repo: endpoint, method, expected payload format, expected success response, possible error codes, fallback behavior. Changes in third-party contracts are detectable via error-rate monitoring per endpoint.
- **NFR36**: Switching the operational LLM provider (Groq → another OpenAI-compatible provider) happens by changing credentials and variables, without altering the workflow topology. This preserves the architecture in case of Groq policy change or service discontinuation.
- **NFR37**: Webhooks received from Telegram and Evolution API are validated for signature/origin when supported by the platform, and rejected if validation fails. This protects against message spoofing.

### Portability & Replicability

- **NFR38**: The template export artifact (`exports/n8n-workflow-template-v1.json`) is self-contained and does not depend on external state from the development n8n instance. Importing into a clean n8n produces a functional workflow after credentials configuration.
- **NFR39**: All firm-specific information (names, city, practice areas, TTS voice, specific prompts) is externalized via variables, credentials, or prompt files. Changing firm is changing these values, not reengineering the workflow.
- **NFR40**: The template is accompanied by documentation sufficient for a developer familiar with n8n to perform a deployment at a new client in ≤ 8 hours of work, per Innovation 5 validation criterion.
- **NFR41**: Template dependencies are explicitly listed: minimum n8n version (2.x), required SQL tables, required community nodes, credentials by type. n8n upgrades may require adjustments but are versioned in `exports/workflow-snapshots/`.

### Cost

- **NFR42**: Total monthly operating cost for running the bot in production at ~100 conversations/day stays at ≤ R$ 100/month, including marginal API costs (Groq paid in fallback case), external services (Cloudflare, etc.), and shared infrastructure (BorgStack, Comadre). The normal-regime target is R$ 0-50/month using only free tiers.
- **NFR43**: The frontier model used in the refinement cycle (Claude Opus 8.6 via Claude Code) is operated exclusively under the operator's fixed monthly subscription, resulting in zero marginal cost per refinement cycle regardless of how many cycles are executed.
- **NFR44**: When monthly cost exceeds 80% of budget, the system issues an alert to the operator with breakdown by provider and recommendation for action (prompt optimization to reduce tokens, rate limit adjustment, etc.).

### Data Residency & Sovereignty

- **NFR45**: Client data (profiles, conversations, transcripts, handoff packages) resides exclusively in the client firm's BorgStack-mini PostgreSQL, under the firm's operational control. No persistent copy in third-party infrastructure.
- **NFR46**: In production, sensitive data transiting external services (Groq for LLM and Whisper) is limited to the strict minimum: current conversation window plus system prompt, never full history or aggregated data from multiple clients. The external provider never receives non-masked PII beyond the content of the client's current message.
- **NFR47**: Audit bundles exported for analysis in Claude Code during the post-launch refinement cycle pass through an anonymization filter (names, phones, CPFs, addresses replaced with placeholders) before leaving BorgStack. The filter is operated and validated by Galvani before every export.
