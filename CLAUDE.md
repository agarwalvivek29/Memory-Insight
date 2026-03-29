# CLAUDE.md — Agent Contract

> This file governs how Claude Code operates in this repository.
> It also applies to all other AI agents (Copilot, Codex, Cursor, etc.) — `AGENTS.md` is an alias to this file.
> Read this before taking **any** action.

---

## Project Context

Read these before every session:
- **[ARCHITECTURE.md](ARCHITECTURE.md)** — system overview, service map, auth model, database schema, queue design, failure paths, constraints
- **[DAG.md](DAG.md)** — 33-issue build plan with dependency graph and parallel lanes
- **Per-service PRD** — `docs/prd-{agent,api,worker,lambda,web}.md` for the service you're working on

> **Missing file check**: If `ARCHITECTURE.md` does not exist, **stop immediately** and tell the human before proceeding.

---

## Quick Start

| Situation | Action | gstack skill |
|---|---|---|
| Evaluating a new idea | Think first, then use office hours | `/office-hours` |
| Writing a plan (>2 files) | Spec Gate + Plan Gate → EnterPlanMode | `/plan-eng-review` before exiting |
| Architectural decision | ADR Gate — create ADR first | — |
| Pre-landing code check | Run before pushing | `/review` |
| Creating a PR | Full PR workflow | `/ship` |
| CI check failing | Fix root cause — never `--no-verify` | `/debug` |
| Stuck / blocked | Diagnose root cause | `/debug` |
| Feature added / changed / removed | See CORE_RULES Rule 15 | — |
| Unclear requirements | `AskUserQuestion` — never assume | — |
| Potential secret in code | Flag it, do not commit | — |

---

## Mandatory Pre-Flight Checks

Before starting any task, you MUST:

1. **Read `docs/CORE_RULES.md`** — The binding rules for this repository. No exceptions.
2. **Find the GitHub Issue** — Task context, requirements, and acceptance criteria come from the linked issue. Cross-reference `DAG.md` to confirm dependencies are met.
3. **Check for a spec** — Look in `docs/specs/` for a file matching the issue number. If none exists for a feature task, stop and create one before proceeding.
4. **Check relevant ADRs** — Scan `docs/adr/` for decisions that affect the area you're working in. Key ADRs: 0001 (monorepo), 0002 (auth), 0003 (queue architecture).
5. **Read the service AGENTS.md** — If you're modifying a specific service, read `{service}/AGENTS.md` (once it exists) before touching any code.
6. **Check `api/src/db/schema.ts`** — If your task involves any data type (entity, enum, event), verify it's defined there. If not, define it first before writing service code.
7. **Check gstack artifacts** — Look in `~/.gstack/projects/` for test plans, design docs, and review logs. Don't redo work that's already been saved.

---

## Workflow Gates

### Gate 1: Spec Gate
- **Non-trivial features require a spec.** If `docs/specs/[ISSUE-NUMBER]-*.md` does not exist, create it using `docs/specs/TEMPLATE.md` before writing any implementation code.
- Bug fixes, dependency updates, and documentation changes are exempt.

### Gate 2: Plan Gate
- **Changes touching more than 2 files or introducing new architecture require an approved plan.**
- Use `EnterPlanMode` to explore, design, and present your plan. Do not write production code during planning.
- **Before exiting plan mode, run `/plan-eng-review` on your written plan.** Exit only after the plan is approved by the user.

### Gate 3: ADR Gate
- **Any architectural decision requires an ADR.** See `docs/adr/README.md` for what triggers an ADR.
- Create the ADR in `docs/adr/[NNNN]-[title].md` before implementing the decision.

---

## File and Code Rules

- **Prefer editing existing files over creating new ones.**
- **Never skip git hooks.** Do not use `--no-verify` or `--no-gpg-sign`.
- **Never add AI attribution.** Do not add `Co-Authored-By: Claude` or any AI mention in commits or PRs.
- **Never commit to `main` directly.** All changes go through a PR on a feature branch.
- **Never push to `main` directly.** The `.husky/pre-push` hook enforces this.
- **Never hardcode secrets.** Use `.env` files (gitignored). Update `.env.example` with placeholder values.
- **Claude API key lives in AWS SSM Parameter Store only.** Never in env vars, never committed.
- **Never modify `infra/docker-compose.yml` or AWS configs without explicit human approval.**

---

## Task Tracking Discipline

Use the built-in task system for any multi-step work:

```
TaskCreate  → when starting a complex task
TaskUpdate  → mark in_progress before beginning, completed when done
TaskList    → check for next task after completing one
```

Break large tasks into smaller, independently completable units. Never mark a task complete if tests are failing or implementation is partial.

---

## Memory Management

Maintain `.claude/memory/` to preserve context across sessions:

- `MEMORY.md` — high-level summary, always loaded (keep under 200 lines)
- Topic files (e.g., `auth.md`, `database.md`) — detailed notes linked from `MEMORY.md`

Write to memory when you discover:
- Stable architectural patterns in this repo
- Non-obvious conventions or gotchas
- Important file paths and entry points
- Solutions to recurring problems

Do NOT write: session-specific state, incomplete conclusions, or anything that duplicates `CORE_RULES.md` or `ARCHITECTURE.md`.

### gstack Artifacts

