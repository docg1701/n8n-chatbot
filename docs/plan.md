# Project Plan — Maruzza Teixeira Legal Intake Chatbot

> Version 1.0 — April 2026
>
> This plan was built from the deep research documented in `research/`. Each section references its source files. For implementation details beyond what is covered here, consult the original research.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Solution Overview](#2-solution-overview)
3. [Architecture](#3-architecture)
4. [Message Ingestion Layer](#4-message-ingestion-layer)
5. [Media Processing](#5-media-processing)
6. [Deterministic Router](#6-deterministic-router)
7. [Triage Agent](#7-triage-agent)
8. [Memory Architecture](#8-memory-architecture)
9. [Context Management](#9-context-management)
10. [Response and TTS](#10-response-and-tts)
11. [Security Layers](#11-security-layers)
12. [Lead Qualification and HITL Handoff](#12-lead-qualification-and-hitl-handoff)
13. [Error Handling and Resilience](#13-error-handling-and-resilience)
14. [Evaluation and Prompt Improvement](#14-evaluation-and-prompt-improvement)
15. [Monitoring, Analytics, and Reporting](#15-monitoring-analytics-and-reporting)
16. [Cost Tracking and Optimization](#16-cost-tracking-and-optimization)
17. [Conversation Lifecycle](#17-conversation-lifecycle)
18. [Testing and Staging](#18-testing-and-staging)
19. [Database Schema](#19-database-schema)
20. [Project Structure](#20-project-structure)
21. [Delivery Phases](#21-delivery-phases)
22. [Research References](#22-research-references)

---

## 1. Problem Statement

Maruzza Teixeira Advocacia is a law firm in São Luís, Maranhão, with 1,500+ clients across eight practice areas. They are the largest social security (previdenciário) firm in the city. Potential clients find the firm online and reach out via WhatsApp. An attorney currently handles all incoming conversations personally — screening, qualifying, and gathering information.

The bottleneck: attorney time is expensive, and most incoming leads are either unqualified, outside the firm's scope, or need basic information that does not require a lawyer. The firm needs a way to give every contact a warm, professional experience while only routing genuinely valuable leads to the human attorney.

### Practice Areas

| Area | Portuguese Term | Typical Client Need |
|------|----------------|---------------------|
| Social Security | Previdenciário | Denied INSS benefits, retirement issues, disability claims, LOAS |
| Labor | Trabalhista | Wrongful termination, workplace accidents, wage disputes, CLT violations |
| Banking | Bancário | Abusive interest rates, debt renegotiation, credit card disputes |
| Healthcare | Saúde | Medical malpractice, health insurance denials, treatment access |
| Family | Família | Divorce, child custody, spousal support, paternity, inheritance |
| Real Estate | Imobiliário | Property transactions and disputes |
| Corporate | Empresarial | Business recovery and restructuring |
| Tax | Tributário | Tax compliance, disputes, liability mitigation |

Source: [maruzzateixeira.com](https://www.maruzzateixeira.com)

---

## 2. Solution Overview

Build an AI-powered chatbot that acts as the firm's virtual receptionist:

1. **Greet** every potential client warmly
2. **Identify** their legal need and match it to a practice area
3. **Qualify** the lead by asking targeted questions
4. **Collect** key information (name, city, case summary, urgency)
5. **Route** qualified leads to a human attorney with a complete brief
6. **Decline gracefully** when the case is not a fit, suggesting alternatives

The chatbot communicates via text and voice (bidirectional audio). It starts on Telegram (faster to develop and test) and migrates to WhatsApp (via Evolution API) when proven.

**Infrastructure**: n8n on BorgStack-mini at `10.10.10.205` (PostgreSQL+pgvector, Redis, Caddy, Cloudflare tunnel). TTS via Comadre at `10.10.10.207`.

---

## 3. Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                     MESSAGING CLIENT                             │
│              Telegram (MVP) → WhatsApp (Phase 3)                 │
│                  text / voice / documents                        │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│  LAYER 1 — MESSAGE INGESTION                                     │
│  Telegram Trigger node (webhook, auto-registered)                │
│  Session ID: chat.id (unique per user)                           │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│  LAYER 2 — MEDIA PROCESSING (sub-workflow)                       │
│  Voice → Groq Whisper STT → text (flag: inputWasAudio)           │
│  Text → sanitize, normalize, extract metadata                    │
│  Images/Docs → extract or flag for human review                  │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│  LAYER 3 — SECURITY PRE-CHECK                                    │
│  Rate limiting (Redis counter per user)                          │
│  Input sanitization (strip HTML, truncate, remove zero-width)    │
│  Guardrails node: jailbreak detection, topical alignment         │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│  LAYER 4 — DETERMINISTIC ROUTER (Code + Switch, no LLM)          │
│  /start, /menu, /help → static responses                        │
│  Greetings → welcome + ask how to help                           │
│  FAQ patterns → canned answers (areas, address, hours, pricing)  │
│  Thanks, goodbye → polite closure                                │
│  Everything else → falls through to Layer 5                      │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│  LAYER 5 — TRIAGE AGENT (AI Agent node)                          │
│  Model: GPT-4o-mini or Claude Haiku (cheap, fast)                │
│  Memory: Postgres Chat Memory (10-15 msg window)                 │
│  System prompt: legal intake persona for Maruzza Teixeira        │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ Tools (4):                                               │    │
│  │  • save_client_info → write to client_profiles table     │    │
│  │  • lookup_practice_areas → static reference data         │    │
│  │  • qualify_lead → deterministic scoring logic             │    │
│  │  • request_human_handoff → trigger HITL notification      │    │
│  └──────────────────────────────────────────────────────────┘    │
│  Output guardrails: PII check before sending response            │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                     ┌────┴────┐
                     ▼         ▼
         ┌───────────────┐  ┌─────────────────────────────────────┐
         │ NORMAL REPLY  │  │ QUALIFIED LEAD → HITL HANDOFF       │
         │ Text always   │  │ Compose lead package (name, area,   │
         │ + TTS if user │  │ case summary, urgency, transcript)  │
         │   sent audio  │  │ Notify attorney via Telegram group  │
         └───────────────┘  │ Wait for accept / reject / timeout  │
                            └─────────────────────────────────────┘
```

**Research sources**: `01-agent-patterns-and-optimization.md` §1 (orchestration patterns), `01b-agent-patterns-supplementary.md` §1.3-1.5 (sub-workflows, chaining), `03-media-integrations-and-security.md` §1.3 (Telegram), §4 (security)

---

## 4. Message Ingestion Layer

### Telegram (MVP)

- n8n built-in **Telegram Trigger** node
- Webhook auto-registers on workflow activation
- Receives: text, voice, photos, documents, callback queries
- Session ID: `chat.id` (unique per user, persists across conversations)
- Bot token from [@BotFather](https://t.me/BotFather)
- n8n must be HTTPS-accessible (handled by Caddy + Cloudflare tunnel)
- Constraint: one webhook URL per bot — only one workflow can listen

### WhatsApp via Evolution API (Phase 3)

- Evolution API already running on BorgStack
- Receives messages via `MESSAGES_UPSERT` webhook event
- Sends via `POST /message/sendText/{instance}` and similar endpoints
- Media via `POST /message/sendWhatsAppAudio/{instance}`
- A message normalization layer will abstract platform differences so the triage agent logic remains unchanged

### Multi-Platform Normalization Pattern

When WhatsApp is added, a Code node normalizes incoming messages into a unified schema before they reach the router:

```javascript
{
  platform: 'telegram' | 'whatsapp',
  userId: string,        // telegram chat.id or whatsapp phone number
  userName: string,
  messageType: 'text' | 'audio' | 'image' | 'document',
  text: string,          // original or STT-transcribed text
  mediaUrl: string,      // if applicable
  timestamp: ISO8601,
  inputWasAudio: boolean
}
```

**Research sources**: `03-media-integrations-and-security.md` §1.1 (Evolution API), §1.3 (Telegram), §1.8 (normalization)

---

## 5. Media Processing

Implemented as a **sub-workflow** called by the main chatbot workflow. This keeps the main flow clean and allows reuse when WhatsApp is added.

### Voice Messages (STT)

1. Telegram sends voice as OGG file → download via Telegram "Get File" operation
2. Send audio to **Groq Whisper** STT API (near-instant, free tier, supports pt-BR natively)
3. Extract transcribed text → continue as a text message
4. Preserve flag `inputWasAudio: true` so the response layer knows to generate audio back

**Why Groq Whisper**: sub-second latency, free tier available, outperforms local Whisper setups on CPU-only hardware. Alternative: OpenAI Whisper ($0.006/min) if Groq is unavailable.

### Images (Future — Phase 2+)

- Send to GPT-4o-mini vision for description/OCR
- Use case: client sends a photo of a document (benefit denial letter, contract)
- Flag all images for human review alongside the conversation

### Documents (Future — Phase 2+)

- n8n **Extract From File** node handles PDF, XLSX, CSV, DOCX
- For complex PDFs: LlamaParse or PDF.co community nodes
- Use case: client sends legal documents, contracts, court notices

### Text Preprocessing

A Code node normalizes all text input before routing:

```javascript
// Strip HTML, normalize whitespace, remove zero-width chars, truncate
const raw = input.message;
const clean = raw
  .replace(/<[^>]*>/g, '')
  .replace(/[\u200B-\u200D\uFEFF]/g, '')
  .replace(/\s+/g, ' ')
  .trim();
const truncated = clean.length > 4000
  ? clean.substring(0, 4000) + '...[truncated]'
  : clean;
```

**Research sources**: `03-media-integrations-and-security.md` §2 (attachment processing), §3 (TTS/STT), `01-agent-patterns-and-optimization.md` §7 (data preprocessing)

---

## 6. Deterministic Router

A **Code node** + **Switch node** before any LLM call. Handles the highest-volume, lowest-complexity messages at zero cost.

### Intent Classification (Code Node)

```javascript
const msg = input.text.toLowerCase().trim();

// Telegram commands
if (msg.startsWith('/'))
  return { intent: 'command', cmd: msg.split(' ')[0] };

// Greetings (Portuguese)
if (/^(oi|olá|ola|bom dia|boa tarde|boa noite|hey|hello|hi)\b/.test(msg))
  return { intent: 'greeting' };

// Gratitude
if (/^(obrigad[oa]|valeu|thanks|thank you)\b/.test(msg))
  return { intent: 'thanks' };

// Farewell
if (/^(tchau|até mais|bye|adeus|falou)\b/.test(msg))
  return { intent: 'bye' };

// FAQ: contact info
if (/endere[çc]o|localiza[çc]|onde fica|hor[áa]rio|telefone|contato/.test(msg))
  return { intent: 'faq_contact' };

// FAQ: practice areas
if (/[áa]reas?|atua[çc]|especialidade|tipos? de caso/.test(msg))
  return { intent: 'faq_areas' };

// FAQ: pricing
if (/pre[çc]o|valor|custo|cobran[çc]|honor[áa]rio|quanto custa/.test(msg))
  return { intent: 'faq_pricing' };

// Everything else → AI
return { intent: 'ai_triage' };
```

### Static Responses (Set Nodes)

Each deterministic intent maps to a pre-written response:

- **greeting**: Welcome message + brief introduction + ask how to help
- **faq_contact**: Address, phone, hours, WhatsApp number
- **faq_areas**: List of practice areas with one-line descriptions
- **faq_pricing**: Explain that the initial consultation is free, fees depend on the case
- **thanks/bye**: Warm closing message
- **command /start**: Welcome + menu of options
- **command /help**: Available commands and how to talk to a human

### Expected Savings

15-25% of all messages handled without any LLM call. These are typically the highest-volume interactions: greetings, FAQs, and pleasantries.

**Research sources**: `01-agent-patterns-and-optimization.md` §3 (deterministic filters), `01b-agent-patterns-supplementary.md` §3.3-3.7 (regex detection, FAQ lookup, decision tree)

---

## 7. Triage Agent

### Model Selection

**Primary model**: GPT-4o-mini or Claude Haiku — cheap, fast, adequate for structured intake conversations. The agent follows a predefined conversational script; it does not need deep reasoning.

**Fallback model**: configure a secondary model in the AI Agent node so the workflow does not fail if the primary provider is down.

**Future**: use n8n's **Model Selector node** to dynamically route complex cases to a more capable model (Sonnet/GPT-4o) based on conversation complexity or practice area.

### System Prompt

The system prompt is the agent's behavioral contract. It is stored as a versioned file (`docs/system-prompts/triage-agent-v1.md`) and loaded into n8n as a variable or database record — never hardcoded in the workflow.

**Structure**:

```
[IDENTITY]
Virtual assistant for Maruzza Teixeira Advocacia.
Located in São Luís/MA. Speaks Brazilian Portuguese naturally.

[GOAL]
Welcome potential clients. Understand their legal need.
Identify the practice area. Gather qualifying information.
Route qualified leads to a human attorney.

[CONVERSATION FLOW]
1. Greet warmly and ask what legal issue brought them here
2. Identify which practice area applies
3. Ask qualifying questions specific to that area
4. Collect: full name, city, brief case description, urgency
5. If qualified → confirm info with client → call request_human_handoff
6. If not a fit → explain kindly and suggest alternatives

[QUALIFYING CRITERIA BY AREA]
(one paragraph per area — what makes a case worth pursuing)

[TOOLS]
- save_client_info: call when you have name + practice area + case summary
- lookup_practice_areas: call if unsure which area applies
- qualify_lead: call after gathering enough info to score the lead
- request_human_handoff: call only when lead is qualified and confirmed

[RULES]
1. Never give legal advice — you gather information only
2. Never promise outcomes, timelines, or costs
3. Always be empathetic — these people have real problems
4. Keep responses concise (2-3 sentences max per message)
5. If unsure about the area, ask clarifying questions
6. After gathering info, always confirm with the client before handoff
7. Speak the client's language — avoid legal jargon
8. If asked something outside legal topics, redirect politely

[OUTPUT FORMAT]
Plain text in Brazilian Portuguese. No markdown, no emojis unless
the client uses them first. Short paragraphs.
```

### Agent Tools

Four tools, well under the 5-7 per agent limit recommended by research:

| Tool | Type | Purpose |
|------|------|---------|
| `save_client_info` | Sub-workflow (Postgres Insert) | Persist extracted client data to `client_profiles` table |
| `lookup_practice_areas` | Code node (static JSON) | Return practice areas with qualifying criteria for the agent to reference |
| `qualify_lead` | Code node (scoring logic) | Score the lead based on practice area match, info completeness, urgency |
| `request_human_handoff` | Sub-workflow (notification) | Compose lead package and notify attorney via Telegram group |

### Dynamic Context Injection

Before the agent runs, a Set node injects additional context into the user message using n8n expressions:

- Current date and time (so the agent can say "bom dia" vs "boa noite")
- Client profile from `client_profiles` table (if returning client)
- Conversation count (first visit vs returning)

**Research sources**: `01-agent-patterns-and-optimization.md` §4 (system prompts), §1.4 (tool limits), `01b-agent-patterns-supplementary.md` §4.2-4.8 (prompt structure, persona, dynamic injection), `01-agent-patterns-and-optimization.md` §2.1 (Model Selector)

---

## 8. Memory Architecture

n8n on BorgStack-mini runs in **queue mode** (editor + webhook + worker). This means Simple Memory (in-process RAM) does not work — successive requests may hit different worker processes. All memory must use external backends.

### Tier 1 — Short-Term (Current Conversation)

- **Postgres Chat Memory** node attached to the AI Agent
- Context window: 10-15 messages (sweet spot per research)
- Session key: Telegram `chat.id`
- Table auto-created by n8n in the project's PostgreSQL instance
- Provides immediate conversational context

### Tier 2 — Long-Term (Client Profile)

A separate `client_profiles` table in PostgreSQL, populated by the `save_client_info` tool:

| Column | Type | Purpose |
|--------|------|---------|
| `telegram_id` | BIGINT | Unique client identifier |
| `name` | VARCHAR | Client's full name |
| `phone` | VARCHAR | Phone number (if provided) |
| `city` | VARCHAR | Client's city |
| `practice_area` | VARCHAR | Identified legal area |
| `case_summary` | TEXT | Brief description of the legal issue |
| `urgency` | VARCHAR | low / normal / high / urgent |
| `lead_score` | INTEGER | Qualification score (0-100) |
| `status` | VARCHAR | new / qualified / handed_off / closed |
| `created_at` | TIMESTAMPTZ | First contact |
| `updated_at` | TIMESTAMPTZ | Last interaction |

On new conversations, query by `telegram_id` and inject the profile into the system prompt so the agent knows this is a returning client and what was previously discussed.

### Tier 3 — Semantic Search (Future — RAG)

When the firm's knowledge base grows (legal FAQs, procedure guides, internal policies), add pgvector embeddings:

- Document ingestion workflow: source → text splitter → embedding model → pgvector table
- Query workflow: agent calls Vector Store Tool → semantic search → inject relevant chunks
- Not needed for MVP — the system prompt + static tools cover the initial scope

**Research sources**: `02-memory-rag-and-context.md` §1 (short-term memory, queue mode limitation), §2 (long-term, hybrid architecture), §3 (RAG with pgvector)

---

## 9. Context Management

### Context Rot Prevention

As conversations grow, accumulated context degrades response quality through three mechanisms:

1. **Lost-in-the-middle**: models lose track of information in the middle of long contexts
2. **Attention dilution**: too much context causes the model to miss important details
3. **Distractor interference**: irrelevant old messages confuse current reasoning

### Context Compaction Strategy

For conversations exceeding the 10-15 message window:

1. **Chat Memory Manager** node checks memory size after each agent response
2. If messages exceed threshold → extract key facts into a summary using a cheap LLM call
3. Delete old messages from memory → insert the summary as a system message
4. This preserves essential context while keeping token costs stable

Implementation:
```
Agent responds → Code node counts messages →
  If > threshold:
    Chat Memory Manager (Get Messages) →
    LLM Chain (summarize key facts) →
    Chat Memory Manager (Delete All) →
    Chat Memory Manager (Insert summary as system message)
```

### Context Isolation

When future sub-agents are added (appointment scheduling, document collection), each manages its own context window and memory. This prevents context pollution — billing context should not leak into a legal intake conversation.

**Research sources**: `02-memory-rag-and-context.md` §4 (context rot), §5 (compaction and summary reintroduction), `01-agent-patterns-and-optimization.md` §8 (context engineering strategies)

---

## 10. Response and TTS

### Text Responses

Always sent via **Telegram Action** node (Send Message operation). Supports HTML formatting. Keep messages short (2-3 sentences).

### Audio Responses

When the incoming message was voice (`inputWasAudio: true`):

1. Agent generates text response (same as always)
2. Send text response via Telegram (so client has both text and audio)
3. HTTP Request to Comadre TTS:
   ```
   POST http://10.10.10.207:8000/v1/audio/speech
   Headers: { "Authorization": "Bearer API_KEY", "Content-Type": "application/json" }
   Body: { "input": "response text", "voice": "kokoro/pm_santa" }
   ```
4. Receive audio binary → Telegram "Send Voice" operation → client hears the response

**Voice selection**: `kokoro/pm_santa` — natural Brazilian Portuguese male voice, optimized for Kokoro fp32 engine. Fast synthesis, suitable for real-time conversation.

**Research sources**: `03-media-integrations-and-security.md` §3 (TTS/STT bidirectional flow), Comadre README at `~/dev/comadre/`

---

## 11. Security Layers

Defense in depth — each layer catches a different class of problem. Cheaper layers run first.

### Layer 1 — Input Sanitization (Code Node, zero cost)

- Strip HTML tags
- Remove zero-width and invisible Unicode characters
- Normalize whitespace
- Truncate to 4,000 characters (prevent token abuse)
- Extract metadata (has URL, has email, estimated token count)

### Layer 2 — Rate Limiting (Redis, zero cost)

- Redis INCR + EXPIRE per `telegram_id`
- Threshold: e.g., 30 messages per 5 minutes
- Exceeding → polite "please slow down" message, skip LLM

### Layer 3 — Guardrails Input (n8n Guardrails Node)

- **Jailbreak Detection**: LLM-based, catches prompt injection attempts (threshold 0.7)
- **Topical Alignment**: rejects off-topic messages (non-legal subjects)
- **Keywords**: block specific prohibited terms if needed
- **Custom Regex**: catch patterns specific to legal context

### Layer 4 — System Prompt Guardrails

The system prompt itself is a security boundary:
- Explicit refusal instructions for out-of-scope requests
- Numbered rules the model follows more reliably than prose
- No capability to access other clients' data (tools are scoped per session)

### Layer 5 — Output Guardrails (n8n Guardrails Node)

Before sending the response to the client:
- **PII check**: ensure the agent is not leaking data it should not share
- **Content validation**: verify the response stays within legal intake scope

### Layer 6 — Infrastructure Security

- PostgreSQL on Docker internal network only (no internet exposure)
- n8n credentials system for API keys (encrypted at rest with `N8N_ENCRYPTION_KEY`)
- Cloudflare tunnel + WAF for external access
- Caddy reverse proxy with TLS termination
- n8n version must be ≥ 1.120.4 (patches CVE-2025-68613 RCE vulnerability)

### Layer 7 — Audit Trail

All conversations logged in `chat_analytics` table with:
- Session ID, user ID, message role, content
- Model used, tokens consumed
- Whether the message was audio
- Whether escalation was triggered
- Timestamp

**Research sources**: `03-media-integrations-and-security.md` §4 (complete security section: guardrails, prompt injection, rate limiting, PII, OWASP, Cloudflare, CVE), `01-agent-patterns-and-optimization.md` §4.4 (guardrails node types)

---

## 12. Lead Qualification and HITL Handoff

### Qualification Scoring

The `qualify_lead` tool uses deterministic logic (Code node, no LLM):

```javascript
let score = 0;

// Practice area match (firm handles this area?)
if (SUPPORTED_AREAS.includes(practiceArea)) score += 30;

// Information completeness
if (clientName) score += 15;
if (city) score += 10;
if (caseSummary && caseSummary.length > 20) score += 20;

// Urgency bonus
if (urgency === 'high' || urgency === 'urgent') score += 15;

// Location bonus (São Luís or Maranhão)
if (/são luís|sao luis|maranhão|maranhao/i.test(city)) score += 10;

return { score, qualified: score >= 60 };
```

### Handoff Flow

When `request_human_handoff` is triggered:

1. **Compose lead package** (Code node):
   - Client name, Telegram handle, phone (if provided)
   - Practice area, urgency level, lead score
   - Case summary (agent-written)
   - Full conversation transcript (from Postgres Chat Memory)

2. **Notify attorney** (Telegram Action node):
   - Send to a private Telegram group or channel dedicated to leads
   - Format as a structured message with all key data
   - Include inline keyboard buttons: Accept / Need More Info / Decline

3. **Wait for response** (n8n Wait node):
   - Timeout: 4 hours for first response
   - On accept → tell client "An attorney will follow up shortly", update status to `handed_off`
   - On "need more info" → bot asks client the specified follow-up questions
   - On decline → tell client politely, suggest alternatives, update status to `closed`
   - On timeout → escalate to backup attorney or schedule callback for next business day

4. **Create case record** (Postgres Insert):
   - Insert into a `cases` table (or update `client_profiles` status)
   - Timestamp the handoff for SLA tracking

### Non-Qualified Leads

Leads that score below threshold still get a good experience:
- The bot explains that their issue may require a different type of assistance
- Suggests relevant public resources (INSS website, labor tribunal, consumer protection)
- Invites them to contact the firm directly for a second opinion
- Status set to `closed` with reason

**Research sources**: `04-production-operations.md` §1 (HITL patterns, approval gates, escalation), `04-production-operations.md` §1.4 (Wait node + notification pattern, community handoff nodes)

---

## 13. Error Handling and Resilience

### Node-Level Error Handling

- **Retry on Fail**: enable for external API calls (Groq STT, Comadre TTS, Telegram API) with exponential backoff and jitter
- **Continue on Fail**: enable for non-critical operations (analytics logging) so a logging failure does not break the conversation
- **Error Output**: route errors to a dedicated error-handling path

### Workflow-Level Error Handling

- Dedicated **Error Workflow** triggered by the Error Trigger node
- Captures: workflow name, node that failed, error message, input data
- Routes errors by severity:
  - Critical (agent completely failed) → immediate Telegram alert to admin
  - Warning (TTS failed, fell back to text-only) → log for review
  - Info (analytics insert failed) → log silently

### Fallback Behavior

- If STT fails → ask client to type their message instead
- If TTS fails → send text-only response (no audio)
- If primary LLM is down → fallback model in AI Agent node takes over
- If Telegram API rate-limits → queue messages with backoff

### Graceful Degradation

The chatbot should never leave a client without a response. Every error path ends with a friendly message: "I'm having a temporary issue. Let me connect you with a team member." → trigger HITL handoff.

**Research sources**: `04-production-operations.md` §4 (monitoring and alerting, error handling architecture), `01-agent-patterns-and-optimization.md` §1.6 (reliability patterns)

---

## 14. Evaluation and Prompt Improvement

### Manual Phase (MVP)

During early operation with low volume:
1. Human reviews conversation logs weekly
2. Identifies failure patterns (wrong area classification, missed qualifying questions, awkward responses)
3. Updates system prompt manually based on findings
4. Tests changes against golden dataset before deploying

### Automated Phase (Post-MVP)

When volume justifies it, implement the meta-prompting cycle:

```
Weekly scheduled workflow:
  → Pull conversations from chat_analytics (PostgreSQL)
  → Filter: low lead scores, escalated conversations, long conversations
  → n8n Evaluation node scores each conversation:
      - Did the agent identify the correct practice area?
      - Did it collect all required information?
      - Was the tone appropriate?
      - Was the lead correctly qualified?
  → Low-scoring conversations sent to a stronger LLM (Claude Sonnet/Opus)
  → Stronger LLM analyzes patterns and rewrites the system prompt
  → New prompt A/B tested: 80% old prompt, 20% new
  → After N conversations, compare metrics
  → Winner promoted to production
```

### Golden Dataset

Build a curated set of test conversations covering:
- Each practice area (at least 3 conversations each)
- Edge cases: ambiguous area, unqualified lead, aggressive client, out-of-scope question
- Returning clients with existing profiles
- Voice-originated messages (transcription artifacts)

Use n8n's **Light Evaluations** during development and **Metric-Based Evaluations** in production.

**Research sources**: `01-agent-patterns-and-optimization.md` §5 (auto-evaluation, peer review), §6 (meta-prompting techniques), `04-production-operations.md` §3 (QA workflows, built-in evaluations, LLM-as-judge)

---

## 15. Monitoring, Analytics, and Reporting

### Real-Time Monitoring

- n8n Insights dashboard: execution success/failure rates, response times
- Error workflow sends critical alerts to Telegram admin group
- Redis counter anomaly detection: sudden traffic spikes trigger a warning

### Chat Analytics

All conversations logged in `chat_analytics` table. Key metrics:

| Category | Metrics |
|----------|---------|
| Operational | Response time (avg, p95), error rate, conversation volume, escalation rate |
| Quality | Resolution rate (no escalation), LLM-as-judge scores, hallucination rate |
| Business | Cost per conversation, leads qualified per day, human time saved, practice area distribution |

### Weekly Reports

Scheduled workflow generates and delivers reports:

1. **Schedule Trigger** (Monday 9 AM)
2. **PostgreSQL** node runs aggregate queries (total conversations, escalation rate, cost, top areas)
3. **Code node** formats into a structured report
4. **Telegram** node sends to the admin group

### Dashboard (Future)

Metabase (already available in BorgStack cluster mode) for business metrics. Grafana for operational monitoring. Not needed for MVP — weekly report workflow is sufficient.

**Research sources**: `04-production-operations.md` §2 (chat reporting, PostgreSQL schema, aggregate queries, dashboard options), §4 (monitoring and alerting)

---

## 16. Cost Tracking and Optimization

### Token Tracking

Log `tokens_used` in `chat_analytics` for every LLM call. Track input and output tokens separately.

### Cost Reduction Strategies

1. **Deterministic pre-filters**: handle 15-25% of messages without any LLM call
2. **Cheap model**: GPT-4o-mini / Haiku for the triage agent (~$0.15/M input tokens)
3. **Context window limit**: 10-15 messages prevents runaway token costs
4. **Context compaction**: summarize old messages instead of keeping them all
5. **Prompt caching**: if the LLM provider supports it, cache the system prompt prefix (90% discount)
6. **Response caching**: for repeated FAQ-like questions that reach the agent, consider caching common responses in Redis

### Budget Alerts

Scheduled workflow checks weekly token consumption:
- Warning at 80% of monthly budget → Telegram alert
- Critical at 100% → downgrade model or increase deterministic coverage

**Research sources**: `04-production-operations.md` §5 (cost tracking, budget alerts, caching strategies), `01-agent-patterns-and-optimization.md` §2.5 (30x cost reduction)

---

## 17. Conversation Lifecycle

### Session Management

- Session ID: Telegram `chat.id` (persists across conversations for the same user)
- New session detection: if no messages from user in >24 hours, treat as a new conversation topic (but keep profile data)
- Postgres Chat Memory persists indefinitely until explicitly cleared

### Conversation States

```
NEW → client just started talking
QUALIFYING → agent is asking questions, gathering info
QUALIFIED → lead scored, ready for handoff
HANDED_OFF → attorney notified, waiting for response
ACTIVE_HUMAN → human has taken over the conversation
CLOSED → conversation ended (qualified or not)
```

State stored in `client_profiles.status` column. The Check Assignment pattern prevents the bot from interfering when a human is actively handling the conversation.

### Graceful Endings

- After handoff: "An attorney from our team will reach out to you. Is there anything else I can help with?"
- After decline: "I understand. If your situation changes, feel free to reach out anytime."
- Inactivity timeout: if no message for 30 minutes during qualification, send a gentle follow-up. After 24h of no response, close the conversation with a warm message.

### Returning Clients

Query `client_profiles` on every new message. If a profile exists:
- Inject profile data into system prompt context
- Agent greets by name: "Welcome back, [Name]! How can I help today?"
- Skip already-answered qualifying questions

**Research sources**: `04-production-operations.md` §6 (conversation lifecycle, session strategies, state machines), `02-memory-rag-and-context.md` §2.3 (user profile persistence)

---

## 18. Testing and Staging

### Development Testing

- Use Telegram **test bot** (separate bot token via BotFather)
- n8n test webhook URLs (`/webhook-test/`) for manual testing
- **Pin Data** feature: freeze node outputs for reproducible testing
- Test each layer independently: router regex, agent responses, handoff flow

### Evaluation Testing

- Build golden dataset with representative conversations
- Run n8n **Light Evaluations** to compare agent output against expected answers
- Test system prompt changes against the golden dataset before deploying

### Staging Environment

- Separate n8n workflow with test bot token and test database tables
- Same PostgreSQL instance, different table names (prefix `test_`)
- Or: dedicated staging workflow on the same n8n instance

### Pre-Production Checklist

- [ ] All deterministic routes tested with representative messages
- [ ] Agent correctly identifies all 8 practice areas
- [ ] Lead scoring produces reasonable results across test cases
- [ ] HITL handoff notification arrives with complete data
- [ ] STT works for Portuguese voice messages
- [ ] TTS generates intelligible audio responses
- [ ] Rate limiting blocks excessive messages
- [ ] Guardrails catch prompt injection attempts
- [ ] Error workflow fires on simulated failures
- [ ] Returning client is recognized and greeted by name

**Research sources**: `04-production-operations.md` §7 (testing and staging, webhook URLs, pin data, regression testing, pre-activation checklist)

---

## 19. Database Schema

```sql
-- Client profiles (long-term memory, Tier 2)
CREATE TABLE client_profiles (
  id SERIAL PRIMARY KEY,
  telegram_id BIGINT UNIQUE NOT NULL,
  name VARCHAR(255),
  phone VARCHAR(50),
  city VARCHAR(100),
  practice_area VARCHAR(50),
  case_summary TEXT,
  urgency VARCHAR(20) DEFAULT 'normal',
  lead_score INTEGER DEFAULT 0,
  status VARCHAR(20) DEFAULT 'new',
  handed_off_at TIMESTAMPTZ,
  assigned_attorney VARCHAR(255),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Chat analytics (reporting and monitoring)
CREATE TABLE chat_analytics (
  id SERIAL PRIMARY KEY,
  session_id VARCHAR(100) NOT NULL,
  telegram_id BIGINT NOT NULL,
  message_role VARCHAR(20) NOT NULL,
  message_content TEXT,
  model_used VARCHAR(50),
  tokens_input INTEGER,
  tokens_output INTEGER,
  response_time_ms INTEGER,
  was_audio BOOLEAN DEFAULT FALSE,
  escalated BOOLEAN DEFAULT FALSE,
  error_occurred BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_analytics_session ON chat_analytics(session_id);
CREATE INDEX idx_analytics_created ON chat_analytics(created_at);
CREATE INDEX idx_analytics_telegram ON chat_analytics(telegram_id);

-- n8n Postgres Chat Memory table is auto-created by the memory node
```

**Research sources**: `04-production-operations.md` §2.2 (analytics schema), `02-memory-rag-and-context.md` §2.1-2.3 (memory tables, user profiles)

---

## 20. Project Structure

```
n8n-chatbot/
├── CLAUDE.md                              # Project conventions
├── research/                              # Deep research (completed)
│   ├── 01-agent-patterns-and-optimization.md
│   ├── 01b-agent-patterns-supplementary.md
│   ├── 02-memory-rag-and-context.md
│   ├── 03-media-integrations-and-security.md
│   └── 04-production-operations.md
├── docs/                                  # Architecture and decisions
│   ├── plan.md                            # This document
│   └── system-prompts/
│       └── triage-agent-v1.md             # Versioned system prompt
├── workflows/                             # n8n workflow JSON exports
│   ├── main-chatbot.json                  # Primary chatbot workflow
│   └── sub-workflows/
│       ├── media-processor.json           # STT/TTS pipeline
│       ├── save-client-info.json          # Client data persistence
│       ├── lead-handoff.json              # HITL notification
│       └── error-handler.json             # Error routing workflow
├── sql/                                   # Database migrations
│   └── 001-init-tables.sql                # client_profiles + chat_analytics
└── config/
    └── telegram-bot-setup.md              # Bot creation guide
```

---

## 21. Delivery Phases

### Phase 1 — MVP: Text-Only Triage on Telegram

- Telegram Trigger + deterministic router + AI triage agent
- Postgres Chat Memory + client_profiles table
- HITL handoff to attorney via Telegram group
- Security: input sanitization, guardrails, rate limiting
- Analytics logging in chat_analytics table
- Error handling workflow
- Weekly report workflow

### Phase 2 — Bidirectional Audio

- Groq Whisper STT for incoming voice messages
- Comadre TTS for outgoing audio responses
- Full voice conversation flow
- Audio flag tracking in analytics

### Phase 3 — WhatsApp Migration

- Evolution API webhook integration
- Message normalization layer (unified schema)
- Same triage agent logic, different I/O adapter
- Handle WhatsApp-specific features (interactive lists, buttons)

### Phase 4 — Advanced Features

- Appointment scheduling (Google Calendar or custom)
- Case status queries (integration with firm's case management)
- Document collection and validation (receive docs via chat, confirm receipt)
- Active follow-ups (scheduled reminders for pending documents)
- Commemorative date messages (birthdays, case milestones)
- Targeted marketing content to validated client segments
- Evaluation automation (meta-prompting cycle)
- RAG knowledge base (firm's legal FAQ, procedure guides)

---

## 22. Research References

This plan draws from the following research files, all located in `research/`:

| File | Key Topics Used |
|------|-----------------|
| `01-agent-patterns-and-optimization.md` | Multi-agent patterns (§1), Model Selector node and tiered routing (§2), deterministic pre-filters and regex patterns (§3), system prompt structure and guardrails (§4), auto-evaluation and peer review (§5), meta-prompting cycle (§6), data preprocessing pipeline (§7), context engineering strategies (§8) |
| `01b-agent-patterns-supplementary.md` | Orchestrator with AI Agent Tool pattern (§1.3), Call n8n Workflow Tool for sub-workflows (§1.5), Text Classifier + Switch routing (§1.4), memory and context sharing between agents (§1.7), cost-aware routing (§2.4), FAQ lookup without LLM (§3.4), prompt maintenance and versioning (§4.9), dynamic context injection with expressions (§4.8) |
| `02-memory-rag-and-context.md` | Memory sub-nodes comparison and queue mode limitation (§1), Postgres Chat Memory config (§2.1), hybrid three-tier memory architecture (§2.2), user profile persistence patterns (§2.3), RAG architecture with pgvector (§3), context rot mechanisms and prevention (§4), context compaction with Chat Memory Manager (§5), knowledge base management (§6) |
| `03-media-integrations-and-security.md` | WhatsApp via Evolution API (§1.1), Telegram nodes and constraints (§1.3), multi-platform normalization schema (§1.8), STT with Whisper/Groq (§2.1, §3.2), TTS integration with OpenAI-compatible API (§3.1), bidirectional audio flow (§3.3), n8n Guardrails system with 10 validation types (§4.1), prompt injection prevention (§4.2), rate limiting patterns (§4.3), PII detection (§4.5), OWASP LLM Top 10 (§4.8), Cloudflare security (§4.9), CVE-2025-68613 (§4.10), Evolution API webhooks and message endpoints (§5) |
| `04-production-operations.md` | HITL native features and approval patterns (§1.1-1.2), escalation and handoff patterns (§1.4), chat analytics schema and aggregate queries (§2.2-2.3), n8n Evaluations system and LLM-as-Judge (§3.1-3.2), A/B testing prompts (§3.4), error handling architecture (§4), cost tracking approaches and budget alerts (§5), conversation lifecycle and session management (§6), testing/staging with webhook URLs and pin data (§7) |
