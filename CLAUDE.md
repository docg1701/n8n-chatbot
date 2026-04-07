# n8n-chatbot

## Language

All code, comments, commits, docs, and file contents in this repository must be in native English.

## Project Workflow

The project follows strict sequential phases:

1. **Research** (current phase) — gather information, explore possibilities, understand requirements
2. **Planning** — define architecture, stack, technical decisions based on research
3. **Project creation** — Claude creates the project structure
4. **n8n workflow** — the user builds the workflow in n8n with Claude's guidance
5. **Review & adjustments** — the user brings feedback and concerns, and we finalize together

Respect the current phase. Do not jump ahead to implementation before research and planning are complete.

## Permissions

- **Read-only access** to `~/dev/` — reference other projects for research, but never modify files outside this repository.
- **Write access** is restricted to `/home/galvani/dev/n8n-chatbot/` only.

## Infrastructure

- **Deployment tool:** [BorgStack](https://github.com/docg1701/borgstack) — Docker Swarm IaC with Ansible playbooks (source at `~/dev/borgstack/`)
- **Target environment:** borgstack-mini (single-node) at `10.10.10.205` (local VPS, already running)
- **Available services on mini:** PostgreSQL (pgvector), Redis, n8n (queue mode: editor + webhook + worker), Caddy, Cloudflare tunnel, Homer, Authelia, lldap
- **Deployment path:** local VPS first (`10.10.10.205`), then internet (via Cloudflare tunnel)
- **TTS engine:** [Comadre](https://github.com/docg1701/comadre) at `10.10.10.207` — dual-engine TTS (Kokoro fp32 + Qwen3-TTS), OpenAI-compatible API (`POST /v1/audio/speech`), optimized for Brazilian Portuguese (source at `~/dev/comadre/`)
