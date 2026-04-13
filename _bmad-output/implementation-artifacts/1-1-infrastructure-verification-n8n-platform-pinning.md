# Story 1.1: Infrastructure verification & n8n platform pinning

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator (Galvani),
I want to verify borgstack-mini is healthy and that n8n is running a 2.x release, and to produce a timestamped verification document,
so that I have a documented, reproducible baseline before any workflow construction begins and every later epic can trust that the platform foundation is sound.

Evolution API is **explicitly out of scope** for this story — it is not part of borgstack-mini (confirmed in `~/dev/borgstack/templates/stack-mini.yml.j2` and `~/dev/borgstack/docs/mini-architecture.md`). Its verification belongs to Epic 8 Story 8.2 as part of the mini → cluster swap prefix. Do not create the `evolution_api` credential in this story; do not call any Evolution API health endpoint.

## Acceptance Criteria

**AC1 — n8n editor reachable over HTTPS**

- **Given** borgstack-mini is deployed at `10.10.10.205` and the n8n editor is exposed via Cloudflare Tunnel + Caddy at the `mini_n8n_domain` configured in `~/dev/borgstack/inventory/mini/group_vars/all/main.yml`
- **When** the operator opens the n8n editor URL in a browser
- **Then** the editor loads successfully over HTTPS (no TLS warnings, no 5xx)

**AC2 — n8n version is 2.x**

- **Given** the n8n editor is open
- **When** the operator navigates to Settings and reads the displayed n8n version
- **Then** the version is **any 2.x release**. Do **not** demand a specific minor. BorgStack pins the n8n image in `~/dev/borgstack/inventory/shared/images.yml` (currently `n8nio/n8n:2.11.3`, `pixeloddity/n8n-worker:2.11.3`) and the operator upgrades on an independent cadence; any 2.x is acceptable and the CVE-2025-68613 fix floor (1.120.4+) is met by every 2.x release (AR4)
- **And** the exact running version string is captured for the verification document

**AC3 — Postgres reachable and `pgvector` already provisioned in `n8n_db`**

- **Given** n8n is running in queue mode (editor + webhook + worker) on borgstack-mini
- **When** the operator creates a temporary verification workflow with a Postgres node bound to a **disposable ad-hoc credential** pointing at borgstack-mini's Postgres (host `postgresql`, port `5432`, database `n8n_db`, user `n8n_user`, password from the `n8n_db_password` Docker Swarm secret — do **not** create or use the locked credential name `postgres_main`, which Story 1.2 owns)
- **Then** a manual execution of `SELECT 1 AS ping;` returns a row (connection is alive)
- **And** a second manual execution of `SELECT extname FROM pg_extension WHERE extname = 'vector';` against `n8n_db` returns exactly one row, confirming that the `pgvector` extension provisioned by borgstack-mini's init script (`~/dev/borgstack/config/postgresql/init-databases-mini.sh`, `CREATE EXTENSION IF NOT EXISTS vector` on line ~87) is present and usable — this is verification, not provisioning

**AC4 — Redis reachable (SET/GET roundtrip)**

- **Given** Redis is part of borgstack-mini
- **When** the operator creates a temporary verification workflow with a Redis node bound to a **disposable ad-hoc credential** (host `redis`, port `6379`, db `0`, password from the `redis_password` Docker Swarm secret — do **not** create or use the locked name `redis_main`, which Story 1.2 owns) that performs `SET verify:infra "ok" EX 60` followed by `GET verify:infra`
- **Then** both operations succeed and the returned value equals `"ok"`
- **And** the test key uses a 60 s TTL so no residual state remains, and the key is outside the `n8n:` prefix so it does not touch n8n's queue namespace

**AC5 — `n8n_encryption_key` Docker Swarm secret present, Ansible Vault source encrypted, vault password backed up out-of-band**

- **Given** borgstack-mini manages the n8n encryption key as a Docker Swarm external secret named `n8n_encryption_key` (declared in `~/dev/borgstack/templates/stack-mini.yml.j2`, mounted inside the n8n editor, webhook, and worker containers at `/run/secrets/n8n_encryption_key` via the `N8N_ENCRYPTION_KEY_FILE` env var)
- **When** the operator runs `docker secret ls` on `10.10.10.205`
- **Then** the output lists a secret named `n8n_encryption_key` (present and created)
- **And** the operator confirms that the source-of-truth value for `n8n_encryption_key` lives in `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml`, and that file is **Ansible Vault-encrypted** at rest (the first line begins with `$ANSIBLE_VAULT;1.1;AES256` or equivalent — never plaintext)
- **And** the Ansible Vault password itself is stored in the firm's out-of-band encrypted backup location (e.g., password manager) — if it is not yet backed up, the operator pauses Story 1.1 and completes the backup before continuing, because this is load-bearing for FR41 and for disaster-recovery of every credential Story 1.2 onwards will register
- **And** the operator never reads, echoes, or copies the secret value itself into any workflow field, Code node, log output, exported JSON, markdown file, or chat window
- **And** `docs/infra-verification.md` records only a **reference-only pointer** to where the Ansible Vault password is stored (e.g., `"stored in Galvani's password manager under entry borgstack-mini/ansible-vault"`), never the vault password and never the encryption key value

