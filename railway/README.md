# Archon on Railway

This directory contains a Railway-oriented deployment shape for Archon without changing the upstream Docker Compose defaults.


## Docker Image Choice

Do **not** use the docs-only CLI command as the Railway app service:

```bash
docker run --rm -v "$PWD:/workspace" ghcr.io/coleam00/archon:latest workflow list
```

That command is for running the published Archon CLI image against a local project mounted from your laptop. Railway cannot provide `$PWD:/workspace` as a bind mount.

For this template, build `archon-app` from the repository `Dockerfile` instead. The app is self-contained, starts the web/server process via `docker-entrypoint.sh`, and stores managed repositories/worktrees under the persistent `/.archon` volume.

## Topology

- `archon-caddy`: public HTTP service. Railway terminates TLS at the edge, then routes to Caddy on `$PORT`.
- `archon-auth`: private form auth sidecar used by Caddy `forward_auth`.
- `archon-app`: private Archon app service built from the upstream root `Dockerfile`.
- `Postgres`: Railway managed PostgreSQL plugin/service.
- `archon-data`: Railway volume attached to `archon-app` at `/.archon`.

Only `archon-caddy` should have a public Railway domain. The app and auth services should communicate over Railway private networking with internal hostnames such as `archon-app.railway.internal` and `archon-auth.railway.internal`.

## Files

- `railway/Caddyfile`: Railway Caddy config. It listens on `:{$PORT}`, protects the UI/API with form auth, and bypasses `/webhooks/*` plus `/api/health`.
- `railway/caddy/Dockerfile`: tiny Caddy image that copies the Railway Caddyfile.
- `railway/app/railway.json`: app config-as-code example with Dockerfile build, healthcheck, and Postgres migration pre-deploy command.
- `railway/caddy/railway.json`: Caddy config-as-code example.
- `railway/auth-service/railway.json`: auth sidecar config-as-code example.
- `railway.template.json`: draft metadata for the intended multi-service template.

## Service Setup

Create four services in one Railway project/environment:

1. Add a managed PostgreSQL service named `Postgres`.
2. Add `archon-app` from this repo. In service settings, set the config-as-code file to `/railway/app/railway.json`.
3. Add `archon-auth` from this repo. Set the config-as-code file to `/railway/auth-service/railway.json`.
4. Add `archon-caddy` from this repo. Set the config-as-code file to `/railway/caddy/railway.json`.

Generate a public domain only for `archon-caddy`. Do not generate public domains for `archon-app` or `archon-auth`.

## Variables

Set these on `archon-app`:

```bash
ARCHON_DOCKER=true
ARCHON_HOME=/.archon
DATABASE_URL=${{Postgres.DATABASE_URL}}
HOST=::
PORT=8000
DEFAULT_AI_ASSISTANT=claude
AUTH_USERNAME=admin
AUTH_PASSWORD_HASH=<bcrypt hash>
COOKIE_SECRET=<64+ random hex chars>
```

Set one assistant credential path:

```bash
CLAUDE_CODE_OAUTH_TOKEN=<token from claude setup-token>
# or CLAUDE_API_KEY=<anthropic api key>
# or CODEX_ID_TOKEN=<...> plus CODEX_ACCESS_TOKEN=<...>
```

Set platform variables only for adapters you use:

```bash
GITHUB_TOKEN=<github token>
WEBHOOK_SECRET=<github webhook secret>
GITHUB_ALLOWED_USERS=<optional comma-separated users>
SLACK_BOT_TOKEN=<optional>
SLACK_APP_TOKEN=<optional>
DISCORD_BOT_TOKEN=<optional>
TELEGRAM_BOT_TOKEN=<optional>
```

Set these on `archon-auth`:

> Railway note: paste the bcrypt hash exactly as generated. Do **not** escape `$` as `$$`; that escaping is only for Docker Compose.


