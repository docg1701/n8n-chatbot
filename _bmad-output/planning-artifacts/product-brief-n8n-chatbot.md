---
title: "Product Brief: Legal Intake AI"
status: "complete"
created: "2026-04-07"
updated: "2026-04-07"
inputs:
  - docs/plan.md
  - research/01-agent-patterns-and-optimization.md
  - research/01b-agent-patterns-supplementary.md
  - research/02-memory-rag-and-context.md
  - research/03-media-integrations-and-security.md
  - research/04-production-operations.md
---

# Product Brief: Legal Intake AI

## Executive Summary

Half the people who message a law firm on WhatsApp will never become paying clients. But an attorney has to talk to every single one of them to find out. In a firm where 50%+ of revenue comes from internet leads, the intake process is both the most critical and the most wasteful part of the operation — and hiring humans to do it is an increasingly losing bet in Brazil's talent market.

Legal Intake AI is a self-improving chatbot that handles the first conversation with every potential client. It greets them naturally in Brazilian Portuguese (text and voice), identifies their legal need, asks the right qualifying questions, and either routes a complete brief to a human attorney or helps the person directly — so even unqualified leads leave feeling genuinely served. Under the hood, cheap fast models handle daily conversations while a frontier model periodically audits their performance and refines every prompt, guardrail, and filter. The system gets measurably better every week.

The first deployment is Maruzza Teixeira Advocacia in São Luís, MA — 1,500+ clients, 8 practice areas, the largest social security firm in the city. This live environment with real volume and real edge cases accelerates iteration faster than any synthetic pilot. The proven case becomes the portfolio to sell the solution to other law firms and eventually other industries.

## The Problem

A mid-sized Brazilian law firm spends heavily on digital marketing to attract clients via WhatsApp. The leads arrive — but 50-80% are cold, out of scope, or just looking for free advice. In some practice areas like labor law, up to 80% of contacts will never convert — people who prefer to consult a lawyer for free online rather than search Google. An attorney (billing at R$300-500/hour) personally screens every one of them. You only discover who is a good lead by doing the intake.

The alternative — hiring and training salespeople — is expensive, slow, and fragile. Good ones take months to develop, cost real money, and leave the moment a competitor offers 5% more salary or they decide to "start their own business." Qualified labor for client intake in Brazil is extremely rare and getting scarcer every year.

The result: the firm's most expensive resource is doing its least valuable work, the labor market offers no reliable alternative, and the problem compounds as digital marketing costs rise and lead quality degrades.

## The Solution

An AI-powered chatbot deployed on Telegram (MVP) and WhatsApp that acts as the firm's virtual intake specialist:

- **Conversations, not forms.** Natural dialogue in Brazilian Portuguese — text and voice. Clients talk to it the way they'd talk to a receptionist. It doesn't feel like a bot.
- **Smart qualification.** The chatbot maps the client's need to one of the firm's practice areas, asks targeted questions, collects name, city, case summary, and urgency — all conversationally.
- **Graceful routing.** Qualified leads get packaged (data + full transcript) and sent to the attorney with action buttons (Accept / Need More Info / Decline). Unqualified leads are helped genuinely with curated public resources (Defensoria Pública, INSS website, Procon, labor tribunals) — attorney-approved content, not AI-generated legal advice. They leave thinking "that was useful," not "I got rejected."
- **Self-improving intelligence.** Cheap operational models (GPT-4o-mini / Claude Haiku) handle every conversation. A weekly audit cycle works as follows: a scheduled workflow samples recent conversation logs and qualification outcomes, sends them to a frontier model (Claude Opus / GPT-4o) which scores responses, identifies failure patterns, and proposes prompt revisions. New prompts are evaluated against a held-out golden dataset; if they outperform the current version, the attorney reviews and approves the change before it goes live. The system improves continuously with minimal human effort.

## What Makes This Different

**Self-improving, not static.** Every competitor in Brazil (Sabio Adv, ChatADV, WhatsBot GPT) ships static prompts that degrade silently. Legal Intake AI runs a meta-prompting cycle grounded in established methodology (DSPy, AutoPrompt): frontier model audits cheap model behavior, refines prompts and guardrails, tests changes against a golden dataset, and promotes winners after human sign-off. The system compounds its advantage every week it runs.

**Data stays in the firm.** Client conversations never leave the firm's infrastructure. Self-hosted on the firm's own servers — no data flowing to third-party SaaS platforms. In a profession where attorney-client privilege is non-negotiable and LGPD compliance is mandatory, this is the single most important differentiator. Every SaaS competitor forces firms to trust their client data to external servers. This solution eliminates that risk entirely.

