# Memory Insight — Build DAG

This file defines the dependency graph and recommended execution order for all 33 implementation issues.

---

## Dependency Graph

```
MILESTONE 1: Foundation
───────────────────────
#1  Monorepo scaffold (turborepo, shared tsconfig, root package.json)
#2  DB schema + Drizzle migrations          ← depends on #1
#3  Better Auth shared config               ← depends on #1
#4  AWS infrastructure (VPC, S3, SQS, ECR, ECS, IAM roles)
#5  Fly.io config (api + web apps, Redis, env vars)
#6  CI/CD pipelines (GitHub Actions: lint, test, deploy, release) ← depends on #1

MILESTONE 2: API (Fastify)
──────────────────────────
#7  Auth middleware (sessionAuth + apiKeyAuth via Better Auth)  ← depends on #2 #3
#8  Machine registration + machine token issuance               ← depends on #7
#9  Long-poll + Redis pub/sub (GET /jobs/pending)               ← depends on #5 #7
#10 Job lifecycle endpoints (complete + fail + SQS publish)     ← depends on #4 #9
#11 S3 presigned URL endpoint                                    ← depends on #4 #7
#12 Namespace CRUD endpoints                                     ← depends on #7
#13 Schedules + BullMQ wiring                                   ← depends on #5 #9
#14 Results + findings read endpoints                           ← depends on #7

MILESTONE 3: Agent (Go)
────────────────────────
#15 Agent skeleton + AVML embedding (go.mod, embed, Makefile)   ← depends on #1
#16 Register command (POST /machines/register, config.yaml)     ← depends on #8 #15
#17 Long-poll loop (poller goroutine + exponential backoff)     ← depends on #9 #15
#18 Dump pipeline (preflight, AVML, multipart S3 upload, SHA-256) ← depends on #11 #15
#19 Heartbeat goroutine + systemd unit + install.sh             ← depends on #16 #17
#20 GitHub Actions release (static amd64 + arm64 binaries)     ← depends on #6 #19

MILESTONE 4: Analysis Pipeline
───────────────────────────────
#21 Worker Docker image + ISF symbol files bundled              ← depends on #1
#22 Plugin runner + network isolation (unshare --net)           ← depends on #21
#23 Severity classification (classify.py rule table)            ← depends on #22
#24 Claude AI summarizer (top-10 findings, SSM key fetch)       ← depends on #23 #4
#25 Worker DB writes + analysis lifecycle (done/failed)         ← depends on #2 #23
#26 Lambda dispatcher (SQS → ECS RunTask + analyses record)     ← depends on #4 #10 #25
#27 ECS Fargate task definitions (small/medium/large)           ← depends on #4 #21

MILESTONE 5: Web UI (Next.js)
──────────────────────────────
#28 Better Auth client + auth pages (login, register, GitHub OAuth) ← depends on #3 #7
#29 Onboarding flow (create-org, join-org, pending screen)      ← depends on #12 #28
#30 Dashboard + Machines page (fleet summary, namespace grouping) ← depends on #8 #28
#31 Jobs page (history, trigger modal, status timeline, SWR polling) ← depends on #10 #28
#32 Results page (findings cards, AI summary, raw output toggle) ← depends on #14 #28
#33 Org settings (namespaces, members, API keys)                ← depends on #12 #28
```

---

## Parallel Execution Lanes

Issues without shared dependencies can be built in parallel.

```
WEEK 1 — Foundation (serial, everything else blocks on this)
─────────────────────────────────────────────────────────────
  #1  → #2 + #3 + #4 + #5 + #6    (all parallelizable after #1)

WEEK 2 — API core (must complete before Agent + Web UI start)
─────────────────────────────────────────────────────────────
  Lane A: #7 → #8 → #9 → #10 → #11 → #12 → #13 → #14   (mostly sequential)

WEEK 3 — Agent + Worker (parallel with each other, after API)
─────────────────────────────────────────────────────────────
  Lane B: #15 → #16 → #17 → #18 → #19 → #20   (Go Agent, sequential)
  Lane C: #21 → #22 → #23 → #24 → #25          (Worker, sequential)
           then #26 + #27                        (Lambda + ECS, parallel)

WEEK 4 — Web UI (parallel with Agent + Worker)
───────────────────────────────────────────────
  Lane D: #28 → #29 + #30 + #31 + #32 + #33   (#28 blocks all, rest parallel)

WEEK 5 — Integration + polish
──────────────────────────────
  Full end-to-end: register agent → trigger job → analysis → UI results
```

