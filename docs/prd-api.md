# PRD: Fastify API

**Service:** `api/`
**Language:** TypeScript (Node 20)
**Framework:** Fastify 4
**Deployment:** Fly.io

---

## Overview

The Fastify API is the central nervous system of Memory Insight. It handles machine registration, issues machine tokens, drives the long-poll job delivery mechanism, manages schedules, generates S3 presigned URLs, and triggers analysis dispatch via SQS. It also exposes read endpoints for results and findings consumed by the web UI.

All authentication is delegated to Better Auth — no custom auth code exists in this service.

---

## Goals

- Serve as the single API surface for both the Go agent (API key / machine token auth) and the Next.js web UI (session auth).
- Issue per-machine tokens at registration so the org API key is never stored on agent machines.
- Deliver jobs to agents via long-poll (30s hang, Redis pub/sub wakeup) that works correctly across multiple Fastify instances.
- Dispatch analysis jobs to Fargate via SQS with explicit failure handling.
- Enforce org-scoped RBAC on all session-authed endpoints via Better Auth's access plugin.

## Non-goals

- Does not touch dump bytes — S3 upload is direct agent → S3.
- Does not run Volatility3 — that is the worker's job.
- Does not handle email sending for org invitations (Better Auth handles that).

---

## Tech stack decisions

| Concern | Choice | Reason |
|---------|--------|--------|
| Auth | Better Auth (`organization` + `apiKey` + `access` plugins) | Zero custom auth code |
| ORM | Drizzle ORM | Type-safe, lightweight, migration-first |
| Job scheduling | BullMQ + Redis | Repeating cron jobs, dedup by key |
| Long-poll wakeup | Redis pub/sub | Works across multiple Fastify instances |
| Analysis dispatch | AWS SQS (standard queue) | Decouples API from Fargate cold start |
| Validation | Zod | Request schema validation on every route |

---

## Module structure

```
api/
├── src/
│   ├── auth.ts                  Better Auth instance (same config as web/)
│   ├── server.ts                Fastify app factory + plugin registration
│   ├── middleware/
│   │   ├── sessionAuth.ts       auth.api.getSession() → req.user, req.orgId, req.role
│   │   └── apiKeyAuth.ts        auth.api.verifyApiKey() → req.orgId, req.machineId
│   ├── routes/
│   │   ├── machines.ts          register, list, heartbeat, deregister
│   │   ├── jobs.ts              create, long-poll, complete, fail
│   │   ├── dumps.ts             presigned URL generation
│   │   ├── results.ts           findings + analyses read
│   │   ├── schedules.ts         CRUD + BullMQ wiring
│   │   └── namespaces.ts        CRUD (admin only)
│   ├── db/
│   │   ├── schema.ts            Drizzle schema (app tables only)
│   │   ├── index.ts             DB connection pool
│   │   └── migrations/
│   └── lib/
│       ├── s3.ts                Presigned URL helpers (aws-sdk v3)
│       ├── queue.ts             BullMQ workers + Redis pub/sub publish
│       └── sqs.ts               SQS publish helper
├── package.json
└── .env.example
```

---

## Environment variables

```
DATABASE_URL             postgres://... (shared with web/)
BETTER_AUTH_SECRET       long random string (shared with web/)
REDIS_URL                redis://...
AWS_REGION               us-east-1
AWS_ACCESS_KEY_ID        ...
AWS_SECRET_ACCESS_KEY    ...
S3_BUCKET                memoryinsight-dumps
SQS_QUEUE_URL            https://sqs.us-east-1.amazonaws.com/...
PORT                     3001
```

---

## API contract

### Machine registration

```
POST /machines/register
Authorization: Bearer sk_live_xxxx   (org API key)

Request body:
{
  namespace_id: string   // must belong to the org resolved from the API key
  hostname:     string
  os:           "linux"
  distro:       string   // e.g. "Ubuntu 22.04"
  arch:         "amd64" | "arm64"
  total_memory_gb: number
  agent_version: string
}

Response 201:
{
  machine_id:    string   // UUID
  machine_token: string   // "mt_..." — stored in agent config.yaml
}

Errors:
  401 — invalid org API key
  400 — namespace_id missing or invalid format
  403 — namespace does not belong to the org
  409 — hostname already registered in this namespace
```

### Long-poll

```
GET /jobs/pending
Authorization: Bearer mt_xxxx   (machine token)

Behaviour:
  - Validates machine token via Better Auth verifyApiKey
  - Confirms metadata.machine_id matches
  - Subscribes to Redis channel "machine:{machine_id}:jobs"
  - Hangs up to 30s
  - On Redis PUBLISH: responds 200 { job_id, type: "dump" }
  - On 30s timeout: unsubscribes, responds 204
  - On Redis error: unsubscribes, responds 204 (never crashes)

Fastify config required:
  server.keepAliveTimeout = 36000   (36s, longer than 30s poll window)
```

### Job lifecycle

