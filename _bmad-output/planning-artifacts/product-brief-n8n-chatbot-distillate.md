---
title: "Product Brief Distillate: n8n-chatbot"
type: llm-distillate
source: "product-brief-n8n-chatbot.md"
created: "2026-04-07"
purpose: "Token-efficient context for downstream PRD creation"
---

# Product Brief Distillate: Legal Intake AI

## User and Stakeholder Context

- **Firm owner (Galvani)** builds and maintains the system himself using n8n + Claude guidance; intermediate technical skill level
- **Managing attorney** is the primary beneficiary — receives pre-qualified leads with structured data, skips exhausting early intake steps
- **Attorney time is the most expensive resource** in the firm; client intake is repetitive, draining, and currently unavoidable
- **50%+ of firm revenue** comes from internet-sourced clients via digital marketing campaigns
- **50-80% of leads are cold/wrong** depending on practice area; labor law is worst at ~80% — people seeking free advice online instead of Googling
- **Bad leads can occasionally convert** but most never will; you only know which is which after doing the intake
- **Digital marketing costs are high**, lead quality is degrading, and the trend is worsening year over year
- **Qualified intake staff in Brazil is extremely scarce**: months to train, expensive to retain, high turnover — good salespeople leave for 5% more salary or to "start their own business"
- **The chatbot must not feel like a chatbot** — success means natural, convincing conversation that clients don't question

## Detailed User Scenarios

- **Unqualified lead in labor law**: person describes workplace issue, chatbot identifies labor area, asks qualifying questions, determines case is weak or out of scope — bot provides genuinely helpful curated resources (Defensoria Pública, labor tribunal info), warm farewell. Client feels helped, not rejected. Brand preserved
- **Qualified social security lead**: person describes denied INSS benefit, chatbot identifies previdenciário area, collects name, city (São Luís — location bonus), case details, urgency (high). Score >= 60. Bot confirms data with client, triggers handoff. Attorney receives structured package with full transcript and action buttons
- **Returning client**: system queries client_profiles by telegram_id, injects profile into system prompt, greets by name, skips already-answered questions, picks up where last conversation left off
- **Voice message flow** (Phase 2): client sends voice on Telegram → Groq Whisper STT transcribes → triage agent processes text → response generated → Comadre TTS synthesizes Brazilian Portuguese audio → client receives both text and voice response
- **Attorney handoff**: lead package sent to private Telegram group with Accept / Need More Info / Decline buttons. 4-hour timeout. On accept → client notified. On "need more info" → bot asks follow-up. On decline → graceful closure. On timeout → escalate to backup or schedule next business day

## Competitive Intelligence

- **Sabio Adv (Brazil)**: WhatsApp Business API-authorized AI for law firms. Closed SaaS, no self-improvement engine, firm has zero control over model behavior or data pipeline
- **ChatADV (Brazil)**: GPT-5-based legal AI with WhatsApp and web. Generic nationwide, not firm-specific, no n8n integration, no self-refinement loop, opaque model updates
- **WhatsBot GPT / Intelizap (Brazil)**: Low-cost WhatsApp chatbot builders (~R$80/mo). Template-driven, shallow legal domain knowledge, no RAG, no memory beyond session, no lead scoring
- **LawDroid / Smith.ai / Intaker (US)**: Specialized legal intake with AI lead scoring. English-only, no WhatsApp, SaaS pricing scales poorly, no self-hosted option
- **Perspective AI / Lawmatics QualifyAI (US)**: Conversational AI replacing web forms, 2.5x conversion lift. Web-chat only, no WhatsApp/Telegram, no Brazilian legal knowledge, closed-source
- **Key gap across all competitors**: no self-improvement engine, no self-hosting option, no firm-specific customization at the practice-area level
- **No dominant bespoke legal intake solution exists in Brazil** — the market is generic SaaS tools leaving a gap for firm-specific, self-hosted systems

## Regulatory and Compliance Context

