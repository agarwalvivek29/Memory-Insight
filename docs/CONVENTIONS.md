# Coding Conventions

Language-specific standards for this monorepo. All contributions must follow these conventions.
Agents must read the relevant section before modifying or generating code.

---

## General (All Languages)

- **No magic numbers** — use named constants
- **No commented-out code** — delete dead code, use version control to recover it
- **No TODOs in merged code** — convert to GitHub Issues before merging
- **Descriptive names** — variable and function names must explain what they do
- **Single responsibility** — functions do one thing; files have a clear, single purpose
- **Error handling at the boundary** — validate at system entry points (API handlers, queue consumers); trust internal code
- **Structured logging only** — never `console.log`, `print()`, or `fmt.Println` in production code

---

## TypeScript (api/, web/)

### Tooling
- **Formatter**: Prettier (config in `.prettierrc`)
- **Linter**: ESLint with `@typescript-eslint`
- **Runtime**: Node.js LTS
- **Package manager**: pnpm

### Rules
- `strict: true` in `tsconfig.json` — no exceptions
- No `any` — use `unknown` and narrow types explicitly
- No `ts-ignore` or `ts-expect-error` without a comment explaining why
- Use `interface` for object shapes, `type` for unions/intersections/aliases
- Path aliases via `tsconfig.json` `paths` — no relative `../../../` chains beyond 2 levels
- Use `zod` for runtime validation of external data (API request bodies, env vars)
- `async/await` throughout — no raw `.then()` chains

### Environment variables
```typescript
// Validated env module — never import process.env directly in business logic
import { env } from '@/config/env'

// Use zod for env validation at startup
import { z } from 'zod'
const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  BETTER_AUTH_SECRET: z.string().min(32),
  REDIS_URL: z.string().url(),
  // ...
})
export const env = envSchema.parse(process.env)
```

### API service (api/ — Fastify)
```
api/src/
├── index.ts          # entry point, server bootstrap
├── auth.ts           # Better Auth instance initialization
├── db/
│   ├── schema.ts     # Drizzle schema (source of truth for all app tables)
│   └── index.ts      # db client
├── routes/           # Fastify route handlers
├── middleware/       # sessionAuth, apiKeyAuth (from Better Auth)
├── domain/           # business logic, pure functions
└── config/
    └── env.ts        # validated env
```

**Fastify route pattern:**
```typescript
// Every route applies auth middleware first
fastify.get('/machines', { preHandler: [sessionAuth] }, async (req, reply) => {
  // req.user, req.session, req.org available after middleware
})

// Agent routes use apiKeyAuth
fastify.post('/jobs/:id/complete', { preHandler: [apiKeyAuth] }, async (req, reply) => {
  // req.machineId available after middleware
})
```

**Auth middleware (do not reimplement):**
```typescript
// api/src/middleware/auth.ts
export const sessionAuth: FastifyPluginCallback = async (req, reply) => {
  const session = await auth.api.getSession({ headers: req.headers })
  if (!session) return reply.status(401).send({ error: 'Unauthorized' })
  req.user = session.user
  req.session = session.session
}

export const apiKeyAuth: FastifyPluginCallback = async (req, reply) => {
  const token = req.headers.authorization?.replace('Bearer ', '')
  const result = await auth.api.verifyApiKey({ key: token })
  if (!result) return reply.status(401).send({ error: 'Unauthorized' })
  req.machineId = result.keyMetadata?.machine_id
  req.orgId = result.key.orgId
}
```

**Long-poll pattern (GET /jobs/pending):**
```typescript
// Hangs up to 30s. Returns 200 { job_id, type } or 204 on timeout.
// Redis pub/sub subscription; resolve with 204 on disconnect or timeout.
// Never crash the Fastify process if Redis disconnects.
```

### Web UI (web/ — Next.js 15)
```
web/
├── app/              # App Router pages
├── lib/
│   └── auth-client.ts  # Better Auth client (createAuthClient)
├── components/
└── public/
```

- Use Better Auth client for all auth operations — no custom session handling
- Server Components by default; Client Components only when needed (interactivity)
- No `getServerSideProps` / `getStaticProps` — use App Router conventions

