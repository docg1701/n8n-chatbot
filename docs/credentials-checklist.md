# Credentials checklist — n8n chatbot template

**Operator:** Galvani
**Scope:** Story 1.2 — the eight locked n8n credential names that the chatbot template references by exact spelling across every workflow. Seven credentials are created in this story; `evolution_api` is **reserved** and created later in Epic 8 Story 8.2.

## Why this file exists

The chatbot template is designed to be imported by a new firm without a single workflow edit. The only per-firm substitution is **credential values** (tokens, API keys, database passwords) — the **names** stay identical so every node reference keeps resolving. A typo or rename at the credential layer breaks every downstream workflow silently.

This checklist is the authoritative list of the eight locked names, spelled exactly. When the template is deployed against a new instance, the operator recreates these credentials with their own values; the list below does not change.

**Zero secret values appear in this file.** Every entry documents where the value comes from — never the value itself (NFR9, Architecture §Secrets and credentials).

## Summary table

| # | Name | n8n credential type | Purpose | Created in |
|---|---|---|---|---|
| 1 | `telegram_bot_refinement` | Telegram API | Refinement-environment bot used during Epic 1–9 | Story 1.2 |
| 2 | `telegram_bot_production` | Telegram API | Production-environment bot, **reserved** (not wired until Epic 10 Story 10.5) | Story 1.2 |
| 3 | `groq_free` | OpenAI (base URL override) | Primary operational LLM (Groq free tier) | Story 1.2 |
| 4 | `groq_paid` | OpenAI (base URL override) | Paid-tier fallback for AI Agent failover | Story 1.2 |
| 5 | `comadre_tts` | Header Auth | Comadre TTS Bearer authentication | Story 1.2 |
| 6 | `postgres_main` | Postgres | Main application database (chat history, profiles, analytics, staging) | Story 1.2 |
| 7 | `redis_main` | Redis | Rate limits, idempotency keys, queue-mode state | Story 1.2 |
| 8 | `evolution_api` | HTTP Header Auth | WhatsApp Business API (Evolution API) | **Reserved — Epic 8 Story 8.2** |

After Story 1.2 is complete, the n8n Credentials list contains exactly **seven** entries (rows 1–7). Row 8 is documented here as a locked name only; no credential entity is instantiated in this story.

## Per-credential detail

### 1. `telegram_bot_refinement`

- **n8n credential type:** Telegram API.
- **Purpose:** refinement-environment bot used during Epic 1–9 for client-facing conversations, handoff notifications, and ops alerts.
- **Value source:** `@BotFather` on Telegram — `/newbot`, capture the token, and store it in the operator's password manager (entry suggestion: `maruzza-refinement-bot-token`).
- **Creation steps:**
  1. Open `@BotFather` in the operator's personal Telegram account.
  2. `/newbot` — set an intelligible bot name and username (e.g., `Maruzza Refinement Bot` / `@maruzza_refinement_bot`). Capture the token.
  3. In n8n: **Settings → Credentials → New → Telegram API**. Name the credential exactly `telegram_bot_refinement`; paste the token into the Access Token field.
  4. Save, then run **Test step** (n8n issues `getMe`); confirm success.

### 2. `telegram_bot_production`

- **n8n credential type:** Telegram API.
- **Purpose:** production-environment bot reserved for the Launch Gate flip (Epic 10 Story 10.5). **Not** wired to any workflow during Epic 1–9. The credential exists so the template contract is ready the moment Launch Gate fires.
- **Value source:** `@BotFather` — a **distinct** second bot with its own token (never shared with `telegram_bot_refinement`).
- **Creation steps:**
  1. `/newbot` on `@BotFather` a second time to create the production bot (e.g., `Maruzza Bot` / `@maruzza_bot`, or whatever username is available). Capture the token; store under a distinct password-manager entry.
  2. **Do not** activate webhooks, group memberships, or any runtime posture yet — the production bot stays dormant until Epic 10.
  3. In n8n: new Telegram API credential named exactly `telegram_bot_production`; paste the production token.
  4. Run **Test step** to confirm the token is valid. The credential is never referenced by any workflow in Epic 1–9; Epic 10 Story 10.5 clones `main-chatbot-refinement` → `main-chatbot-production` and swaps the credential reference.