- **OAB Recommendation 001/2024**: advisory (not prohibitive) guidelines — disclose AI use, no legal advice, attorney supervision, data confidentiality. Current window is permissive before stricter regulation arrives
- **CNJ Resolução 615/2025**: additional regulatory framework for AI in legal practice
- **LGPD (Brazil's GDPR)**: mandates informed consent, data minimization, encryption at rest, access controls, clear retention policies. Intake conversations contain sensitive legal details
- **The chatbot must never provide legal advice** — it gathers information and routes. All content shown to unqualified leads (public resources, referrals) must be pre-approved by attorney and stored as static responses
- **Attorney-client privilege**: conversations stored on-premise only; LLM API calls process only the current message window, not full history
- **AI disclosure required**: chatbot must identify itself as AI assistant when interacting with clients

## Technical Context and Constraints

- **Infrastructure (all running)**: BorgStack-mini single-node Docker Swarm at 10.10.10.205 — PostgreSQL+pgvector, Redis, n8n (queue mode: editor+webhook+worker), Caddy, Cloudflare tunnel
- **n8n queue mode constraint**: successive requests may hit different worker processes, so Simple Memory (in-process RAM) does not work — all memory must use external backends (Postgres, Redis)
- **TTS engine (running)**: Comadre at 10.10.10.207 — dual-engine (Kokoro fp32 + Qwen3-TTS), OpenAI-compatible API, kokoro/pm_santa voice (natural Brazilian Portuguese male)
- **STT**: Groq Whisper API — sub-second latency, free tier, native pt-BR. Fallback: OpenAI Whisper ($0.006/min)
- **Operational models**: GPT-4o-mini or Claude Haiku (~$0.15/M input tokens). Fallback model configured in AI Agent node for provider resilience
- **Frontier audit model**: Claude Opus or GPT-4o for the meta-prompting cycle
- **n8n security**: version must be >= 1.120.4 (patches CVE-2025-68613 RCE vulnerability)
- **WhatsApp**: Evolution API already running on BorgStack, deferred to Phase 3 to avoid integration investment during MVP
- **Telegram MVP rationale**: faster to develop, n8n has built-in Telegram Trigger node with auto-registering webhook, qualification logic is platform-independent and transfers directly to WhatsApp

## Self-Improvement Engine (Core Differentiator)

- **Architecture**: two-tier model system — cheap models (Haiku/GPT-4o-mini) handle live conversations; frontier model (Opus/GPT-4o) periodically audits and refines
- **Audit cycle**: weekly scheduled workflow → sample recent conversation logs and qualification outcomes → frontier model scores responses, identifies failure patterns → proposes prompt revisions → new prompts evaluated against held-out golden dataset → if they outperform, attorney reviews and approves before promotion
- **Scope of refinement**: system prompts, guardrails, deterministic router filters, qualifying criteria, response templates — everything the cheap model uses as instructions
- **Methodology grounding**: DSPy (documented 46.2% → 64.0% accuracy improvement), AutoPrompt, Meta-Rewarding Language Models
- **Critical constraint**: autonomous prompt changes in legal context require human sign-off — a frontier model optimizing for user satisfaction could inadvertently soften qualification criteria, producing pleasant conversations with worse leads
- **Risk**: adversarial prompt research shows meta-prompting can bypass safety filters 62% of the time — human-in-the-loop review before deploying prompt changes is essential, not optional
- **Risk**: prompt overfitting to test cases and bias amplification if frontier model encodes its own biases into refined prompts — periodic human audits of meta-prompting outputs required
- **Golden dataset requirements**: 3+ conversations per practice area, edge cases (ambiguous area, unqualified lead, aggressive client, out-of-scope), returning clients, voice-originated messages

## Scope Signals

- **In MVP**: text Telegram, deterministic router, AI triage agent, Postgres Chat Memory (10-15 msg window), client profiles, deterministic lead scoring, HITL handoff, input sanitization, Redis rate limiting, guardrails (jailbreak/topical), analytics logging, error handling workflow, weekly reports
- **Phase 2**: bidirectional voice (STT + TTS) — infrastructure already exists
- **Phase 3**: WhatsApp via Evolution API with normalization layer
- **Future (post-Phase 3)**: post-intake client lifecycle (document collection, case status updates, appointment reminders, renewal nudges), RAG knowledge base, appointment scheduling, multi-firm tenancy, evaluation automation (meta-prompting cycle)
- **Explicitly out of scope**: referral network for rejected leads (raised and rejected), productization pricing/commercial model (future work, far after implementation), multi-firm tenancy in v1

## Rejected Ideas (with rationale)

- **Referral network for rejected leads** (routing unqualified leads to partner firms for referral fees): out of scope — adds complexity and partnership management before the core product is proven
- **Productization pricing model**: premature — the system needs to be built, tested, and refined at one firm before commercial questions are relevant
- **RAG knowledge base in MVP**: system prompt + static tools cover initial scope; RAG deferred until knowledge base grows and volume justifies the investment
- **WhatsApp in MVP**: requires n8n + Evolution API integration work; Telegram MVP avoids that cost while the qualification logic is platform-independent
- **Fully autonomous prompt changes**: legal context requires human sign-off before deploying any prompt revision — even though the self-improvement engine is a core feature, it must include attorney review as a gate

## Open Questions

- **Replication cost**: when the time comes to deploy at Firm #2, how much is configuration (swap practice areas, criteria, firm name) vs. engineering? This determines whether the future productization vision is viable as a product or a consulting service
- **Telegram audience validation**: the firm's actual leads arrive via WhatsApp; Telegram MVP tests the technology on a potentially different audience — conversion of real leads won't be measurable until Phase 3
- **Attorney-side UX**: the handoff notification format, acknowledgment flow, and how conversion outcomes (contract signed / not interested) feed back into the system as ground truth for the audit cycle need detailed design
- **Urgent/distress escalation**: how does the system handle someone describing an urgent situation (criminal arrest, domestic violence) that requires immediate human intervention beyond normal handoff?
- **STT quality for regional accents**: Maranhão regional Portuguese accent + legal terminology + budget smartphone microphones + background noise may degrade Groq Whisper transcription — fallback behavior and confidence thresholds need testing

## Market Timing

- OAB's advisory (not prohibitive) AI guidelines create a permissive adoption window before stricter regulation arrives
- Cheap operational models now match 2023-era frontier performance at 10-50x lower cost, making the two-tier architecture economically viable
- Meta-prompting frameworks matured from research to production-ready in 2025-2026
- No Brazilian competitor offers a self-improving chatbot — every product uses static prompts
- 67% of potential clients abandon traditional intake forms; conversational AI achieves 75-85% completion rates
- 72% of legal consumers comfortable with AI for initial intake (83% for under-45s)
- Brazilian legal AI sector entered "augmented intelligence" phase in 2026 (TOTVS report)