---

## Python (worker/, lambda/)

### Tooling
- **Formatter**: ruff format
- **Linter**: ruff
- **Type checker**: mypy (strict mode)
- **Dependency management**: uv + `pyproject.toml` — never pip, poetry, or conda directly
- **Test framework**: pytest

### Rules
- Type hints required on all function signatures (enforced by mypy)
- No bare `except:` — always catch specific exceptions
- Use `pydantic` for data models and config validation
- Use `structlog` for structured logging — no `print()`
- Docstrings only on public APIs and non-obvious logic

### Project layout (worker/)
```
worker/
├── src/
│   └── worker/
│       ├── __init__.py
│       ├── main.py          # ECS Fargate entrypoint
│       ├── analysis.py      # Volatility3 subprocess orchestration
│       ├── findings.py      # finding classification + Claude summarization
│       ├── storage.py       # S3 download, Postgres writes
│       └── config.py        # pydantic settings
├── tests/
│   └── fixtures/            # small .lime samples for testing
├── pyproject.toml
└── Dockerfile
```

### Project layout (lambda/)
```
lambda/
├── src/
│   └── dispatcher/
│       ├── __init__.py
│       ├── handler.py       # SQS event handler
│       └── config.py        # pydantic settings
├── tests/
├── pyproject.toml
└── Dockerfile
```

### Worker subprocess rules (CRITICAL)
```python
import subprocess

# ALWAYS use unshare --net for the Volatility3 subprocess
result = subprocess.run(
    ["unshare", "--net", "python3", "-m", "volatility3", ...],
    capture_output=True,
    timeout=300,
)
# Parent process retains network access.
# Subprocess has ZERO network access. This is non-negotiable.
```

### SSM for secrets
```python
import boto3

def get_claude_api_key() -> str:
    """Fetch Claude API key from SSM at startup. Cache the result."""
    ssm = boto3.client('ssm', region_name=settings.aws_region)
    response = ssm.get_parameter(
        Name=f'/memory-insight/{settings.env}/claude-api-key',
        WithDecryption=True
    )
    return response['Parameter']['Value']
# Never read Claude API key from env vars.
```

### Environment variables
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    aws_region: str = "us-east-1"
    s3_bucket: str
    env: str = "dev"
    # Claude API key is NOT here — comes from SSM at runtime

    class Config:
        env_file = ".env"

settings = Settings()
```

### Failure handling
```python
# Worker: no matching ISF → write FAILED analysis record and exit (no retry)
# Lambda: SQS publish fail → raise exception (SQS will retry via visibility timeout)
# Always write structured error records to Postgres before exiting
```

---

## Go (agent/)

### Tooling
- **Formatter**: gofmt (automatic)
- **Linter**: golangci-lint
- **Dependency management**: Go modules (`go.mod`)
- **Test framework**: standard `testing` + testify

### Rules
- Follow the [Standard Go Project Layout](https://github.com/golang-standards/project-layout)
- Error wrapping: `fmt.Errorf("context: %w", err)` — always add context
- No `panic` in service code — return errors
- Use `slog` (stdlib) for structured logging
- Use `context.Context` as first parameter for all I/O operations
- Interfaces defined at the consumer side, not the implementation side
- Use table-driven tests

### Project layout
```
agent/
├── cmd/
│   └── agent/
│       └── main.go          # entry point, config loading
├── internal/
│   ├── api/                 # HTTP client for platform API
│   │   ├── client.go        # API client (machine token auth)
│   │   └── longpoll.go      # GET /jobs/pending long-poll loop
│   ├── dump/                # AVML invocation, file handling
│   ├── upload/              # S3 upload with retry
│   └── config/
│       └── config.go        # config loading from file + env
├── pkg/                     # exported utilities if needed
├── go.mod
└── Dockerfile
```

### Machine token authentication
```go
// All API calls after registration use machine token
type Client struct {
    baseURL     string
    machineToken string // mt_xxxx — from config file after registration
    httpClient  *http.Client
}

