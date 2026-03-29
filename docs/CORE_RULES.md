# Core Repository Rules

> **This document is law.** It applies equally to every contributor — human or AI agent.
> No exception is valid unless it is itself captured in an ADR.

---

## 1. Spec-First Development

**No feature code may be written without a corresponding spec file.**

- Every feature, enhancement, or significant refactor requires a spec at `docs/specs/[ISSUE-NUMBER]-[feature-name].md`
- Use `docs/specs/TEMPLATE.md` as the starting point
- The spec must exist and be linked in the GitHub Issue **before** any implementation begins
- Agents must verify the spec exists before starting any non-trivial coding task
- Cross-reference `DAG.md` to confirm issue dependencies are resolved before starting

Exceptions: bug fixes, typo corrections, dependency updates, and documentation-only changes.

---

## 2. Architecture Decision Records (ADRs)

**Any decision affecting more than one service, the data layer, or infrastructure requires an ADR.**

- ADRs live in `docs/adr/[NNNN]-[short-title].md`
- Use `docs/adr/README.md` to find the next sequential number
- ADRs are append-only — once accepted, they are never deleted, only superseded

Triggers for an ADR:
- Choosing a new database, queue, or caching layer
- Changing API protocol
- Adding a new shared package or cross-service dependency
- Significant infra change (new AWS service, new docker-compose service)
- Changing authentication/authorization strategy
- Any decision that future contributors will ask "why did we do it this way?"

Current ADRs to read before any architectural work:
- `docs/adr/0001-monorepo-structure.md` — service layout and isolation
- `docs/adr/0002-auth-architecture.md` — Better Auth, machine tokens, session sharing
- `docs/adr/0003-queue-architecture.md` — Redis pub/sub vs BullMQ vs SQS separation

---

## 3. Plan-Before-Code

**Non-trivial work requires an approved plan before implementation.**

- **Agents**: Use `EnterPlanMode` for any task touching more than 2 files or introducing new architecture. Exit only when the plan is documented and approved.
- **Humans**: Write a brief implementation plan as a comment on the GitHub Issue.
- A "plan" must include: what files change, why, and what the rollback strategy is.
- Agents must **not** write production code during the planning phase.

---

## 4. Conventional Commits

**All commits must follow the Conventional Commits specification. No AI authorship attribution.**

Format: `<type>(<scope>): <description>`

Allowed types:
- `feat` — new feature
- `fix` — bug fix
- `docs` — documentation only
- `refactor` — code change that neither fixes a bug nor adds a feature
- `test` — adding or correcting tests
- `chore` — build process, dependency updates, tooling
- `perf` — performance improvement
- `ci` — CI/CD changes

Scope is the service name (e.g., `feat(api): add machine registration endpoint`). Valid scopes: `agent`, `api`, `worker`, `lambda`, `web`, `infra`, `docs`.

Hooks must **never** be skipped (`--no-verify` is forbidden unless explicitly approved in an ADR).

**No AI attribution in commits or PRs:**
- Never add `Co-Authored-By: Claude` or any AI model as a co-author
- Never mention Claude, Anthropic, or any AI tool in PR titles, PR bodies, or commit messages

---

## 5. No Direct Commits to Main

**Main is a protected branch. All changes arrive via Pull Request.**

- PRs require at least one approval
- All CI checks must pass before merge
- Squash merges are preferred to keep history clean
- Branch naming: `feat/[ISSUE-NUMBER]-[short-description]`, `fix/[ISSUE-NUMBER]-[short-description]`

---

## 6. Service Isolation

**Each service is a self-contained unit.**

Services: `agent/` (Go), `api/` (TypeScript/Fastify), `worker/` (Python), `lambda/` (Python), `web/` (TypeScript/Next.js)

- Every service has its own `Dockerfile`, `.env.example`, and `AGENTS.md`
- Services communicate only via well-defined interfaces: HTTP (agent→api), SQS (api→lambda→worker), Redis pub/sub (api→agent long-poll)
- Shared code: none currently. If needed, create `packages/` with explicit versioning — never copy-paste between services

---

## 7. Docker-First Local Development

**All services and infrastructure run via `docker-compose`.**

- Every service must have a working `Dockerfile` with multi-stage build (dev + prod targets)
- `infra/docker-compose.yml` is the single source of truth for local infra (Postgres, Redis)
- Environment variables via `.env` files (gitignored); `.env.example` is always committed
- No service should require manual steps beyond `docker compose up` for local infra

---

## 8. Test Coverage

**PRs must not reduce test coverage below the established baseline.**

Test frameworks by service:
- `agent/` (Go): `go test ./...` + testify. Table-driven tests.
- `api/` (TypeScript): Vitest + supertest. Route integration tests.
- `worker/` (Python): pytest. Fixture with a real small `.lime` sample.
- `lambda/` (Python): pytest. Mock SQS events.
- `web/` (TypeScript): Playwright for the critical user path (register → trigger job → view finding).

Unit tests live alongside code. Integration tests in `tests/` at the service root. CI enforces coverage gate.

---

## 9. Secret Hygiene

**Secrets and credentials are never committed to the repository.**

- `.env` files are always gitignored
- `.env.example` is committed with placeholder (non-functional) values
- Never hardcode API keys, passwords, or tokens in source code
- CI secrets managed through GitHub Actions secrets
- **Claude API key**: AWS SSM Parameter Store only (`/memory-insight/{env}/claude-api-key`). Never in env vars.
- **AWS credentials**: IAM role (Fargate task role, Lambda execution role). Never hardcoded.
- If a secret is accidentally committed: revoke it immediately, rotate it, then remove from git history