### 3. `groq_free`

- **n8n credential type:** OpenAI (with base URL override — Groq is OpenAI-compatible).
- **Purpose:** primary operational LLM for the AI Agent, triage classifier, guardrails, and structured-extraction sub-workflow. Free tier is the committed operational default (project policy: Groq free tier first, paid tier only when needed for failover).
- **Value source:** `https://console.groq.com/keys` — create an API key on the operator's free-tier account.
- **Creation steps:**
  1. In n8n: **Settings → Credentials → New → OpenAI**. Name exactly `groq_free`.
  2. Override the base URL to `https://api.groq.com/openai/v1`.
  3. Paste the free-tier API key into the API Key field.
  4. Save. Probe: throwaway HTTP Request node calling `GET https://api.groq.com/openai/v1/models` with credential `groq_free` — confirm HTTP 200 and a JSON model list; discard the probe node without saving.

### 4. `groq_paid`

- **n8n credential type:** OpenAI (with base URL override).
- **Purpose:** failover credential used by the AI Agent's retry policy when the free tier trips its quota or the free-tier account is degraded. Same base URL as `groq_free`; different API key on a paid-tier Groq account (or a distinct paid key on the same account).
- **Value source:** `https://console.groq.com/keys` — create or retrieve the paid-tier API key.
- **Creation steps:**
  1. New OpenAI credential named exactly `groq_paid`.
  2. Base URL: `https://api.groq.com/openai/v1`.
  3. Paste the paid-tier key.
  4. Probe the same `GET /v1/models` endpoint against `groq_paid`; discard the probe.

### 5. `comadre_tts`