---

## Milestone → Issue Mapping

| # | Title | Milestone | Depends On |
|---|-------|-----------|-----------|
| 1 | Monorepo scaffold | Foundation | — |
| 2 | DB schema + Drizzle migrations | Foundation | #1 |
| 3 | Better Auth shared config | Foundation | #1 |
| 4 | AWS infrastructure | Foundation | — |
| 5 | Fly.io config + Redis | Foundation | — |
| 6 | CI/CD pipelines | Foundation | #1 |
| 7 | Auth middleware (sessionAuth + apiKeyAuth) | API | #2 #3 |
| 8 | Machine registration + machine token | API | #7 |
| 9 | Long-poll + Redis pub/sub | API | #5 #7 |
| 10 | Job lifecycle (complete + fail + SQS) | API | #4 #9 |
| 11 | S3 presigned URL endpoint | API | #4 #7 |
| 12 | Namespace CRUD | API | #7 |
| 13 | Schedules + BullMQ | API | #5 #9 |
| 14 | Results + findings endpoints | API | #7 |
| 15 | Agent skeleton + AVML embedding | Agent | #1 |
| 16 | Register command | Agent | #8 #15 |
| 17 | Long-poll loop (poller goroutine) | Agent | #9 #15 |
| 18 | Dump pipeline (AVML + S3 upload) | Agent | #11 #15 |
| 19 | Heartbeat + systemd + install.sh | Agent | #16 #17 |
| 20 | GitHub Actions release (static binaries) | Agent | #6 #19 |
| 21 | Worker Docker image + ISF files | Analysis Pipeline | #1 |
| 22 | Plugin runner + network isolation | Analysis Pipeline | #21 |
| 23 | Severity classification rules | Analysis Pipeline | #22 |
| 24 | Claude AI summarizer + SSM key fetch | Analysis Pipeline | #4 #23 |
| 25 | Worker DB writes + analysis lifecycle | Analysis Pipeline | #2 #23 |
| 26 | Lambda dispatcher (SQS → ECS RunTask) | Analysis Pipeline | #4 #10 #25 |
| 27 | ECS Fargate task definitions | Analysis Pipeline | #4 #21 |
| 28 | Better Auth client + auth pages | Web UI | #3 #7 |
| 29 | Onboarding flow | Web UI | #12 #28 |
| 30 | Dashboard + Machines page | Web UI | #8 #28 |
| 31 | Jobs page + SWR polling | Web UI | #10 #28 |
| 32 | Results page (findings + AI summary) | Web UI | #14 #28 |
| 33 | Org settings (namespaces, members, API keys) | Web UI | #12 #28 |

---

## Critical Path

The longest blocking chain determines the minimum calendar time:

```
#1 (monorepo) → #3 (Better Auth) → #7 (middleware) → #9 (long-poll)
→ #10 (job lifecycle) → #26 (Lambda) → end-to-end test
```

**Bottom line:** #1 → #3 → #7 → #8 → #16 → #17 → #18 → full agent flow.
Agent cannot register until API auth middleware is done. Worker cannot run until Lambda dispatcher is done.

---

## Labels Reference

| Label | Color | Meaning |
|-------|-------|---------|
| `milestone:foundation` | `#0075ca` | DB, auth config, infra, CI |
| `milestone:api` | `#e4e669` | Fastify endpoints |
| `milestone:agent` | `#d73a4a` | Go daemon |
| `milestone:worker` | `#a2eeef` | Python Volatility3 worker |
| `milestone:web` | `#0e8a16` | Next.js UI |
| `type:feat` | `#7057ff` | New feature |
| `type:infra` | `#b60205` | Infrastructure / IaC |
| `type:security` | `#e11d48` | Security-sensitive change |
| `blocked` | `#d93f0b` | Blocked on dependency |
| `good first issue` | `#7fc97f` | Good entry point |
