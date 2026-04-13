# Infrastructure verification — BorgStack mini + Comadre

**Verification completed:** 2026-04-12T23:26:28-03:00 (ISO 8601, America/São_Paulo, -03:00 offset)
**Operator:** Galvani
**Scope:** Story 1.1 ground-truth verification of `borgstack-mini` (`10.10.10.205`) and Comadre TTS (`10.10.10.207`). This is the baseline every later epic will trust.

## Summary

All eight infrastructure acceptance criteria for Story 1.1 passed. No production workflows were built; no locked credentials were created. The three temporary verification workflows and their two disposable ad-hoc credentials (plus an inline Bearer header) were discarded as part of Story 1.1 Task 7, leaving `borgstack-mini` clean for Story 1.2.

Verification covered:

- n8n editor reachability over HTTPS and version on the 2.x floor
- PostgreSQL connectivity and the `pgvector` extension, both against the shared `n8n_db`
- Redis SET/GET roundtrip with a 60-second TTL
- `n8n_encryption_key` Docker Swarm secret presence, Ansible Vault encrypted at rest, and Vault password backed up out-of-band
- Comadre TTS reachability, Bearer authentication, and valid Brazilian Portuguese audio output

Evolution API was **not** verified here — see the dedicated section below.

### Acceptance criteria at a glance

| AC | Requirement | Status | Section |
|---|---|---|---|
| AC1 | n8n editor reachable over HTTPS | Pass | n8n editor and version |
| AC2 | n8n version is 2.x | Pass | n8n editor and version |
| AC3 | Postgres reachable and `pgvector` in `n8n_db` | Pass | PostgreSQL and pgvector |
| AC4 | Redis SET/GET roundtrip | Pass | Redis roundtrip |
| AC5 | `n8n_encryption_key` + Ansible Vault + out-of-band backup | Pass | n8n encryption key and Ansible Vault |
| AC6 | Comadre TTS reachable with Bearer + valid pt-BR audio | Pass | Comadre TTS |
| AC7 | Verification document committed, English, zero secret values | Pass | this document as a whole |
| AC8 | Temporary workflows and credentials discarded | Pass | How this was verified and cleanup |

## n8n editor and version (AC1, AC2)

- **URL:** n8n editor behind Cloudflare Tunnel + Caddy at `mini_n8n_domain` (currently `n8n.pixeloddity.org`).
- **HTTPS:** editor loaded cleanly via the Cloudflare Tunnel production path (Browser → Cloudflare edge → `cloudflared` on `borgstack-mini` → Caddy → n8n editor — not a LAN-direct short-circuit) — no TLS warnings, no 5xx.
- **Running version:** `2.11.3`, observed in Settings.
- **2.x floor satisfied:** `2.11.3` is a 2.x release; AR4 (the platform version pinning requirement — "n8n is any 2.x release") is met. The CVE-2025-68613 fix shipped in `1.120.4`, and every release in the 2.x series inherits it by release lineage (not by lexical string ordering), so the security floor is also satisfied regardless of which 2.x minor BorgStack pins.
- **Upgrade cadence:** BorgStack pins the image in `~/dev/borgstack/inventory/shared/images.yml` (currently `n8nio/n8n:2.11.3`, `pixeloddity/n8n-worker:2.11.3`) and upgrades on operator cadence. This project does not demand a specific minor.

## PostgreSQL and pgvector (AC3)

Verified via a temporary workflow (`_scratch_verify-postgres`) built from an empty canvas, with a disposable ad-hoc credential (`_scratch_verify_postgres_adhoc` — **not** `postgres_main`, which is locked for Story 1.2).

| Check | Query | Result |
|---|---|---|
| Connectivity | `SELECT 1 AS ping;` | one row returned: `ping = 1` |
| `pgvector` provisioned | `SELECT extname FROM pg_extension WHERE extname = 'vector';` | exactly one row returned: `extname = vector` |

- **Connection:** host `postgresql`, port `5432`, database `n8n_db`, user `n8n_user`. The ad-hoc credential pinned `database: n8n_db` explicitly, so the `pg_extension` query binds to `n8n_db`'s extension set and not to any other database in the cluster. A belt-and-braces check could add `SELECT current_database();` but AC3 did not require one.
- **Single shared database:** `borgstack-mini` ships one application database. All chatbot tables will coexist in `n8n_db` alongside n8n's own tables. There is no separate `chatbot` database.
- **Password source:** the `n8n_db_password` Docker Swarm secret, populated at deploy time from `vault_n8n_db_password` in Ansible Vault (`~/dev/borgstack/inventory/mini/group_vars/all/vault.yml`).
- **Initialization reference:** `~/dev/borgstack/config/postgresql/init-databases-mini.sh` — `CREATE EXTENSION IF NOT EXISTS vector` on `n8n_db` at first container boot. This story verifies the extension is already present; no provisioning happens here.

## Redis roundtrip (AC4)

Verified via a temporary workflow (`_scratch_verify-redis`) with a disposable ad-hoc credential (`_scratch_verify_redis_adhoc` — **not** `redis_main`, which is locked for Story 1.2).

