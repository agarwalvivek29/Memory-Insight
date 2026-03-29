# AGENTS.md — Instructions for AI Coding Agents

This file tells AI coding agents (Claude Code, Codex, Cursor, etc.) how to work in this repo. Read it before touching any code.

---

## What this project is

Memory Insight is a memory forensics platform. A Go daemon on Linux machines takes scheduled memory dumps, uploads them to S3, and triggers a Volatility3 analysis pipeline on AWS Fargate. Findings are classified by severity and summarised by Claude. Users manage everything through a Next.js web UI.

**Source of truth:** `ARCHITECTURE.md` — read it before anything else.

---

## Monorepo layout

```
agent/      Go 1.22 — daemon binary, AVML embedded, systemd service
api/        TypeScript / Fastify — platform API on Fly.io
worker/     Python 3.11 — Volatility3 analysis container (AWS Fargate)
lambda/     Python 3.11 — SQS dispatcher (AWS Lambda)
web/        TypeScript / Next.js 15 — UI on Fly.io
docs/       Per-service PRDs (read these before working on a service)
```

---

## Before you write any code

1. **Read `ARCHITECTURE.md`** — it has every architectural decision, the database schema, the auth model, the queue design, and the failure paths.
2. **Read the PRD for your service** — `docs/prd-{agent,api,worker,lambda,web}.md`.
3. **Check `DAG.md`** — your issue's dependencies must be merged before you start.
4. **Check the GitHub issue** — acceptance criteria are the definition of done.

---

## Auth model (do not invent your own)

Better Auth handles everything: sessions, API keys, orgs, RBAC.

- **Web UI:** uses Better Auth session cookies. Client: `createAuthClient()` from `web/lib/auth-client.ts`.
- **Fastify API:** initialises the same Better Auth instance (`api/src/auth.ts`) pointing at the same `DATABASE_URL` and `BETTER_AUTH_SECRET`. Two middleware functions:
  - `sessionAuth` — calls `auth.api.getSession()`, attaches user + org + role to request
  - `apiKeyAuth` — calls `auth.api.verifyApiKey()`, attaches org_id to request
- **Agent daemon:** uses a **machine token** (`mt_xxxx`), not the org API key. Machine token is a Better Auth `apiKey` record with `metadata: { machine_id }`. The org API key (`sk_live_xxxx`) is used once at registration and never stored on the agent machine.

Do not build custom key hashing, session tables, or RBAC middleware. It already exists in Better Auth.

---

## Database

- ORM: **Drizzle** (`api/src/db/schema.ts`)
- Better Auth manages its own tables (`user`, `session`, `account`, `organization`, `member`, `invitation`, `apikey`). Do not create these manually.
- App tables (in `schema.ts`): `namespaces`, `machines`, `jobs`, `dumps`, `analyses`, `findings`, `schedules`.
- All foreign keys that reference users or orgs use `TEXT` (Better Auth uses string IDs, not UUIDs).
- `ON DELETE RESTRICT` on `machines.namespace_id` — deleting a namespace requires deregistering machines first. Never change this to CASCADE without a migration that handles machine cleanup.

---

## Queue rules

Two queues. Do not mix them.

| Queue | Purpose | Implementation |
|-------|---------|----------------|
| Redis pub/sub | Wake up long-poll connections | `PUBLISH machine:{id}:jobs {job_id}` |
| BullMQ | Repeating schedule jobs | Creates `jobs` record, then publishes to Redis |
| SQS | Trigger Fargate analysis | Published in `POST /jobs/:id/complete` handler |

The BullMQ and SQS paths are completely independent. A job going to SQS does not go through BullMQ.

---

## Agent communication contract

The agent uses **machine tokens** (`mt_xxxx`) as `Authorization: Bearer` on every request after registration.

```
Registration:   POST /machines/register  — Bearer sk_live_xxxx (org key, once only)
All other calls: Authorization: Bearer mt_xxxx
```

Long-poll: `GET /jobs/pending` — hangs up to 30s, returns `200 { job_id, type }` or `204` on timeout. Agent re-polls immediately on 204.

---

## Failure handling rules

- **Redis lost during long-poll:** resolve the pending reply with 204, log, reconnect. Never crash the Fastify process.
- **SQS publish fails after job complete:** set `job.status = FAILED`, `error_code = SQS_PUBLISH_FAILED`, return 500 to agent. Do not silently discard.
- **Worker: no matching ISF:** write a `FAILED` analysis record and exit. Do not retry.
- **Agent AVML_FAILED / PERMISSION_DENIED:** no retry, call POST /jobs/:id/fail immediately.
- **Agent UPLOAD_FAILED:** retry 3× with exponential backoff (5s, 25s, 125s), then /fail.

---

## Fargate worker rules

- The **subprocess** running Volatility3 plugins must use `unshare --net`. It has no network access.
- The **parent Python process** retains HTTPS for: Postgres writes, Claude API calls, SSM parameter fetch.
- Claude API key comes from AWS SSM Parameter Store at startup via `boto3`. Never in env vars.
- `ephemeralStorageGiB` must match the task definition: 32 (small), 64 (medium), 200 (large).

---

## Code style

- TypeScript: strict mode, no `any`. Use Zod for request validation.
- Go: standard `gofmt`, errors wrapped with `fmt.Errorf("context: %w", err)`.
- Python: type hints everywhere, `ruff` for linting.
- No comments that restate what the code does. Comments explain *why*.
- No TODO comments in committed code — open a GitHub issue instead.

---

## Testing

- Go: `go test ./...` with `testify`. Table-driven tests for anything with multiple inputs.
- TypeScript: Vitest + supertest for route integration tests.
- Python: pytest. Worker tests use a fixture with a real small `.lime` sample.
- E2E: Playwright for the critical user path (register → trigger job → view finding).

---

## Do not

- Store the org API key in `config.yaml` on agent machines (use machine token).
- Write custom auth middleware — use Better Auth.
- Add `ON DELETE CASCADE` to `machines.namespace_id` without a data migration.
- Pull ISF symbol files at Volatility3 runtime — they must be bundled in the Docker image.
- Give the Volatility3 subprocess network access.
- Store the Claude API key anywhere except SSM Parameter Store.
- Hard-code AWS region — read from `AWS_REGION` env var (default: `us-east-1`).
