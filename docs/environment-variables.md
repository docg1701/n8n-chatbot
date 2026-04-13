# Environment variables — n8n chatbot template

**Operator:** Galvani
**Scope:** Story 1.2 — the eleven locked environment variables injected into the n8n containers via BorgStack's `stack-mini.yml.j2` template and read from workflow nodes via `{{ $env.VAR_NAME }}` expressions.

## Why this file exists

Every firm-specific value and every operational tunable the chatbot template exposes is externalized as either a credential (see [`credentials-checklist.md`](credentials-checklist.md)) or an environment variable. A new firm importing the template edits only these eleven values and the credential secrets — the workflow topology and prompts stay unchanged (NFR39, Architecture §Environment Variable Naming).

Env vars live at the **container** level, not at the n8n editor-session level. They are set in BorgStack's `~/dev/borgstack/templates/stack-mini.yml.j2` environment blocks for **all three** n8n services (`n8n-editor`, `n8n-webhook`, `n8n-worker`) so the queue-mode worker sees the same values at execution time that the editor sees at design time. The n8n runtime exposes them inside expressions via `{{ $env.VAR_NAME }}`, which requires `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` — already set by BorgStack and confirmed in Story 1.1 (`stack-mini.yml.j2` lines ≈ 245 / 330 / 415).

**Zero secret values appear in this file.** The two Telegram group chat IDs are operational routing values, not secrets — a random Telegram user cannot post to either group without first being added as a member, and the bot tokens that authorize posting live in the encrypted credential store (see [`credentials-checklist.md`](credentials-checklist.md)).

## Summary table

All eleven variables use `SCREAMING_SNAKE_CASE` and appear here in the committed order (Architecture §Environment Variable Naming):

| # | Name | Type | Default (Maruzza) | Per-firm? |
|---|---|---|---|---|
| 1 | `ENVIRONMENT` | string enum (`refinement` \| `production`) | `refinement` | No — lifecycle-controlled |
| 2 | `RATE_LIMIT_BURST` | positive integer | `30` | Yes — tune to traffic profile |
| 3 | `RATE_LIMIT_BURST_WINDOW_SEC` | positive integer (seconds) | `300` | Yes — tune to traffic profile |
| 4 | `RATE_LIMIT_DAILY` | positive integer | `200` | Yes — tune to traffic profile |
| 5 | `MODEL_PRIMARY` | Groq model identifier | `llama-3.3-70b-versatile` | No — stack-wide |
| 6 | `MODEL_WHISPER` | Groq Whisper model identifier | `whisper-large-v3-turbo` | No — stack-wide |
| 7 | `TTS_VOICE` | Comadre voice preset | `kokoro/pm_santa` | Yes — voice casting |
| 8 | `FIRM_NAME` | string (user-facing) | `Maruzza Teixeira Advocacia` | **Yes** — firm identity |
| 9 | `FIRM_CITY` | string (user-facing, format `City/State`) | `São Luís/MA` | **Yes** — firm identity |
| 10 | `HANDOFF_GROUP_CHAT_ID` | negative integer (Telegram group id) | captured in Story 1.2 Task 8 | **Yes** — per-deployment |
| 11 | `OPS_GROUP_CHAT_ID` | negative integer, **distinct** from `HANDOFF_GROUP_CHAT_ID` | captured in Story 1.2 Task 8 | **Yes** — per-deployment |

## Per-variable detail

### 1. `ENVIRONMENT`

- **Type:** string enum — exactly `refinement` or `production`.
- **Default (this instance):** `refinement`.
- **Purpose:** signals which lifecycle phase the live stack is in. Downstream workflows (Epic 9 audit bundle exporter, Epic 10 compliance checks, weekly report tagging) branch on this value.
- **Refinement vs production:** `refinement` holds during Epic 1–9 while the chatbot is being built and refined in isolation with synthetic and internal-human test data. Epic 10 Story 10.5 (Launch Gate) flips the value to `production` and simultaneously clones `main-chatbot-refinement` → `main-chatbot-production`, swapping the credential reference from `telegram_bot_refinement` to `telegram_bot_production`. The credential split in [`credentials-checklist.md`](credentials-checklist.md) mirrors this env var flip exactly.
- **Per-firm tuning:** none. Every deployment uses the same two-value alphabet.

### 2. `RATE_LIMIT_BURST`

