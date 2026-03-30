---
id: local-setup
title: Local Development Setup
sidebar_label: Local Setup
---

# Local Development Setup

This guide walks you through running the Memintel backend on your local machine — from installing dependencies through to verifying a working server.

---

## Prerequisites

Install the following before continuing:

| Dependency | Minimum version | Notes |
|---|---|---|
| Python | 3.11+ | [python.org/downloads](https://www.python.org/downloads/) |
| PostgreSQL | 15+ | Local install or [Supabase](https://supabase.com) / [Neon](https://neon.tech) free tier |
| Redis | 7+ | Local install or [Upstash](https://upstash.com) free tier |
| Git | Any recent | To clone the repository |

Optional — only needed if you use the LLM-assisted pipeline (`/execute/full`):

| Dependency | Notes |
|---|---|
| Anthropic API key | [console.anthropic.com/settings/api-keys](https://console.anthropic.com/settings/api-keys) |

---

## Step 1 — Clone the repository

```bash
git clone https://github.com/your-org/memintel.git
cd memintel/backend/memintel-backend
```

---

## Step 2 — Install Python dependencies

```bash
python -m venv .venv

# macOS / Linux
source .venv/bin/activate

# Windows (PowerShell)
.\.venv\Scripts\Activate.ps1

pip install -r requirements.txt
```

---

## Step 3 — Set environment variables

Copy the example startup script and fill in your values:

```powershell
# Windows PowerShell
Copy-Item start_server.ps1.example start_server.ps1
```

```bash
# macOS / Linux — create an equivalent shell script
cp start_server.ps1.example start_server.sh
```

Open `start_server.ps1` (or `start_server.sh`) and replace every placeholder with a real value. The complete variable reference is below.

:::warning
Never commit `start_server.ps1` or `start_server.sh` — they contain real credentials. The `.example` file is safe to commit. Keep your local copy out of version control.
:::

### Environment variable reference

#### Core — required for all routes

| Variable | Purpose | Required | Example |
|---|---|---|---|
| `MEMINTEL_CONFIG_PATH` | Path to `memintel_config.yaml` | Yes | `memintel_config.yaml` |
| `DATABASE_URL` | PostgreSQL connection URL for the primary Memintel database (definitions, graphs, jobs, feedback) | Yes | `postgresql://user:pass@localhost:5432/memintel` |
| `REDIS_URL` | Redis connection URL for the execution result cache | Yes | `redis://localhost:6379` |

#### Security

| Variable | Purpose | Required | Example |
|---|---|---|---|
| `MEMINTEL_ELEVATED_KEY` | Secret key for privileged endpoints — pass as `X-Elevated-Key` header. See [Elevated Key](#the-elevated-key) below. | Yes | `a-long-random-secret` |

#### LLM provider

| Variable | Purpose | Required | Example |
|---|---|---|---|
| `ANTHROPIC_API_KEY` | Anthropic API key for the LLM-assisted pipeline | Only if `USE_LLM_FIXTURES=false` | `sk-ant-...` |
| `USE_LLM_FIXTURES` | Set to `false` in production to use a real LLM provider. Defaults to `true` (fixture mode) — safe for local dev without an API key | No | `false` |

#### Data connector credentials

These variables correspond to connectors declared in `memintel_config.yaml`. Add only the variables for connectors you have configured.

| Variable | Connector | Purpose | Example |
|---|---|---|---|
| `ANALYTICS_DB_HOST` | `postgres.analytics` | Hostname of the analytics PostgreSQL instance | `analytics-db.internal` |
| `ANALYTICS_DB_USER` | `postgres.analytics` | Database user | `analytics_reader` |
| `ANALYTICS_DB_PASSWORD` | `postgres.analytics` | Database password | `••••••••` |
| `ACCOUNTS_DB_HOST` | `postgres.accounts` | Hostname of the accounts PostgreSQL instance | `accounts-db.internal` |
| `ACCOUNTS_DB_USER` | `postgres.accounts` | Database user | `accounts_reader` |
| `ACCOUNTS_DB_PASSWORD` | `postgres.accounts` | Database password | `••••••••` |
| `BILLING_API_URL` | `rest.billing_api` | Base URL of the billing REST API | `https://billing.example.com` |
| `BILLING_API_TOKEN` | `rest.billing_api` | Bearer token for billing API auth | `••••••••` |

---

## Step 4 — Run database migrations

Memintel uses [Alembic](https://alembic.sqlalchemy.org/) to manage the schema. Run migrations once after cloning and again after each update that includes new migration files:

```bash
alembic upgrade head
```

You should see output like:

```
INFO  [alembic.runtime.migration] Running upgrade  -> abc123, create conditions table
INFO  [alembic.runtime.migration] Running upgrade abc123 -> def456, create execution_graphs table
...
```

If the database already contains the latest schema (e.g. on a re-run), Alembic prints nothing and exits cleanly — this is expected.

---

## Step 5 — Start the server

**Windows (PowerShell):**

```powershell
.\start_server.ps1
```

**macOS / Linux:**

```bash
# Set variables inline or export them from start_server.sh
export MEMINTEL_CONFIG_PATH=memintel_config.yaml
export DATABASE_URL=postgresql://user:pass@localhost:5432/memintel
export REDIS_URL=redis://localhost:6379
export MEMINTEL_ELEVATED_KEY=your-elevated-key
# ... add remaining variables ...

uvicorn app.main:app --host 0.0.0.0 --port 8000
```

The server starts on port 8000. You should see:

```
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

:::tip Hot reload for development
Add `--reload` to the `uvicorn` command to automatically restart the server when source files change:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```
:::

---

## Step 6 — Verify the server is running

```bash
curl http://127.0.0.1:8000/openapi.json
```

A successful response returns the full OpenAPI specification as JSON. You can also open the interactive docs in a browser:

```
http://127.0.0.1:8000/docs
```

:::note Use 127.0.0.1, not localhost
On some systems `localhost` resolves to `::1` (IPv6) but the server binds to `0.0.0.0` (IPv4). Using `127.0.0.1` directly avoids a ~2-second TCP timeout on every request.
:::

---

## The Elevated Key

Certain endpoints require the `X-Elevated-Key` header in addition to the standard `X-API-Key`. These are privileged operations that modify shared registry state:

| Endpoint | Operation |
|---|---|
| `POST /compile` | Compile a concept to an execution graph |
| `POST /conditions/register` | Register a new condition |
| `POST /concepts/register` | Register a new concept |
| `POST /actions/register` | Register a new action |
| `POST /conditions/apply-calibration` | Apply a calibration recommendation |

Set the header value to match `MEMINTEL_ELEVATED_KEY`:

```bash
curl -X POST http://127.0.0.1:8000/compile \
  -H "X-Elevated-Key: your-elevated-key" \
  -H "Content-Type: application/json" \
  -d '{ "concept": { ... } }'
```

Use a long, randomly generated secret. Generate one with:

```bash
# macOS / Linux
openssl rand -hex 32

# Python (any platform)
python -c "import secrets; print(secrets.token_hex(32))"
```

---

## Troubleshooting

**`connection refused` on port 8000**
The server is not running, or it started on a different port. Check the terminal where you ran `uvicorn`.

**`startup_failed` — database connection error**
Verify `DATABASE_URL` is correct and the PostgreSQL instance is reachable. Test the connection:

```bash
psql "$DATABASE_URL" -c "SELECT 1;"
```

**`startup_failed` — Redis connection error**
Verify `REDIS_URL` is correct and Redis is running:

```bash
redis-cli -u "$REDIS_URL" ping
# Expected output: PONG
```

**`alembic upgrade head` fails with `relation already exists`**
The database was partially initialised. Reset it and re-run migrations:

```bash
# Drop and recreate the database (local dev only — destroys all data)
psql "$DATABASE_URL" -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
alembic upgrade head
```

**`HTTP 403` on compile or register endpoints**
These endpoints require the `X-Elevated-Key` header. See [The Elevated Key](#the-elevated-key) above.
