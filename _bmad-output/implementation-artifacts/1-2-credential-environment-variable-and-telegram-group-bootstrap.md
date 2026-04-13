# Story 1.2: Credential, environment variable, and Telegram group bootstrap

Status: in-progress

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator (Galvani),
I want to register all locked n8n credentials and environment variables on borgstack-mini, plus the handoff and ops Telegram groups with captured chat IDs,
so that every subsequent workflow story can reference credentials and env vars by their committed names — with zero hardcoded values anywhere in the template — and the Epic 4 handoff flow + Epic 6 ops alerting have real delivery addresses before any wiring begins (AR41, AR42, NFR9, NFR39).

Story 1.1 left the n8n Credentials list and workflow canvas **empty**. Story 1.2 registers the 8 locked credential names (minus `evolution_api`, deferred to Epic 8), 11 locked environment variables, and 2 Telegram groups. Story 1.3 then consumes these names when creating empty-canvas workflow shells.

**`evolution_api` is explicitly out of scope** — it is a cluster-only service not shipped by borgstack-mini (confirmed in `~/dev/borgstack/templates/stack-mini.yml.j2` and `~/dev/borgstack/docs/mini-architecture.md`). Its credential is registered in Epic 8 Story 8.2 alongside the mini → cluster swap. Attempting to create it here would fail and mixes two epics' concerns.

## Acceptance Criteria

**AC1 — Telegram refinement bot created; production bot identity reserved**

- **Given** Story 1.1 is complete and the n8n credentials list is empty
- **When** the operator opens Telegram and creates a new bot via @BotFather for the refinement environment
- **Then** the operator captures the refinement bot token and sets an intelligible bot name + username (e.g., `Maruzza Refinement Bot` / `@maruzza_refinement_bot`)
- **And** the operator separately creates (or reserves) a **distinct production bot identity** with its own token via @BotFather — **not activated yet** (AR5); the production token is held ready for Epic 10 Story 10.5's Launch Gate credential switch, never wired to any workflow in this story
- **And** the two tokens are distinct (never share a token across environments)

**AC2 — Seven locked credentials registered in n8n with exact committed names (`evolution_api` intentionally deferred)**