**AC6 — Comadre TTS reachable over HTTPS with API key and returns valid pt-BR audio**

- **Given** Comadre is deployed at `10.10.10.207` (source at `~/dev/comadre`), runs its own Caddy reverse proxy on ports 80/443, and requires an API key (enforced at service start by `~/dev/comadre/compose.yaml`: `API_KEY: ${API_KEY:?API_KEY must be set}`)
- **When** the operator creates a temporary verification workflow with a single HTTP Request node configured as:
  - Method: `POST`
  - URL: `https://<comadre-domain>/v1/audio/speech` (or `https://10.10.10.207/v1/audio/speech` with "Ignore SSL Issues" enabled if Comadre is running with its default self-signed cert per `~/dev/comadre/deploy/caddy/Caddyfile`)
  - Authentication: a temporary ad-hoc Bearer credential or an inline header `Authorization: Bearer <COMADRE_API_KEY>` (do **not** create the locked `comadre_tts` credential — Story 1.2 owns it)
  - Body content type: `JSON`
  - Body: `{"input": "Olá, teste de verificação de infraestrutura.", "voice": "kokoro/pm_santa"}`
  - Response format: `File` (binary)
  - and executes it manually
- **Then** the response returns HTTP **200** with an audio binary in the body
- **And** the operator saves the binary as `.ogg` (Comadre's default output format is OGG Opus, 24 kbps, 16 kHz, mono — not MP3) and plays it back locally
- **And** the playback is valid Brazilian Portuguese speech in the `kokoro/pm_santa` (neural male) voice; duration is consistent with the input text
- **And** if Comadre returns a non-200 or unintelligible audio, the operator stops Story 1.1 and debugs Comadre (consulting `~/dev/comadre/` read-only) before continuing — this path is load-bearing for Epic 7 (voice modality)

**AC7 — Verification document committed**

- **Given** AC1–AC6 are all green
- **When** the operator authors `docs/infra-verification.md` and commits it to the repo
- **Then** the document records:
  - Timestamp of the verification run (ISO 8601 with `-03:00` offset, per Architecture §Timestamp Handling)
  - The exact running n8n version string observed in Settings (confirmed to be a 2.x release)
  - Postgres connectivity confirmation against `n8n_db` via `SELECT 1`
  - `pgvector` confirmation via `SELECT extname FROM pg_extension WHERE extname='vector'` returning exactly one row
  - Redis `SET`/`GET` roundtrip confirmation on the `verify:infra` key
  - `n8n_encryption_key` Docker secret confirmation (`docker secret ls` output mentioning the secret by name)
  - Confirmation that `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml` is Ansible Vault-encrypted at rest
  - Reference-only pointer to where the Ansible Vault password is stored out-of-band (never the value)
  - Comadre HTTP 200 confirmation plus subjective playback confirmation (pt-BR, `kokoro/pm_santa`, OGG Opus)
- **And** the document is in **English** (per project `CLAUDE.md` language rule and Architecture §Language Policy)
- **And** the document contains **zero** secret values (no encryption keys, no vault passwords, no API keys, no Bearer tokens)
- **And** the document explicitly states that Evolution API was **not** verified in this story and explains why (not part of borgstack-mini; Epic 8 prefix)
- **And** FR41 (credential encryption at rest) is architecturally satisfied — the encryption key exists in Swarm, its source-of-truth is encrypted in Ansible Vault, and the vault password is recoverable from out-of-band backup

**AC8 — Temporary verification workflows and credentials discarded**

- **Given** all verification workflows were intentionally temporary
- **When** verification is complete and `docs/infra-verification.md` is committed
- **Then** every temporary verification workflow used for AC3, AC4, AC6 is **deleted** from the n8n editor
- **And** every disposable ad-hoc credential created for verification (Postgres, Redis, Comadre Bearer) is **deleted** from the n8n Credentials list, so Story 1.2 begins with an empty credentials list and registers the 8 locked credential names cleanly (minus `evolution_api`, which Story 8.2 will register)
- **And** no unnamed / undocumented workflow remains on the canvas when Story 1.2 begins

## Tasks / Subtasks

- [x] **Task 1 — Open n8n editor and confirm reachability + 2.x version (AC: 1, 2)**
  - [x] Open the n8n editor URL in a browser (the `mini_n8n_domain` configured in `~/dev/borgstack/inventory/mini/group_vars/all/main.yml` — currently `n8n.pixeloddity.org`)
  - [x] Confirm HTTPS loads cleanly (no warnings, no 5xx) → satisfies AC1
  - [x] Navigate Settings → read the n8n version string
  - [x] Confirm it is a 2.x release (any 2.x is fine; do not force an upgrade) → satisfies AC2
  - [x] Capture the exact version string for AC7's verification document

- [x] **Task 2 — Verify Postgres + `pgvector` via temporary workflow (AC: 3)**
  - [x] On an empty canvas, create a temporary workflow named `_scratch_verify-postgres` (to be deleted in Task 7)
  - [x] Add a Manual Trigger, then a Postgres node
  - [x] Create a **disposable ad-hoc** Postgres credential in n8n with: host `postgresql`, port `5432`, database `n8n_db`, user `n8n_user`, password from the `n8n_db_password` Docker Swarm secret (the operator may need to read the secret from the running BorgStack host once to populate the credential — handle the secret carefully)
  - [x] Do **not** name the credential `postgres_main` — that name is locked for Story 1.2
  - [x] Set the Postgres node operation to Execute Query: `SELECT 1 AS ping;` — run, confirm the row
  - [x] Change the query to: `SELECT extname FROM pg_extension WHERE extname = 'vector';` — run, confirm exactly one row is returned
  - [x] Capture both confirmations for AC7

- [x] **Task 3 — Verify Redis via temporary workflow (AC: 4)**
  - [x] On an empty canvas, create a temporary workflow named `_scratch_verify-redis`
  - [x] Add a Manual Trigger, then a Redis node
  - [x] Create a **disposable ad-hoc** Redis credential with: host `redis`, port `6379`, db `0`, password from the `redis_password` Docker Swarm secret
  - [x] Do **not** name the credential `redis_main` — that name is locked for Story 1.2
  - [x] Operation 1: `SET verify:infra "ok" EX 60` — confirm success
  - [x] Operation 2: `GET verify:infra` — confirm the returned value equals `"ok"`
  - [x] Capture both confirmations for AC7

- [x] **Task 4 — Verify `n8n_encryption_key` Docker secret + Ansible Vault source + out-of-band backup (AC: 5)**
  - [x] SSH into the borgstack-mini host at `10.10.10.205` (or whatever approach is already set up for Galvani to operate the box)
  - [x] Run `docker secret ls` and confirm the output contains a row named `n8n_encryption_key`
  - [x] Open `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml` locally and confirm its first line begins with `$ANSIBLE_VAULT;1.1;AES256` or similar — **the file must be encrypted at rest, never plaintext**
  - [x] Confirm the Ansible Vault password (the password that decrypts `vault.yml`) is stored in the firm's out-of-band encrypted backup location (password manager, encrypted vault, etc.)
  - [x] If the Ansible Vault password is **not** yet backed up, **pause this story** and complete the backup first — this is load-bearing for FR41 and for recovering every credential Story 1.2+ registers
  - [x] **Do not** read, echo, or copy the encryption key value or the vault password into any file, workflow field, chat window, or log. Handle both as strict out-of-band secrets
  - [x] Capture a reference-only pointer (e.g., `"Galvani 1Password → entry borgstack-mini/ansible-vault"`) for AC7 — the pointer, not the value

- [x] **Task 5 — Verify Comadre TTS reachability, auth, and audio output via temporary workflow (AC: 6)**
  - [x] On an empty canvas, create a temporary workflow named `_scratch_verify-comadre`
  - [x] Add a Manual Trigger, then an HTTP Request node configured:
    - Method: `POST`
    - URL: `https://<comadre-domain>/v1/audio/speech` (consult `~/dev/comadre/.env` or the operator's notes for the actual `DOMAIN` value — if it is `localhost` / default self-signed, use `https://10.10.10.207/v1/audio/speech` and enable "Ignore SSL Issues" on the HTTP Request node)
    - Authentication: add a header `Authorization: Bearer <COMADRE_API_KEY>` — paste the key inline for this one-time verification, or use a temporary Generic Credential Type. Do **not** create or use the locked name `comadre_tts` — that name is locked for Story 1.2
    - Body Content Type: `JSON`
    - Body: `{"input": "Olá, teste de verificação de infraestrutura.", "voice": "kokoro/pm_santa"}`
    - Response format: `File` (binary)
  - [x] Execute the node. Confirm HTTP 200 and that an audio binary is returned
  - [x] Download the binary from the execution output and save as `test.ogg` (default output is OGG Opus per `~/dev/comadre/README.md`)
  - [x] Play it back locally and confirm: valid Brazilian Portuguese speech, `kokoro/pm_santa` (neural male) voice, duration consistent with the input text
  - [x] If Comadre returns non-200 or unintelligible audio, stop and debug against `~/dev/comadre/` before continuing
  - [x] Capture HTTP status + subjective playback confirmation for AC7

- [x] **Task 6 — Author `docs/infra-verification.md` and commit (AC: 7)**
  - [x] Create `docs/infra-verification.md` with the fields listed in AC7 (timestamp, n8n version, Postgres + `pgvector` against `n8n_db`, Redis roundtrip, `n8n_encryption_key` Docker secret + Ansible Vault encrypted confirmation + reference-only pointer to vault password backup, Comadre HTTP 200 + playback confirmation)
  - [x] Language: **English** (per `CLAUDE.md` and Architecture §Language Policy)
  - [x] Include a short "How this was verified" section summarizing that the verification was performed via four temporary manual-trigger workflows on an empty canvas and that they (and their ad-hoc credentials) were deleted in Task 7
  - [x] Include an explicit paragraph noting that **Evolution API was not verified** in this story because it is not part of borgstack-mini; its verification is the responsibility of Epic 8 Story 8.2 as a prefix step, alongside the mini → cluster swap or a standalone Evolution API deployment
  - [x] **Explicitly confirm** in the document that no secret values are present in the file and that FR41 is satisfied
  - [x] `git add docs/infra-verification.md` and commit with an English, imperative, under-72-char subject (suggestion: `Verify borgstack-mini infrastructure for n8n-chatbot`)

- [x] **Task 7 — Discard temporary verification workflows and credentials (AC: 8)**
  - [x] In the n8n editor, delete the four temporary workflows created in Tasks 2, 3, 5 (`_scratch_verify-postgres`, `_scratch_verify-redis`, `_scratch_verify-comadre`)
  - [x] Delete the ad-hoc credentials used solely for verification (the temporary Postgres, Redis, and Comadre Bearer credentials) so Story 1.2 starts from an empty credentials list
  - [x] Visually confirm the n8n workflow list and credentials list are clean before moving on to Story 1.2

## Dev Notes

### Nature of this story

This is the **first** story of Epic 1 — there is no previous story intelligence to inherit. It is a **verification and documentation** story: the deliverable is a commit to `docs/infra-verification.md` plus a green signal for every infrastructure dependency the rest of the project assumes. **No production workflow construction happens in this story.** The empty-canvas workflow shells are Story 1.3's scope, the locked credential set is Story 1.2's scope, and the real Telegram echo bot is Story 1.4's scope.

### Ground truth sources consulted while writing this story

Every concrete value below (database name, secret name, host name, port, file path, domain) was extracted from the actual BorgStack and Comadre IaC, not from planning documents:

- `~/dev/borgstack/templates/stack-mini.yml.j2` — borgstack-mini service topology
- `~/dev/borgstack/config/postgresql/init-databases-mini.sh` — database and user provisioning, `pgvector` init
- `~/dev/borgstack/inventory/shared/images.yml` — Docker image pins (including the intentional n8n 2.11.3 pin)
- `~/dev/borgstack/inventory/mini/group_vars/all/main.yml` — domains, timezone
- `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml` — Ansible Vault-encrypted secrets
- `~/dev/borgstack/docs/mini-architecture.md` — confirmation that Evolution API is cluster-only
- `~/dev/comadre/compose.yaml` — API key is mandatory at service start
- `~/dev/comadre/deploy/caddy/Caddyfile` — Comadre's own HTTPS reverse proxy layout
- `~/dev/comadre/README.md` — the OpenAI-compatible `POST /v1/audio/speech` contract with `Authorization: Bearer <API_KEY>`

### Relevant architecture patterns and constraints

- **n8n is 2.x; do not force a specific minor.** BorgStack's pin in `~/dev/borgstack/inventory/shared/images.yml` is intentional and updated on Galvani's cadence. The CVE-2025-68613 fix floor (1.120.4+) is above every 2.x release, so any 2.x meets the security requirement. (Architecture §Platform version pinning; AR4)
- **borgstack-mini ships a single application database `n8n_db`.** All chatbot tables (`client_profiles`, `chat_analytics`, `static_responses`, `staging_*`, `n8n_chat_histories`) coexist in `n8n_db` alongside n8n's own operational tables. There is **no** separate `chatbot` database. The `pgvector` extension is already enabled in `n8n_db` at container init. (Architecture §Data Architecture)
- **Evolution API is NOT on borgstack-mini.** It ships only in the full BorgStack cluster (`stack-worker-*.yml.j2`). Epic 1 through Epic 7 run on mini with no Evolution API dependency. Epic 8 triggers the mini → cluster swap (or a standalone Evolution API deploy) as a prefix step. **This story must not attempt to verify, reach, or create a credential for Evolution API.** (Architecture §Starter Template & Foundation; `~/dev/borgstack/docs/mini-architecture.md`)
- **Comadre runs its own Caddy on `10.10.10.207`.** Access is HTTPS, authentication is mandatory via `Authorization: Bearer <COMADRE_API_KEY>`, default voice is `kokoro/pm_santa`, default output is OGG Opus. The old `http://10.10.10.207:8000` pattern that appeared in early planning documents was incorrect and has been corrected throughout the planning artifacts. (`~/dev/comadre/compose.yaml`, `~/dev/comadre/deploy/caddy/Caddyfile`, `~/dev/comadre/README.md`)
- **`n8n_encryption_key` is a Docker Swarm external secret.** It is mounted inside the n8n containers at `/run/secrets/n8n_encryption_key` via the `N8N_ENCRYPTION_KEY_FILE` env var pattern — not set as a plain env var. Its source-of-truth value lives in Ansible Vault (`inventory/mini/group_vars/all/vault.yml`), encrypted at rest. The Ansible Vault password is what must be backed up out-of-band. (Architecture §Platform version pinning, §Authentication & Security)
- **Empty-canvas discipline.** The four temporary verification workflows created here are built from an empty canvas (never from a community template) and are deleted in Task 7 (Architecture §Seed decision — empty canvas).
- **Visual-first, no Code nodes, no Python.** Every AC is satisfied by native nodes only (Manual Trigger, Postgres, Redis, HTTP Request). A Code node in this story would be an unjustified exception. Python is never used in Code nodes in this project. (Architecture §Code Node Acceptance Criteria; project CLAUDE.md memory rule)
- **Node naming rule.** Every non-trigger node gets a descriptive Title Case name (imperative verb + object). Trigger nodes keep their default name with a descriptive suffix (e.g., `Manual Trigger — Verify Postgres`). Apply this even in the temporary scratch workflows to build muscle memory. (Architecture §n8n Workflow Naming)
- **English for technical artifacts.** `docs/infra-verification.md` is a technical document and must be in English. The single Portuguese string in this story is the transient TTS verification input text in AC6, which is test data and not persisted anywhere. (`CLAUDE.md`; Architecture §Language Policy)
- **Temporary credentials here, locked credentials in Story 1.2.** The four temporary credentials (Postgres, Redis, Comadre Bearer — there is no temporary credential for AC1/AC2/AC5 because those do not need one) must not use the locked names `postgres_main`, `redis_main`, `comadre_tts`, `evolution_api`. Story 1.2 owns those names and writes `docs/credentials-checklist.md` to document them.

### Source tree components touched

This story writes **one file** in the Git repository:

- `docs/infra-verification.md` — new, committed at the end of Task 6

It does **not** write:

- Any file under `exports/workflow-snapshots/` — the first snapshot belongs to Story 1.3
- Any file under `sql/` — database schema files are Epic 2/3 scope; `pgvector` is only *verified* here, not consumed
- Any file under `prompts/` — the first prompt belongs to Epic 2
- Any SQL migration — the `n8n_db` database is assumed to exist, provisioned by BorgStack's `init-databases-mini.sh`
- Any `package.json`, `node_modules/`, `.github/workflows/`, `src/`, `tests/` — none of those exist or should exist in this repo (Architecture §Notable absences)

### Testing standards summary

- **No unit tests, no integration tests, no CI pipeline** — this project's testing story is the offline refinement pipeline (`golden-dataset/`, `synthetic-scenarios/`, `red-team-battery/`, `human-test-logs/`) introduced by Epic 9. None of that infrastructure exists yet.
- **"Test" for this story = live manual execution in the n8n editor** of the four temporary verification workflows, verified by human observation and captured in `docs/infra-verification.md`.
- **Green gate:** every AC is a binary pass/fail. Partial completion is not allowed. If AC2 is red (n8n not 2.x), stop and investigate. If AC5 is red (vault password not backed up out-of-band), stop and back it up. If AC6 is red (Comadre 4xx/5xx or bad audio), stop and debug Comadre.
- **Evidence format:** the commit to `docs/infra-verification.md` is the test artifact.

### Dev Agent Guardrails

- **Do not verify or touch Evolution API.** It is not on borgstack-mini. Any attempt to verify it is out of scope and based on an outdated premise.
- **Do not register the locked Story-1.2 credentials (`postgres_main`, `redis_main`, `comadre_tts`, `evolution_api`) here.** Use ad-hoc, disposable credentials only.
- **Do not copy the `n8n_encryption_key` value, the Ansible Vault password, or the `COMADRE_API_KEY` into any n8n workflow field, Code node, log output, markdown file, or chat window.** The document only records reference-only pointers.
- **Do not use Python in any Code node; do not introduce a Code node at all in this story.**
- **Do not force a specific n8n minor version.** If the version is 2.x, that is sufficient regardless of the minor number.
- **Do not propose Edge TTS, Piper, Kokoro Web, OpenAI TTS, or OpenEdAI.** Comadre is the only TTS engine in this project.
- **Do not propose ChatGPT, Claude, GPT-4o, GPT-4o-mini, or Claude Haiku for operational calls.** This story makes no LLM calls, but even as an aside, the operational posture is Groq free → Groq paid → flat-rate monthly provider.
- **Do not output n8n workflow JSON as part of this story.** This is a no-code, visual-first project — the operator builds workflows manually in the editor. The permitted artifact is a markdown file (`docs/infra-verification.md`) plus a Git commit.
- **Do not run destructive commands outside the repo.** Read-only access to `~/dev/` is permitted for consulting BorgStack and Comadre sources. Write access is strictly `/home/galvani/dev/n8n-chatbot/` (per `CLAUDE.md` permissions).

### Library, framework, and version requirements relevant to this story

| Component | Version / identity | Source |
|---|---|---|
| n8n | **any 2.x release** (current BorgStack pin: `2.11.3`, intentional) | `~/dev/borgstack/inventory/shared/images.yml`; AR4 |
| PostgreSQL + `pgvector` | image `pgvector/pgvector:pg18` (Postgres 18, `pgvector` enabled in `n8n_db` at init) | `~/dev/borgstack/inventory/shared/images.yml`, `~/dev/borgstack/config/postgresql/init-databases-mini.sh` |
| Redis | image `redis:8-alpine` | `~/dev/borgstack/inventory/shared/images.yml` |
| Comadre | own Caddy on `10.10.10.207`, OpenAI-compatible `/v1/audio/speech`, voice `kokoro/pm_santa`, OGG Opus, Bearer auth mandatory | `~/dev/comadre/compose.yaml`, `~/dev/comadre/deploy/caddy/Caddyfile`, `~/dev/comadre/README.md` |
| Evolution API | **not on mini** — cluster-only, verification deferred to Epic 8 | `~/dev/borgstack/docs/mini-architecture.md`, `~/dev/borgstack/templates/stack-mini.yml.j2` |
| borgstack-mini host | `10.10.10.205` | `~/dev/borgstack/inventory/mini/hosts` |
| Comadre host | `10.10.10.207` | `~/dev/comadre/deploy/ansible/inventory/production.yml.example` |
| borgstack-mini n8n domain | `mini_n8n_domain` variable in `~/dev/borgstack/inventory/mini/group_vars/all/main.yml` (currently `n8n.pixeloddity.org`) | same |

No new dependencies, no `package.json`, no npm install — this is a pure verification story.

### File structure requirements

- **Write:** `docs/infra-verification.md` (one file, English, markdown, zero secret values)
- **Commit:** English, imperative mood, under 72 chars in the subject
- **Do not write:** anything under `exports/`, `sql/`, `prompts/`, `audit-bundles/`, `refinement-log/`, `golden-dataset/`, `synthetic-scenarios/`, `red-team-battery/`, `human-test-logs/`
- **Do not create:** `package.json`, `node_modules/`, `src/`, `tests/`, `.github/workflows/` (deliberately absent per Architecture §Notable absences)

### Project Structure Notes

- **`docs/infra-verification.md`** is the canonical location. It sits alongside later Story 1.2 artifacts `docs/credentials-checklist.md` and `docs/environment-variables.md`, forming a `docs/` prefix convention for operator-facing documentation (Architecture §Surface 1 — Git Repository Structure).
- **No conflicts detected** with project structure. The `docs/` directory already exists with two unrelated files (`plan.md`, `claude-code-instruction-quality-guide.md`); the new file adds to that directory without collision.
- **No `exports/workflow-snapshots/` directory is created in this story.** The first export belongs to Story 1.3.
- **Alignment with unified project structure:** confirmed against Architecture §Surface 1. The file name follows the lowercase-hyphenated convention.
- **Detected variances:** none.

### References

All technical details cite source paths and sections:

- [Source: `_bmad-output/planning-artifacts/epics.md` § Story 1.1 — Infrastructure verification & n8n platform pinning] — canonical acceptance criteria foundation (post-correction)
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Platform version pinning] — n8n 2.x floor, Docker Swarm secret model for `n8n_encryption_key`, Ansible Vault source of truth
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Bootstrap checklist ("Step 1 — Infrastructure Setup")] — 14-step bootstrap sequence; this story covers items 1–4, 10 (partially), 12 (verification of preconditions only, not execution)
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Technical Constraints & Dependencies] — BorgStack never runs local inference, TTS is exclusively Comadre with voice `kokoro/pm_santa`, JavaScript-only Code nodes
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Evolution API integration — HTTP Request node] — Availability note stating Evolution API is cluster-only; HTTP Request pattern is mandated when it becomes available
- [Source: `_bmad-output/planning-artifacts/architecture.md` § n8n Workflow Naming] — descriptive node naming, trigger-with-suffix convention
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Language Policy] — English-only for technical artifacts including `docs/*.md`
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Code Node Acceptance Criteria] — JavaScript only; Code nodes require documented justification
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Data Architecture] — single database `n8n_db`, all chatbot tables coexist there
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Surface 1 — Git Repository Structure] — canonical location for `docs/infra-verification.md`
- [Source: `_bmad-output/planning-artifacts/architecture.md` § Authentication & Security] — `n8n_encryption_key` model, credential encryption at rest
- [Source: `_bmad-output/planning-artifacts/prd.md` § Infrastructure Topology] — three domains: borgstack-mini (10.10.10.205, no Evolution API), Comadre (10.10.10.207, own Caddy + Bearer auth), Groq (external)
- [Source: `_bmad-output/planning-artifacts/prd.md` § Progression Sequence item 1] — infrastructure setup verification
- [Source: `~/dev/borgstack/config/postgresql/init-databases-mini.sh`] — `n8n_db` and `n8n_user` creation, `CREATE EXTENSION IF NOT EXISTS vector` at init
- [Source: `~/dev/borgstack/templates/stack-mini.yml.j2`] — service topology, Docker Swarm secrets declarations, environment variable wiring
- [Source: `~/dev/borgstack/inventory/shared/images.yml`] — n8n image pin (`n8nio/n8n:2.11.3`, `pixeloddity/n8n-worker:2.11.3`) — read but do not force an upgrade
- [Source: `~/dev/borgstack/inventory/mini/group_vars/all/main.yml`] — `mini_n8n_domain`, `timezone`, resource limits
- [Source: `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml`] — Ansible Vault-encrypted source-of-truth for `n8n_encryption_key` (encrypted at rest)
- [Source: `~/dev/borgstack/inventory/mini/hosts`] — borgstack-mini host `10.10.10.205`
- [Source: `~/dev/borgstack/docs/mini-architecture.md`] — confirmation that Evolution API is cluster-only (line 47–50)
- [Source: `~/dev/comadre/compose.yaml`] — `API_KEY: ${API_KEY:?API_KEY must be set}` — mandatory API key enforcement
- [Source: `~/dev/comadre/deploy/caddy/Caddyfile`] — Comadre's own Caddy reverse proxy on its `DOMAIN`
- [Source: `~/dev/comadre/README.md`] — OpenAI-compatible `POST /v1/audio/speech`, Bearer auth examples, default OGG Opus output
- [Source: `~/dev/comadre/deploy/ansible/inventory/production.yml.example`] — Comadre host `10.10.10.207`, default voice `kokoro/pm_santa`
- [Source: project `CLAUDE.md` § Infrastructure] — borgstack-mini at 10.10.10.205, Comadre at 10.10.10.207, queue mode in use, TTS is Comadre
- [Source: project `CLAUDE.md` § Permissions] — read-only `~/dev/`, write access restricted to this repo
- [Source: Functional Requirement FR41] — credential encryption at rest, architecturally satisfied by AC5 + AC7
- [Source: Non-Functional Requirement NFR9] — credential encryption enforced by `n8n_encryption_key` presence verified in AC5
- [Source: Non-Functional Requirement NFR41] — explicit dependency list; the n8n version string recorded in `docs/infra-verification.md` is the first such entry