func (c *Client) do(ctx context.Context, req *http.Request) (*http.Response, error) {
    req.Header.Set("Authorization", "Bearer "+c.machineToken)
    return c.httpClient.Do(req.WithContext(ctx))
}
```

### Long-poll pattern
```go
// Retry immediately on 204 (timeout). Back off only on errors.
func (c *Client) PollForJob(ctx context.Context) (*Job, error) {
    for {
        job, err := c.getNextJob(ctx)
        if err != nil {
            // log, back off, retry
            continue
        }
        if job == nil {
            // 204 — re-poll immediately
            continue
        }
        return job, nil
    }
}
```

### Upload retry pattern
```go
// Retry 3× with exponential backoff: 5s, 25s, 125s
// After 3 failures, call POST /jobs/:id/fail — do not silently discard
var backoffs = []time.Duration{5 * time.Second, 25 * time.Second, 125 * time.Second}

func uploadWithRetry(ctx context.Context, ...) error {
    for i, delay := range backoffs {
        err := upload(ctx, ...)
        if err == nil {
            return nil
        }
        if i < len(backoffs)-1 {
            time.Sleep(delay)
        }
    }
    return fmt.Errorf("upload failed after %d retries", len(backoffs))
}
```

### Failure modes
```go
// AVML_FAILED / PERMISSION_DENIED → no retry, call /jobs/:id/fail immediately
// UPLOAD_FAILED → retry 3× with backoff, then /jobs/:id/fail
// Registration failure → exit with error (not retry)
```

---

## Database (Drizzle / PostgreSQL)

### Schema conventions
```typescript
// api/src/db/schema.ts
import { pgTable, text, timestamp, integer } from 'drizzle-orm/pg-core'

// String IDs for Better Auth compatibility (not UUID)
export const machines = pgTable('machines', {
  id: text('id').primaryKey().$defaultFn(() => createId()),
  namespaceId: text('namespace_id')
    .notNull()
    .references(() => namespaces.id, { onDelete: 'restrict' }), // NEVER change to cascade
  // ...
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
})
```

- Primary keys: `text` with `cuid2` or `nanoid` (not serial/uuid) — matches Better Auth's ID format
- Timestamps: always `timestamp with time zone` (`timestamptz`)
- Foreign keys to Better Auth tables: `text` (Better Auth uses string IDs)
- `ON DELETE RESTRICT` on `machines.namespace_id` — sacred, never CASCADE without migration

### Migration workflow
```bash
# After changing schema.ts
cd api && pnpm drizzle-kit generate  # generates migration SQL
pnpm drizzle-kit migrate             # applies in dev
# Commit both schema.ts changes AND generated migration files
```

---

## Queue Architecture

Do not mix queue types. Each has exactly one purpose.

| Queue | Purpose | When to publish |
|---|---|---|
| Redis pub/sub | Wake long-poll connections | `PUBLISH machine:{id}:jobs {job_id}` — after creating a job record |
| BullMQ | Repeating scheduled jobs | Creates `jobs` record, then publishes to Redis |
| SQS | Trigger Fargate analysis | Published in `POST /jobs/:id/complete` handler |

BullMQ and SQS paths are completely independent. A job going to SQS does not go through BullMQ.

---

## API Design

- RESTful resource naming: `GET /machines`, `POST /machines`, `GET /machines/:id`
- Consistent error format: `{ error: string, code?: string }`
- Status codes: 200 (ok), 201 (created), 204 (no content / long-poll timeout), 400 (bad request), 401 (unauthorized), 403 (forbidden), 404 (not found), 500 (server error)
- Pagination: cursor-based for lists (`?cursor=&limit=`)
- All request bodies validated with Zod schemas before processing

---

## Docker

### Multi-stage builds (required)
```dockerfile
# Development stage
FROM node:22-alpine AS dev
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install
COPY . .
CMD ["pnpm", "dev"]

# Production stage
FROM node:22-alpine AS prod
WORKDIR /app
COPY --from=dev /app/dist ./dist
COPY --from=dev /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

### Agent binary (Go — static build)
```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o agent ./cmd/agent

FROM scratch
COPY --from=builder /app/agent /agent
ENTRYPOINT ["/agent"]
```

The agent distribution artifact is a static binary — no Docker required on user machines.