- **Given** the operator has both Telegram tokens and the existing infra coordinates verified in Story 1.1
- **When** the operator opens n8n Settings → Credentials and registers each entry below with its **exact** committed name (Architecture §Credential Naming):
  - `telegram_bot_refinement` — Telegram API credential; token = refinement bot token from AC1
  - `telegram_bot_production` — Telegram API credential; token = production bot token from AC1 (reserved; no workflow will reference this credential until Epic 10 Story 10.5)
  - `groq_free` — OpenAI-compatible credential pointing at `https://api.groq.com/openai/v1` with the free-tier API key; primary operational LLM credential (Architecture §LLM provider failover)
  - `groq_paid` — OpenAI-compatible credential pointing at the same Groq base URL with a paid-tier API key; fallback credential for Epic 2+ AI Agent failover
  - `comadre_tts` — HTTP Header Auth (or Generic with `Authorization` header) credential; header name `Authorization`, header value `Bearer <COMADRE_API_KEY>` (the key value lives in `~/dev/comadre/.env` or Galvani's password manager — `API_KEY` is enforced mandatory at Comadre service start per `~/dev/comadre/compose.yaml`)
  - `postgres_main` — Postgres credential; host `postgresql`, port `5432`, database `n8n_db`, user `n8n_user`, password = source-of-truth value of the `n8n_db_password` Docker Swarm secret (the single application database shipped by borgstack-mini; hosts `n8n_chat_histories`, `client_profiles`, `chat_analytics`, `static_responses`, staging tables)
  - `redis_main` — Redis credential; host `redis`, port `6379`, database index `0`, password = source-of-truth value of the `redis_password` Docker Swarm secret
- **Then** all seven credentials exist in n8n with the committed names spelled exactly as listed (AR41), each successfully tested via n8n's built-in "Test connection" / "Test step" affordance where the credential type supports it (Postgres, Redis, Groq via a probe HTTP call; Comadre via a `POST /v1/audio/speech` probe)
- **And** `evolution_api` is **NOT** created in this story — the locked name is documented in the checklist (AC6) as reserved for Epic 8, but no credential entity is instantiated now
- **And** no credential value (token, API key, password) appears in plain text in any workflow field, Code node, log output, markdown file, Git commit, chat window, or exported JSON (NFR9, reinforces the Story 1.1 hygiene that got briefly deviated from for the Comadre key)

**AC3 — Eleven locked environment variables registered on the n8n containers and readable from workflows**

- **Given** the locked env var list from Architecture §Environment Variable Naming
- **When** the operator edits `~/dev/borgstack/templates/stack-mini.yml.j2` (editable IaC — see Dev Notes for the exact location) and appends the following entries to the `environment:` block of **all three** n8n services (`n8n-editor`, `n8n-webhook`, `n8n-worker`), with values either inline in the Ansible template or sourced from `~/dev/borgstack/inventory/mini/group_vars/all/main.yml` (non-secret values) or `vault.yml` (any firm-specific value considered sensitive)
- **Then** the following 11 env vars are present and set as specified:
  - `ENVIRONMENT=refinement` (flipped to `production` by Epic 10 Launch Gate; this story sets it to `refinement`)
  - `RATE_LIMIT_BURST=30`
  - `RATE_LIMIT_BURST_WINDOW_SEC=300`
  - `RATE_LIMIT_DAILY=200`
  - `MODEL_PRIMARY=llama-3.3-70b-versatile` (or the currently-chosen Groq 7–12B multilingual model identifier — record whichever value is live)
  - `MODEL_WHISPER=whisper-large-v3-turbo` (or the currently-chosen Groq Whisper model identifier)
  - `TTS_VOICE=kokoro/pm_santa`
  - `FIRM_NAME=Maruzza Teixeira Advocacia`
  - `FIRM_CITY=São Luís/MA`
  - `HANDOFF_GROUP_CHAT_ID=<value from AC4>` (negative integer captured in AC4)
  - `OPS_GROUP_CHAT_ID=<value from AC5>` (negative integer captured in AC5)
- **And** after redeploying the stack (`ansible-playbook` against `inventory/mini` or the operator's usual borgstack-mini deploy command), all three n8n containers are healthy and running
- **And** every env var is readable from a **temporary** verification workflow's Set node via `{{ $env.VAR_NAME }}` expressions (AR44 expression style), producing the exact values registered — verified by executing the workflow once and eyeballing the output
- **And** after a full n8n container restart (`docker service update --force` on the three n8n services, or equivalent), every env var is still present and unchanged — confirming the values are container-env-level, not editor-session state
- **And** `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` is already set in `stack-mini.yml.j2` (verified in Story 1.1 context; allows `$env.*` access in workflow expressions) — no change required here, just a confirmation that this prerequisite holds

**AC4 — Private Telegram handoff group created and `chat_id` captured**

- **Given** the `telegram_bot_refinement` credential from AC2 is registered
- **When** the operator creates a **private** Telegram group for handoff notifications (suggested name: `Maruzza — Handoff Leads`), adds the refinement bot as a member with permission to post messages, and obtains the group's `chat_id`
- **Then** the `chat_id` is a **negative integer** (Telegram convention for groups / supergroups; positive integers are private user chats)
- **And** the operator sets this value as the `HANDOFF_GROUP_CHAT_ID` env var in AC3's stack-mini update

**AC5 — Private Telegram ops group created and `chat_id` captured, distinct from handoff**

- **Given** the handoff group from AC4 exists with its `chat_id` captured
- **When** the operator creates a **second private** Telegram group for ops alerts and weekly reports (suggested name: `Maruzza — Ops Alerts`), adds the refinement bot as a member with post permission, and obtains the second group's `chat_id`
- **Then** the `chat_id` is a negative integer and **distinct** from `HANDOFF_GROUP_CHAT_ID` (AR38 — two separate groups to prevent HITL lead notifications from being lost in ops noise and vice versa)
- **And** the operator sets this value as the `OPS_GROUP_CHAT_ID` env var in AC3's stack-mini update

**AC6 — `docs/credentials-checklist.md` authored and committed**

- **Given** credentials are registered in n8n (AC2) and Telegram bots exist (AC1)
- **When** the operator authors `docs/credentials-checklist.md`
- **Then** the file lists **all 8 locked credential names** in the committed order: `telegram_bot_refinement`, `telegram_bot_production`, `groq_free`, `groq_paid`, `comadre_tts`, `postgres_main`, `redis_main`, `evolution_api`
- **And** each entry records: exact committed name, n8n credential type (e.g., "Telegram API", "OpenAI / Generic with base URL override", "HTTP Header Auth", "Postgres", "Redis"), its purpose in one sentence, and step-by-step creation instructions referencing where the value comes from (BotFather, Groq console, Comadre `.env`, Docker Swarm secret, etc.) — **never the actual value**
- **And** the `evolution_api` entry explicitly notes it is **reserved** for Epic 8 Story 8.2, **not created in Story 1.2**, and states the availability gate (mini → cluster swap or standalone Evolution API deploy)
- **And** the `comadre_tts` entry includes a short "known follow-up" note flagging the optional key rotation window at this registration point — a neutral structural cue promoted from Story 1.1's review findings, with **no** narrative about the prior transcript exposure (Story 1.1 Dev Agent Record already documents the incident)
- **And** the file is in English (Architecture §Language Policy) and contains **zero** secret values (NFR9)
- **And** the file is committed to the repo in English with an imperative-mood, under-72-char commit subject

**AC7 — `docs/environment-variables.md` authored and committed**

- **Given** env vars are registered on the n8n containers (AC3) and Telegram group IDs are captured (AC4, AC5)
- **When** the operator authors `docs/environment-variables.md`
- **Then** the file lists **all 11 locked env vars** in the committed order from Architecture §Environment Variable Naming: `ENVIRONMENT`, `RATE_LIMIT_BURST`, `RATE_LIMIT_BURST_WINDOW_SEC`, `RATE_LIMIT_DAILY`, `MODEL_PRIMARY`, `MODEL_WHISPER`, `TTS_VOICE`, `FIRM_NAME`, `FIRM_CITY`, `HANDOFF_GROUP_CHAT_ID`, `OPS_GROUP_CHAT_ID`
- **And** each entry records: exact `SCREAMING_SNAKE_CASE` name, purpose, expected type (integer / string / model identifier / negative integer for group chats), default value for this firm (Maruzza), and per-firm tuning notes for the refinement → production journey
- **And** the `ENVIRONMENT` entry explicitly flags the refinement-vs-production semantics: value `refinement` during Epic 1–9, flipped to `production` at Epic 10 Launch Gate
- **And** the `telegram_bot_refinement` vs `telegram_bot_production` credential split is cross-referenced here (credentials are not env vars, but the environment split is identical — call this out to future operators)
- **And** `HANDOFF_GROUP_CHAT_ID` and `OPS_GROUP_CHAT_ID` are documented as **negative integers** (Telegram group convention) with a reminder that the two IDs must be distinct (AR38)
- **And** the file is in English and contains **zero** secret values (no bot tokens, no API keys, no passwords; the chat IDs are not secrets — they are locked operational values that a random Telegram user cannot post to without also being in the group)
- **And** the file is committed to the repo in English with an imperative-mood, under-72-char commit subject

**AC8 — Verification workflow run and then discarded; credentials list contains exactly 7 entries**

- **Given** AC2, AC3 are green
- **When** the operator creates a **temporary** verification workflow named `_scratch_verify-bootstrap` on an empty canvas containing a Manual Trigger + Set node that outputs every one of the 11 env vars via `{{ $env.VAR_NAME }}` expressions, plus small probe nodes per credential type (a Postgres node running `SELECT 1`, a Redis node running `SET _verify_bootstrap "ok" EX 30`, an HTTP Request node calling Groq `GET /v1/models` with credential `groq_free`, and an HTTP Request node calling Comadre `POST /v1/audio/speech` with credential `comadre_tts`)
- **Then** the workflow executes successfully, the Set node output shows the 11 env var values verbatim, and each credential probe returns a successful response (Postgres `ping=1`, Redis `OK`, Groq 200 with a JSON model list, Comadre 200 with OGG Opus binary)
- **And** after evidence is captured into the task log, the `_scratch_verify-bootstrap` workflow is **deleted** from the n8n editor so Story 1.3 begins with an empty workflow canvas
- **And** the n8n Credentials list shows **exactly 7 entries** (the seven committed names, no more, no fewer) and the Workflows list is empty

## Tasks / Subtasks

- [ ] **Task 1 — Create Telegram refinement bot and reserve production bot identity (AC: 1)**
  - [ ] Open @BotFather in the operator's personal Telegram account
  - [ ] `/newbot` — name: `Maruzza Refinement Bot`, username: `@maruzza_refinement_bot` (or whatever is available); capture the token
  - [ ] `/newbot` again for the production bot — name: `Maruzza Bot`, username: `@maruzza_bot` (or whatever is available); capture the token but **do not** activate or configure webhooks yet
  - [ ] Store both tokens in the operator's password manager under distinct entries (`maruzza-refinement-bot-token`, `maruzza-production-bot-token`) — **do not** write them to any repo file, chat window, or n8n Code node
  - [ ] Confirm the two tokens are different strings

- [ ] **Task 2 — Register `telegram_bot_refinement` and `telegram_bot_production` in n8n (AC: 2)**
  - [ ] Open n8n Settings → Credentials → New credential → "Telegram API"
  - [ ] Name: `telegram_bot_refinement` (exact spelling, lowercase with underscores); paste the refinement token from Task 1
  - [ ] Click "Save", then "Test step" (n8n performs a `getMe` call); confirm success
  - [ ] Repeat for `telegram_bot_production` with the production token; confirm "Test step" success
  - [ ] Neither token value is copied into any workflow field, Code node, export, or chat window

- [ ] **Task 3 — Register `groq_free` and `groq_paid` in n8n (AC: 2)**
  - [ ] Obtain (or locate existing) free-tier and paid-tier Groq API keys from `https://console.groq.com/keys` (separate accounts or separate keys on the same account — operator's choice)
  - [ ] Credentials → New → choose "OpenAI" credential type (Groq is OpenAI-compatible); override base URL to `https://api.groq.com/openai/v1`
  - [ ] Name: `groq_free`; paste the free-tier key; Save
  - [ ] Quick probe: create a throwaway HTTP Request node (do not save the workflow) calling `GET https://api.groq.com/openai/v1/models` with credential `groq_free` — confirm 200 and a JSON model list is returned
  - [ ] Repeat for `groq_paid` with the paid-tier key; same probe; confirm 200
  - [ ] Discard the throwaway probe node without saving

- [ ] **Task 4 — Register `comadre_tts` in n8n with Bearer header (AC: 2)**
  - [ ] Fetch `COMADRE_API_KEY` value from `~/dev/comadre/.env` (read-only access to `~/dev/` per `CLAUDE.md`) or from Galvani's password manager — whichever is the current source of truth
  - [ ] Consider rotating the key now as the Story 1.1 review's promoted "known follow-up" suggests: generate a new random key, update `~/dev/comadre/.env`, restart Comadre (`docker compose restart` in `~/dev/comadre/`), then use the new value here. This is optional but encouraged at this natural "clean slate" moment (LAN-only service, no external consumers, trivial rotation cost)
  - [ ] Credentials → New → choose "Header Auth" credential type (or Generic credential with a custom header)
  - [ ] Name: `comadre_tts`; header name `Authorization`; header value `Bearer <THE_KEY_VALUE_PASTED_INLINE_HERE_ONCE>`
  - [ ] Save. The value is now encrypted at rest by `n8n_encryption_key`; never copy it out again
  - [ ] Probe: create a throwaway HTTP Request node calling `POST https://<comadre-domain>/v1/audio/speech` with credential `comadre_tts`, JSON body `{"input": "teste", "voice": "kokoro/pm_santa"}`, response format `File` — confirm HTTP 200 and a valid OGG Opus binary
  - [ ] Discard the probe node without saving

- [ ] **Task 5 — Register `postgres_main` in n8n (AC: 2)**
  - [ ] Fetch the `n8n_db_password` Docker Swarm secret's source-of-truth value from `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml` (Ansible Vault-decrypt with the stored vault password; operator's existing flow)
  - [ ] Credentials → New → "Postgres"
  - [ ] Name: `postgres_main`; host `postgresql`; port `5432`; database `n8n_db`; user `n8n_user`; password = the value just fetched; SSL mode: disable (internal Docker network — same convention Story 1.1 used for its ad-hoc Postgres credential)
  - [ ] Save; "Test step" (n8n probes the connection); confirm success
  - [ ] Quick probe: a throwaway `SELECT 1 AS ping;` execution confirms end-to-end connectivity

- [ ] **Task 6 — Register `redis_main` in n8n (AC: 2)**
  - [ ] Fetch the `redis_password` Docker Swarm secret's source-of-truth value from `vault.yml` (same way as Task 5)
  - [ ] Credentials → New → "Redis"
  - [ ] Name: `redis_main`; host `redis`; port `6379`; database index `0`; password = the value just fetched
  - [ ] Save; probe with a throwaway `SET _verify_redis_main "ok" EX 10` + `GET _verify_redis_main` → confirm `ok`

- [ ] **Task 7 — Confirm `evolution_api` is deliberately absent (AC: 2)**
  - [ ] Visually confirm the n8n Credentials list contains exactly the 7 credentials from Tasks 2–6, and **no** `evolution_api` entry exists
  - [ ] If `evolution_api` somehow appeared (e.g., residual from an earlier experiment), delete it now — Epic 8 Story 8.2 owns its registration

- [ ] **Task 8 — Create the two private Telegram groups and capture chat IDs (AC: 4, 5)**
  - [ ] In Telegram: new group → name `Maruzza — Handoff Leads` → add the refinement bot as a member → set bot permission to post messages (in Telegram group settings, the bot must not be restricted)
  - [ ] Capture the group's `chat_id`:
    - Method A (simplest): temporarily disable group privacy on the bot via @BotFather (`/setprivacy` → Disable), send any message in the group, then hit `https://api.telegram.org/bot<REFINEMENT_TOKEN>/getUpdates` from a browser or curl — the response JSON's `result[*].message.chat.id` is the group chat ID (negative integer). Re-enable privacy after capture (`/setprivacy` → Enable).
    - Method B: add a third-party helper bot like `@RawDataBot` temporarily, read the `chat.id` value from its echo, then remove the helper bot.
    - Method C: create a temporary n8n workflow with a Telegram Trigger on the refinement bot, send `/id` to the group, read the `message.chat.id` from the execution output, then discard the workflow.
  - [ ] Record the handoff chat ID (negative integer) for AC4 / Task 9
  - [ ] Repeat the entire step for the second group `Maruzza — Ops Alerts`; capture its distinct chat ID for AC5 / Task 9
  - [ ] Confirm the two chat IDs are **different** numbers

- [ ] **Task 9 — Append 11 env vars to `stack-mini.yml.j2` and redeploy borgstack-mini (AC: 3)**
  - [ ] Edit `~/dev/borgstack/templates/stack-mini.yml.j2` (the operator's own BorgStack fork; Galvani has write access there outside this repo's permissions — the `CLAUDE.md` read-only rule for `~/dev/` applies to the n8n-chatbot project's Claude context, not to Galvani's operator actions)
  - [ ] Locate the three n8n service `environment:` blocks: `n8n-editor` (around line 210), `n8n-webhook` (around line 296), `n8n-worker` (around line 381) — verify the exact line numbers against the current file before editing
  - [ ] Append the following 11 entries to **each** of the three environment lists (identical list in all three, because worker executes the same workflows as editor and webhook and must see the same env):
    ```yaml
          # Chatbot configuration (Story 1.2)
          - ENVIRONMENT=refinement
          - RATE_LIMIT_BURST=30
          - RATE_LIMIT_BURST_WINDOW_SEC=300
          - RATE_LIMIT_DAILY=200
          - MODEL_PRIMARY=llama-3.3-70b-versatile
          - MODEL_WHISPER=whisper-large-v3-turbo
          - TTS_VOICE=kokoro/pm_santa
          - FIRM_NAME=Maruzza Teixeira Advocacia
          - FIRM_CITY=São Luís/MA
          - HANDOFF_GROUP_CHAT_ID=<value from Task 8>
          - OPS_GROUP_CHAT_ID=<value from Task 8>
    ```
  - [ ] Use an Ansible-native approach if preferred: move the values into `inventory/mini/group_vars/all/main.yml` (e.g., as a `chatbot_env` dict) and reference them with `{{ }}` expansion in the Jinja template. Either approach is fine — the simpler inline form is acceptable for a single-firm install
  - [ ] Redeploy: run the operator's usual `ansible-playbook -i inventory/mini playbooks/deploy-mini.yml` (or equivalent), wait for the three n8n services to report healthy
  - [ ] `docker service ls` confirms `borgstack-mini_n8n-editor`, `_n8n-webhook`, `_n8n-worker` are all `1/1` replicas up
  - [ ] `docker exec` into any n8n container and run `env | grep -E '^(ENVIRONMENT|RATE_LIMIT|MODEL|TTS_VOICE|FIRM|HANDOFF_GROUP_CHAT_ID|OPS_GROUP_CHAT_ID)='` — confirm all 11 values are present with the expected values
  - [ ] Commit the `stack-mini.yml.j2` + `main.yml` changes inside the **borgstack repo** (`~/dev/borgstack`) — this is an operator action outside the n8n-chatbot repo, not part of this story's Git activity, but the commit discipline still applies

- [ ] **Task 10 — Run `_scratch_verify-bootstrap` workflow to confirm env vars are reachable via `$env.*` and all 7 credentials probe green (AC: 3, 8)**
  - [ ] On an empty n8n canvas, create a temporary workflow `_scratch_verify-bootstrap`
  - [ ] Add a Manual Trigger
  - [ ] Add a Set node `Echo All Env Vars` (Title Case + imperative + object — AR40) with fields mapping each `$env.VAR_NAME` to an output of the same name:
    - `ENVIRONMENT = {{ $env.ENVIRONMENT }}`
    - `RATE_LIMIT_BURST = {{ $env.RATE_LIMIT_BURST }}`
    - `RATE_LIMIT_BURST_WINDOW_SEC = {{ $env.RATE_LIMIT_BURST_WINDOW_SEC }}`
    - `RATE_LIMIT_DAILY = {{ $env.RATE_LIMIT_DAILY }}`
    - `MODEL_PRIMARY = {{ $env.MODEL_PRIMARY }}`
    - `MODEL_WHISPER = {{ $env.MODEL_WHISPER }}`
    - `TTS_VOICE = {{ $env.TTS_VOICE }}`
    - `FIRM_NAME = {{ $env.FIRM_NAME }}`
    - `FIRM_CITY = {{ $env.FIRM_CITY }}`
    - `HANDOFF_GROUP_CHAT_ID = {{ $env.HANDOFF_GROUP_CHAT_ID }}`
    - `OPS_GROUP_CHAT_ID = {{ $env.OPS_GROUP_CHAT_ID }}`
  - [ ] Add a Postgres node `Probe postgres_main` running `SELECT 1 AS ping;` with credential `postgres_main`
  - [ ] Add a Redis node `Probe redis_main` running `SET _verify_bootstrap "ok" EX 30` with credential `redis_main` (run within a single execution; recall the Story 1.1 lesson that TTL-bounded operations must fire back-to-back inside one run)
  - [ ] Add an HTTP Request node `Probe groq_free` calling `GET https://api.groq.com/openai/v1/models` with credential `groq_free`
  - [ ] Add an HTTP Request node `Probe comadre_tts` calling `POST https://<comadre-domain>/v1/audio/speech` with credential `comadre_tts`, JSON body `{"input": "teste bootstrap", "voice": "{{ $env.TTS_VOICE }}"}`, response format `File`
  - [ ] Execute Workflow → confirm: all 11 env var values in the Set output match Task 9's expected values; all 4 probes return HTTP 200 / success
  - [ ] Send a one-line test message to the handoff group and the ops group each (via a throwaway Telegram Send Text node using `telegram_bot_refinement` and the respective `chat_id`) — confirm the messages arrive in the correct groups. This validates both that the bot was added with post permission and that the chat IDs are wired to the right groups

- [ ] **Task 11 — Restart the n8n services to confirm env var persistence (AC: 3)**
  - [ ] Run `docker service update --force borgstack-mini_n8n-editor borgstack-mini_n8n-webhook borgstack-mini_n8n-worker` (or equivalent `docker service scale 0 → 1` cycle)
  - [ ] Wait for all three services to report healthy again
  - [ ] Re-execute `_scratch_verify-bootstrap` once — confirm the env var values are unchanged and the credentials still pass their probes

- [ ] **Task 12 — Delete `_scratch_verify-bootstrap` workflow; confirm workflow list is empty (AC: 8)**
  - [ ] In the n8n editor, delete `_scratch_verify-bootstrap`
  - [ ] Confirm the Workflows list is empty (Story 1.3 begins with an empty canvas)
  - [ ] Confirm the Credentials list contains exactly the 7 committed names from Tasks 2–6 — no more, no fewer

- [ ] **Task 13 — Author `docs/credentials-checklist.md` and commit (AC: 6)**
  - [x] Create `docs/credentials-checklist.md` in the n8n-chatbot repo
  - [x] Contents (English; zero secret values):
    - Short intro: "This checklist lists the 8 locked credential names used by the template. A new firm importing the workflow recreates these credentials with their own values; the names stay identical so the workflow does not need to change."
    - A table or itemized list covering all 8 names in the committed order from Architecture §Credential Naming: `telegram_bot_refinement`, `telegram_bot_production`, `groq_free`, `groq_paid`, `comadre_tts`, `postgres_main`, `redis_main`, `evolution_api`
    - For each entry: committed name (monospace), n8n credential type, one-sentence purpose, step-by-step creation instructions that reference where the value comes from (BotFather, Groq console, Comadre `.env`, Docker Swarm secret, etc.)
    - `evolution_api` row is marked **"Reserved — created in Epic 8 Story 8.2"** with a one-sentence explanation (cluster-only service; mini does not ship it)
    - `comadre_tts` row includes a one-line "Known follow-up" note recommending key rotation at the moment of registration
    - Explicit NFR9 reminder: no credential value ever enters a workflow field, Code node, log, export, or chat window
  - [x] Language: English (per `CLAUDE.md` and Architecture §Language Policy)
  - [ ] Commit: `docs/credentials-checklist.md`; imperative subject under 72 chars (suggestion: `Document locked credential checklist for template import`) — **pending operator approval; file authored on disk**

- [ ] **Task 14 — Author `docs/environment-variables.md` and commit (AC: 7)**
  - [x] Create `docs/environment-variables.md` in the n8n-chatbot repo
  - [x] Contents (English; zero secret values):
    - Short intro: "This document lists the 11 locked environment variables used by the template. They are injected into the n8n containers via BorgStack's `stack-mini.yml.j2` template and read from workflow nodes via `{{ $env.VAR_NAME }}` expressions (requires `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`, already set by BorgStack)."
    - A table covering all 11 names in the committed order from Architecture §Environment Variable Naming
    - For each entry: exact name (monospace), type (integer / string / model identifier / negative integer / etc.), default value for this firm (Maruzza), per-firm tuning notes, refinement-vs-production notes
    - `ENVIRONMENT` row: explicit note that it is `refinement` during Epic 1–9 and flipped to `production` at Epic 10 Launch Gate (credentials `telegram_bot_refinement` / `telegram_bot_production` follow the same split; cross-reference to `credentials-checklist.md`)
    - `HANDOFF_GROUP_CHAT_ID` / `OPS_GROUP_CHAT_ID` rows: negative integer format; must be distinct (AR38); creation method documented briefly
    - Operational note at the bottom: "To add a new env var to the template, edit `stack-mini.yml.j2` environment blocks for all three n8n services (editor, webhook, worker), redeploy, and update this file."
  - [x] Language: English
  - [ ] Commit: `docs/environment-variables.md`; imperative subject under 72 chars (suggestion: `Document locked environment variables for template import`) — **pending operator approval; file authored on disk**

## Dev Notes

### Nature of this story

This is the **second** story of Epic 1. Story 1.1 left n8n with an **empty** credentials list and an **empty** workflow canvas. Story 1.2 populates the foundation that every future workflow story consumes by name: 8 locked credential names (one deferred), 11 locked env var names, 2 Telegram group chat IDs. The deliverable is **operational state inside n8n and borgstack-mini** plus **two markdown documents in the repo**. Story 1.3 then consumes these names when creating empty-canvas workflow shells.

This is a **configuration and documentation** story. There is **no workflow topology built here** beyond a single disposable `_scratch_verify-bootstrap` workflow used to verify that env vars and credentials resolve correctly; that workflow is deleted in Task 12.

### Ground truth sources consulted while writing this story

Every concrete value below was extracted from the actual BorgStack, Comadre, and project IaC — not from planning documents:

- `~/dev/borgstack/templates/stack-mini.yml.j2` — the exact `environment:` blocks of `n8n-editor`, `n8n-webhook`, `n8n-worker` services (lines ~210, ~296, ~381). `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` is set here; no hook for chatbot-scoped env vars exists yet, so Task 9 appends them directly to the three environment lists
- `~/dev/borgstack/inventory/mini/group_vars/all/main.yml` — `mini_n8n_domain`, `timezone`, `databases:` list confirms `n8n_db` / `n8n_user` / `n8n_db_password` as the single app DB
- `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml` — Ansible Vault-encrypted source of truth for `n8n_db_password`, `redis_password`, `n8n_encryption_key` (verified in Story 1.1)
- `~/dev/borgstack/docs/mini-architecture.md` — confirms Evolution API is cluster-only, justifying `evolution_api` deferral
- `~/dev/comadre/compose.yaml` — `API_KEY: ${API_KEY:?API_KEY must be set}` proves the Bearer auth is mandatory
- `~/dev/comadre/deploy/caddy/Caddyfile` — Comadre's own Caddy reverse proxy (HTTPS); not `:8000`
- `~/dev/comadre/README.md` — OpenAI-compatible `POST /v1/audio/speech`, default OGG Opus output, `kokoro/pm_santa` default voice
- `_bmad-output/planning-artifacts/architecture.md` §Credential Naming — the 8 committed credential names
- `_bmad-output/planning-artifacts/architecture.md` §Environment Variable Naming — the 11 committed env var names
- `_bmad-output/planning-artifacts/architecture.md` §Language Policy — English for all technical artifacts
- `_bmad-output/planning-artifacts/architecture.md` §Bootstrap checklist — items 5–10, 13 are covered by this story; item 11 (Evolution API) is deferred
- `_bmad-output/implementation-artifacts/1-1-infrastructure-verification-n8n-platform-pinning.md` — previous story's discipline around empty canvas, scratch naming, deletion, secret handling

### Relevant architecture patterns and constraints

- **Credential names are load-bearing contracts, not labels.** Every future workflow story references credentials by the exact name. A typo or rename at this stage breaks every downstream story silently. Triple-check spelling in Tasks 2–6 against Architecture §Credential Naming (lines ~1015–1029) — all lowercase, underscore-separated, no trailing whitespace
- **Env var names are equally load-bearing.** `FIRM_NAME`, `MODEL_PRIMARY`, `TTS_VOICE`, etc. are interpolated into system prompts, HTTP bodies, Telegram messages, and handoff cards by downstream stories. A misspelled `MODEL_PRIMARYY` produces silent empty strings in workflow expressions, not a hard error — hence the `_scratch_verify-bootstrap` Task 10 enforcement
- **Env vars on borgstack-mini are container-env-level, not n8n-editor-session state.** Unlike n8n's editor-UI "Environments" feature, which applies per-workflow inside the editor session, container env vars (`stack-mini.yml.j2` → `environment:`) persist across n8n restarts and queue-mode worker spawns. This is the correct layer for `ENVIRONMENT`, `FIRM_NAME`, etc. (Architecture §Platform version pinning implicit; AC3 "after restart" test)
- **`N8N_BLOCK_ENV_ACCESS_IN_NODE=false` is already set by BorgStack** (`stack-mini.yml.j2` line ~245 / ~330 / ~415). This allows `$env.*` access inside workflow expressions, which is required by AR44 / AR42. Do **not** change this value in Task 9 — just confirm it is still set after the deploy
- **Worker sees the same env as editor.** n8n queue mode executes workflows in the worker container, not the editor. If env vars are added only to the editor service, workflow runs will see undefined values at execution time. Task 9 explicitly requires appending to **all three** environment blocks
- **`N8N_ENCRYPTION_KEY` makes credential values unreadable at rest without the key.** Once a credential is saved in n8n, its value is encrypted with the Swarm secret verified in Story 1.1 (AC5). Backups are useless without the key; hence Story 1.1's out-of-band vault-password backup gate. This story adds 7 encrypted credential values to the encrypted-at-rest pool — meaning a vault-password loss now has a larger recovery cost, reinforcing why that backup was load-bearing (Architecture §Secrets and credentials)
- **`evolution_api` is a cluster-only integration.** borgstack-mini does not ship Evolution API (`stack-mini.yml.j2` has no `evolution` service). Creating the credential now would require inventing fake values, which defeats the "credential names are contracts" principle. Epic 8 Story 8.2 registers `evolution_api` as a prefix step alongside the mini → cluster swap (or a standalone Evolution API deploy). Do **not** try to create a stub credential (Architecture §Evolution API integration; `~/dev/borgstack/docs/mini-architecture.md`)
- **Telegram chat IDs for groups are negative integers; private user chats are positive.** When a group is upgraded to a supergroup, the chat ID format changes (still negative, larger absolute value, with a `-100` prefix). Both forms are acceptable values of `HANDOFF_GROUP_CHAT_ID` / `OPS_GROUP_CHAT_ID`; the downstream workflows interpolate them as-is without arithmetic
- **Two Telegram groups, distinct by design (AR38).** The handoff group receives lead notifications that attorneys act on (`Accept` / `Decline` button callbacks in Epic 4). The ops group receives system alerts and weekly reports (Epic 6, Epic 9). Mixing the two would cause HITL leads to get lost in ops noise, or cause an ops alert cascade to flood the handoff channel during an incident. The `chat_id` distinctness check is a hard AC
- **Node naming convention applies even to disposable workflows.** `_scratch_verify-bootstrap` follows the `_scratch_` prefix that Story 1.1 established for temporary verification workflows. Nodes inside use Title Case imperative verb + object names (e.g., `Echo All Env Vars`, `Probe postgres_main`) to build muscle memory for later stories (Architecture §n8n Workflow Naming; AR40)
- **English for technical artifacts.** `docs/credentials-checklist.md` and `docs/environment-variables.md` are technical documents and must be in English. `FIRM_NAME` and `FIRM_CITY` values are in Portuguese because they represent Brazilian firm data (user-facing strings), which is permissible content (Architecture §Language Policy)
- **Visual-first, no Code nodes, JavaScript-only if a Code node becomes unavoidable.** None of the ACs in this story require a Code node. Any Code node here would be an unjustified exception (Architecture §Code Node Acceptance Criteria; project CLAUDE.md + user memory `feedback_javascript_only_in_code_nodes.md`)

### Secret handling discipline (reinforces NFR9)

This story **writes secret values into n8n credentials** for the first time (Story 1.1 used only disposable ad-hoc credentials and inline headers). Every handling gesture must keep the value **inside n8n's encrypted storage** and **nowhere else**:

- ✅ Paste the value **once** into the n8n credential form field; click Save; never copy it out again
- ✅ Keep password-manager entries for each secret (BotFather tokens, Groq API keys, Comadre key, Docker Swarm secret values) as the permanent source of truth; n8n holds encrypted copies
- ❌ Do **not** paste any secret value into a workflow field, Code node, Set node, HTTP Request header override, log output, exported JSON, markdown file, Git commit, issue tracker, or chat window
- ❌ Do **not** echo secrets to terminal (the Story 1.1 review's promoted "known follow-up" was triggered by exactly this — a `grep API_KEY=...` emission to stdout). If you must read `.env` contents, redirect to a known-private path or use a tool that masks the value
- ❌ Do **not** include secret values in `docs/credentials-checklist.md` — it documents **names** and **types**, never values

### Dev Agent Guardrails

- **Do not create the `evolution_api` credential.** Any attempt is out of scope and based on an outdated premise. The locked name is listed in `docs/credentials-checklist.md` as reserved; no credential entity is instantiated in this story
- **Do not rename the locked credentials or env vars.** All 8 credential names and all 11 env var names are the committed template contract. A rename here cascades into every downstream story silently. If you think a name should change, stop and escalate — do not "fix" it
- **Do not set `ENVIRONMENT=production` in this story.** The value is `refinement` for Epic 1–9. Epic 10 Story 10.5 owns the flip
- **Do not activate `telegram_bot_production` against any workflow.** The credential is registered (committed contract) but remains unused until Epic 10 Story 10.5 clones `main-chatbot-refinement` to `main-chatbot-production` with the production credential substituted
- **Do not propose Edge TTS, Piper, Kokoro Web, OpenAI TTS, or OpenEdAI.** Comadre is the only TTS engine (project user-memory `feedback_tts_always_comadre.md`)
- **Do not propose Haiku or GPT-4o-mini or any paid frontier model for `MODEL_PRIMARY`.** The current and committed primary is a Groq free-tier 7–12B multilingual model (`llama-3.3-70b-versatile` or current equivalent). Future migration is to a flat-rate monthly API (project user-memory `feedback_model_selection_strategy.md`)
- **Do not use Python in any Code node.** JavaScript only if a Code node becomes unavoidable — which it does not here (project user-memory `feedback_javascript_only_in_code_nodes.md`)
- **Do not output n8n workflow JSON as part of this story.** This is a no-code, visual-first project. The operator builds in the editor. The permitted Git artifacts are two markdown files plus commits. No `exports/workflow-snapshots/` file is written in this story — Story 1.3 owns the first snapshot (project user-memory `feedback_no_n8n_workflow_code.md`)
- **Do not hardcode firm-specific values in workflows.** All firm data comes from `FIRM_NAME`, `FIRM_CITY`, `TTS_VOICE`, etc. The Maruzza values are defaults for this instance; a new firm re-imports the template and changes only these values (NFR39, AR42)
- **Do not skip Task 11 (restart test).** If env vars were accidentally set via a non-durable mechanism (e.g., `docker exec container export VAR=...` or only in the editor UI), Task 10 would pass but Task 11 would fail. The restart test is the real durability gate
- **Do not push to the borgstack-mini deploy without operator confirmation.** Task 9 modifies BorgStack IaC and redeploys the stack. That is a shared-environment action; confirm Galvani's readiness before applying (especially if other services on borgstack-mini might be perturbed by a redeploy window)
- **Do not write destructive operations outside `/home/galvani/dev/n8n-chatbot/`.** Read-only access to `~/dev/` is permitted for BorgStack and Comadre source consultation. Edits to `~/dev/borgstack/` in Task 9 are operator actions performed by Galvani, not agent-initiated writes (project `CLAUDE.md` permissions)

### Library, framework, and version requirements relevant to this story

| Component | Version / identity | Source |
|---|---|---|
| n8n | any 2.x release (current BorgStack pin `2.11.3`) — reads `$env.*` via `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` | `~/dev/borgstack/inventory/shared/images.yml`, `stack-mini.yml.j2:245` |
| n8n credential types used | "Telegram API", "OpenAI" (with base URL override for Groq), "Header Auth", "Postgres", "Redis" | n8n built-in credential catalog |
| Groq base URL | `https://api.groq.com/openai/v1` (OpenAI-compatible) | Architecture §LLM call pattern, AR30 |
| Comadre endpoint | `POST https://<comadre-domain>/v1/audio/speech` (HTTPS via Comadre's own Caddy) | `~/dev/comadre/README.md`, `~/dev/comadre/deploy/caddy/Caddyfile` |
| Comadre auth | Bearer token via `Authorization: Bearer <COMADRE_API_KEY>` header | `~/dev/comadre/compose.yaml` |
| Comadre default voice | `kokoro/pm_santa` (neural male, pt-BR) | `~/dev/comadre/deploy/ansible/inventory/production.yml.example` |
| `MODEL_PRIMARY` candidate | `llama-3.3-70b-versatile` (Groq) or current chosen Groq 7–12B multilingual model | Architecture §Environment Variable Naming |
| `MODEL_WHISPER` candidate | `whisper-large-v3-turbo` (Groq) | Architecture §Environment Variable Naming |
| borgstack-mini host | `10.10.10.205` | `~/dev/borgstack/inventory/mini/hosts` |
| Comadre host | `10.10.10.207` | `~/dev/comadre/deploy/ansible/inventory/production.yml.example` |
| Docker Swarm secrets consumed (read for pasting into n8n credentials, never exposed) | `n8n_db_password`, `redis_password` | `stack-mini.yml.j2:207–208` |

No new project-side dependencies. No `package.json`. No npm install. The only IaC-side dependency change is the 11 env var lines appended to `stack-mini.yml.j2`.

### File structure requirements

- **Write:** `docs/credentials-checklist.md` (new, English, zero secret values)
- **Write:** `docs/environment-variables.md` (new, English, zero secret values)
- **Commit:** English, imperative mood, under 72 chars per subject (two separate commits, one per file, or one bundled commit — operator's preference)
- **Do not write (in n8n-chatbot repo):** anything under `exports/`, `sql/`, `prompts/`, `audit-bundles/`, `refinement-log/`, `golden-dataset/`, `synthetic-scenarios/`, `red-team-battery/`, `human-test-logs/`. None of those scopes are open yet
- **Do not create:** `package.json`, `node_modules/`, `src/`, `tests/`, `.github/workflows/` (deliberately absent per Architecture §Notable absences)

### Testing standards summary

- **No unit tests, no integration tests, no CI pipeline.** Same as Story 1.1. The testing story for this project lives in Epic 9's offline refinement pipeline, which is still several epics away
- **"Test" for this story = live execution of `_scratch_verify-bootstrap`** against the real borgstack-mini with the real credentials and the real Telegram groups, verified by human observation. Task 10 and Task 11 together are the green gate
- **Green gate:** every AC is binary pass/fail.
  - If AC2 credential probes fail (bad password, wrong host, expired token) → stop and fix the credential before moving on
  - If AC3 env var readback shows empty strings → stop; the env was likely added to the editor only (not worker/webhook), or the stack was not redeployed
  - If AC4/AC5 chat IDs are equal → stop; create a genuinely distinct second group
  - If AC8's final credentials count is not exactly 7 → stop and reconcile (the `_scratch_verify-bootstrap` credentials must all be **reused** locked credentials, not new ad-hoc ones)
- **Evidence format:** commit messages for the two markdown files + Dev Agent Record → Completion Notes List on this story file, with execution timestamps, credential probe outcomes, chat ID capture method used (A / B / C), and the n8n container restart test result

### Previous story intelligence (Story 1.1 → Story 1.2)

From `_bmad-output/implementation-artifacts/1-1-infrastructure-verification-n8n-platform-pinning.md`:

- **Secret-handling discipline was the single failure mode.** Story 1.1 executed cleanly except for one incident: the `COMADRE_API_KEY` value was echoed to the conversation transcript via a `grep` command. Galvani exercised an override at the time (LAN-only service, no external consumers), but promoted a "known follow-up" asking Story 1.2 to consider rotating the key at registration. **Action:** Task 4 explicitly invites key rotation at the `comadre_tts` registration moment as the natural "clean slate" window
- **`_scratch_*` prefix convention is in place.** Story 1.1's three verification workflows used this naming convention. This story extends it with `_scratch_verify-bootstrap`. The prefix is a contract-free hint to future operators that these are disposable — not an n8n feature
- **Credential cleanup worked.** Story 1.1 ended with an **empty** credentials list, so Task 7's "confirm exactly 7 entries" check has a clean starting point. If Galvani re-entered n8n between stories and added experimentation credentials, reconcile manually in Task 7
- **Redis 60s TTL race was a live hazard.** Story 1.1 noted that `SET ... EX 60` followed by a separate manual `GET` can race if the operator pauses to read. This story's Task 10 runs the Redis probe inside a **single** Execute Workflow run with `EX 30`, back-to-back, so the race cannot happen
- **AC-by-AC pass/fail matrix is expected in the final commit.** Story 1.1's review promoted this structural improvement for documentation files. Apply it to `docs/credentials-checklist.md` and `docs/environment-variables.md` where it aids the reader (e.g., a "Status" column indicating which credentials/env vars are created vs reserved)
- **Visual-first + no Code nodes + JavaScript-only held cleanly in Story 1.1.** No Code nodes were needed there; none are needed here
- **English language policy held cleanly.** Both new files continue the discipline; `FIRM_NAME` / `FIRM_CITY` Portuguese values are data, not documentation

### Git intelligence summary

Recent commits (most relevant — most recent first):

- `9771261 Apply Story 1.1 code review findings and close the story` — finalizes Story 1.1 after the 2026-04-12 adversarial review; sets the review discipline baseline this story inherits
- `19c07df Align planning artifacts with borgstack-mini + Comadre ground truth` — updated PRD/architecture/epics to reflect borgstack-mini constraints (Evolution API deferral, Comadre HTTPS-via-Caddy, single `n8n_db`). Story 1.2 consumes these post-correction values directly
- `c8ce4b1 Mark Story 1.1 as review (infrastructure verification done)` — status transition that promoted Story 1.1 past its green gate
- `daa11e7 Document borgstack-mini infrastructure verification (Story 1.1)` — the `docs/infra-verification.md` baseline. This story adds `docs/credentials-checklist.md` and `docs/environment-variables.md` as siblings in the same `docs/` directory, forming the foundation-documentation cluster
- `b4a6c41 Complete Phase 3 solutioning and kick off Phase 4 sprint` — Phase 4 implementation authorization

**Actionable pattern to reuse:** Story 1.1's `docs/infra-verification.md` used an AC-by-AC pass/fail table, explicit cross-references to `~/dev/borgstack/` and `~/dev/comadre/` ground-truth paths, and a concluding "no secret values present" confirmation. Mirror this structure in both new docs for operator-reader consistency.

### Project Structure Notes

- **`docs/credentials-checklist.md`** and **`docs/environment-variables.md`** are both canonical locations per Architecture §Surface 1 — Git Repository Structure (lines ~1498–1499). They sit alongside the Story 1.1 deliverable `docs/infra-verification.md` and the later `docs/deployment-guide.md` (Epic 11), forming the `docs/` operator-facing documentation cluster
- **Alignment with unified project structure:** confirmed against Architecture §Surface 1. Both filenames follow the lowercase-hyphenated convention
- **No conflicts detected.** The `docs/` directory already exists with `infra-verification.md`, `plan.md`, and `claude-code-instruction-quality-guide.md`. The two new files add to the directory without collision
- **Detected variances:** none
- **No `exports/workflow-snapshots/` file is created in this story.** Story 1.3 owns the first snapshot

### References

All technical details cite source paths and sections:

- [Source: `_bmad-output/planning-artifacts/epics.md` § Story 1.2: Credential, environment variable, and Telegram group bootstrap (lines ~503–541)] — canonical acceptance criteria foundation
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Credential Naming (lines ~1015–1029)] — the 8 committed credential names, snake_case format, template-contract rationale
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Environment Variable Naming (lines ~1031–1047)] — the 11 committed env var names, SCREAMING_SNAKE_CASE format, per-firm externalization principle
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Bootstrap checklist ("Step 1 — Infrastructure Setup") items 5–10 and 13 (lines ~371–440)] — the exact operator sequence; this story covers items 5, 7, 8, 9, 10, and 13 in full, and item 6 as "reserved" (production bot identity)
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Secrets and credentials (lines ~666–673)] — n8n Credentials as the only secret store, encryption at rest via `N8N_ENCRYPTION_KEY`, no external vault
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Evolution API integration — HTTP Request node + Availability note (lines ~343–370)] — Evolution API cluster-only; deferral to Epic 8 Story 8.2
- [Source: `_bmad-output/planning-artifacts/architecture.md` § n8n Workflow Naming (lines ~973–1013)] — node naming convention, trigger-with-suffix rule
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Language Policy (lines ~956–972)] — English for all technical artifacts including `docs/*.md`
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Code Node Acceptance Criteria] — JavaScript only; Code nodes require documented justification (none required here)
- [Source: `_bmad-output/planning-artifacts/architecture.md` § LLM call pattern (lines ~675–695)] — Groq OpenAI-compatible endpoint; `groq_free` primary, `groq_paid` fallback failover mechanism
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Surface 1 — Git Repository Structure (lines ~1471–1503)] — `docs/credentials-checklist.md` and `docs/environment-variables.md` canonical locations
- [Source: `_bmad-output/planning-artifacts/prd.md` § FR41 (n8n Credentials system + `N8N_ENCRYPTION_KEY`)] — credential encryption at rest requirement
- [Source: `_bmad-output/planning-artifacts/prd.md` § NFR9 (credential encryption)] — no credential value in plain text anywhere
- [Source: `_bmad-output/planning-artifacts/prd.md` § NFR39 (firm-specific values externalized)] — no hardcoded firm data in workflows
- [Source: `_bmad-output/planning-artifacts/prd.md` § NFR29 (operator-readability)] — 2-hour visual understanding target; credential names and env var names must be consistent with Architecture conventions to satisfy it
- [Source: `_bmad-output/planning-artifacts/prd.md` § NFR41 (explicit dependency list)] — env var list documented in `docs/environment-variables.md` is part of the dependency contract
- [Source: `_bmad-output/implementation-artifacts/1-1-infrastructure-verification-n8n-platform-pinning.md`] — previous story; secret-handling discipline, `_scratch_*` workflow convention, Redis TTL race lesson, clean credentials-list handoff
- [Source: `~/dev/borgstack/templates/stack-mini.yml.j2` lines 200–247, 286–332, 371–417] — the three n8n service environment blocks where Task 9 appends chatbot env vars
- [Source: `~/dev/borgstack/templates/stack-mini.yml.j2` lines 245 / 330 / 415] — `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` confirmation
- [Source: `~/dev/borgstack/inventory/mini/group_vars/all/main.yml`] — domain config, timezone, databases list; optional location for non-secret chatbot env values if the operator prefers Ansible variable extraction over Jinja inline
- [Source: `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml`] — Ansible Vault source for `n8n_db_password`, `redis_password` values consumed by Tasks 5 and 6
- [Source: `~/dev/borgstack/docs/mini-architecture.md`] — Evolution API cluster-only confirmation, justifying `evolution_api` deferral
- [Source: `~/dev/comadre/compose.yaml`] — mandatory `API_KEY`; Bearer auth contract
- [Source: `~/dev/comadre/deploy/caddy/Caddyfile`] — Comadre's own Caddy reverse proxy; HTTPS on ports 80/443
- [Source: `~/dev/comadre/README.md`] — OpenAI-compatible `POST /v1/audio/speech`, default voice `kokoro/pm_santa`, OGG Opus output
- [Source: project `CLAUDE.md` § Language and § Permissions] — English-only for technical artifacts, read-only `~/dev/`, write access restricted to this repo
- [Source: project user-memory `feedback_tts_always_comadre.md`] — Comadre is the only TTS engine
- [Source: project user-memory `feedback_model_selection_strategy.md`] — Groq free-tier first for `MODEL_PRIMARY`; no paid frontier models for operational calls
- [Source: project user-memory `feedback_javascript_only_in_code_nodes.md`] — never propose Python for Code nodes
- [Source: project user-memory `feedback_no_n8n_workflow_code.md`] — n8n is no-code; no workflow JSON output in story deliverables
- [Source: project user-memory `project_repo_visibility.md`] — repo is private forever; infrastructure IPs and domain names may appear in commits (but never secret values)

