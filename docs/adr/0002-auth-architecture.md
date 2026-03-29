# 0002 Better Auth as Unified Auth Layer

**Date**: 2026-03-29
**Status**: Accepted
**Deciders**: Vivek Agarwal
**Issue**: N/A (established during architecture design)

## Context

The platform needs:
- Email/password sign-up and sign-in
- GitHub OAuth
- Organization creation, invitations, and membership management
- Role-based access control (admin/write/read)
- API key generation and validation for the agent daemon
- A machine-token mechanism for per-machine identity (distinct from org API keys)

The agent daemon authenticates with the API on behalf of a specific machine. The org API key is used once at registration; subsequent calls use a machine-scoped token stored locally.

## Decision

We will use **Better Auth** as the single auth layer for all services.

Better Auth runs as a shared instance initialized with the same `DATABASE_URL` and `BETTER_AUTH_SECRET` in both Next.js (web/) and Fastify (api/). No session bridging. No shared-secret JWT. No custom key table.

**Plugins enabled:**
- `emailPassword` — email/password sign-up and sign-in
- `socialProvider("github")` — GitHub OAuth
- `organization` — org creation, invitations, membership, role assignment
- `apiKey` — key generation (SHA-256 stored), validation, per-key rate limiting, expiry
- `access` — RBAC mapping org roles to permissions

**Machine token convention**: Machine tokens (`mt_xxxx`) are Better Auth `apiKey` records with `metadata: { machine_id }`. The org API key (`sk_live_xxxx`) is used **once** at `POST /machines/register` and never stored on agent machines.

## Consequences

### Positive
- Zero custom auth code: no key hashing, no session bridging, no RBAC middleware
- Better Auth owns `user`, `session`, `account`, `organization`, `member`, `invitation`, `apikey` tables — no manual management
- Machine token revocation is a standard Better Auth API key operation
- Org management UI routes are handled by Better Auth's built-in handlers

### Negative
- Tight coupling to Better Auth's data model and upgrade path
- Both Next.js and Fastify must be initialized with the same env vars (operationally simple, but a misconfiguration causes auth failures across both services)

### Neutral
- Better Auth is initialized in both services — this is by design, not a bug

## Alternatives Considered

### Option A: Custom JWT auth with shared secret
Rejected: requires building and maintaining key hashing, rotation, session management, org membership. Significant implementation surface and security risk.

### Option B: Auth0 / Clerk / Okta
Rejected: vendor lock-in, cost at scale, and the org/RBAC model is product-critical — we need direct database control for machine token queries and audit logging.

### Option C: NextAuth.js (Fastify gets JWT tokens from Next.js)
Rejected: requires session bridging between Next.js and Fastify. Better Auth's shared-instance model eliminates this complexity entirely.
