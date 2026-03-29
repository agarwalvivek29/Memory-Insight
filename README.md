# Memory Insight

**"VirusTotal for live memory" — scheduled memory surveillance for Linux servers.**

Memory Insight is an open-core platform that gives sysadmins and security researchers scheduled memory forensics without the enterprise price tag. A lightweight Go daemon runs on your Linux machines, takes scheduled memory dumps, and ships them to a cloud analysis pipeline powered by Volatility3. Findings are classified by severity, summarised in plain English by Claude, and surfaced in a web UI.

---

## What it does

```
curl -sSL https://get.memoryinsight.io | bash -s -- \
  --key sk_live_xxxx \
  --namespace ns_xxxxxxxx
```

That one command installs a daemon that:
- Registers the machine with your org
- Long-polls for jobs (scheduled or manual)
- Takes a full memory dump via AVML
- Uploads directly to S3
- Triggers Volatility3 analysis in an isolated Fargate container
- Returns structured findings (malfind, pslist, netscan, cmdline, proc.Maps, envars) with severity classification and AI summaries

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                        Web UI (Next.js)                  │
│  login · orgs · namespaces · machines · jobs · results   │
└────────────────────────┬─────────────────────────────────┘
                         │ Better Auth session
                         ▼
┌──────────────────────────────────────────────────────────┐
│                    Fastify API (Fly.io)                   │
│  machine tokens · long-poll · jobs · schedules · results │
│  BullMQ (Redis) for scheduling · SQS publish on complete │
└──────┬──────────────────────────────────────┬────────────┘
       │ long-poll (30s, Redis pub/sub wakeup) │ presigned PUT URL
       ▼                                       ▼
┌─────────────────┐                   ┌───────────────────┐
│  Go Agent       │                   │    AWS S3         │
│  (systemd daemon│──── dump ────────▶│  (us-east-1)      │
│  on Linux host) │                   └────────┬──────────┘
└─────────────────┘                            │ POST /jobs/:id/complete
                                               ▼
                                    ┌──────────────────────┐
                                    │  SQS → Lambda        │
                                    │  dispatcher          │
                                    └──────────┬───────────┘
                                               │ ECS RunTask
                                               ▼
                                    ┌──────────────────────┐
                                    │  ECS Fargate Worker  │
                                    │  Python + Volatility3│
                                    │  (network-isolated)  │
                                    └──────────────────────┘
```

**Phase 1:** Full pipeline — scheduled dumps, Volatility3 analysis, findings + AI summaries.
**Phase 2:** Diff alerting — surface processes that appeared or disappeared between scans.
**Phase 3:** ML severity scoring, threat intel enrichment.

---

## Repository Structure

```
memory-insight/
├── agent/          Go daemon installed on Linux machines
├── api/            Fastify API (Fly.io)
├── worker/         Python Volatility3 analysis container
├── lambda/         Python SQS dispatcher
├── web/            Next.js UI
├── docs/           Per-service PRDs
├── ARCHITECTURE.md Full design doc (source of truth)
├── AGENTS.md       Instructions for AI coding agents
├── DAG.md          Issue dependency graph and build order
└── README.md       This file
```

---

## Quick start (local dev)

```bash
# Prerequisites: Node 20+, Go 1.22+, Python 3.11+, Docker, bun

# 1. Start Postgres + Redis
docker compose up -d

# 2. Run Better Auth migrations
cd api && bun run db:migrate

# 3. Start the API
cd api && bun run dev

# 4. Start the web UI
cd web && bun run dev

# 5. Build the agent (local test only — requires Linux for AVML)
cd agent && make build
```

For the full analysis pipeline (worker + Lambda + Fargate), see `docs/prd-worker.md`.

---

## Services

| Service | Language | Runtime | Deployment |
|---------|----------|---------|------------|
| `agent/` | Go 1.22 | Binary | GitHub Releases |
| `api/` | TypeScript | Node 20 / Fastify | Fly.io |
| `worker/` | Python 3.11 | Docker | AWS ECS Fargate |
| `lambda/` | Python 3.11 | Zip | AWS Lambda |
| `web/` | TypeScript | Next.js 15 | Fly.io |

---

## Contributing

See `AGENTS.md` for how to work with AI coding agents on this repo.
Issues are tracked on GitHub — see `DAG.md` for the recommended build order.