**Bespoke, not generic.** Each deployment is tuned to the firm's specific practice areas, qualifying criteria, voice, and regional context. Social security law in Brazil's Northeast has distinct claim types, local INSS office patterns, and regional court backlogs. A system trained on real caseload from São Luís will outperform any generic national product in that region — and that hyper-local precision becomes the template for each new deployment.

**Doesn't quit for 5% more.** The chatbot works every hour, handles every conversation identically well, never has a bad day, and never walks out the door taking institutional knowledge with it.

## Who This Serves

**Primary: The managing attorney.** Receives only qualified, pre-briefed leads with enough data to begin substantive work immediately. Reclaims hours previously spent on repetitive screening. Tracks lead quality, conversion, and chatbot performance through weekly reports that surface actionable decisions: which practice areas convert best, where the chatbot struggles, and whether the latest prompt revision improved or degraded performance.

**Secondary: The potential client.** Gets an immediate, warm, professional response at any hour — no "we'll get back to you." Whether they become a client or not, they receive a respectful experience that reflects well on the firm's brand.

**Future: Other firms and businesses.** Once proven at Maruzza Teixeira, the solution — architecture, workflows, self-improvement engine — becomes a productized offering. Medical clinics, accounting firms, real estate brokers, and insurance agencies in Brazil face the identical problem: expensive licensed professionals doing low-value screening over WhatsApp.

## Compliance and Data Governance

- **OAB compliance:** The chatbot discloses it is an AI assistant. It never provides legal advice — it gathers information and routes. All substantive content shown to unqualified leads (public resources, referrals) is pre-approved by the attorney and stored as static responses, not generated dynamically.
- **LGPD:** Informed consent captured at conversation start. Data retention and deletion policies defined per firm. Access to conversation logs restricted to authorized personnel. All data encrypted at rest (PostgreSQL with `N8N_ENCRYPTION_KEY`).
- **Attorney-client privilege:** Conversations stored on-premise, never transmitted to external services beyond the LLM API call (which processes only the current message window, not the full history).

## Success Criteria

| Metric | Target | How Measured |
|--------|--------|-------------|
| Conversation quality | Users engage naturally; post-conversation survey >=4/5 | Optional end-of-conversation rating prompt |
| Practice area accuracy | Correct identification >90% | Human review of random sample vs. chatbot classification |
| Attorney time saved | 60%+ reduction in intake screening | Before/after time tracking over first month |
| Data completeness | >85% of handoffs include name, area, summary, urgency | Automated check on lead package fields |
| Graceful dismissal | Unqualified leads feel helped, not rejected | Post-conversation survey; absence of complaints |
| Self-improvement | Qualification accuracy and dismissal quality improve across prompt versions | Delta in evaluation scores between prompt versions on golden dataset |
| Response time | <30 seconds, 24/7 | Automated latency logging |

## Scope

**MVP (Phase 1):** Text-only Telegram chatbot. Telegram is chosen for the MVP to avoid the upfront investment in n8n + WhatsApp (Evolution API) integration — the core qualification logic is platform-independent and transfers directly to WhatsApp in Phase 3. Deterministic router handles greetings, FAQs (office address, hours, practice areas, pricing), commands, and pleasantries without LLM. AI triage agent for everything else. Postgres-backed conversation memory (10-15 message window) and client profiles. Deterministic lead qualification scoring. Attorney handoff via Telegram group with Accept/Decline buttons. Security: input sanitization, Redis rate limiting, guardrails (jailbreak detection, topical alignment). Analytics logging. Error handling workflow. Weekly report workflow.

**Phase 2:** Bidirectional voice — Groq Whisper STT for incoming voice messages, Comadre TTS (native Brazilian Portuguese, kokoro/pm_santa voice) for audio responses. Infrastructure for voice already exists and is proven.

**Phase 3:** WhatsApp migration via Evolution API (already running on infrastructure) with platform normalization layer so the triage logic remains unchanged.

**Not in scope for v1:** RAG knowledge base, appointment scheduling, document collection, case status queries, multi-firm tenancy.

## Vision

Maruzza Teixeira is the first client and the live laboratory. Within weeks, the chatbot is handling real conversations and improving itself. Within months, statistical evidence accumulates: better qualification rates, faster response times, more contracts closed, lower cost per lead. The deployment is documented as a structured case study with before/after metrics from day one.

Beyond intake, the same infrastructure naturally extends into the post-conversion client lifecycle: document collection and validation, case status updates, appointment reminders, and renewal nudges. A one-time intake tool becomes a continuous client relationship platform — increasing lifetime value without acquiring new leads.

That data becomes the sales portfolio. The architecture — n8n workflows, self-improvement engine, multi-platform adapters — is designed to be replicable. The next deployment is another law firm. Then a dental clinic, a real estate agency, any business where lead qualification is manual, repetitive, and expensive.

The self-improvement engine is the moat. Every week the system runs, it gets better. Competitors with static prompts fall further behind. The gap compounds.
