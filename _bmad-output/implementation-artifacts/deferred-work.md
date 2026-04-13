# Deferred work

Items flagged during reviews / retros that are real issues but not actionable in the change that surfaced them. Revisit when the surrounding context is in scope.

## Deferred from: code review of story-1-1-infrastructure-verification-n8n-platform-pinning (2026-04-12)

- **Ansible Vault decrypt test for AC5.** The `$ANSIBLE_VAULT;1.1;AES256` header check is spoofable — true tamper-evidence is `ansible-vault view` succeeding with the backed-up password, which would also close AC5's second boundary (proving the paper-backup password is still the correct one). `docs/infra-verification.md:62-63`. Pre-existing: AC5 spec (story `_bmad-output/implementation-artifacts/1-1-infrastructure-verification-n8n-platform-pinning.md:49`) defines the weaker header-only check. Revisit when AC5-style infrastructure verification stories are templatized, or if a DR dry-run is scheduled.

- **`n8n_encryption_key` value integrity check.** `docker secret ls` confirms existence by name only; a zero-byte or malformed secret would still appear in the listing. n8n fails only on first encrypted-credential decrypt, which the disposable ad-hoc credentials in this story never exercise long-term. `docs/infra-verification.md:59-64`. Pre-existing: AC5 spec defines the check as "present and created". Revisit if an n8n credential-decrypt smoke test is ever added to the bootstrap checklist.

- **MITM caveat for LAN "Ignore SSL Issues" pattern.** The doc normalizes TLS verification bypass for Comadre IP access without a LAN-MITM warning. `docs/infra-verification.md:73`. Pre-existing: AC6 spec lines 58-59 explicitly authorize the self-signed LAN path. Revisit when Comadre moves to a real domain (a step likely triggered by Epic 7), at which point the bypass becomes unnecessary.

- **Postgres `execution_entity` rows from manual verification executions.** The doc's "No residual state" claim covers the `verify:infra` Redis key only. Manual executions from the three scratch workflows wrote rows to `execution_entity` in `n8n_db` that persist according to n8n's prune settings. `docs/infra-verification.md:54-55, 115`. Platform-level concern, not a Story 1.1 compliance gap. Revisit if AC8's "clean slate for Story 1.2" is ever interpreted strictly enough to require a Postgres-side cleanup check.

- **Version-snapshot refresh policy.** `docs/infra-verification.md` records `n8n 2.11.3` and specific image pins as a point-in-time baseline. When BorgStack bumps the pin on its independent cadence, a future reader cannot tell from the doc whether it is stale or was verified against a different version. `docs/infra-verification.md:27, 129`. Pre-existing: spec doesn't require a refresh policy. Revisit as part of a project-wide documentation convention (e.g., an "as of <date>" line or an auto-refreshed section), likely during Epic 10 or 11.
