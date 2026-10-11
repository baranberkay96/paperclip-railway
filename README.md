# Deploy and Host Paperclip with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/paperclip-ai-company-hq)

Paperclip is an open-source control plane for running a company staffed by AI agents. You set goals, an org chart and budgets, then hire agents such as Claude Code, Codex, OpenCode or Gemini CLI that work through tasks on a schedule and ask for your approval when needed. This template runs it with Postgres, a volume and object storage.

## About Hosting Paperclip

Paperclip runs a Node.js server that serves the board UI and API, wakes agents on heartbeats and launches their CLIs as local processes. It needs Postgres for companies, agents, tasks and users, a persistent home directory for agent workspaces, CLI logins and backups, and somewhere durable for attachments. This community template uses the official `ghcr.io/paperclipai/paperclip` image pinned to a release, Railway Postgres on the private network, a volume at `/paperclip` and a Railway Bucket through Paperclip's S3 storage provider. It runs in authenticated mode with every signing secret generated for you.

## Common Use Cases

- Run a small team of coding agents against your repositories with budgets, approvals and an audit trail
- Give a solo founder or small team an always-on set of agents working through a backlog
- Coordinate different agent runtimes (Claude Code, Codex, OpenCode, Gemini) under one org chart and goal tree
- Evaluate Paperclip on a real HTTPS URL before moving it to your own infrastructure

## Dependencies for Paperclip Hosting

- Postgres (included, private network)
- A Railway volume for `/paperclip` and a Railway Bucket for attachments (included)
- An API key or subscription login for at least one agent runtime (Anthropic, OpenAI, Google and others)

### Deployment Dependencies

- Paperclip repository and releases: https://github.com/paperclipai/paperclip
- Paperclip documentation: https://docs.paperclip.ing/
- Railway Storage Buckets: https://docs.railway.com/storage-buckets
- Railway SSH (for subscription logins): https://docs.railway.com/cli/ssh

### Implementation Details

| Service | Image | Purpose |
|---|---|---|
| Paperclip | `ghcr.io/paperclipai/paperclip:2026.1005.0` | Board UI, API, heartbeat scheduler, agent CLIs; volume at `/paperclip` |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl:18` | Primary database |
| Bucket | Railway Storage Bucket | Attachments and uploads (S3 API) |

**First login (do this right after the deploy turns green)**

1. Open the Paperclip URL, create an account and sign in.
2. On the setup screen choose **Claim this instance** to become the instance admin. Until someone claims it, the first signed-in user can, so claim it before sharing the URL.
3. Create your company, set goals and budgets, and hire agents.

**Connecting agents**: add `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` or `GEMINI_API_KEY` to the Paperclip service, or use a subscription. For subscriptions, Paperclip's "Connect a model" screen shows a login command. Run it on the server with `railway ssh` and then `gosu node sh -c '{that command}'`. The login is stored on the volume.

**Scaling**: keep one Paperclip replica, because agents run as local processes against workspaces on the volume. Railway scales the service's RAM and CPU up to your plan limit as more agents work in parallel. Postgres can be moved to Railway's HA Postgres later if needed.

**Pinning and upgrades**: change the image tag to a newer release from the GitHub releases page and redeploy. Migrations run automatically, and Paperclip keeps hourly database backups on the volume (7 days by default). Keep a copy of `PAPERCLIP_SECRETS_MASTER_KEY`: it decrypts the company secrets stored in Postgres.

### Why Deploy Paperclip on Railway?

Paperclip only works if agents can wake up on schedule and the board stays reachable. Railway runs the server, database and storage together on a private network with persistent volumes, HTTPS, logs and SSH, so you can focus on what your agent company should do instead of maintaining servers.