## Dev Agent Record

### Agent Model Used

Claude Opus 4.6 (1M context) — `claude-opus-4-6[1m]`

### Debug Log References

- None yet. Operator tasks (1–12) execute live in BotFather, n8n editor, BorgStack IaC, and Telegram; their evidence will land here (or in Completion Notes) once executed.

### Completion Notes List

**2026-04-13 — Agent-authored documentation complete; operator tasks pending.**

Story 1.2 splits cleanly into two execution surfaces:

1. **Agent-authored (done in this session):**
   - `docs/credentials-checklist.md` written. Lists all 8 locked credential names in committed order, each with n8n credential type, one-sentence purpose, and step-by-step creation instructions referencing where values come from (BotFather / Groq console / Comadre `.env` / Docker Swarm secret), never the values themselves. `evolution_api` row explicitly marked reserved for Epic 8 Story 8.2 with availability-gate justification (cluster-only service not shipped by `borgstack-mini`). `comadre_tts` row carries a neutral "Known follow-up" note on the optional rotation window per the Story 1.1 review follow-up. English; zero secret values (AC6).
   - `docs/environment-variables.md` written. Lists all 11 locked env vars in committed order, each with type, Maruzza default, per-firm tuning note, and refinement-vs-production notes where applicable. `ENVIRONMENT` row flags the Epic 10 Launch Gate flip and cross-references the matching `telegram_bot_refinement` → `telegram_bot_production` credential split. `HANDOFF_GROUP_CHAT_ID` / `OPS_GROUP_CHAT_ID` rows document negative-integer Telegram-group convention (including `-100` supergroup prefix) and the AR38 distinctness constraint. Operational notes at the bottom cover the "edit all three n8n services" rule for adding new env vars. English; zero secret values (AC7).
   - Neither file has been committed to Git. The commit step is flagged for operator approval before it runs (system policy: commits require explicit user request).