- **n8n credential type:** Header Auth (or a Generic credential configured with a custom `Authorization` header).
- **Purpose:** Bearer-authenticated access to Comadre TTS at `https://10.10.10.207/v1/audio/speech` (OpenAI-compatible). The Epic 7 voice-out path depends on this.
- **Value source:** `~/dev/comadre/.env` (`API_KEY` variable) or the operator's password manager — both should stay in sync. Comadre enforces `API_KEY` as mandatory at service start (`~/dev/comadre/compose.yaml`).
- **Creation steps:**
  1. Fetch the current Comadre API key from the source of truth.
  2. In n8n: new **Header Auth** credential named exactly `comadre_tts`. Header name: `Authorization`. Header value: `Bearer <key>` (paste the key once, inline, into the value field — n8n encrypts it at rest via `N8N_ENCRYPTION_KEY`).
  3. Probe: throwaway HTTP Request node calling `POST https://10.10.10.207/v1/audio/speech` with credential `comadre_tts`, JSON body `{"input": "teste", "voice": "kokoro/pm_santa"}`, response format `File`, and **Ignore SSL Issues** enabled (Comadre's Caddy serves a self-signed certificate for the IP vhost — same posture as Story 1.1 infrastructure verification). Confirm HTTP 200 and a valid OGG Opus binary; discard the probe.
- **Known follow-up:** key rotation is natural at this registration moment. Comadre is LAN-only with no external consumers, so the operational cost is trivial — generate a new random key, update `~/dev/comadre/.env`, `docker compose restart` in `~/dev/comadre/`, and paste the new value into this credential. Optional but encouraged as the clean-slate bootstrap window.

### 6. `postgres_main`

- **n8n credential type:** Postgres.
- **Purpose:** the single application database shipped by `borgstack-mini`. Hosts n8n's own tables plus chatbot tables: `n8n_chat_histories`, `client_profiles`, `chat_analytics`, `static_responses`, and the Epic 9 staging tables.
- **Value source:** the `n8n_db_password` Docker Swarm secret, whose source of truth is `vault_n8n_db_password` in `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml` (Ansible Vault-encrypted).
- **Creation steps:**
  1. Ansible Vault-decrypt `vault.yml` with the stored vault password; read `vault_n8n_db_password`.
  2. In n8n: new **Postgres** credential named exactly `postgres_main`. Host: `postgresql`. Port: `5432`. Database: `n8n_db`. User: `n8n_user`. SSL mode: disable (internal Docker network).
  3. Paste the password fetched in step 1. Save.
  4. **Test step** success required. Optional end-to-end probe: a throwaway `SELECT 1 AS ping;` execution.

### 7. `redis_main`

- **n8n credential type:** Redis.
- **Purpose:** rate-limit counters, idempotency keys, and queue-mode state. Uses database index `0` alongside n8n's own queue keys (n8n queue keys live under the `n8n:*` prefix; chatbot keys live outside it, so no collision).
- **Value source:** the `redis_password` Docker Swarm secret; source of truth is `vault_redis_password` in `vault.yml`.
- **Creation steps:**
  1. Ansible Vault-decrypt `vault.yml`; read `vault_redis_password`.
  2. In n8n: new **Redis** credential named exactly `redis_main`. Host: `redis`. Port: `6379`. Database: `0`.
  3. Paste the password. Save.
  4. Probe: throwaway `SET _verify_redis_main "ok" EX 10` immediately followed by `GET _verify_redis_main` — both inside the same Execute Workflow run (the Story 1.1 TTL-race lesson applies: do not pause between the Set and the Get). Confirm the Get returns `ok`.

### 8. `evolution_api` — reserved (not created in Story 1.2)

- **n8n credential type (expected):** HTTP Header Auth. Epic 8 Story 8.2 locks the exact credential type when the service becomes reachable.
- **Purpose:** API key authentication against Evolution API for WhatsApp message ingestion and outbound sending.
- **Availability gate:** Evolution API is **cluster-only** on BorgStack. `borgstack-mini` does not ship it — confirmed in `~/dev/borgstack/templates/stack-mini.yml.j2` (no `evolution` service) and `~/dev/borgstack/docs/mini-architecture.md`. The credential becomes creatable when the operator either swaps from `borgstack-mini` to the full cluster or deploys a standalone Evolution API instance alongside the mini.
- **When it gets created:** Epic 8 Story 8.2 — "WhatsApp ingestion via Evolution API webhook, payload normalization, idempotency" — as the prefix step before any WhatsApp workflow wiring.
- **Do not create a stub credential here.** Inventing fake values would defeat the "credential names are contracts" discipline, and any probe would fail the Test step.

## Secret-handling discipline (NFR9)

Every locked credential above holds an encrypted secret value inside n8n. The discipline is load-bearing:

- Paste each value **once** into the n8n credential form field, click Save, and never copy it back out.
- Keep password-manager entries for each secret (BotFather tokens, Groq API keys, Comadre key, Docker Swarm secret values) as the permanent source of truth; n8n holds encrypted copies via `N8N_ENCRYPTION_KEY`.
- **Never** paste a secret value into a workflow field, Code node, Set node, HTTP Request header override, log output, exported JSON, markdown file, Git commit, issue tracker, or chat window.
- **Never** echo secrets to the terminal (Story 1.1 review flagged exactly this — a `grep API_KEY=...` emission to stdout is the canonical anti-pattern).

## References

- [Architecture §Credential Naming](../_bmad-output/planning-artifacts/architecture.md) — the eight committed names, `snake_case` format, template-contract rationale.
- [Architecture §Secrets and credentials](../_bmad-output/planning-artifacts/architecture.md) — n8n Credentials as the only secret store, encryption at rest via `N8N_ENCRYPTION_KEY`.
- [Architecture §Evolution API integration](../_bmad-output/planning-artifacts/architecture.md) — Evolution API cluster-only; deferred to Epic 8 Story 8.2.
- [Architecture §LLM call pattern](../_bmad-output/planning-artifacts/architecture.md) — Groq OpenAI-compatible endpoint; `groq_free` primary, `groq_paid` failover.
- [Story 1.2 — this story](../_bmad-output/implementation-artifacts/1-2-credential-environment-variable-and-telegram-group-bootstrap.md) — ACs, operator tasks, creation-step detail.
- [Story 1.1 infrastructure verification](infra-verification.md) — baseline posture; where `N8N_ENCRYPTION_KEY` and the Ansible Vault chain were proven.
- [`environment-variables.md`](environment-variables.md) — the eleven locked env var names consumed by workflows alongside these credentials; the `telegram_bot_refinement` ↔ `telegram_bot_production` split mirrors `ENVIRONMENT=refinement` ↔ `ENVIRONMENT=production`.