---

## 10. Auth Middleware (Better Auth — Mandatory)

**Every Fastify API endpoint must be protected by Better Auth middleware.**

Two middleware functions — use the right one per credential type:

| Credential | Middleware | Used by |
|---|---|---|
| Session cookie | `sessionAuth` | Web UI → API calls |
| Bearer `mt_xxxx` | `apiKeyAuth` | Agent daemon (machine token) |
| Bearer `sk_live_xxxx` | `apiKeyAuth` | Agent registration only |

Rules:
- Middleware must be the **first** in the chain (before logging, before rate limiting)
- On auth failure → `401 Unauthorized`
- Never build custom key hashing, session tables, or RBAC middleware — Better Auth handles it
- **Exempt paths** (explicit allowlist, not unprotected by default):
  - `GET /health`
  - `GET /metrics`
  - Better Auth's own handler routes (`/api/auth/*`)

**Machine token rule**: The org API key (`sk_live_xxxx`) is used **once** at registration (`POST /machines/register`). After that, all agent calls use machine token (`mt_xxxx`). Never store the org API key on agent machines.

---

## 11. Agent Operating Discipline

**Agents must follow additional rules beyond those for humans.**

- Read `AGENTS.md` in every service being modified before writing any code
- Consult `docs/specs/` and `docs/adr/` before proposing solutions
- Never modify infrastructure (docker-compose, AWS configs, ECS task definitions) without explicit human approval
- Always prefer editing existing files over creating new ones
- Maintain memory files (`.claude/memory/`) to preserve context across sessions
- If a task is unclear, stop and ask — do not make assumptions that affect architecture
- Document any new tool dependency in the service's `AGENTS.md`

---

## 12. Drizzle Schema-First (Database Layer)

**All database entities are defined in `api/src/db/schema.ts` before any service code references them.**

Better Auth owns (do not touch):
- `user`, `session`, `account`, `organization`, `member`, `invitation`, `apikey`

App tables (all in `api/src/db/schema.ts`):
- `namespaces`, `machines`, `jobs`, `dumps`, `analyses`, `findings`, `schedules`

Process for new tables or columns:
1. Define in `api/src/db/schema.ts`
2. Run `cd api && pnpm drizzle-kit generate` to create the migration
3. Commit schema + migration files together before writing service code
4. Import types from schema — never redefine them in service code

**`ON DELETE RESTRICT` on `machines.namespace_id` is sacred** — never change to CASCADE without a migration that handles machine cleanup and explicit human approval.

---

## 13. Worker Network Isolation (Security — Non-Negotiable)

**The Volatility3 subprocess must have zero network access.**

- Use `unshare --net` on the subprocess that runs Volatility3 plugins
- The **parent Python process** retains HTTPS for: Postgres writes, Claude API, SSM
- ISF symbol files must be **bundled** in the Docker image — never pulled at runtime
- `ephemeralStorageGiB` must match the task definition: 32 (small), 64 (medium), 200 (large)
- Never remove or loosen this isolation — the worker processes potentially hostile memory dumps

---

## 14. No Example or Placeholder Data in Production Code

**Scaffolded example content must be removed before any code ships.**

Must be removed:
- Sample model instances left by generators
- Demo or scaffold routes (`GET /example`, `GET /hello`)
- Scaffold comments: `# TODO: replace with your logic`
- Hardcoded test credentials or placeholder tokens in source files

Allowed to stay:
- `.env.example` placeholder values (required by Rule 9)
- `tests/fixtures/` files used exclusively by automated tests

Agents: scan the full diff before pushing. Fix placeholder content in the same branch.

---

## 15. Feature Change Protocol

**Any time a feature is added, changed, or removed, execute this protocol.**

### Step 1 — Update the Drizzle schema
- Identify every table in `api/src/db/schema.ts` affected by the change
- Apply changes; run `cd api && pnpm drizzle-kit generate`
- Commit schema + migration files before any service code is written

### Step 2 — Update open GitHub issues
- For each open issue whose acceptance criteria or technical notes are now stale, edit the issue body
- Pay attention to: field renames, new required fields, changed status lifecycles

### Step 3 — Create follow-up issues for affected modules
- If the change impacts a service with no open issue tracking the required update, create one

### Step 4 — Update ARCHITECTURE.md
- If the change affects the core domain model (new entity, renamed field, changed lifecycle), update `ARCHITECTURE.md` in the same commit as the schema change

---

## Checklist: Before Opening a PR

- [ ] Spec file exists in `docs/specs/` and is linked in the issue
- [ ] ADR created if an architectural decision was made
- [ ] All commits follow `type(scope): description` format
- [ ] `AGENTS.md` updated if service behavior or architecture changed
- [ ] New tables/columns defined in `api/src/db/schema.ts` with migration generated
- [ ] Unit tests written for new domain logic
- [ ] Integration tests written for new API endpoints / queue handlers
- [ ] Coverage not reduced
- [ ] `.env.example` updated if new env vars were added
- [ ] No secrets committed (especially Claude API key, AWS keys)
- [ ] No scaffold/example/placeholder code remaining in the diff
- [ ] No AI attribution in any commit message or PR body
- [ ] Auth middleware applied to all new Fastify routes
- [ ] Worker subprocess still uses `unshare --net` if modified
- [ ] `docker compose up` still works
- [ ] `/review` run — pre-landing code review passed
- [ ] `/ship` used to create the PR