2. **Operator-executed (Galvani, pending — Tasks 1–12):**
   - Task 1: create refinement Telegram bot via `@BotFather`; reserve a distinct production bot identity; both tokens in password manager (AC1).
   - Task 2: register `telegram_bot_refinement` and `telegram_bot_production` in n8n with exact names; `getMe` probes green.
   - Task 3: register `groq_free` and `groq_paid` as OpenAI credentials with base URL override `https://api.groq.com/openai/v1`; `GET /v1/models` probes green (AC2).
   - Task 4: register `comadre_tts` as Header Auth with `Authorization: Bearer <key>`; `POST /v1/audio/speech` probe returns OGG Opus; optional key rotation per the Story 1.1 follow-up (AC2).
   - Task 5: register `postgres_main` (host `postgresql`, port 5432, db `n8n_db`, user `n8n_user`) with password from `vault_n8n_db_password`; Test step green (AC2).
   - Task 6: register `redis_main` (host `redis`, port 6379, db 0) with password from `vault_redis_password`; `SET/GET` probe in a single run per the Story 1.1 TTL-race lesson (AC2).
   - Task 7: confirm the Credentials list contains exactly 7 entries with no `evolution_api` stub (AC2).
   - Task 8: create two distinct private Telegram groups (`Maruzza — Handoff Leads`, `Maruzza — Ops Alerts`), add the refinement bot with post permission, capture negative-integer `chat_id`s using any of methods A/B/C in the task; confirm the two IDs differ (AC4, AC5).
   - Task 9: append the 11 chatbot env vars to all three n8n service `environment:` blocks in `~/dev/borgstack/templates/stack-mini.yml.j2` (editor / webhook / worker); redeploy via Ansible; verify via `docker exec … env | grep` and `docker service ls` (AC3).
   - Task 10: build `_scratch_verify-bootstrap` workflow on empty canvas with Manual Trigger → Set node echoing all 11 `$env.*` values → probes for `postgres_main`, `redis_main`, `groq_free`, `comadre_tts` → Telegram messages to both groups to confirm the chat IDs route correctly (AC3, AC8).
   - Task 11: `docker service update --force` on all three n8n services; re-run `_scratch_verify-bootstrap`; confirm env values persist across restart (AC3).
   - Task 12: delete `_scratch_verify-bootstrap`; confirm Workflows list is empty and Credentials list has exactly 7 entries (AC8).
   - When Tasks 1–12 are executed, record evidence (timestamps, probe outcomes, chat-id capture method, restart verification result) either here in Completion Notes or in a separate debug log reference, and check off the corresponding subtasks. Only then does the story transition to `review`.

