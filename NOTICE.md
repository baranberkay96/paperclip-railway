# NOTICE: Paperclip template

## Software deployed

| Component | Source | License | How it is used |
|---|---|---|---|
| Paperclip | https://github.com/paperclipai/paperclip, image `ghcr.io/paperclipai/paperclip:2026.1005.0` | MIT, Copyright (c) 2025 Paperclip AI | Official image pulled unmodified (it also bundles the agent CLIs listed below). No upstream files are copied into this template. |
| Agent CLIs inside the Paperclip image | Claude Code, OpenAI Codex CLI, OpenCode, Gemini CLI, Kimi Code (installed by upstream's Dockerfile) | Each under its vendor's license/terms | Not modified; use is subject to each vendor's terms |
| PostgreSQL | `ghcr.io/railwayapp-templates/postgres-ssl:18` (definition from `templates/_railway-base/postgres.json`) | PostgreSQL License | Railway's standard Postgres service |
| Storage Bucket | Railway Storage Buckets | Railway service | Attachments via Paperclip's S3 provider |

Paperclip is a trademark of its owners. This is a community template and is not affiliated with or endorsed by Paperclip AI.

## Upstream files checked (tag `v2026.1005.0`, checked 2026-10-10)

- `Dockerfile` (production stage): Node 24 trixie base, preinstalled agent CLIs, `HOME=/paperclip`, `PAPERCLIP_DEPLOYMENT_MODE=authenticated`, `PAPERCLIP_DEPLOYMENT_EXPOSURE=private`, `tini` → `docker-entrypoint.sh`
- `scripts/docker-entrypoint.sh`: volume ownership repair, `gosu node`
- `server/src/config.ts`: bind/host, public URL, allowed hostnames, S3 storage variables, DB backup defaults
- `server/src/index.ts`: `PAPERCLIP_MIGRATION_AUTO_APPLY`
- `server/src/auth/better-auth.ts`, `server/src/agent-auth-jwt.ts`, `server/src/services/tool-content-guards.ts`: `BETTER_AUTH_SECRET`, `PAPERCLIP_AGENT_JWT_SECRET`, `PAPERCLIP_TOOL_ACTION_SIGNING_SECRET`
- `server/src/secrets/local-encrypted-provider.ts`: `PAPERCLIP_SECRETS_MASTER_KEY` formats (64-char hex accepted)
- `server/src/middleware/private-hostname-guard.ts`, `server/src/middleware/trust-proxy.ts`: allow-list and `TRUST_PROXY`
- `server/src/routes/access.ts` (`/bootstrap/claim`), `server/src/routes/health.ts`, `cli/src/commands/auth-bootstrap-ceo.ts`: first-admin flows
- `server/src/services/local-ai-login.ts`: subscription login commands shown in the UI
- `packages/db/src/backup-lib.ts`: backups fall back to a JavaScript engine when `pg_dump` is absent
- `docs/deploy/docker.md`, `docs/deploy/deployment-modes.md`, `docs/deploy/environment-variables.md`, `docs/deploy/storage.md`, `docs/deploy/database.md`, `doc/DOCKER.md`, `doc/DEPLOYMENT-MODES.md`

## Market templates and repositories reviewed (ideas only, nothing copied)

| Template / repo | License | Use |
|---|---|---|
| `paperclip-official-1` (image-only template, inline helper in variables) | not stated | Reviewed for configuration choices (official image, private exposure, healthcheck host). Its inline helper code was not used. |
| `paperclip` / `Lukem121/paperclip-railway-template` | none (all rights reserved) | Read only. No files or text copied. |
| `paperclip-ai-company` / `praveen-ks-2001/paperclip-railway-template` | none (all rights reserved) | Read only. No files or text copied. |

All files in this package (TEMPLATE.md, template.json, README.md, this NOTICE) were written for this template.
