# Paperclip AI Company HQ: Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Paperclip AI Company HQ** (LISTINGS.md name, **no version label**: Paperclip is date-versioned and the pinned tag `v2026.1005.0` does not read naturally in a name, e.g. "Paperclip v2026.1005 AI Company HQ") |
| Description | Run a team of AI agents with goals, budgets and org charts. (77 chars, unchanged) |
| Category | AI/ML |
| Tags | paperclip, ai-agents, multi-agent, claude-code, codex, orchestration, postgres |

The description is delivered as written: Paperclip's companies, goals, budgets and org charts in Postgres, agent workspaces and CLI logins on a volume, and attachments in a Railway Bucket.

## Base market template

- **Base: `paperclip-official-1`** ("Paperclip (Official Image)", 16 active / 24 recent, the only market template on the official image). Kept: official `ghcr.io/paperclipai/paperclip` image (no wrapper repo), Railway Postgres, `/paperclip` volume, `authenticated` + `private` mode with the browser first-admin claim, `/api/health` healthcheck, `healthcheck.railway.app` in the hostname allow-list, auto-applied migrations, rate limiting, all three signing secrets. Not kept: its ~300-line inline JavaScript "connect helper" injected through an environment variable and a long custom start command that re-installs `@latest` CLIs on every boot and symlinks over files inside `node_modules` (fragile, unpinned, defeats the image tag). It also pins Postgres 17 and an older Paperclip release (2026.916.1).
- `paperclip` (Lukem121/paperclip-railway-template, 262 active, most deploys): builds Paperclip **from source** (old `v2026.722.0`) plus a custom wrapper server, and creates the first-admin invite by inserting rows directly into Paperclip's internal `invites` table (breaks if the schema changes). **No license**, read only. It does not set `PAPERCLIP_TOOL_ACTION_SIGNING_SECRET`, so signed tool-action approvals cannot be issued.
- `paperclip-ai-company` (praveen-ks-2001/paperclip-railway-template, 234 active): custom wrapper image on `node:20-slim` with its own setup page that captures the bootstrap invite URL, `public` exposure, `RAILWAY_RUN_UID=0` (runs as root), no healthcheck, no tool-action signing secret. **No license**, read only.
- No files or README text were copied from any of them.

## Pinned versions

| Service | Image | Tag | Released | Checked |
|---|---|---|---|---|
| Paperclip | `ghcr.io/paperclipai/paperclip` (official) | `2026.1005.0` (release `v2026.1005.0`, latest stable) | 2026-10-06 | 2026-10-10 |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl` (Railway base) | `18` | Railway base | 2026-10-10 |
| Bucket | Railway Storage Bucket | n/a | n/a | n/a |

Agent CLIs come with the official image (`claude`, `codex`, `opencode`, `gemini`, `kimi`, installed by upstream's Dockerfile at image build time). No extra Dockerfile layer is needed.

Upgrade: change the Paperclip tag to a newer release from https://github.com/paperclipai/paperclip/releases and redeploy. `PAPERCLIP_MIGRATION_AUTO_APPLY=true` migrates the database on start. Paperclip also takes hourly database backups to the volume before you upgrade (upstream default: enabled, 60 min, 7 days retention).

## Architecture

```
  Browser ── HTTPS (Railway edge) ──► Paperclip :3100   /api/health
                                        │  board UI + API + heartbeat scheduler
                                        │  spawns agent CLIs (claude, codex, opencode, gemini)
                                        │  volume /paperclip  (HOME: agent workspaces, CLI logins,
                                        │                      instance config, DB backups)
                     private network    ├──► Postgres :5432  (companies, agents, tasks, users)
                                        └──► Bucket (S3 API)  (attachments, uploads)
```

## Services

### Paperclip

| Setting | Value |
|---|---|
| Source | image `ghcr.io/paperclipai/paperclip:2026.1005.0` |
| Start command | none (image default: `tini` → `docker-entrypoint.sh` fixes volume ownership → `gosu node node server/dist/index.js`) |
| Healthcheck | `/api/health`, timeout 300 s (first boot runs all migrations) |
| Public domain | yes, target port 3100 |
| Volume | `/paperclip` (= `HOME` and `PAPERCLIP_HOME` in the image) |
| Restart policy | ON_FAILURE, 10 retries |
| Replicas | 1 (agents run as local processes and workspaces live on the volume) |

Variables (descriptions in `template.json`):

| Variable | Default | Why |
|---|---|---|
| `PORT`, `HOST` | `3100`, `0.0.0.0` | Upstream image defaults, explicit. |
| `DATABASE_URL` | `${{Postgres.DATABASE_URL}}` | External Postgres instead of the embedded database. |
| `PAPERCLIP_DEPLOYMENT_MODE` / `_EXPOSURE` | `authenticated` / `private` | Login required. `private` is upstream's documented Docker "single public URL" setup and the only mode with the browser first-admin claim. |
| `PAPERCLIP_PUBLIC_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Auth callbacks, invite links, allow-list. |
| `PAPERCLIP_ALLOWED_HOSTNAMES` | `healthcheck.railway.app,${{RAILWAY_PRIVATE_DOMAIN}}` | Railway healthcheck Host header and private-network callers pass the private-host guard. |
| `BETTER_AUTH_SECRET`, `PAPERCLIP_AGENT_JWT_SECRET`, `PAPERCLIP_TOOL_ACTION_SIGNING_SECRET` | `${{secret(64, hex)}}` each | Session, agent-token and approval signing, kept separate. |
| `PAPERCLIP_SECRETS_MASTER_KEY` | `${{secret(64, hex)}}` | Company-secret encryption key held in Railway variables instead of only a file on the volume. |
| `PAPERCLIP_MIGRATION_AUTO_APPLY` | `true` | Non-interactive upgrades. |
| `PAPERCLIP_AUTH_RATE_LIMIT_ENABLED`, `TRUST_PROXY` | `true`, `1` | Brute-force protection behind Railway's edge. |
| `PAPERCLIP_STORAGE_PROVIDER` + `PAPERCLIP_STORAGE_S3_*`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | `s3` + `${{Bucket.*}}` | Attachments in the Railway Bucket (upstream recommends S3 for production). |
| `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY` | empty | Optional API-key auth for the local adapters. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `30` | Clean stop of heartbeats and agent processes. |