## Dev Agent Record

### Agent Model Used

Claude Opus 4.6 (1M context) — `claude-opus-4-6[1m]`

### Debug Log References

None. No errors, no retries, no HALT conditions beyond the AC5 pause described in Completion Notes.

### Completion Notes List

**Outcome:** all 6 acceptance criteria green, verified live against `borgstack-mini` (`10.10.10.205`) and Comadre (`10.10.10.207`) on 2026-04-12. Final Comadre HTTP 200 observed at `2026-04-12T23:26:28-03:00`, which anchors the verification-completion timestamp recorded in `docs/infra-verification.md`.

- **AC1 green:** n8n editor loaded cleanly over HTTPS at `mini_n8n_domain` (currently `n8n.pixeloddity.org`) — no TLS warnings, no 5xx.
- **AC2 green:** n8n version `2.11.3` observed in Settings — 2.x floor (AR4) and CVE-2025-68613 fix floor both met.
- **AC3 green:** `SELECT 1 AS ping;` returned `ping = 1`; `SELECT extname FROM pg_extension WHERE extname = 'vector';` returned exactly one row (`extname = vector`) against `n8n_db`. `pgvector` provisioning from `~/dev/borgstack/config/postgresql/init-databases-mini.sh` is intact.
- **AC4 green:** `SET verify:infra "ok" EX 60` then `GET verify:infra` returned `ok`. First GET attempt returned `null` because the 60s TTL expired during the multi-step walkthrough (discussion of wiring and Set vs Get semantics); a single Execute Workflow run made both operations fire back-to-back within the TTL window and returned the expected value. Noted here as a documentation cue for Story 1.2+ authors who use 60s TTLs in teaching examples.
- **AC5 green, with a brief pause:** `docker secret ls` on `10.10.10.205` listed `n8n_encryption_key` (ID omitted from the committed doc on purpose — it adds no verification value beyond the name). `vault.yml` first line confirmed as `$ANSIBLE_VAULT;1.1;AES256` by direct file inspection. The Ansible Vault password was initially reported as memorized-only; per AC5's explicit pause instruction, Story 1.1 was halted until Galvani confirmed the password is stored on paper in his home safe (reference-only pointer `"Paper note stored in Galvani's home safe"` committed to the doc). FR41 is architecturally satisfied.
- **AC6 green:** `POST https://10.10.10.207/v1/audio/speech` with Bearer auth, voice `kokoro/pm_santa`, and OGG Opus default output returned HTTP 200, `content-type: audio/ogg`, `content-length: 9108` bytes (~3 s at 24 kbps — consistent). Playback through n8n's built-in binary viewer confirmed valid Brazilian Portuguese speech in the `kokoro/pm_santa` (neural male) voice.
- **AC7 green:** `docs/infra-verification.md` written, in English, zero secret values, explicit Evolution API exclusion, FR41/NFR9/NFR41 notes included. Committed as `daa11e7` locally (not pushed — awaiting user decision).
- **AC8 green:** three `_scratch_verify-*` workflows archived and then permanently deleted; two ad-hoc credentials (`_scratch_verify_postgres_adhoc`, `_scratch_verify_redis_adhoc`) deleted; no Comadre credential was auto-created (Bearer was inline in HTTP Request headers). n8n workflow list and credentials list are both empty, so Story 1.2 begins on a clean slate.