- **Type:** positive integer (messages per burst window, per-user).
- **Default:** `30`.
- **Purpose:** upper bound on messages a single user can send inside the rolling burst window before the Epic 5 guardrails sub-workflow rejects further input. Enforced by Redis counters against the `redis_main` credential.
- **Per-firm tuning:** raise if the firm expects legitimate multi-step media-heavy conversations; lower if abuse patterns emerge during Epic 9 red-team cycles.

### 3. `RATE_LIMIT_BURST_WINDOW_SEC`

- **Type:** positive integer, seconds.
- **Default:** `300` (5 minutes).
- **Purpose:** sliding window size for the burst counter. A user who sends `RATE_LIMIT_BURST` messages inside `RATE_LIMIT_BURST_WINDOW_SEC` seconds is throttled until the window slides.
- **Per-firm tuning:** tune alongside `RATE_LIMIT_BURST`; the ratio matters more than either value in isolation.

### 4. `RATE_LIMIT_DAILY`

- **Type:** positive integer (messages per rolling 24h window, per-user).
- **Default:** `200`.
- **Purpose:** per-user daily ceiling independent of the burst counter. Enforced by a second rolling Redis counter.
- **Per-firm tuning:** firms serving a concentrated daytime client window may lower this; firms with 24/7 self-service journeys may raise it.

### 5. `MODEL_PRIMARY`

- **Type:** Groq model identifier (must match a model id returned by `GET https://api.groq.com/openai/v1/models`).
- **Default:** `llama-3.3-70b-versatile` — whichever Groq 7–12B multilingual model is currently recommended for Brazilian Portuguese conversations is acceptable; record the actually-deployed id in this env var.
- **Purpose:** drives every operational LLM call — triage, structured extraction, lead qualification, guardrails rewrite (Epic 2–6).
- **Per-firm tuning:** none. The primary is a stack-wide choice tied to the Groq free tier (project policy: Groq free tier first, then flat-rate monthly API; no paid frontier models for operational calls). Do **not** substitute Haiku, GPT-4o-mini, or any other paid frontier model here.

### 6. `MODEL_WHISPER`

- **Type:** Groq Whisper model identifier.
- **Default:** `whisper-large-v3-turbo`.
- **Purpose:** audio transcription in the Epic 7 voice modality (incoming Telegram voice notes, later WhatsApp voice notes via Evolution API).
- **Per-firm tuning:** none — stack-wide.

### 7. `TTS_VOICE`

- **Type:** Comadre voice preset id (see `~/dev/comadre/README.md` for the current catalog).
- **Default:** `kokoro/pm_santa` — neural male, Brazilian Portuguese, matches Maruzza's brand voice.
- **Purpose:** TTS voice used by the Epic 7 voice-out path when the Comadre call is made. Interpolated into the `POST /v1/audio/speech` request body against the `comadre_tts` credential.
- **Per-firm tuning:** voice casting is a per-firm brand decision. Any voice in Comadre's catalog is acceptable; smoke-test the new value against a sample script before committing it.

### 8. `FIRM_NAME`

- **Type:** string, arbitrary, user-facing.
- **Default:** `Maruzza Teixeira Advocacia`.
- **Purpose:** interpolated into every greeting, disclosure, handoff card, and LGPD notice. Also appears in Epic 9 weekly report subjects.
- **Per-firm tuning:** **required.** Every firm must set its own value. This is the most visible per-firm knob.
- **Language:** the firm's native language and orthography — Portuguese for Brazilian firms. This is user-facing content, not technical documentation, so the English-only language policy (Architecture §Language Policy) does not apply here.

### 9. `FIRM_CITY`

- **Type:** string, arbitrary, user-facing — conventional format `City/State` (e.g., `São Luís/MA`).
- **Default:** `São Luís/MA`.
- **Purpose:** used by the triage agent for area-specific tone cues, jurisdiction hints, and occasional filler ("lawyers based in…"). Not used for any routing or legal logic.
- **Per-firm tuning:** **required.** Every firm sets its own locality.

### 10. `HANDOFF_GROUP_CHAT_ID`