### Postgres

Copied from `templates/_railway-base/postgres.json` (image `ghcr.io/railwayapp-templates/postgres-ssl:18`, volume `/var/lib/postgresql/data`, generated password, service name `Postgres`).

### Bucket

Railway Storage Bucket named `Bucket`. Credentials are referenced with `${{Bucket.BUCKET}}`, `${{Bucket.REGION}}`, `${{Bucket.ENDPOINT}}`, `${{Bucket.ACCESS_KEY_ID}}`, `${{Bucket.SECRET_ACCESS_KEY}}`. Railway buckets use virtual-hosted URLs, which matches Paperclip's default `forcePathStyle: false`.

## First login (first-admin bootstrap)

1. When the deploy is green, open the Paperclip URL **right away**.
2. Create an account and sign in.
3. On the setup screen choose **Claim this instance**. You become the instance admin. Until this happens, the first signed-in user on the URL can claim it (upstream's `authenticated/private` contract), so do not share the URL before claiming.
4. Create your company, set goals and budgets, and hire agents.

Connecting agent runtimes:
- **API keys**: set `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GEMINI_API_KEY` on the Paperclip service and redeploy.
- **Subscriptions (Claude Pro/Max, ChatGPT)**: Paperclip's "Connect a model" flow shows a login command to run on the server. Run it through Railway SSH as the app user: `railway ssh` (Paperclip service), then `gosu node sh -c '<command shown in Paperclip>'`. The credentials are written under `/paperclip`, so they survive redeploys.

## Improvements over market templates

1. **Official image, pinned to the current release** and run with its own entrypoint (tini + ownership fix + non-root `node` user). No source builds, wrapper servers, root override or boot-time `@latest` installs.
2. **All signing secrets present and separate** (session, agent JWT, tool-action approvals). Two of three market templates miss the tool-action secret, which breaks approvals.
3. **Secrets master key in Railway variables**, so company secrets stay decryptable even if the volume is recreated.
4. **Railway Bucket for attachments** via Paperclip's native S3 provider. No market template uses object storage.
5. **Private-host guard configured for Railway**: public URL, healthcheck host and private domain allowed, so the healthcheck passes and other Railway services can call the API privately.
6. **Brute-force protection on** (`PAPERCLIP_AUTH_RATE_LIMIT_ENABLED=true`, `TRUST_PROXY=1`).
7. **Current Postgres 18 base** from the Railway template, graceful draining on redeploy.

## Resource estimate

Railway prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month (Bucket storage is billed separately by Railway and is negligible for attachments).

| Service | Idle RAM / CPU | Typical RAM / CPU | Notes |
|---|---|---|---|
| Paperclip | 0.5 GB / 0.05 vCPU | 1.5 GB avg, 3-4 GB peaks / 0.4 vCPU | Each running agent CLI (Claude Code, Codex) adds 300-600 MB while it works |
| Postgres | 0.1 GB / 0.02 vCPU | 0.2 GB / 0.05 vCPU | |
| Volumes | 1 GB total | 5 GB (workspaces, backups) + 1 GB (Postgres) | |

Estimated monthly cost: **idle ≈ $8**, **typical ≈ $28** (1.7 GB × $10 + 0.45 vCPU × $20 + 6 GB × $0.15). Model usage is billed by the provider or your subscription.

Minimum plan: **Hobby**. Trial's 1 GB per service is not enough once agents run CLI adapters.

## Verification checklist

- [ ] Paperclip logs: migrations applied, server listening on 3100, no "Invalid PAPERCLIP_SECRETS_MASTER_KEY" or Better Auth secret errors.
- [ ] `GET https://<domain>/api/health` returns 200 and shows `bootstrapStatus: bootstrap_pending` before the claim and `ready` after it.
- [ ] Sign up → **Claim this instance** works; a second account created afterwards gets no admin rights.
- [ ] Upload an attachment to an issue; it appears under the `paperclip/` prefix in the Bucket and downloads again.
- [ ] Hire a Claude Code (or Codex) agent with an API key, assign a small task, and watch one heartbeat run to completion; approve a tool action (proves `PAPERCLIP_TOOL_ACTION_SIGNING_SECRET`).
- [ ] Persistence: redeploy Paperclip; companies, agents, workspaces and CLI logins are intact; stored company secrets still decrypt.

## Open questions

- `authenticated/public` exposure would disable the browser claim and require `paperclipai auth bootstrap-ceo`, which reads an instance `config.json` that an env-only container does not have (market wrappers work around this with custom setup servers or direct inserts into internal tables, which we avoided for upgrade safety). The template therefore uses upstream's `private` appliance mode with a public URL. The residual risk is the short window between deploy and claim.
- Whether Paperclip strips server variables (database URL, bucket keys, auth secrets) from the environment of spawned agent CLIs could not be confirmed in the time available. Market templates have the same exposure. A sandboxed environment provider is the upstream answer for untrusted work.
- `TRUST_PROXY=1` assumes Railway's edge is the only hop that appends to `X-Forwarded-For`. If logs show Railway-internal IPs as client IPs, raise it to `2`.
- Railway Postgres 18 versus upstream's tested Postgres 17: no incompatibility is known, but upstream's compose file still uses 17.