| Step | Operation | Result |
|---|---|---|
| 1 | `SET verify:infra "ok" EX 60` | succeeded (n8n's Redis Set node returns an empty item on success) |
| 2 | `GET verify:infra` | returned the string `ok` |

- **Connection:** host `redis`, port `6379`, database `0`.
- **Password source:** the `redis_password` Docker Swarm secret, populated from `vault_redis_password` in Ansible Vault.
- **Key namespace:** `verify:*` is deliberately outside n8n's `n8n:*` queue-mode key namespace, so this test cannot collide with queue state.
- **TTL:** the key expires after 60 seconds, so no residual state remains after verification. No cleanup was needed.
- **Execution-time constraint:** the two operations must fire inside that 60-second window. In our verification session a first step-through reached the second operation after the TTL had expired (we discussed node wiring between the two runs), and `GET` correctly returned `null`. A single Execute Workflow run — both Set and Get fired back-to-back — returned `ok` as expected. Anyone reproducing this check should trigger the full workflow in one go rather than executing the two Redis nodes individually with a pause between them.

## n8n encryption key and Ansible Vault (AC5)

- **Docker Swarm secret present:** `docker secret ls` on `10.10.10.205` returned a row named `n8n_encryption_key` (created with the rest of the stack's secrets). Alongside it: `n8n_db_password`, `redis_password`, `postgresql_password`, `cloudflare_tunnel_token`, and the Authelia/lldap secret family — all expected and documented in BorgStack's mini inventory.
- **Secret mount (per `~/dev/borgstack/templates/stack-mini.yml.j2`):** the secret is mounted at `/run/secrets/n8n_encryption_key` inside the n8n editor, webhook, and worker containers; n8n reads it via the `N8N_ENCRYPTION_KEY_FILE` environment variable. The value is never set as a plaintext `N8N_ENCRYPTION_KEY` env var.
- **Source of truth:** `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml`, Vault key `vault_n8n_encryption_key`.
- **Ansible Vault encrypted at rest:** the first line of `vault.yml` begins with `$ANSIBLE_VAULT;1.1;AES256` — confirmed by direct inspection. The file is never committed in plaintext and never echoed in any chat, log, or repo artifact.
- **Ansible Vault password — out-of-band backup:** *Paper note stored in Galvani's home safe.* This is a reference-only pointer; the password itself does not appear in this document, in this repository, in any n8n credential, or in any chat transcript.
- **FR41 (credential encryption at rest):** architecturally satisfied. The Docker secret exists; the source-of-truth value is encrypted in Ansible Vault; the Ansible Vault password is recoverable from out-of-band backup. Every credential Story 1.2 onwards registers in n8n will be protected by this chain.

Zero secret values appear anywhere in this document. The Docker secret IDs from `docker secret ls` are intentionally omitted — they are stable identifiers that offer no verification value beyond the secret name itself.

## Comadre TTS (AC6)

Verified via a temporary workflow (`_scratch_verify-comadre`) with a single HTTP Request node. A disposable inline `Authorization: Bearer` header was used; **no** locked `comadre_tts` credential was created here — Story 1.2 owns that name.

- **Endpoint:** `POST https://10.10.10.207/v1/audio/speech`
- **TLS:** Comadre runs its own Caddy reverse proxy; the host is an IP (`10.10.10.207`), so the certificate is self-signed and the n8n HTTP Request node was configured with **Ignore SSL Issues** enabled. This is the path AC6 contemplates explicitly.
- **Host-header match:** Comadre's Caddyfile binds the vhost to `{$DOMAIN:localhost}` and only responds when the inbound `Host` header matches `DOMAIN`. On this host Comadre was deployed with `DOMAIN=10.10.10.207` in `/home/galvani/comadre/.env`, so the IP-direct request matches. Anyone reproducing the check against a Comadre instance that still runs with the default `DOMAIN=localhost` will get a 308/404 from Caddy even with TLS verification disabled — the fix is to set `DOMAIN` to the hostname or IP the n8n HTTP Request node will use. See `~/dev/comadre/docs/troubleshooting.md` for the upstream troubleshooting note.
- **Auth:** `Authorization: Bearer <COMADRE_API_KEY>`. The key lives on `10.10.10.207` in `/home/galvani/comadre/.env` and is not recorded anywhere in this repository.
- **Request body (JSON):**
  ```json
  {
    "input": "Olá, teste de verificação de infraestrutura.",
    "voice": "kokoro/pm_santa"
  }
  ```
- **Response:**
  - Status: **HTTP 200 OK**
  - `content-type: audio/ogg`
  - `content-length: 9108` bytes (≈ 3 seconds at 24 kbps OGG Opus, consistent with the input phrase)
  - Security headers present: HSTS (`max-age=63072000; includeSubDomains; preload`), CSP, X-Frame-Options `DENY`, X-Content-Type-Options `nosniff`, Referrer-Policy `strict-origin-when-cross-origin`, Permissions-Policy disabling microphone/camera/geolocation.
- **Playback:** valid Brazilian Portuguese speech in the `kokoro/pm_santa` (neural male) voice, intelligible, duration consistent with the input, played through n8n's built-in binary viewer.
- **Default output format:** OGG Opus (24 kbps, 16 kHz, mono) per Comadre's documented defaults — MP3 is not Comadre's default and is not used.

Comadre is available and operational for Epic 7's voice modality (STT + TTS).

## Evolution API — not verified (intentional) (AC7)

Evolution API is **explicitly out of scope** for this verification story.

**Why:** Evolution API is not part of `borgstack-mini`. It ships only in the full BorgStack cluster (`~/dev/borgstack/templates/stack-worker-*.yml.j2`), and `~/dev/borgstack/docs/mini-architecture.md` confirms the mini topology excludes it. Between Epic 1 and Epic 7 the chatbot has no Evolution API dependency — Telegram is the sole client channel through Epic 7, and WhatsApp lights up only with Epic 8.

**When it gets verified:** Epic 8 Story 8.2 — "WhatsApp ingestion via Evolution API" — owns the Evolution API reachability check as a prefix step, alongside either the `mini → cluster` swap or a standalone Evolution API deployment.

**Credential posture:** the locked credential name `evolution_api` is **not** created in this story and is not created by Story 1.2 either. Story 8.2 registers it when it first becomes needed.

## How this was verified and cleanup (AC7, AC8)

Verification was performed live in the n8n editor against the running `borgstack-mini` deployment (not a staging copy, not a local Docker Compose sandbox). Three temporary workflows were created on an empty canvas:

- `_scratch_verify-postgres` — Manual Trigger → Postgres (Execute Query, two queries)
- `_scratch_verify-redis` — Manual Trigger → Redis (Set) → Redis (Get)
- `_scratch_verify-comadre` — Manual Trigger → HTTP Request

Host-side checks:

- `docker secret ls` on `10.10.10.205` for `n8n_encryption_key` presence.
- Reading the first line of `~/dev/borgstack/inventory/mini/group_vars/all/vault.yml` for the Ansible Vault encrypted-at-rest signature.

Each n8n workflow used a disposable, ad-hoc credential — never one of the locked names Story 1.2 owns (`postgres_main`, `redis_main`, `comadre_tts`, `evolution_api`). The two ad-hoc credentials (`_scratch_verify_postgres_adhoc`, `_scratch_verify_redis_adhoc`), the inline Comadre Bearer header (which was never saved as an n8n credential entity), and the three temporary workflows were all removed in Story 1.1 Task 7, leaving the n8n workflow list and credentials list empty for Story 1.2.

## Compliance notes (AC7)

- **Language:** English, per project `CLAUDE.md` and Architecture §Language Policy. The single Portuguese string in this document — `"Olá, teste de verificação de infraestrutura."` — is transient TTS input test data, not a persisted or committed artifact elsewhere.
- **Secret values:** zero. Every secret is referenced by Docker secret name, Ansible Vault variable name, or reference-only pointer to where it is stored.
- **FR41 (credential encryption at rest):** architecturally satisfied by the `n8n_encryption_key` chain verified in the AC5 section above.
- **NFR9 (credential encryption enforced):** satisfied by `n8n_encryption_key` presence in Swarm; every credential n8n registers from this point onwards is automatically encrypted at rest.
- **NFR41 (explicit dependency list):** this document records the first set of observed dependency identities for the project — see the "Platform and tool versions observed" table below.

## Platform and tool versions observed

| Component | Image pin / identity | Source |
|---|---|---|
| n8n | `2.11.3` (any 2.x acceptable) | observed in editor Settings; pinned in `~/dev/borgstack/inventory/shared/images.yml` |
| PostgreSQL + `pgvector` | image `pgvector/pgvector:pg18`; `vector` extension present in `n8n_db` at boot | `~/dev/borgstack/inventory/shared/images.yml`, `~/dev/borgstack/config/postgresql/init-databases-mini.sh` |
| Redis | image `redis:8-alpine` | `~/dev/borgstack/inventory/shared/images.yml` |
| Comadre | own Caddy on `10.10.10.207`, OpenAI-compatible `POST /v1/audio/speech`, default voice `kokoro/pm_santa`, default output OGG Opus, Bearer auth mandatory | `~/dev/comadre/compose.yaml`, `~/dev/comadre/deploy/caddy/Caddyfile`, `~/dev/comadre/README.md` |
| `borgstack-mini` host | `10.10.10.205` | `~/dev/borgstack/inventory/mini/hosts` |
| Comadre host | `10.10.10.207` | `~/dev/comadre/deploy/ansible/inventory/production.yml.example` |

## Known follow-ups

- **Rotate `COMADRE_API_KEY` at Story 1.2's `comadre_tts` credential registration.** Story 1.2 is the natural clean-slate moment for Comadre auth — the credential list transitions from empty to the seven locked names Story 1.2 owns, and the operational cost of a rotation is trivial. Rotating then closes any residual exposure envelope accumulated during Story 1.1's verification pass (including any transient capture of the key in operator terminals or session transcripts).