```bash
source <(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)
# $SLUG is now set (e.g. agarwalvivek29-memory-insight)
```

| Path | Contents |
|---|---|
| `~/.gstack/projects/$SLUG/*-test-plan-*.md` | QA test plans from `/plan-eng-review` |
| `~/.gstack/projects/$SLUG/*-design-*.md` | Design docs from `/office-hours` |
| `~/.gstack/projects/$SLUG/review-log.jsonl` | Review history |

---

## Capabilities and Tools

### gstack Skills

| Skill | When to use |
|---|---|
| `/office-hours` | Evaluating a new idea before any planning |
| `/plan-eng-review` | Before exiting plan mode — reviews architecture, tests, performance |
| `/plan-ceo-review` | For significant product direction changes |
| `/plan-design-review` | When UI/UX changes are involved |
| `/review` | Pre-landing code review — run before pushing |
| `/ship` | Creating the PR — handles VERSION, CHANGELOG, branch sync |
| `/qa` | Browser-based QA after implementation |
| `/debug` | Systematic root-cause analysis when stuck or CI is failing |
| `/retro` | Weekly retrospective |
| `/document-release` | Post-ship doc updates |

---

## Service Modification Checklist

When modifying a service (`agent/`, `api/`, `worker/`, `lambda/`, `web/`):

- [ ] Read the service's `AGENTS.md` (or `ARCHITECTURE.md` + PRD if AGENTS.md doesn't exist yet)
- [ ] Spec exists in `docs/specs/`
- [ ] ADR created if architectural decision needed
- [ ] Plan approved (EnterPlanMode for >2 files, `/plan-eng-review` before exiting)
- [ ] Auth middleware applied to all new Fastify routes (`sessionAuth` or `apiKeyAuth`)
- [ ] No custom auth logic — use Better Auth (see ADR 0002)
- [ ] New DB tables/columns added to `api/src/db/schema.ts` first, migration generated
- [ ] `ON DELETE RESTRICT` preserved on `machines.namespace_id` (never change to CASCADE without a migration)
- [ ] Worker subprocess uses `unshare --net` — Volatility3 has no network access
- [ ] Claude API key fetched from SSM at startup — never in env vars or hardcoded
- [ ] Unit tests written for new domain functions
- [ ] E2E/integration tests written for new API endpoints
- [ ] `.env.example` updated if new env vars added
- [ ] `AGENTS.md` updated if service behavior or architecture changed
- [ ] No scaffold/example/placeholder code in the diff
- [ ] All commits follow `type(scope): description` format
- [ ] `/review` run before pushing
- [ ] `/ship` used to create the PR

---

## Drizzle Schema Rule (Critical)

Before writing any type definition in service code:

1. Check if the entity/table exists in `api/src/db/schema.ts`
2. If not → define the table there first
3. Generate migration: `cd api && pnpm drizzle-kit generate`
4. Commit schema + migration files together
5. Import types from the schema in service code — never redefine them

Better Auth owns: `user`, `session`, `account`, `organization`, `member`, `invitation`, `apikey` — **do not create these manually**.

App tables: `namespaces`, `machines`, `jobs`, `dumps`, `analyses`, `findings`, `schedules` — all defined in `api/src/db/schema.ts`.

---

## Auth Rules (Critical — See ADR 0002)

- **All Fastify routes** must use `sessionAuth` (session cookie) or `apiKeyAuth` (Bearer token) middleware
- **Never build custom key hashing, session tables, or RBAC middleware**
- **Machine tokens** (`mt_xxxx`) are Better Auth `apiKey` records with `metadata: { machine_id }`
- **Org API key** (`sk_live_xxxx`) is used once at agent registration only — never stored on agent machines
- **Better Auth instance** is shared: Next.js + Fastify initialize from the same `DATABASE_URL` + `BETTER_AUTH_SECRET`

---

## Worker Security Rules (Critical)

- Volatility3 subprocess must use `unshare --net` — zero network access
- Parent Python process retains HTTPS for: Postgres writes, Claude API, SSM fetch
- `ephemeralStorageGiB` must match task definition: 32 (small), 64 (medium), 200 (large)
- ISF symbol files must be bundled in the Docker image — never pulled at runtime
- Claude API key: fetched from AWS SSM Parameter Store (`boto3`) at startup — never in env vars

---

## Multi-Agent Strategy

| Scenario | Tool |
|---|---|
| N GitHub Issues in parallel | `EnterWorktree` per issue — each gets an isolated branch |
| 1 issue, N parallel sub-tasks | gstack Teams (`TeamCreate` + `Agent` tool) |
| Sequential work (default) | Neither needed |

**For worktree work:**
- Each worktree is isolated from `main` and every other session
- Push your feature branch when done; open a PR via `/ship`
- `.claude/` is shared across worktrees — memory accumulates correctly
- Never touch another worktree's branch or `main` directly

---

## Behavior Boundaries

Without explicit human approval, you MUST NOT:

- Push to remote branches
- Create, close, or comment on GitHub Issues or PRs
- Modify CI/CD pipeline configurations
- Drop or migrate databases in production
- Change infrastructure (AWS, docker-compose services, ECS task definitions)
- Delete files that may represent in-progress work
- Force-push any branch
- Modify SSM Parameter Store values

When in doubt: stop, explain what you were about to do, and ask.

---