**Documented deviation — Comadre API key was not rotated after brief transcript exposure:**

During Task 5 setup, the Comadre `API_KEY` value was captured in this conversation transcript when Galvani ran a `grep` command that emitted `API_KEY=...` to stdout. I recommended rotation per the story's Dev Agent Guardrail that forbids copying the key "into any ... chat window". Galvani exercised an explicit override, on the grounds that (a) Comadre is LAN-only at `10.10.10.207`, (b) there are no external consumers of the key yet (this verification test was its first use), and (c) he accepts the residual risk for the duration of offline refinement. Recorded here — not in the committed `docs/infra-verification.md` — so a code reviewer or future-self has full context. Suggest revisiting at Story 1.2 when the locked `comadre_tts` credential is formally registered: rotating at that point would be a natural "clean slate" moment, and the ops cost is trivial.

**Note on Task 7 cleanup mechanics:** the story's Task 7 line reads "four temporary workflows" but then lists only three. Three were created (one per verification: Postgres, Redis, Comadre); three were deleted. The Task 1 n8n-editor-reachability check did not require a workflow and therefore did not create one. This is a story-text inconsistency, not a verification gap.

### File List

New:

- `docs/infra-verification.md`

Modified (tracking only, no logic change):

- `_bmad-output/implementation-artifacts/sprint-status.yaml` — Story 1.1 status transitions: `ready-for-dev` → `in-progress` → `review`
- `_bmad-output/implementation-artifacts/1-1-infrastructure-verification-n8n-platform-pinning.md` — this file: Status field, Tasks/Subtasks checkboxes, Dev Agent Record, Change Log

Deleted:

- n/a (no code or config was removed from the repository)

## Change Log

| Date | Change | Author |
|---|---|---|
| 2026-04-12 | Story 1.1 executed end-to-end. All 6 ACs verified green against `borgstack-mini` and Comadre. `docs/infra-verification.md` committed (`daa11e7`). One deliberate deviation logged: Comadre API key not rotated after brief transcript exposure (Galvani override, LAN-only). Story status → `review`. | Claude Opus 4.6 + Galvani |