- **Type:** **negative integer** — Telegram's convention for group chats. Supergroups carry a `-100` prefix (e.g., `-1001234567890`); plain groups are small negatives (e.g., `-987654321`). Both forms are acceptable; workflows interpolate the value as-is without arithmetic.
- **Default:** captured live during Story 1.2 Task 8 via one of: (A) temporary `/setprivacy Disable` on the refinement bot + a browser hit to `https://api.telegram.org/bot<REFINEMENT_TOKEN>/getUpdates`, (B) a third-party helper bot such as `@RawDataBot`, or (C) a temporary n8n workflow with a Telegram Trigger.
- **Purpose:** the private Telegram group that receives lead notifications for attorney pickup. Epic 4's handoff flow posts here; Accept / Decline button callbacks return from attorneys through this same chat. For this instance the group is suggested as `Maruzza — Handoff Leads`.
- **Per-firm tuning:** **required.** Every firm creates its own private group (with the refinement bot added, post permission granted), captures the chat id, and sets this env var to that value.
- **Distinctness constraint:** **must be different** from `OPS_GROUP_CHAT_ID` (AR38). Mixing leads into ops noise — or ops alerts into the handoff channel — causes dropped leads and/or missed incidents.

### 11. `OPS_GROUP_CHAT_ID`

- **Type:** **negative integer** (same Telegram-group convention as `HANDOFF_GROUP_CHAT_ID`).
- **Default:** captured live during Story 1.2 Task 8 — a **second, distinct** private Telegram group.
- **Purpose:** ops/alerting Telegram group for system-level alerts (Epic 6 error handler's threshold routing) and Epic 9 weekly reports. For this instance the group is suggested as `Maruzza — Ops Alerts`.
- **Per-firm tuning:** **required.** Every firm creates a second distinct private group for operational noise and captures its id.
- **Distinctness constraint:** **must not equal** `HANDOFF_GROUP_CHAT_ID`.

## Operational notes

- **Adding a new env var to the template.** Edit the `environment:` block of **all three** n8n services in `~/dev/borgstack/templates/stack-mini.yml.j2` (`n8n-editor`, `n8n-webhook`, `n8n-worker`), redeploy with the operator's usual Ansible command (e.g., `ansible-playbook -i inventory/mini playbooks/deploy-mini.yml`), then update this file. Omitting any of the three services will cause design-time vs runtime drift (editor sees the value, worker does not, workflows read empty strings at execution time).
- **Editing a value on a live instance.** Change the value in `stack-mini.yml.j2` (and/or `inventory/mini/group_vars/all/main.yml` if the operator extracted non-secret env values to Ansible variables), then redeploy. `docker service update --force` on the three n8n services picks up new values without a full stack redeploy, but the Ansible playbook is the canonical path.
- **Reading an env value inside a workflow.** `{{ $env.VAR_NAME }}` expression in any node's expression-enabled field. AR44 fixes this as the sole access pattern. The three n8n services all export the same env set, so the value is identical at design time (editor) and execution time (worker).
- **Redeploy gate.** After any env var change, re-execute the Story 1.2 `_scratch_verify-bootstrap` pattern (or its successor) to confirm every expected value is present and the n8n services are healthy (`docker service ls` shows 1/1 replicas for each).
- **Refinement vs production credential split.** `ENVIRONMENT=refinement` is paired with `telegram_bot_refinement`; `ENVIRONMENT=production` is paired with `telegram_bot_production`. Epic 10 Story 10.5 owns the coordinated flip. This is the only env-var-to-credential coupling worth calling out explicitly; see [`credentials-checklist.md`](credentials-checklist.md) for the full credential inventory.

## References

- [Architecture §Environment Variable Naming](../_bmad-output/planning-artifacts/architecture.md) — the eleven committed names, `SCREAMING_SNAKE_CASE` format, per-firm externalization principle.
- [Architecture §LLM call pattern](../_bmad-output/planning-artifacts/architecture.md) — Groq OpenAI-compatible endpoint drives `MODEL_PRIMARY` and `MODEL_WHISPER` identity.
- [Architecture §Language Policy](../_bmad-output/planning-artifacts/architecture.md) — English for technical artifacts; user-facing strings (`FIRM_NAME`, `FIRM_CITY`) follow the firm's language.
- [Story 1.2 — this story](../_bmad-output/implementation-artifacts/1-2-credential-environment-variable-and-telegram-group-bootstrap.md) — ACs, operator tasks, the exact `stack-mini.yml.j2` edit locations (lines ≈ 210, 296, 381).
- [`credentials-checklist.md`](credentials-checklist.md) — the eight locked credential names; the `telegram_bot_refinement` / `telegram_bot_production` split mirrors `ENVIRONMENT` exactly.
- [Story 1.1 infrastructure verification](infra-verification.md) — baseline where `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` was first confirmed in `stack-mini.yml.j2`.