```
POST /jobs/:id/complete
Authorization: Bearer mt_xxxx

Request body:
{
  s3_key:      string
  size_bytes:  number
  checksum:    string   // SHA-256 hex of the dump file
}

Behaviour:
  - Verifies job.machine.org_id === resolved org_id from machine token
  - Updates job.status → QUEUED, sets dump record
  - Publishes SQS message { job_id, s3_key, size_bytes }
  - On SQS failure: sets job.status = FAILED, error_code = SQS_PUBLISH_FAILED, returns 500

Response: 200 { ok: true }

---

POST /jobs/:id/fail
Authorization: Bearer mt_xxxx

Request body:
{
  error_code:    "AVML_FAILED" | "UPLOAD_FAILED" | "DISK_FULL" | "PERMISSION_DENIED"
  error_message: string
}

Response: 200 { ok: true }
```

### Presigned URL

```
GET /dumps/presigned-url
Authorization: Bearer mt_xxxx
Query: job_id, size_bytes

Response 200:
{
  upload_url: string   // S3 presigned PUT URL, 15-min expiry
  s3_key:     string   // "dumps/{org_id}/{machine_id}/{job_id}.lime"
}
```

### Namespace management (session auth, admin only)

```
POST   /namespaces           Create namespace in caller's active org
GET    /namespaces           List namespaces for org
DELETE /namespaces/:id       Delete (blocked if machines exist → 409)
```

### Schedules

```
POST  /schedules              Create cron schedule for a machine (write+ role)
GET   /schedules/:machine_id  Get machine's active schedule (read+ role)
PATCH /schedules/:id          Update cron expression or enabled flag (write+ role)
DELETE /schedules/:id         Delete schedule (write+ role)
```

### Results

```
GET /results/:job_id      Full analysis: analyses record + findings array (read+ role)
GET /findings/:job_id     Findings only with severity (read+ role)
```

---

## Long-poll implementation detail

```typescript
// jobs.ts — GET /jobs/pending
fastify.get('/jobs/pending', { preHandler: [apiKeyAuth] }, async (req, reply) => {
  const machineId = req.machineId
  const channel = `machine:${machineId}:jobs`

  return new Promise((resolve) => {
    const sub = redis.duplicate()  // dedicated subscriber connection
    let resolved = false

    const timeout = setTimeout(() => {
      if (!resolved) {
        resolved = true
        sub.unsubscribe(channel)
        sub.quit()
        resolve(reply.code(204).send())
      }
    }, 30_000)

    sub.subscribe(channel, (err) => {
      if (err) {
        clearTimeout(timeout)
        resolved = true
        sub.quit()
        resolve(reply.code(204).send())  // Redis error → clean 204, not crash
      }
    })

    sub.on('message', (_, message) => {
      if (!resolved) {
        resolved = true
        clearTimeout(timeout)
        sub.unsubscribe(channel)
        sub.quit()
        const { job_id } = JSON.parse(message)
        resolve(reply.send({ job_id, type: 'dump' }))
      }
    })
  })
})
```

---

## Schedule engine

BullMQ repeating jobs fire server-side. On API startup:

1. Read all enabled schedules from DB
2. For each: `queue.add('dump', { machine_id }, { repeat: { cron: schedule.cron_expr }, jobId: `schedule:${schedule.id}` })`
3. BullMQ worker processes the job: creates a `jobs` DB record, then Redis PUBLISH to wake any waiting long-poll

New schedules created via API are registered with BullMQ immediately (no restart).
Deleted schedules: `queue.removeRepeatable('dump', { cron: ..., jobId: ... })`.

---

## RBAC enforcement

Every session-authed route runs `sessionAuth` middleware which resolves:
- `req.user` — Better Auth user object
- `req.orgId` — active org from session
- `req.role` — `admin | write | read`

Then route handlers check: `await auth.api.hasPermission({ userId: req.user.id, orgId: req.orgId, permission: 'machine:trigger' })`.

Permission map (matches ARCHITECTURE.md RBAC table):

| Permission string | Admin | Write | Read |
|-------------------|-------|-------|------|
| `org:manage` | ✓ | — | — |
| `namespace:manage` | ✓ | — | — |
| `apikey:manage` | ✓ | — | — |
| `machine:trigger` | ✓ | ✓ | — |
| `schedule:edit` | ✓ | ✓ | — |
| `machine:view` | ✓ | ✓ | ✓ |
| `results:view` | ✓ | ✓ | ✓ |

---

## Failure paths

See `ARCHITECTURE.md → API Failure Paths` for Redis lost mid-poll and SQS publish failure handling.

---

## Acceptance criteria (Phase 1)

- [ ] `POST /machines/register` creates machine record + returns machine token. Org API key not required after this point.
- [ ] `GET /jobs/pending` hangs 30s, returns 204 on timeout. Returns 200 with job when Redis PUBLISH fires.
- [ ] Redis failure during long-poll returns 204, not 500.
- [ ] `POST /jobs/:id/complete` with SQS failure sets job.status = FAILED with SQS_PUBLISH_FAILED error code.
- [ ] RBAC: Read-role user cannot trigger job (403). Write-role user can.
- [ ] Duplicate hostname in same namespace returns 409.
- [ ] Namespace deletion with active machines returns 409.
- [ ] All routes validate request body with Zod. Invalid input returns 400 with field-level error.