```bash
PORT=8000
# AUTH_PORT=9000 / AUTH_SERVICE_PORT=9000 also work outside Railway; prefer PORT=8000 on Railway.
AUTH_USERNAME=${{archon-app.AUTH_USERNAME}}
AUTH_PASSWORD_HASH=${{archon-app.AUTH_PASSWORD_HASH}}
COOKIE_SECRET=${{archon-app.COOKIE_SECRET}}
COOKIE_MAX_AGE=86400
```

Set these on `archon-caddy`:

```bash
ARCHON_APP_PRIVATE_URL=http://archon-app.railway.internal:8000
AUTH_SERVICE_PRIVATE_URL=http://archon-auth.railway.internal:8000
```

If you rename services, update the `*.railway.internal` hostnames.

## Auth Setup

Generate a bcrypt password hash locally. Do not commit it. Docker Compose is not required; this one-off Node container is enough:

```bash
docker run --rm node:22-alpine sh -lc "npm add bcryptjs >/dev/null && node -e \"require('bcryptjs').hash(process.argv[1], 12).then(console.log)\" 'YOUR_PASSWORD'"
```

Set the resulting hash directly as `AUTH_PASSWORD_HASH` in Railway. Do not double `$` characters in Railway variables; that escaping is only for Docker Compose env interpolation.

Generate a cookie secret locally with either command:

```bash
docker run --rm node:22-alpine node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
# or
openssl rand -hex 32
```

## Persistent Data

Attach one Railway volume to `archon-app`:

- Name: `archon-data`
- Mount path: `/.archon`

Archon stores workspaces, worktrees, artifacts, and local app data under `/.archon` in Docker. PostgreSQL stores relational state separately in the managed Postgres service.

If the app logs volume permission errors, set `RAILWAY_RUN_UID=0` on `archon-app` and redeploy. The upstream Docker entrypoint fixes `/.archon` ownership before dropping to `appuser`.

## Database and Migrations

Archon selects PostgreSQL when `DATABASE_URL` is set. Without it, the app falls back to SQLite; do not omit `DATABASE_URL` on Railway.

The upstream Docker Compose Postgres service auto-runs `migrations/000_combined.sql` only because Compose mounts that SQL file into the Postgres container init directory. Railway managed Postgres does not run repo migrations automatically.

`railway/app/railway.json` therefore uses this app pre-deploy command:

```bash
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f migrations/000_combined.sql
```

The root Dockerfile installs `postgresql-client` and copies `migrations/` into the image, so this command should run inside Railway's pre-deploy container. The combined migration is documented in the SQL header as idempotent and safe to run multiple times. On upgrades, check whether upstream added new migration files after the combined migration and apply the required incrementals if they are not reflected in `000_combined.sql`.

## App Start Command

Use the upstream Dockerfile default for `archon-app`; do not override the start command unless Railway requires it.

The Docker entrypoint runs:

```bash
bun run setup-auth
bun run start
```

The root `start` script resolves to:

```bash
bun --filter @archon/server start
```

and `@archon/server` starts with:

```bash
bun src/index.ts
```

## Validation Checklist

Before publishing this as a template:

- `archon-app` deploy logs show the pre-deploy `psql` migration completed.
- `archon-app` logs show `db.connection_postgresql_selected`, `database_connected`, and `server_listening` on port `3090`.
- `archon-auth` logs show it is listening on `:9000`.
- `archon-caddy` logs show it is listening on Railway `$PORT`.
- `https://<caddy-domain>/api/health` returns JSON without login.
- `https://<caddy-domain>/login` renders the form.
- `https://<caddy-domain>/` redirects to or requires login.
- A bad login is rejected and a valid login reaches the Archon UI.
- GitHub or other webhook delivery to `/webhooks/*` is not blocked by form auth.
- No public domain exists on `archon-app` or `archon-auth`.

## Known Blockers / TODO Checks

- Confirm the final Railway template builder accepts the draft `railway.template.json` service metadata as-is; Railway's normal checked-in config-as-code contract is per-service `railway.json`, while multi-service template publication is usually configured in Railway's template UI.
- Confirm service names before deploy. The Caddy defaults assume `archon-app` and `archon-auth`.
- Confirm upgrade migration procedure against the exact upstream release being deployed.