**Secret-handling posture for this session:** no secret values were read, echoed, or written by the agent. Doc files contain names and procedures only; the `~/dev/comadre/.env`, `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml`, and any BotFather / Groq / Swarm-secret values are handled exclusively by Galvani during operator execution.

### File List

- `docs/credentials-checklist.md` (new; agent-authored in this session; not yet committed)
- `docs/environment-variables.md` (new; agent-authored in this session; not yet committed)
- `_bmad-output/implementation-artifacts/sprint-status.yaml` (updated: `1-2-…` status `ready-for-dev` → `in-progress`; `last_updated` annotation)
- `_bmad-output/implementation-artifacts/1-2-credential-environment-variable-and-telegram-group-bootstrap.md` (this story file — Status, Dev Agent Record, File List, Change Log updated per workflow-permitted sections)

Expected additions when Galvani executes operator tasks (tracked outside the n8n-chatbot repo, recorded here for traceability):

- `~/dev/borgstack/templates/stack-mini.yml.j2` — +11 env var lines appended to each of the three n8n service `environment:` blocks (Task 9). Operator commit inside `~/dev/borgstack`, not in this repo.
- (Optional) `~/dev/borgstack/inventory/mini/group_vars/all/main.yml` — if the operator extracts non-secret chatbot env values into an Ansible dict (Task 9 alternative path).

## Change Log

| Date | Change | Author |
|---|---|---|
| 2026-04-13 | Story 1.2 drafted from Epic 1 AC block; Story 1.1 learnings incorporated; BorgStack IaC ground truth cross-referenced for env var injection mechanism. Status → `ready-for-dev`. | Claude Opus 4.6 |
| 2026-04-13 | Agent-authored `docs/credentials-checklist.md` (AC6) and `docs/environment-variables.md` (AC7) from the story's committed names and architecture anchors; Task 13 / 14 authoring subtasks checked off; commit subtasks deferred pending operator approval; sprint status → `in-progress`; Tasks 1–12 pending operator execution. | Claude Opus 4.6 (1M context) |
