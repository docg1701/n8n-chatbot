# n8n Chatbot Architecture: Deep Research

> Research compiled April 2026. Covers n8n memory management, RAG, context engineering, and knowledge base patterns using sources from 2024-2026.

---

## Table of Contents

1. [Short-Term Memory (Conversation Context)](#1-short-term-memory-conversation-context)
2. [Long-Term Memory (Persistent Across Sessions)](#2-long-term-memory-persistent-across-sessions)
3. [RAG (Retrieval Augmented Generation)](#3-rag-retrieval-augmented-generation)
4. [Context Rot Prevention](#4-context-rot-prevention)
5. [Context Compaction and Summary Reintroduction](#5-context-compaction-and-summary-reintroduction)
6. [Knowledge Base Management](#6-knowledge-base-management)

---

## 1. Short-Term Memory (Conversation Context)

Short-term memory keeps a history of recent messages so the AI can maintain an ongoing conversation rather than treating every interaction as a fresh start. In n8n, memory is attached to the AI Agent node as a sub-node.

### 1.1 Memory Sub-Nodes Available in n8n

n8n provides several memory sub-nodes, each with different storage backends:

| Node | Backend | Persistence | Best For |
|------|---------|-------------|----------|
| **Simple Memory** (formerly Window Buffer Memory) | In-process (RAM) | Volatile -- lost on restart or save | Testing, prototypes |
| **Postgres Chat Memory** | PostgreSQL table | Persistent across restarts | Production workloads |
| **Redis Chat Memory** | Redis key-value store | Persistent with TTL support | Real-time, high-frequency apps |
| **MongoDB Chat Memory** | MongoDB collection | Persistent | Document-oriented stacks |
| **Motorhead** | Motorhead memory server | Persistent | External memory service |
| **Zep** | Zep AI memory service | Persistent | Managed memory with enrichment |
| **Xata** | Xata database | Persistent | Serverless stacks |

Sources:
- [n8n: What's memory in AI?](https://docs.n8n.io/advanced-ai/examples/understand-memory/)
- [Simple Memory node docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorybufferwindow/)

### 1.2 Simple Memory (Buffer Window Memory)

The simplest option. Stores recent chat history in the n8n process memory.

**Key parameters:**

- **Session Key**: Identifies the conversation. Uses the session ID from the Chat Trigger by default to associate memory with specific users or conversation threads. You can set custom session keys to isolate conversations.
- **Session ID Type**: Can be set to "Connected Chat Trigger" (auto), "Custom Key" (manual), or derived from an expression.
- **Context Window Length**: Number of previous interaction pairs (human + AI message = 1 interaction) to retain. Controls how much history the agent sees.

**Limitations:**
- Data disappears when n8n restarts, when the workflow is saved, or when the workflow is deactivated.
- Does **not** work in n8n queue mode, because the runtime cannot guarantee that successive calls reach the same worker process.
- Only suitable for development and testing.

**Recommended context window sizes:**
- 5-10 interactions for simple Q&A bots
- 10-20 interactions for conversational agents (the sweet spot for most use cases)
- Beyond 20 interactions, token costs escalate and stale context degrades quality

Sources:
- [Simple Memory node docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorybufferwindow/)
- [n8n AI Agent Memory Setup Guide](https://towardsai.net/p/machine-learning/n8n-ai-agent-node-memory-complete-setup-guide-for-2026)

### 1.3 Chat Memory Manager Node

For advanced memory manipulation beyond what the memory sub-nodes offer. This node sits in the main workflow (not as a sub-node) and can programmatically manage the contents of any connected memory store.

**Three operation modes:**

1. **Get Many Messages** -- Load messages from memory for inspection or processing.
2. **Insert Messages** -- Add messages to memory. Sub-options:
   - *Insert Messages*: Append alongside existing messages.
   - *Override All Messages*: Replace the entire memory contents.
   - Message types: AI, System, or User.
3. **Delete Messages** -- Remove messages from memory. Sub-options:
   - *Last N*: Delete the N most recent messages.
   - *All Messages*: Wipe the memory entirely.

**Use cases:**
- Check memory size after an Agent response and trim if too long.
- Pre-load a system message or context summary before the agent runs.
- Clear stale context between conversation phases.
- Implement custom summarization logic (load old messages, summarize with an LLM, delete originals, insert summary).

Sources:
- [Chat Memory Manager node docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorymanager/)
- [Chat Memory Manager integrations](https://n8n.io/integrations/chat-memory-manager/)

### 1.4 Session Key Management

Every memory node uses a **session key** to separate conversations. This is critical for multi-user chatbots.

- **Default behavior**: When connected to a Chat Trigger, the session key is automatically set from the trigger's session ID.
- **Webhook/API scenarios**: You must explicitly pass a session ID from your frontend or derive one from a user identifier.
- **Multiple memory instances**: If you add more than one memory node of the same type to a workflow, they share the same memory instance by default. Set different session IDs to create separate memory spaces.
- **Clearing memory**: Use the Chat Memory Manager's Delete mode, or for Postgres, run a direct SQL delete filtered by session ID.

### 1.5 Best Practices for Short-Term Memory

1. **Start with Simple Memory for development**, then switch to Postgres or Redis for production.
2. **Set a reasonable context window length** (10-20 messages). Unlimited history causes token limit errors and increased costs.
3. **Use unique session keys per user** to prevent cross-contamination of conversations.
4. **Monitor token usage** -- each interaction pair added to context increases the token count sent to the LLM on every subsequent call.
5. **Consider Redis for latency-sensitive applications** -- sub-millisecond read/write with TTL for automatic expiration.
6. **If using n8n queue mode**, you must use an external memory backend (Postgres, Redis, etc.), not Simple Memory.

---

## 2. Long-Term Memory (Persistent Across Sessions)

Long-term memory preserves information beyond a single conversation session: user preferences, learned facts, interaction history, and domain knowledge that the chatbot should remember across days, weeks, or months.

### 2.1 PostgreSQL Chat Memory

The production-grade option for persistent conversation history.

**Configuration parameters:**
- **Connection**: Host, port (default 5432), database name, user, password, SSL mode.
- **Table Name**: The table where chat history is stored (e.g., `n8n_chat_histories`). Created automatically if it doesn't exist.
- **Session Key**: Same as Simple Memory -- identifies the conversation thread.
- **Context Window Length**: Number of past interactions to retrieve.

**Auto-created table structure:**
The Postgres Chat Memory node automatically creates a table with columns for:
- Session ID
- Message content (stored as JSONB)
- Message type (human/ai/system)
- Timestamp

**Connection example (Docker):**
When both n8n and PostgreSQL run in Docker containers on the same network, use the PostgreSQL container name as the host (e.g., `pgvector-db`).

Sources:
- [Postgres Chat Memory node docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorypostgreschat/)
- [How to Add PostgreSQL Chat Memory to Your n8n AI Agent](https://itsmepotato.com/2025/08/01/how-to-add-postgresql-chat-memory-to-your-n8n-ai-agent/)

### 2.2 Hybrid Long-Term Memory Architecture

The most capable chatbot implementations combine multiple memory types. A proven pattern from the n8n community uses three tiers:

**Tier 1 -- Temporary Session Memory (Short-Term)**
- Simple Memory or Postgres Chat Memory for the current conversation window.
- Provides immediate conversational context.

**Tier 2 -- Structured SQL Database (Long-Term Facts)**
- A dedicated PostgreSQL table (separate from chat history) storing extracted facts.
- Schema example:
  ```sql
  CREATE TABLE memories (
      id SERIAL PRIMARY KEY,
      memory TEXT NOT NULL,
      date TIMESTAMP DEFAULT NOW(),
      user_id TEXT,
      tags TEXT[]
  );
  ```
- The AI agent decides whether information is worth saving to long-term memory.
- Temporal queries (e.g., "What did we discuss last week?") use SQL with date filtering.

**Tier 3 -- Vector Store (Semantic Long-Term Memory)**
- Every conversation message is also embedded and stored in a vector database.
- Enables semantic search: "What did the user say about their budget?" retrieves relevant past messages by meaning, not keyword.
- Uses a separate `memory_embeddings` table or a vector store like pgvector.

**Retrieval routing logic:**
1. Check if the query is time-related --> use SQL with date filters.
2. Otherwise --> use vector similarity search.
3. Always include the current session memory for immediate context.

Sources:
- [n8n Hybrid Long-Term Memory](https://damiandabrowski.medium.com/day-67-of-100-days-agentic-engineer-challenge-n8n-hybrid-long-term-memory-ce55694d8447)
- [Long Term Memory for LLMs using Vector Store](https://dev.to/einarcesar/long-term-memory-for-llms-using-vector-store-a-practical-approach-with-n8n-and-qdrant-2ha7)

### 2.3 User Profile Persistence

n8n does not have a built-in "user profile" node. Implementing user profiles requires custom storage:

**Pattern 1 -- Database Table:**
- Create a `user_profiles` table with columns like `user_id`, `name`, `preferences` (JSONB), `created_at`, `updated_at`.
- Use n8n's Postgres node to read/write profiles.
- Inject the user profile into the system prompt before each agent call.

**Pattern 2 -- Long-Term Memory with LLM Extraction:**
- After each conversation, pass the transcript to an LLM with a prompt like: "Extract any new facts about the user (preferences, name, context) and summarize in 1-2 sentences."
- Store the extracted facts in a database or Google Doc, keyed by user ID.
- On the next session, retrieve the user's fact sheet and include it in the system prompt.

**Pattern 3 -- Vector-Based User Memory:**
- Store each user's important interactions as embeddings in a vector store, filtered by user ID metadata.
- On new conversations, retrieve the most relevant past interactions and inject them as context.

Sources:
- [9 Context Engineering Strategies for n8n AI Agents](https://www.theaiautomators.com/context-engineering-strategies-to-build-better-ai-agents/)
- [n8n Community: How to Store user_id in PostgreSQL Chat Memory](https://community.n8n.io/t/how-to-store-user-id-in-postgresql-chat-memory-in-n8n/76889)

### 2.4 Redis Chat Memory

Best for applications requiring very low latency (voice assistants, live chat).

**Key parameters:**
- **Session Key**: Unique conversation identifier.
- **TTL (Time to Live)**: Auto-deletes messages after a set duration -- useful for ephemeral conversations.
- **Context Window Length**: Same as other memory nodes.

**Advantages over Postgres:**
- Sub-millisecond read/write.
- Built-in TTL for automatic cleanup.
- Simpler to scale horizontally.

**Disadvantage:**
- No built-in analytics or querying capability like SQL.
- Data loss risk if Redis restarts without persistence configured (use RDB/AOF persistence).

Sources:
- [Redis Chat Memory node docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memoryredischat/)

---

## 3. RAG (Retrieval Augmented Generation)

RAG improves AI responses by grounding them in external knowledge. Instead of relying solely on the LLM's training data, the system retrieves relevant documents and includes them in the prompt.

### 3.1 RAG Architecture in n8n

n8n RAG consists of two workflows:

**Workflow 1 -- Document Ingestion Pipeline:**
```
Source (Drive/API/Upload) --> Data Loader --> Text Splitter --> Embedding Model --> Vector Store
```

**Workflow 2 -- Query and Retrieval:**
```
Chat Trigger --> AI Agent --> Vector Store Tool --> Embedding Model --> LLM --> Response
                    |
                    +--> Memory Sub-Node
```

Sources:
- [RAG in n8n](https://docs.n8n.io/advanced-ai/rag-in-n8n/)
- [Build a Custom Knowledge RAG Chatbot](https://blog.n8n.io/rag-chatbot/)
- [Stop Gluing Services Together: Build RAG Pipelines with n8n](https://blog.n8n.io/rag-pipeline/)

### 3.2 Vector Store Nodes in n8n

n8n supports multiple vector database backends:

| Vector Store | Type | Best For |
|---|---|---|
| **PGVector** (PostgreSQL) | Self-hosted / cloud | Consolidating with existing Postgres |
| **Pinecone** | Managed cloud | Serverless, fully managed |
| **Qdrant** | Self-hosted / cloud | Advanced filtering, sparse vectors |
| **Supabase** | Managed Postgres | Supabase ecosystem users |
| **Chroma** | Self-hosted | Local development |
| **Weaviate** | Self-hosted / cloud | Multi-modal, GraphQL queries |
| **Milvus** | Self-hosted / cloud | Large-scale deployments |
| **MongoDB Atlas** | Managed cloud | MongoDB ecosystem users |
| **Redis** | Self-hosted / cloud | Low-latency retrieval |
| **Azure AI Search** | Azure cloud | Azure ecosystem users |
| **Zep** | Managed | Enriched memory features |
| **In-Memory** | Local process | Quick prototyping (volatile) |

### 3.3 PGVector Configuration (Recommended for This Project)

Using PostgreSQL with pgvector consolidates the database layer: the same instance handles chat memory, structured data, and vector search.

#### Docker Setup

```yaml
# docker-compose.yml
networks:
  n8n_network:
    driver: bridge

services:
  postgres:
    image: pgvector/pgvector:pg17
    container_name: pgvector-db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: chatbot
    networks:
      - n8n_network
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    networks:
      - n8n_network
    ports:
      - "5678:5678"
    volumes:
      - ./n8n-data:/home/node/.n8n
    environment:
      - NODE_FUNCTION_ALLOW_BUILTIN=*
      - NODE_FUNCTION_ALLOW_EXTERNAL=*

volumes:
  pgdata:
```

#### Database Initialization

```sql
-- init.sql
CREATE EXTENSION IF NOT EXISTS vector;

-- For RAG document storage (auto-created by n8n, but can be pre-created)
-- The PGVector Vector Store node will create a table with these columns:
--   id (bigserial), embedding (vector), text (text), metadata (jsonb)
-- Vector dimension must match your embedding model:
--   text-embedding-3-small = 1536 dimensions
--   text-embedding-3-large = 3072 dimensions
--   text-embedding-004 (Google) = 768 dimensions

-- For long-term fact memory (custom)
CREATE TABLE IF NOT EXISTS memories (
    id SERIAL PRIMARY KEY,
    user_id TEXT NOT NULL,
    memory TEXT NOT NULL,
    tags TEXT[],
    created_at TIMESTAMP DEFAULT NOW()
);

-- Chat history is auto-created by Postgres Chat Memory node
-- Table: n8n_chat_histories (session_id, message JSONB, type, timestamp)
```

#### PGVector Vector Store Node -- Operation Modes

The node has four modes:

1. **Get Many** -- Retrieve documents by similarity search. Parameters: table name, search prompt, result limit.
2. **Insert Documents** -- Store new document embeddings. Parameters: table name. Connects to an embedding model and a text splitter.
3. **Retrieve Documents (As Vector Store for Chain/Tool)** -- Used with Vector Store Retriever and Q&A Chain nodes.
4. **Retrieve Documents (As Tool for AI Agent)** -- Connect directly to the AI Agent's tool connector. Parameters: tool name, tool description, table name, result limit.

**Important configuration options:**
- **Collections**: Use collection names to separate different datasets in the same table.
- **Column Names**: Customize column names for id, vector, content, and metadata.
- **Metadata Filtering**: Apply JSON filters to narrow search results.
- **Reranking**: Available for result quality optimization.

**Critical rule**: Use the **same embedding model** for ingestion and retrieval. Mismatched models produce incompatible vector dimensions and poor similarity matching.

Sources:
- [PGVector Vector Store node docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstorepgvector/)
- [Using pgvector as vector store in n8n](https://tsmx.net/using-pgvector-as-vector-store-in-n8n/)
- [n8n + Neon Postgres guide](https://neon.com/guides/n8n-neon)
- [Set up n8n and postgres for RAG](https://iris2020.net/2025-11-19-n8n_postgres_embed/)

### 3.4 Embedding Models

n8n supports these embedding model providers:

| Provider | Model Example | Dimensions | Notes |
|---|---|---|---|
| **OpenAI** | text-embedding-3-small | 1536 | Most widely used |
| **OpenAI** | text-embedding-3-large | 3072 | Higher quality, more expensive |
| **Google Gemini** | text-embedding-004 | 768 | Good for Gemini-stack projects |
| **Azure OpenAI** | (deployment-specific) | varies | Enterprise Azure deployments |
| **Cohere** | embed-v3 | 1024 | Strong multilingual support |
| **HuggingFace** | BGE-M3, etc. | varies | Open-source, self-hosted |
| **Mistral Cloud** | mistral-embed | 1024 | Mistral ecosystem |
| **Ollama** | nomic-embed-text, etc. | varies | Fully local, no API costs |

**Choosing an embedding model:**
- For cloud-hosted n8n: OpenAI `text-embedding-3-small` is the most battle-tested.
- For self-hosted / air-gapped: Ollama with `nomic-embed-text` or `mxbai-embed-large`.
- For cost optimization: Google Gemini's `text-embedding-004` (768 dimensions = smaller storage).
- Match the vector dimension in your database table to the model's output dimension.

### 3.5 Document Ingestion Workflows

#### Document Sources

n8n can ingest from:
- **Google Drive** (via Google Drive Trigger for automatic monitoring)
- **HTTP Request** (fetch from APIs, web pages)
- **Form Upload** (user-uploaded PDFs)
- **File system** (local files mounted in Docker)
- **GitHub** (via GitHub Document Loader)
- **Databases** (via Postgres/MySQL nodes)
- **Web scraping** (via HTTP Request + HTML extraction)

#### Text Splitter Nodes

n8n provides three text splitter sub-nodes:

**1. Recursive Character Text Splitter** (recommended for most use cases)
- Splits text hierarchically: first by paragraphs, then sentences, then characters.
- Maintains natural text boundaries.
- Parameters:
  - **Chunk Size**: Number of characters per chunk (default ~1000).
  - **Chunk Overlap**: Characters shared between adjacent chunks (default ~200).
- Best for: Prose documents, documentation, articles.

**2. Character Text Splitter**
- Splits on a single separator character.
- Parameters: Separator, chunk size, chunk overlap.
- Best for: Structured data with clear delimiters.

**3. Token Text Splitter**
- Splits by token count rather than character count.
- Parameters: Chunk size (in tokens), chunk overlap (in tokens).
- Best for: Precise token budget control. E.g., 1000 tokens per chunk, 250 tokens overlap.

#### Chunking Strategy Recommendations

| Document Type | Chunk Size | Overlap | Splitter |
|---|---|---|---|
| Technical docs | 500-1000 chars | 100-200 chars | Recursive Character |
| Legal/policy text | 1000-2000 chars | 200-400 chars | Recursive Character |
| FAQ / Q&A pairs | 300-500 chars | 50-100 chars | Character (by newline) |
| Code files | 500-1000 tokens | 100-250 tokens | Token |
| Long-form articles | 800-1200 chars | 150-250 chars | Recursive Character |

**General guidance:**
- Larger chunks preserve more context but reduce retrieval precision.
- Smaller chunks improve precision but may lose surrounding context.
- Overlap prevents information from being split across chunk boundaries.
- Start with defaults (1000 chars, 200 overlap) and tune based on retrieval quality.

Sources:
- [Recursive Character Text Splitter docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.textsplitterrecursivecharactertextsplitter/)
- [n8n RAG Text Splitters comparison](https://ryanandmattdatascience.com/n8n-rag-text-splitters/)

### 3.6 Retriever Nodes

n8n offers several retriever patterns:

- **Vector Store Retriever**: Basic similarity search against a vector store. Set `topK` (typically 3-5 documents).
- **Contextual Compression Retriever**: Compresses retrieved documents to only the relevant portions before sending to the LLM. Reduces token usage.
- **MultiQuery Retriever**: Generates multiple reformulations of the user's query and retrieves documents for each, then deduplicates. Improves recall.
- **Workflow Retriever**: Custom retrieval logic implemented as a sub-workflow.

### 3.7 Connecting RAG to the AI Agent

Two primary patterns:

**Pattern A -- Vector Store as Agent Tool (Recommended):**
```
AI Agent Node
  |-- Tool: PGVector Vector Store (mode: "Retrieve Documents as Tool for AI Agent")
  |       |-- Embedding Model: OpenAI
  |       Settings: name="knowledge_base", description="Search the product documentation...", limit=4
  |-- Memory: Postgres Chat Memory
  |-- Model: OpenAI Chat Model (gpt-4o-mini)
```
The agent decides when to search the knowledge base based on the tool description.

**Pattern B -- Q&A Chain (Simpler but Less Flexible):**
```
Chat Trigger --> Question and Answer Chain
                    |-- Vector Store Retriever --> PGVector Vector Store
                    |                              |-- Embedding Model
                    |-- LLM: OpenAI Chat Model
```
Every query triggers a retrieval. No agent reasoning about whether retrieval is needed.

### 3.8 Advanced RAG: Contextual Summaries + Sparse Vectors + Reranking

A community-developed n8n template achieves approximately 49% better retrieval accuracy by combining three techniques (based on Anthropic's contextual retrieval research):

**1. Contextual Summaries:**
Each chunk is processed through an LLM that generates a summary reflecting the chunk's relationship to the full document. The summary is prepended to the chunk before embedding.

**2. Sparse Vectors (BM25/TF-IDF):**
In addition to dense embeddings, generate sparse vectors using TF-IDF (via a Python Code node with scikit-learn). Store both dense and sparse vectors in Qdrant.

**3. Hybrid Search + Reranking:**
Query with both dense and sparse vectors simultaneously. Pass the combined results through Cohere's reranking service (or a local BM25 Retriever node in n8n v1.62.1+) to sort by relevance.

**Qdrant collection configuration for dual vectors:**
```json
{
  "sparse_vectors": {
    "bm42": { "modifier": "idf" }
  },
  "vectors": {
    "default": { "distance": "Cosine", "size": 1024 }
  }
}
```

Sources:
- [Building the Ultimate RAG Setup with Contextual Summaries](https://community.n8n.io/t/building-the-ultimate-rag-setup-with-contextual-summaries-sparse-vectors-and-reranking/54861)
- [Building Agentic RAG with PostgreSQL and n8n](https://aiven.io/blog/building-agentic-rag-with-postgresql-and-n8n)

---

## 4. Context Rot Prevention

### 4.1 What Is Context Rot?

Context rot is the measurable degradation in LLM output quality as input context grows longer. It is not a binary failure at the context window limit -- it is a continuous, progressive decline that begins well before the window is full. A model with a 200K token window can show significant degradation at 50K tokens.

Chroma Research tested 18 frontier models (Claude Opus 4, GPT-4.1, Gemini 2.5 Pro, Qwen3, and others) and found that **every single model** gets worse as input length increases, even on simple tasks.

A Stanford study found that with just 20 retrieved documents (~4,000 tokens), accuracy drops from 70-75% down to 55-60% depending on information placement -- a 15-20 percentage point decline based purely on position.

Sources:
- [Chroma Research: Context Rot](https://www.trychroma.com/research/context-rot)
- [Redis: Context rot explained](https://redis.io/blog/context-rot/)
- [Morph: Context Rot Guide](https://www.morphllm.com/context-rot)

### 4.2 Three Compounding Mechanisms

**1. Lost-in-the-Middle Effect**
Models attend strongly to the beginning and end of the context window but poorly to content in the middle. Research shows a 30%+ accuracy drop when relevant information sits in positions 5-15 versus position 1 or 20 in a list of documents.

**2. Attention Dilution**
Transformer self-attention is quadratic in sequence length. At 100K tokens, the model must track 10 billion pairwise relationships. Each individual token receives proportionally less attention as context grows. The signal-to-noise ratio declines.

**3. Distractor Interference**
Semantically similar but irrelevant content actively misleads the model. This is particularly dangerous in RAG systems where retrieved chunks may be topically related but factually irrelevant to the specific question.

Additional finding: Counterintuitively, models perform worse on coherent, logically-structured haystacks compared to shuffled versions. Narrative flow appears to make irrelevant content more distracting.

### 4.3 Why Chatbots Are Vulnerable

In long-running chatbot conversations, context accumulates continuously:
- Every user message and AI response stays in the context window.
- Tool call results (RAG retrievals, API responses) add bulk.
- By 35 minutes of conversation, an agent may have accumulated 80K-150K tokens.
- The failure loop self-reinforces: degraded outputs cause confusion, leading to more back-and-forth, which adds more noise.

### 4.4 Prevention Strategies

#### Strategy 1: Keep Context Windows Short (Context Isolation)

The primary fix is keeping the context short rather than hoping the model handles long context well.

- **Set a context window length** on your memory node (10-20 interactions, not unlimited).
- **Use sub-agents with isolated contexts**: A lead agent delegates tasks to specialized sub-agents, each with their own clean context window. The sub-agent does deep exploration and returns a condensed summary (1,000-2,000 tokens) to the lead agent. Anthropic's multi-agent approach showed a 90.2% improvement over single-agent performance.
- **Return precise results, not entire documents**: RAG retrievals should return the top 3-5 most relevant chunks, not 20.

#### Strategy 2: Dynamic Retrieval (Just-in-Time Context)

Instead of pre-loading everything into context, retrieve information on demand:

- Maintain lightweight identifiers (file paths, document IDs) rather than full content.
- Use tools that the agent can call when it needs specific information.
- This mirrors human cognition: we look things up when needed rather than memorizing everything.

#### Strategy 3: Relevance Filtering

- Apply **metadata filters** on vector store queries to narrow results before they enter the context.
- Use **Contextual Compression Retriever** nodes to strip retrieved documents down to only the portions relevant to the current query.
- Position critical information at the **beginning or end** of the context, not the middle.

#### Strategy 4: Semantic Caching

Recognize when different queries are semantically equivalent and return cached results:

- "What is context rot?" and "How does context rot work?" are similar enough to use the same retrieval results.
- Reduces redundant processing and prevents the same information from being retrieved and added to context multiple times.
- Redis provides built-in semantic caching capabilities (Redis LangCache).

#### Strategy 5: Context Compaction (see Section 5)

Summarize old conversation turns and replace them with a compressed summary.

#### Strategy 6: Multi-Tiered Memory Architecture

Mirror human memory with separate tiers:

| Tier | Type | Implementation | Content |
|---|---|---|---|
| Working Memory | Short-term | Context window (last N messages) | Current conversation |
| Episodic Memory | Medium-term | Database with temporal metadata | Past interaction summaries |
| Semantic Memory | Long-term | Vector store | Facts, knowledge, preferences |
| Procedural Memory | Long-term | Structured logs | Past action outcomes |

Sources:
- [Anthropic: Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Tribe AI: Context-Aware Memory Systems](https://www.tribe.ai/applied-ai/beyond-the-bubble-how-context-aware-memory-systems-are-changing-the-game-in-2025)
- [Morph: Context Rot](https://www.morphllm.com/context-rot)

### 4.5 Detection and Monitoring

Monitor for context rot with these approaches:

- **Track token counts** per interaction and alert when they cross thresholds (e.g., 50% of model window).
- **Cosine distance monitoring**: Compare embedding distributions of current context versus baseline to detect drift.
- **Response quality scoring**: Periodically evaluate responses for hallucination, relevance, and coherence.
- **User feedback signals**: Track when users rephrase questions or express confusion.

---

## 5. Context Compaction and Summary Reintroduction

Context compaction summarizes older conversation turns and replaces them with a compressed version, allowing conversations to continue indefinitely without hitting token limits or suffering quality degradation.

### 5.1 Core Concept

Instead of keeping the entire conversation history:
```
[Message 1] [Message 2] [Message 3] ... [Message 50] [New Query]
```

Compact to:
```
[Summary of Messages 1-40] [Message 41] [Message 42] ... [Message 50] [New Query]
```

The summary preserves key decisions, facts, and context while discarding redundant back-and-forth.

### 5.2 Compaction Techniques

#### Technique 1: Rolling Summary (Recursive Summarization)

A running summary is maintained and updated incrementally:

```
existing_summary + new_message_chunk --> updated_summary
```

Research ("Recursively Summarizing Enables Long-Term Dialogue Memory in Large Language Models", 2023) shows this generates more consistent responses in long-term conversations, and the summaries are much shorter than full dialogues.

**Process:**
1. After every N interactions (e.g., 10), pass the current summary + the new messages to an LLM.
2. Prompt: "Update this conversation summary with the new information. Preserve key decisions, user preferences, and unresolved questions. Discard redundant exchanges."
3. Replace the old summary + messages with the new summary.
4. Keep the most recent 5-10 messages in full for immediate context.

#### Technique 2: Threshold-Based Auto-Compaction

Trigger compaction when token usage crosses a threshold (e.g., 70-90% of the model's context window):

1. Count current context tokens.
2. When threshold is crossed, run a summarization pass.
3. Replace history with: `[System prompt] + [Summary] + [Recent N messages] + [Current query]`.

This is the approach used by Cursor (at ~90% of model window) and Claude Code.

#### Technique 3: Claim Extraction (Not Summarization)

Instead of traditional summarization, extract discrete factual claims from the conversation:

- "The user's name is Alex."
- "The project uses PostgreSQL with pgvector."
- "The user prefers GPT-4o-mini for cost reasons."

Store claims in structured format (JSON, markdown, or database rows). More precise than prose summaries and easier to update or invalidate individual claims.

#### Technique 4: Observation Masking

Preserve the agent's reasoning and action history but replace older tool/observation outputs with placeholders. JetBrains research found this approach:
- Reduced costs by 50%+ versus unmanaged contexts.
- Matched or exceeded LLM summarization performance in 4 of 5 test configurations.
- Achieved 2.6% higher solve rates while costing 52% less (with Qwen3-Coder).

#### Technique 5: Memory Formation (Selective Retention)

Distinguish between information worth preserving long-term and ephemeral conversation noise:

- **Importance scoring**: Weight conversation elements by future relevance.
- **Decay mechanisms**: Reduce older memories' influence unless consistently referenced.
- **Conflict resolution**: When new information contradicts old, prioritize the more recent or more reliable source.

Sources:
- [Recursively Summarizing Enables Long-Term Dialogue Memory](https://arxiv.org/abs/2308.15022)
- [LLM Chat History Summarization Guide 2025](https://mem0.ai/blog/llm-chat-history-summarization-guide-2025)
- [Context Management and Compaction in LLMs](https://kargarisaac.medium.com/the-fundamentals-of-context-management-and-compaction-in-llms-171ea31741a2)
- [JetBrains: Efficient Context Management](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)
- [OpenAI Community: Best practices for context management](https://community.openai.com/t/best-practices-for-cost-efficient-high-quality-context-management-in-long-ai-chats/1373996)

### 5.3 Implementing Compaction in n8n

n8n does not have a built-in compaction node. You implement it using a combination of existing nodes. Here is a practical workflow pattern:

#### Pattern: Periodic Summary Compaction

```
Chat Trigger
    |
    v
[Code Node: Check Memory Size]
    |
    +--> (memory < threshold) --> AI Agent (normal flow)
    |
    +--> (memory >= threshold) --> [Chat Memory Manager: Get All Messages]
                                       |
                                       v
                                  [LLM Chain: Summarize]
                                       |
                                       v
                                  [Chat Memory Manager: Override All Messages]
                                  (insert: System message with summary + recent N messages)
                                       |
                                       v
                                  AI Agent (continue with compacted context)
```

**Step-by-step implementation:**

1. **Code Node (Check Memory Size):**
   Count the number of messages or estimate token count. Route to compaction if over threshold.
   ```javascript
   const messages = $input.all();
   const messageCount = messages.length;
   const THRESHOLD = 30; // trigger compaction after 30 messages

   return [{
     json: {
       needsCompaction: messageCount >= THRESHOLD,
       messageCount: messageCount
     }
   }];
   ```

2. **Chat Memory Manager (Get Many):**
   Load all messages from the connected memory store.

3. **LLM Chain (Summarize):**
   Send the messages to an LLM with a summarization prompt:
   ```
   Summarize the following conversation history. Preserve:
   - Key decisions and conclusions
   - User preferences and requirements
   - Unresolved questions or pending tasks
   - Important facts and context

   Discard:
   - Redundant greetings and acknowledgments
   - Repeated explanations
   - Exploratory tangents that led nowhere

   Conversation:
   {{ $json.messages }}
   ```

4. **Chat Memory Manager (Override All Messages):**
   Replace the entire memory with:
   - A System message containing the summary.
   - The most recent 5-10 messages preserved verbatim.

5. **Continue to AI Agent** with the compacted context.

### 5.4 Balancing Detail vs. Compression

**Too aggressive** (short summaries): The bot "forgets" important decisions and asks the user to repeat information.

**Too conservative** (verbose summaries): Token savings are minimal and context rot continues.

**Guidelines:**
- Start by maximizing recall (capture everything), then iterate for precision.
- Keep summaries under 500-1000 tokens for conversations with 30+ exchanges.
- Always preserve the most recent 5-10 messages verbatim -- they carry the most immediate relevance.
- Include a "what's still unresolved" section in the summary to maintain task continuity.
- Low-hanging optimization: clear tool result outputs after initial processing (they're often the largest token consumers).

### 5.5 Cost Impact

Smart memory management with compaction typically yields:
- **30-60% reduction** in LLM API costs by minimizing redundant context.
- **80-90% reduction** in token usage for long-running conversations versus full history inclusion.
- **26% improvement** in response quality by reducing noise in the context.

---

## 6. Knowledge Base Management

### 6.1 Building the Knowledge Base

A knowledge base for RAG consists of documents that have been chunked, embedded, and stored in a vector database. The quality of the knowledge base directly determines the quality of RAG responses.

**Document types n8n can ingest:**
- PDF files (via Extract from File node or Default Data Loader)
- Web pages (via HTTP Request + HTML-to-Markdown conversion)
- Google Docs (via Google Drive nodes)
- Plain text / Markdown files
- CSV / spreadsheet data
- API responses (JSON)
- GitHub repositories (via GitHub Document Loader)

### 6.2 Automated Ingestion Pipeline

The recommended pattern monitors a source for changes and automatically re-indexes:

```
Google Drive Trigger (file created/modified)
    |
    v
Switch Node (route by MIME type: PDF, Doc, Sheet, etc.)
    |
    v
Extract Content (type-specific extraction)
    |
    v
Default Data Loader (add metadata: filename, source, date)
    |
    v
Recursive Character Text Splitter (chunk size: 1000, overlap: 200)
    |
    v
Embedding Model (OpenAI / Gemini / Ollama)
    |
    v
PGVector Vector Store (Insert Documents, table: "knowledge_base")
```

**Key features:**
- **Automatic triggering**: The Google Drive Trigger node fires when files are created or modified.
- **MIME-type routing**: A Switch node handles different file types with appropriate extractors.
- **Metadata enrichment**: The Data Loader attaches the filename, source path, and ingestion date to each chunk for traceability.

Sources:
- [Automate document ingestion & RAG system](https://n8n.io/workflows/8312-automate-document-ingestion-and-rag-system-with-google-drive-sheets-and-openai/)
- [Multi-format document processing for RAG](https://n8n.io/workflows/9933-multi-format-document-processing-for-rag-chatbot-with-google-drive-and-supabase/)

### 6.3 Document Versioning and Change Detection

The challenge: When a document is updated, old embeddings become stale. Naive re-ingestion creates duplicates.

**Pattern 1 -- Hash-Based Change Detection:**
- Generate a SHA256 hash of each document's content.
- Store the hash in a tracking table (Google Sheets or PostgreSQL).
- On trigger, compare the new hash with the stored hash.
- Only re-process documents whose hash has changed.

**Pattern 2 -- Metadata-Based Deletion Before Re-Insertion:**
- Before inserting updated embeddings, delete existing vectors with matching metadata (e.g., `source_filename = 'policy.pdf'`).
- Insert the new embeddings.
- This avoids duplicates while ensuring freshness.

**Pattern 3 -- Record Manager (Google Sheets):**
- A community template uses Google Sheets as a record manager to track ingestion states.
- Each row records: filename, hash, ingestion timestamp, chunk count, status.
- The workflow checks this sheet before processing to skip unchanged files.

**Example tracking table schema:**
```sql
CREATE TABLE ingestion_log (
    id SERIAL PRIMARY KEY,
    filename TEXT NOT NULL,
    content_hash TEXT NOT NULL,
    chunk_count INTEGER,
    embedding_model TEXT,
    ingested_at TIMESTAMP DEFAULT NOW(),
    status TEXT DEFAULT 'active'
);
```

### 6.4 Re-Indexing Workflows

**Full re-index** (scheduled):
- Run weekly or monthly via a Schedule Trigger node.
- Delete all vectors in the collection.
- Re-ingest all documents from source.
- Ensures consistency but is expensive for large knowledge bases.

**Incremental re-index** (event-driven):
- Triggered by file change events.
- Only processes changed documents.
- Uses the change detection patterns above.
- More efficient but requires careful state management.

**Re-indexing after embedding model change:**
If you switch embedding models (e.g., from `text-embedding-3-small` to `text-embedding-3-large`), you must re-embed the entire knowledge base. Old embeddings are incompatible with the new model's vector space.

### 6.5 Quality Control of Ingested Content

**Pre-ingestion quality checks:**
- Validate that extracted text is not empty or corrupted.
- Check document length -- very short documents may not be worth indexing.
- Filter out boilerplate content (headers, footers, navigation menus from web pages).

**Post-ingestion validation:**
- Run a set of test queries against the knowledge base and evaluate retrieval quality.
- Check that expected documents appear in top-K results for known queries.
- Monitor embedding dimensionality consistency across all stored vectors.

**Content formatting for RAG quality:**
- Convert HTML to Markdown before ingestion (cleaner, fewer tokens, better for LLMs).
- Use n8n's Markdown node or external services like Firecrawl for web content.
- Remove irrelevant metadata, styling, and navigation elements.
- Preserve document structure (headings, lists) as these help the LLM understand context.

### 6.6 Keeping the Knowledge Base Fresh

**Automated pipelines:**

| Trigger | Frequency | Action |
|---|---|---|
| Google Drive Trigger | Real-time | Re-index on file change |
| Schedule Trigger | Daily/Weekly | Full scan for changes |
| Webhook | On-demand | Manual re-index request |
| RSS/API polling | Hourly | Ingest new external content |

**Staleness prevention checklist:**
1. Set up automated change detection (hash-based or event-driven).
2. Log every ingestion with timestamps.
3. Periodically audit: query the ingestion log for documents not updated in > 30 days.
4. Build a "re-index all" workflow that can be triggered manually when the embedding model changes.
5. Version your knowledge base: before full re-index, back up the current vector table.

### 6.7 The "Just Use Postgres" Architecture

For many n8n chatbot projects, a single PostgreSQL instance with pgvector can serve all three data roles, reducing infrastructure complexity:

| Function | PostgreSQL Feature | n8n Node |
|---|---|---|
| Vector store (RAG) | pgvector extension | PGVector Vector Store |
| Chat memory | Regular table (JSONB) | Postgres Chat Memory |
| Structured data / SQL tools | Standard SQL | Postgres node |
| Long-term fact memory | Custom table | Postgres node |
| Ingestion tracking | Custom table | Postgres node |

The AI agent can be given tools that query the same database in different ways:
- **Vector search tool**: Semantic similarity search for knowledge retrieval.
- **SQL query tool**: Structured queries for factual lookups (dates, numbers, specific records).
- **List documents tool**: `SELECT * FROM document_metadata` to discover available knowledge.

This "agentic RAG" pattern allows the agent to choose the most appropriate retrieval method based on the query type, rather than blindly running vector search for every question.

Sources:
- [Building Agentic RAG with PostgreSQL and n8n](https://aiven.io/blog/building-agentic-rag-with-postgresql-and-n8n)
- [Company knowledge base agent (RAG) template](https://n8n.io/workflows/6538-company-knowledge-base-agent-rag/)

---

## Appendix A: Complete Node Reference for n8n AI Chatbot

### Core Nodes
- **Chat Trigger** -- Starts a chat conversation (webhook-based)
- **AI Agent** -- The orchestrator that combines LLM, memory, and tools
- **Chat** -- Embedded chat widget for testing

### Memory Sub-Nodes
- Simple Memory (Buffer Window Memory)
- Postgres Chat Memory
- Redis Chat Memory
- MongoDB Chat Memory
- Motorhead
- Zep
- Xata

### Memory Management
- Chat Memory Manager (Get/Insert/Delete messages)

### LLM Models
- OpenAI Chat Model
- Anthropic (Claude)
- Google Gemini
- Groq
- Mistral
- Ollama (local)

### Embedding Models
- Embeddings OpenAI
- Embeddings Google Gemini
- Embeddings Azure OpenAI
- Embeddings Cohere
- Embeddings HuggingFace
- Embeddings Mistral Cloud
- Embeddings Ollama

### Vector Stores
- PGVector Vector Store
- Pinecone Vector Store
- Qdrant Vector Store
- Supabase Vector Store
- Chroma Vector Store
- Weaviate Vector Store
- Milvus Vector Store
- MongoDB Atlas Vector Store
- Redis Vector Store
- Azure AI Search Vector Store
- Zep Vector Store
- In-Memory Vector Store

### Text Splitters
- Recursive Character Text Splitter
- Character Text Splitter
- Token Text Splitter

### Document Loaders
- Default Data Loader
- GitHub Document Loader

### Retrievers
- Vector Store Retriever
- Contextual Compression Retriever
- MultiQuery Retriever
- Workflow Retriever

### Chains
- Question and Answer Chain
- Basic LLM Chain

---

## Appendix B: Recommended Architecture for This Project

Based on this research, the recommended architecture for a production n8n chatbot:

```
                    +-------------------+
                    |   Chat Frontend   |
                    | (Webhook / Chat   |
                    |    Trigger)       |
                    +--------+----------+
                             |
                             v
                    +--------+----------+
                    |    AI Agent       |
                    |  (Tools Agent)    |
                    +--+----+----+-----+
                       |    |    |
            +----------+    |    +----------+
            |               |               |
            v               v               v
    +-------+------+  +-----+------+  +-----+-------+
    | Postgres Chat|  | PGVector   |  | Postgres    |
    | Memory       |  | Vector     |  | (SQL Tool)  |
    | (short-term) |  | Store      |  | (facts,     |
    |              |  | (RAG tool) |  |  profiles)  |
    +--------------+  +------------+  +-------------+
            |               |               |
            +-------+-------+-------+-------+
                    |
                    v
            +-------+-------+
            |  PostgreSQL   |
            |  + pgvector   |
            |  (single DB)  |
            +---------------+
```

**Components:**
1. **Chat Trigger** receives user messages.
2. **AI Agent** (Tools Agent type) with GPT-4o-mini or Claude.
3. **Postgres Chat Memory** for conversation history (context window: 15).
4. **PGVector Vector Store** as an agent tool for knowledge base search.
5. **Postgres node** as an agent tool for structured queries and user profile lookup.
6. **Periodic compaction workflow** using Chat Memory Manager + LLM summarization.
7. **Automated ingestion pipeline** triggered by Google Drive changes or schedule.

---

## Appendix C: Key Sources and Further Reading

### Official n8n Documentation
- [RAG in n8n](https://docs.n8n.io/advanced-ai/rag-in-n8n/)
- [What's memory in AI?](https://docs.n8n.io/advanced-ai/examples/understand-memory/)
- [Simple Memory node](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorybufferwindow/)
- [Postgres Chat Memory node](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorypostgreschat/)
- [Chat Memory Manager node](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorymanager/)
- [PGVector Vector Store node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstorepgvector/)
- [Recursive Character Text Splitter](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.textsplitterrecursivecharactertextsplitter/)
- [AI Agent common issues](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/common-issues/)

### n8n Blog and Templates
- [Build a Custom Knowledge RAG Chatbot](https://blog.n8n.io/rag-chatbot/)
- [Build RAG Pipelines with n8n](https://blog.n8n.io/rag-pipeline/)
- [Your Practical Guide to LLM Agents in 2025](https://blog.n8n.io/llm-agents/)
- [Basic RAG chat template](https://n8n.io/workflows/5028-basic-rag-chat/)
- [Company knowledge base agent template](https://n8n.io/workflows/6538-company-knowledge-base-agent-rag/)
- [AI agent chatbot + long-term memory + Telegram template](https://n8n.io/workflows/2872-ai-agent-chatbot-long-term-memory-note-storage-telegram/)
- [Automated document ingestion template](https://n8n.io/workflows/8312-automate-document-ingestion-and-rag-system-with-google-drive-sheets-and-openai/)

### Context Rot and Context Engineering
- [Chroma Research: Context Rot](https://www.trychroma.com/research/context-rot)
- [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Redis: Context rot explained](https://redis.io/blog/context-rot/)
- [Morph: Context Rot Complete Guide](https://www.morphllm.com/context-rot)
- [JetBrains: Efficient Context Management for LLM-Powered Agents](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)

### Memory and Summarization
- [Recursively Summarizing Enables Long-Term Dialogue Memory (arXiv)](https://arxiv.org/abs/2308.15022)
- [LLM Chat History Summarization Guide 2025](https://mem0.ai/blog/llm-chat-history-summarization-guide-2025)
- [Context Management and Compaction in LLMs](https://kargarisaac.medium.com/the-fundamentals-of-context-management-and-compaction-in-llms-171ea31741a2)
- [Tribe AI: Context-Aware Memory Systems 2025](https://www.tribe.ai/applied-ai/beyond-the-bubble-how-context-aware-memory-systems-are-changing-the-game-in-2025)
- [OpenAI Community: Context Management Best Practices](https://community.openai.com/t/best-practices-for-cost-efficient-high-quality-context-management-in-long-ai-chats/1373996)

### Tutorials and Guides
- [9 Context Engineering Strategies for n8n AI Agents](https://www.theaiautomators.com/context-engineering-strategies-to-build-better-ai-agents/)
- [n8n AI Agent Memory Setup Guide (Towards AI)](https://towardsai.net/p/machine-learning/n8n-ai-agent-node-memory-complete-setup-guide-for-2026)
- [Using pgvector as vector store in n8n](https://tsmx.net/using-pgvector-as-vector-store-in-n8n/)
- [n8n + Neon Postgres knowledge base](https://neon.com/guides/n8n-neon)
- [Agentic RAG with n8n, Supabase, pgvector](https://medium.com/@ahsenelmas1/build-an-agentic-rag-chatbot-with-n8n-google-drive-supabase-and-pgvector-step-by-step-e05b1c5d5a18)
- [Building Agentic RAG with PostgreSQL and n8n](https://aiven.io/blog/building-agentic-rag-with-postgresql-and-n8n)
- [Set up n8n + postgres for local RAG](https://iris2020.net/2025-11-19-n8n_postgres_embed/)
- [Hybrid Long-Term Memory in n8n](https://damiandabrowski.medium.com/day-67-of-100-days-agentic-engineer-challenge-n8n-hybrid-long-term-memory-ce55694d8447)
- [Long-Term Memory with n8n and Qdrant](https://dev.to/einarcesar/long-term-memory-for-llms-using-vector-store-a-practical-approach-with-n8n-and-qdrant-2ha7)
- [Ultimate RAG Setup with Contextual Summaries](https://community.n8n.io/t/building-the-ultimate-rag-setup-with-contextual-summaries-sparse-vectors-and-reranking/54861)

### n8n Community Discussions
- [How to maintain conversation context with AI Agent](https://community.n8n.io/t/how-to-maintain-conversation-context-memory-with-ai-agent-for-follow-up-questions/137445)
- [Empty the Window Buffer Memory for a specific key](https://community.n8n.io/t/empty-the-window-buffer-memory-attached-to-an-ai-agent-for-a-specific-key/44585)
- [How to store user_id in PostgreSQL Chat Memory](https://community.n8n.io/t/how-to-store-user-id-in-postgresql-chat-memory-in-n8n/76889)
- [AI Agent token limitation after many iterations](https://community.n8n.io/t/ai-agent-token-limitation-400-error-after-many-iterations-despite-wanting-to-pass-0-context/63179)
