# Agent Patterns, LLM Routing, and Optimization

> Research compiled April 2026. Covers multi-agent architectures, smart LLM routing, deterministic filters, system prompt engineering, auto-evaluation, meta-prompting, data preprocessing, and context engineering for n8n chatbots.

---

## Table of Contents

1. [Multi-Agent Architectures in n8n](#1-multi-agent-architectures-in-n8n)
2. [Smart LLM Routing and Model Right-Sizing](#2-smart-llm-routing-and-model-right-sizing)
3. [Deterministic Pre-Filters (Code Before LLM)](#3-deterministic-pre-filters-code-before-llm)
4. [System Prompt Engineering](#4-system-prompt-engineering)
5. [Auto-Evaluation and Peer-LLM Review](#5-auto-evaluation-and-peer-llm-review)
6. [Meta-Prompting: Stronger LLMs Improving Weaker Ones](#6-meta-prompting-stronger-llms-improving-weaker-ones)
7. [Data Preprocessing for LLM Consumption](#7-data-preprocessing-for-llm-consumption)
8. [Context Engineering Strategies](#8-context-engineering-strategies)

---

## 1. Multi-Agent Architectures in n8n

### 1.1 Core Orchestration Patterns

n8n supports several multi-agent coordination patterns:

| Pattern | Description | Best For |
|---------|-------------|----------|
| **Supervisor/Hierarchical** | Central coordinator agent delegates to specialized sub-agents | Complex assistants spanning multiple domains |
| **Router/Classifier** | Classifier node categorizes input → Switch node routes to specialized branch | Distinct, non-overlapping task categories |
| **Sequential/Pipeline** | Agents process in stages, each building on previous output | Multi-step refinement workflows |
| **Parallel** | Multiple agents work simultaneously, results combined | Independent tasks needing aggregation |
| **Handoff** | Specialized agents pass context between stages dynamically | Context-dependent delegation |

### 1.2 Supervisor Pattern (Recommended for Chatbots)

The most practical pattern for chatbot use cases:

- **AI Agent node** acts as the central coordinator with Simple Memory for conversation context
- Specialized **sub-agents** handle domain-specific tasks (email, search, billing, etc.)
- Sub-agents report results back to the supervisor unidirectionally
- The supervisor synthesizes results and responds to the user

**Implementation in n8n:**
1. Main AI Agent node as coordinator hub
2. Enable Simple Memory (or Postgres Chat Memory) on the main agent
3. Connect sub-agents as tools via the **AI Agent Tool** node
4. Each sub-agent has its own tool set and system prompt

### 1.3 Router Pattern (Cost-Effective Alternative)

For simpler routing where agents don't need to collaborate:

1. **Classifier AI node** categorizes incoming request by intent
2. **Switch node** routes to the appropriate specialized sub-workflow
3. Each branch contains a focused agent with its own tools
4. No shared memory between branches (each is independent)

This pattern is lighter weight than the supervisor pattern and works well when task categories are clearly separable.

### 1.4 Agent Specialization Best Practices

- **Single Responsibility Principle**: each agent handles one clear responsibility
- **5-7 tools per agent** is the sweet spot — beyond that, performance degrades as the LLM must evaluate all tool options for every request, consuming tokens and increasing latency
- **Workflow-as-tools**: wrap multi-step operations in sub-workflows for reuse across agents
- **Dynamic parameters**: tool parameter fields are filled by the LLM at runtime

### 1.5 Context Transfer Between Agents

Two strategies for passing context:

1. **Shared Memory**: add critical intermediate results to shared memory so other agents can access them (via Memory Manager node)
2. **Identifier Passing**: for large data, pass file IDs or URLs instead of full content

### 1.6 Reliability Patterns

- **Validation checkpoints** between agents for critical operations
- **QA validator agent**: checks for hallucinations and policy conflicts before human delivery
- **Execution metadata**: trace/correlation IDs, input classification labels, which agents participated, tool invocation records
- **Error containment**: design so changes to one specialist agent don't cascade
- **Modular testing**: test changes to individual agents in isolation

### 1.7 Parallelization

n8n multi-agent systems work **sequentially by default**. For parallel execution, use:
- **Execute Sub-Workflow node** for parallel agent invocations
- **HTTP Request tool node** for async agent calls

---

## 2. Smart LLM Routing and Model Right-Sizing

### 2.1 The Model Selector Node

n8n provides a dedicated **Model Selector node** for dynamic model routing:

- Connect up to **10 different LLM providers** (OpenAI, Claude, Mistral, Gemini, Azure)
- Output feeds directly into an AI Agent node's Chat Model input
- Two configuration modes:
  - **Rule-Based Selection**: conditional statements matching string values
  - **Index-Based Selection**: returns a numerical index (1-10) for model position

**Cost optimization example with token estimation:**
```javascript
const tokenCount = input.length / 4;
if (tokenCount > 1000) return 1; // GPT-4 (complex)
if (tokenCount > 300) return 2;  // Claude Sonnet (medium)
return 3; // Mistral/Haiku (cheap, simple)
```

### 2.2 Tiered Model Strategy

| Tier | Model Examples | Use Case | Cost |
|------|---------------|----------|------|
| **Zero Cost** | Rule-based code, regex | FAQ matching, intent keywords, data validation | $0 |
| **Cheap/Fast** | Haiku, GPT-4o-mini, Gemini Flash Lite, Mistral | Intent classification, simple extraction, routing | Very low |
| **Mid-Range** | Claude Sonnet, GPT-4o, Gemini Flash | General conversation, summarization | Moderate |
| **Premium** | Claude Opus, GPT-4, o1 | Complex reasoning, nuanced responses, meta-prompting | High |

### 2.3 Centralized Routing Layer

A routing layer acts as a "LLM switchboard" that every AI node calls:

- Abstracts the provider and model selection
- Enforces cost controls and budget limits
- Provides audit trail for all LLM calls
- Enables easy model swaps without changing individual workflows

### 2.4 Model Selection for Multi-Agent Systems

- **Expensive reasoning models** for the planning logic of the main/supervisor agent
- **Cheaper models** for simple sub-agent operations (extraction, classification, formatting)
- **No model at all** for deterministic tasks (regex, code, lookup tables)

### 2.5 Cost Impact

Practitioners report up to **30x cost reduction** by combining:
- Model tiering (right model for each task)
- Deterministic pre-filters (no LLM for simple queries)
- Prompt caching (90% discount on cached prefixes)
- Response caching (semantic deduplication)

---

## 3. Deterministic Pre-Filters (Code Before LLM)

### 3.1 Why Filters Matter

Every LLM call costs money and adds latency. Many chatbot interactions can be resolved without AI:

- Greetings and pleasantries → static responses
- FAQ questions → keyword/pattern matching to canned answers
- Data validation → regex and code
- Status queries → database lookup
- Menu navigation → decision trees

### 3.2 Implementation in n8n

The decision flow before hitting an LLM:

```
User Message
    ↓
[Code Node: normalize text, strip HTML, lowercase]
    ↓
[Switch Node: keyword/regex matching]
    ├── matches greeting pattern → static response
    ├── matches FAQ keyword → lookup response from database
    ├── matches status query → PostgreSQL query → formatted response
    ├── matches menu command → deterministic routing
    └── no match → AI Agent (LLM call)
```

**n8n nodes involved:**
- **Code node**: JavaScript/Python for text normalization and pattern matching
- **Switch node**: conditional routing based on regex or string matches
- **Filter node**: supports regex matching on fields
- **IF node**: simple boolean conditions
- **PostgreSQL node**: FAQ/knowledge lookup
- **Set node**: construct static responses

### 3.3 Regex Patterns for Common Intents

```javascript
// In a Code node before the Switch
const msg = $input.first().json.message.toLowerCase().trim();

const patterns = {
  greeting: /^(oi|olá|ola|bom dia|boa tarde|boa noite|hey|hello|hi)\b/i,
  thanks: /^(obrigad[oa]|valeu|thanks|thank you)\b/i,
  bye: /^(tchau|até mais|bye|adeus)\b/i,
  status: /^(status|pedido|rastreio|tracking)\s*(#?\d+)/i,
  menu: /^(menu|ajuda|help|opções|opcoes)\b/i,
};

let intent = 'unknown';
let extractedData = {};

for (const [key, pattern] of Object.entries(patterns)) {
  const match = msg.match(pattern);
  if (match) {
    intent = key;
    extractedData = { match: match[0], groups: match.slice(1) };
    break;
  }
}

return { json: { intent, extractedData, originalMessage: msg } };
```

### 3.4 Cost Savings

Routing 10-20% of total calls through zero-cost deterministic paths creates a permanent, non-linear reduction in total token spend. These are typically the highest-volume interactions (greetings, simple FAQs).

---

## 4. System Prompt Engineering

### 4.1 From Prompt Engineering to Context Engineering

The 2025-2026 paradigm shift: the question is no longer "how do I craft the perfect prompt" but "which configuration of context leads to the desired behavior." System prompts are one component of a broader context engineering strategy.

### 4.2 System Prompt Structure

A well-structured system prompt for an n8n chatbot agent:

```
[IDENTITY]
You are {name}, a {role} for {company}.

[CAPABILITIES]
You can:
- {capability 1}
- {capability 2}
You cannot:
- {limitation 1}

[TOOLS]
You have access to the following tools:
- {tool_name}: {when and how to use it}

[BEHAVIOR RULES]
1. Always {rule}
2. Never {rule}
3. If uncertain, {fallback behavior}

[OUTPUT FORMAT]
Respond in {format}. Keep responses under {length}.

[GUARDRAILS]
- Do not discuss {topic}
- Do not share {sensitive info}
- If asked about {out of scope}, say {redirect message}

[EXAMPLES] (optional)
User: {example input}
Assistant: {example output}
```

### 4.3 Key Principles

- **Mention tools explicitly** in the system prompt — when you attach tools to an n8n AI agent, the LLM needs to know when and how to use them
- **Numbered rules** for constraints — models follow numbered lists more reliably than prose
- **Specific rejection language** — tell the model exactly what to reject and how
- **Output format constraints** — define JSON schemas, length limits, tone requirements
- **Topical alignment** — define the business scope to keep conversations on-topic (n8n's Guardrails node supports this natively)

### 4.4 n8n Guardrails Node

The built-in Guardrails node provides 10 validation types:

| Type | Purpose |
|------|---------|
| Keywords | Block/flag specific words |
| PII | Detect personal data |
| Secret Keys | Detect API keys, passwords |
| URLs | Control link sharing |
| Custom Regex | Pattern-based validation |
| Jailbreak Detection | Detect prompt injection attempts |
| NSFW | Content moderation |
| Topical Alignment | Keep conversation in scope |
| Custom LLM | LLM-based validation with custom criteria |

Can be applied to both **input** (before AI processing) and **output** (before sending to user).

### 4.5 Versioning and Testing System Prompts

- Store system prompts as variables or in a database, not hardcoded in the workflow
- Version control prompt changes with meaningful descriptions
- Use n8n's Evaluation framework to test prompt changes against golden datasets
- A/B test different prompts by routing a percentage of traffic to each variant

---

## 5. Auto-Evaluation and Peer-LLM Review

### 5.1 n8n's Built-in Evaluation System

Available since n8n v1.95.1+, the Evaluation node supports:

- **LLM-as-a-Judge**: capable models (GPT-5, Claude Sonnet) evaluate outputs from smaller models
- **Correctness (AI-based)**: scores whether answer meaning aligns with reference answers (1-5 scale)
- **Helpfulness (AI-based)**: scores response utility
- **Tools Used**: verifies agents correctly invoke external tools
- **Categorization**: returns 1 for matches, 0 for mismatches
- **Custom Metrics**: define your own scoring criteria
- **Deterministic Metrics**: token count, execution time (tracked automatically)

### 5.2 Evaluation Workflow Architecture

1. **Data Tables**: store ground truth test cases in n8n's database-like structure
2. **Check if Evaluating node**: creates separate paths for evaluation vs production (prevents "test pollution")
3. **Set Metrics node**: maps workflow outputs to result columns and computes metrics
4. **Loop Processing**: iterate through test datasets, capturing performance data

### 5.3 Peer Review System (Multi-Model Consensus)

n8n provides a workflow template (ID: 11660) for multi-model peer review:

- Multiple AI models independently answer the same question
- Each model then reviews and scores the other models' responses
- A final arbiter LLM synthesizes the best answer from all reviews
- The peer review prompt can be customized with domain-specific evaluation criteria

### 5.4 Feedback Loops

- QA validator agents assess output quality before delivery
- Thumbs-up/thumbs-down from users collects satisfaction signals
- These signals feed into prompt refinement and tool access adjustments
- Comparative questions ("Which response is better?") produce more consistent feedback than absolute scoring (1-10 scales)

### 5.5 Continuous Improvement Cycle

```
Agent produces response
    ↓
Evaluator LLM scores response (correctness, tone, safety)
    ↓
Scores stored in PostgreSQL
    ↓
Periodic batch analysis of low-scoring responses
    ↓
Stronger LLM analyzes failure patterns
    ↓
System prompt updated based on findings
    ↓
A/B test new prompt against old
    ↓
Deploy winning prompt
```

---

## 6. Meta-Prompting: Stronger LLMs Improving Weaker Ones

### 6.1 Core Concept

Meta-prompting is using prompts to guide, structure, and optimize other prompts. A stronger model generates or refines prompts that weaker (cheaper) models will use for downstream tasks.

### 6.2 Why It Works

A strong model analyzing and rewriting prompts produces prompts that are clearer, more structured, and more precise. This clarity helps the target model (a less capable or cheaper model) follow instructions more effectively, generating more relevant and accurate responses.

**Real-world example**: Google DeepMind uses Gemini (stronger model) to create detailed prompts for Veo (image/video generator), producing multi-page, richly-specified prompts that yield far better outputs than hand-written ones.

### 6.3 Techniques

| Technique | How It Works | Trade-off |
|-----------|-------------|-----------|
| **Automatic Prompt Engineer (APE)** | Generate candidate prompts → evaluate on test set → iterate with top performers | Finds non-obvious solutions; computationally expensive |
| **Recursive Meta Prompting (RMP)** | LLM creates its own reasoning template → applies it to solve the problem | Adapts to new tasks; risks circularity |
| **Search + Learning** | Algorithmic generation, scored on test cases, top performers seed next iteration | Best results; highest compute cost |
| **Multi-Stage Orchestration** | Higher-level prompts guide which lower-level prompts are generated | Modular and composable |
| **Hybrid (Human + AI)** | Experts provide rough templates; LLMs augment and refine | Combines domain knowledge with optimization |

### 6.4 Implementation Pattern for n8n

```
Worker Agent (cheap model) handles conversations
    ↓
Evaluation workflow scores responses periodically
    ↓
Low-scoring conversations flagged
    ↓
Stronger LLM (Opus/GPT-4) analyzes failure patterns
    ↓
Stronger LLM rewrites/refines worker's system prompt
    ↓
New prompt A/B tested against current
    ↓
Winning prompt deployed to worker agent
    ↓
Cycle repeats
```

**n8n implementation:**
1. **Scheduled trigger** (daily/weekly) pulls conversation logs from PostgreSQL
2. **Code node** filters for low-scoring or flagged conversations
3. **LLM Chain** with premium model analyzes patterns and generates improved prompt
4. **Compare node** diffs old vs new prompt
5. **Human approval step** (optional) before deployment
6. **Set node** updates the system prompt variable/database record

### 6.5 Evaluation Metrics for Prompt Improvement

- Performance benchmarks (accuracy on test cases)
- Token efficiency (fewer input tokens for same output quality)
- Cost per success (meta-prompting at $0.0003 vs chain-of-thought at $0.47 per task)
- Consistency across task types

### 6.6 Security Consideration

If LLMs generate prompts, they could theoretically generate adversarial or jailbreak prompts. Guardrails and human review steps are essential in the meta-prompting pipeline.

---

## 7. Data Preprocessing for LLM Consumption

### 7.1 Why Preprocess

Raw user input is noisy: HTML tags, excessive whitespace, special characters, irrelevant metadata. Cleaning and structuring input before the LLM reduces token usage, improves response quality, and lowers costs.

### 7.2 n8n Nodes for Preprocessing

| Node | Purpose |
|------|---------|
| **HTML node** | Clean Up Text: remove leading/trailing whitespace, line breaks, condense multiple spaces |
| **Code node** | Custom JavaScript/Python for complex transformations |
| **HTML Extract node** | Extract specific content from HTML using CSS selectors |
| **Markdown node** | Convert HTML to Markdown (reduces tokens significantly) |
| **Set node** | Augment input with additional context fields |
| **n8n-nodes-text-manipulation** | Community node for advanced text operations |
| **n8n-nodes-html-cleaner** | Community node for HTML cleaning |

### 7.3 Preprocessing Pipeline

```javascript
// Code node: preprocess user input
const raw = $input.first().json.message;

// 1. Strip HTML tags
const noHtml = raw.replace(/<[^>]*>/g, '');

// 2. Normalize whitespace
const normalized = noHtml.replace(/\s+/g, ' ').trim();

// 3. Remove zero-width characters
const clean = normalized.replace(/[\u200B-\u200D\uFEFF]/g, '');

// 4. Truncate if too long (prevent token abuse)
const maxChars = 4000;
const truncated = clean.length > maxChars
  ? clean.substring(0, maxChars) + '...[truncated]'
  : clean;

// 5. Extract metadata
const hasUrl = /https?:\/\/\S+/.test(truncated);
const hasEmail = /\S+@\S+\.\S+/.test(truncated);
const language = /[à-ú]/.test(truncated) ? 'pt-BR' : 'en';
const charCount = truncated.length;
const estimatedTokens = Math.ceil(charCount / 4);

return {
  json: {
    processedMessage: truncated,
    metadata: { hasUrl, hasEmail, language, charCount, estimatedTokens }
  }
};
```

### 7.4 Format Conversion for Token Efficiency

Converting scraped HTML to Markdown can **significantly reduce token usage**:
- Use n8n's Markdown node or services like Firecrawl.dev
- HTML → Markdown conversion removes styling, scripts, and structural markup
- Only semantic content reaches the LLM

### 7.5 Context Augmentation

Use a Set node or Code node to augment user input with relevant context before the LLM:
- Current date/time
- User profile information (from database)
- Conversation stage/state
- Recently retrieved RAG results

---

## 8. Context Engineering Strategies

### 8.1 The Nine Strategies

Based on current best practices for n8n AI agents:

1. **Short-Term Memory** — recent interactions stored in Simple Memory or Postgres Chat Memory
2. **Long-Term Memory** — persistent facts stored across sessions (database, Google Docs)
3. **Context Expansion via Tool Calling** — dynamically inject live data from external tools
4. **RAG** — vectorized documents retrieved semantically from vector stores
5. **Context Isolation** — split responsibilities across specialized sub-agents with separate context windows
6. **Summarizing Context** — compress raw data through summarization workflows before feeding to agent
7. **Deep Research Blueprint** — manage long-running, data-heavy tasks through deterministic context routing
8. **Formatting Context** — convert data to AI-friendly formats (HTML → Markdown) to reduce tokens
9. **Trimming Context** — limit text length or chunk quantities to fit token constraints

### 8.2 Context Isolation (Critical for Multi-Agent)

Each sub-agent manages its own context window and memory, preventing **context pollution** in the primary agent. This is essential when:
- Different tasks require different context (e.g., billing vs support)
- Context from one domain could confuse another agent
- Token limits would be exceeded by combining all context

### 8.3 Context Trimming Patterns

```javascript
// Limit to first N characters
const trimmed = $json.content.substring(0, 1000);

// Limit RAG chunks returned
// In Vector Store node: set Top K to 3-5 instead of default 10

// Limit memory window
// In Simple Memory node: set Context Window Length to 10-15
```

### 8.4 Key Principle

Proper context engineering prevents:
- **Context poisoning**: irrelevant data degrading response quality
- **Attention dilution**: too much context causing the model to miss important details
- **Contradictory information**: conflicting facts confusing the model
- **Token waste**: paying for context the model doesn't use

---

## Sources

### Multi-Agent Architecture
- [Multi-agent system: Frameworks & tutorial — n8n Blog](https://blog.n8n.io/multi-agent-systems/)
- [Multi Agent Solutions in n8n — HatchWorks](https://hatchworks.com/blog/ai-agents/multi-agent-solutions-in-n8n/)
- [n8n Multi-Agent Orchestration — LogicWorkflow](https://logicworkflow.com/blog/n8n-multi-agent-orchestration/)
- [AI Agent Architectures: Ultimate Guide — Product Compass](https://www.productcompass.pm/p/ai-agent-architectures)

### LLM Routing & Cost Optimization
- [Model Selector Node Guide — AutomateGeniusHub](https://automategeniushub.com/guide-to-use-the-n8n-model-selector-node/)
- [Cost Optimization Guide for n8n AI Workflows — Clixlogix](https://www.clixlogix.com/cost-optimization-guide-for-n8n-ai-workflows/)
- [Dynamic AI Model Router — n8n Template #4237](https://n8n.io/workflows/4237-dynamic-ai-model-router-for-query-optimization-with-openrouter/)
- [Compare LLM Token Costs — n8n Template #12100](https://n8n.io/workflows/12100-compare-llm-token-costs-across-350-models-with-openrouter/)

### System Prompts & Guardrails
- [AI Agent Prompting Best Practices 2025 — Medium/Automation Labs](https://medium.com/automation-labs/ai-agent-prompting-for-n8n-the-best-practices-that-actually-work-in-2025-8511c5c16294)
- [n8n AI Agents Tutorial: System & User Prompts — Width.ai](https://www.width.ai/post/n8n-ai-agents-tutorial-master-system-user-prompts-2026)
- [Guardrails Node Documentation — n8n Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.guardrails/)
- [15 Best Practices for Deploying AI Agents — n8n Blog](https://blog.n8n.io/best-practices-for-deploying-ai-agents-in-production/)

### Auto-Evaluation
- [Building Your Own LLM Evaluation Framework — n8n Blog](https://blog.n8n.io/llm-evaluation-framework/)
- [Introducing Evaluations for AI Workflows — n8n Blog](https://blog.n8n.io/introducing-evaluations-for-ai-workflows/)
- [Peer Review System — n8n Template #11660](https://n8n.io/workflows/11660-generate-consensus-answers-with-multiple-ai-models-and-peer-review-system/)
- [How to Stop AI Agents from Hallucinating — LogRocket](https://blog.logrocket.com/stop-your-ai-agents-from-hallucinating-n8n/)

### Meta-Prompting
- [Meta-Prompting: LLMs Crafting Their Own Prompts — IntuitionLabs](https://intuitionlabs.ai/articles/meta-prompting-llm-self-optimization)
- [Automated LLM Prompt Engineering — IntuitionLabs](https://intuitionlabs.ai/articles/meta-prompting-automated-llm-prompt-engineering)
- [Meta Prompting: Use LLMs to Optimize Prompts — Comet](https://www.comet.com/site/blog/meta-prompting/)
- [Meta Prompting — Prompting Guide](https://www.promptingguide.ai/techniques/meta-prompting)
- [System Prompt Optimization with Meta-Learning — arXiv](https://arxiv.org/html/2505.09666v1/)

### Context Engineering
- [9 Context Engineering Strategies for n8n — The AI Automators](https://www.theaiautomators.com/context-engineering-strategies-to-build-better-ai-agents/)
- [AI Agentic Workflows Guide — n8n Blog](https://blog.n8n.io/ai-agentic-workflows/)
