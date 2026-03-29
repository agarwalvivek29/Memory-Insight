# 0001 Monorepo with Per-Service Isolation

**Date**: 2026-03-29
**Status**: Accepted
**Deciders**: Vivek Agarwal
**Issue**: N/A (established during architecture design)

## Context

Memory Insight is a multi-service system spanning five distinct runtimes:
- Go daemon (agent) — distributed as a static binary to customer machines
- TypeScript/Fastify API — platform backend on Fly.io
- Python Volatility3 worker — long-running analysis on AWS Fargate
- Python Lambda dispatcher — SQS event routing
- TypeScript/Next.js web UI — customer-facing UI on Fly.io

Each service has a radically different runtime, dependency set, deployment target, and security boundary. The Go agent is open-source and distributed as a curl install. The worker processes potentially hostile memory dumps and must be network-isolated.

The question was: single repository or separate repositories?

## Decision

We will use a **monorepo** with per-service isolation: one repository, services in top-level directories (`agent/`, `api/`, `worker/`, `lambda/`, `web/`), each a fully self-contained unit with its own `Dockerfile`, `.env.example`, and `AGENTS.md`.

Services communicate **only via well-defined interfaces**: HTTP (agent→api), SQS (api→lambda→worker), Redis pub/sub (api→agent long-poll). No shared in-process state. No shared runtime dependencies.

## Consequences

### Positive
- Single place for cross-cutting changes (auth protocol, queue contracts, schema)
- Atomic commits across service boundaries when a feature spans multiple services
- Easier onboarding: one repo to clone, one place to understand the system
- DAG.md and docs/ are co-located with implementation

### Negative
- CI must isolate per-service builds (a Python worker change shouldn't trigger Go agent CI)
- Package manager tooling (pnpm workspaces) doesn't naturally span Go and Python services
- More disciplined about keeping services truly isolated (no "just import from ../api/")

### Neutral
- Each service is still independently deployable
- The open-source agent lives in `agent/` — a separate public repo can mirror it if needed

## Alternatives Considered

### Option A: Separate repositories per service
Rejected: atomic cross-service commits become impossible. Auth protocol changes, queue contract changes, and schema changes all span services. Coordinating across 5 repos is significant overhead for a team of one.

### Option B: Shared runtime monorepo (TypeScript-only)
Rejected: the Go agent and Python worker are non-negotiable runtime choices. The agent must be a static Go binary (no Node.js on customer machines). The worker uses Volatility3 which is Python-native.
