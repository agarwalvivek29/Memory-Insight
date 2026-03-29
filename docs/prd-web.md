# PRD: Next.js Web UI

**Service:** `web/`
**Language:** TypeScript
**Framework:** Next.js 15 (App Router)
**Auth:** Better Auth (sessions, GitHub OAuth, email/password)
**Deployment:** Fly.io

---

## Overview

The web UI is the user-facing control plane for Memory Insight. It handles authentication, organisation and namespace management, machine monitoring, job triggering, and result inspection. All state lives in Postgres via the Fastify API — the web UI is a thin client that never writes to the database directly.

---

## Goals

- Onboard new users: create or join an organisation in < 3 clicks after signup
- Surface the machine fleet with real-time status (last seen, last job, namespace grouping)
- Let users trigger manual memory dumps and watch job progress
- Display findings with severity-coloured cards, AI summary by default, raw Volatility3 output expandable
- Let admins manage namespaces, invite team members, and manage API keys
- Work on mobile (responsive, not a full mobile app)

## Non-goals

- Real-time WebSocket streaming of job progress (polling is fine for Phase 1)
- Dark mode (Phase 2)
- Billing / usage dashboard (Phase 2)

---

## Tech stack

| Concern | Choice |
|---------|--------|
| Framework | Next.js 15 App Router |
| Auth | Better Auth (`emailPassword` + `socialProvider("github")` + `organization` + `apiKey` + `access`) |
| Data fetching | SWR for client polling, Server Components for initial load |
| Styling | Tailwind CSS + shadcn/ui |
| Forms | react-hook-form + Zod |
| API calls to Fastify | `fetch` from Server Components / Route Handlers |

---

## Module structure

```
web/
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx           Email/password + GitHub OAuth
│   │   └── register/page.tsx        Email/password signup
│   ├── onboarding/
│   │   ├── create-org/page.tsx      New user: create org (caller becomes Admin)
│   │   └── join-org/page.tsx        New user: request to join existing org by slug
│   ├── org/
│   │   └── [orgId]/
│   │       ├── layout.tsx           Org context provider + sidebar nav
│   │       ├── dashboard/page.tsx   Fleet summary
│   │       ├── machines/page.tsx    Machine list grouped by namespace
│   │       ├── jobs/
│   │       │   ├── page.tsx         Job history + manual trigger
│   │       │   └── [jobId]/page.tsx Job detail + status timeline
│   │       ├── results/
│   │       │   └── [jobId]/page.tsx Findings: severity cards + expandable raw
│   │       └── settings/
│   │           ├── page.tsx         Org overview
│   │           ├── namespaces/page.tsx  Namespace CRUD (Admin)
│   │           ├── members/page.tsx     Member invite/approve/role (Admin)
│   │           └── api-keys/page.tsx    API key create/revoke (Admin)
│   └── api/
│       └── auth/[...all]/route.ts   Better Auth handler
├── components/
│   ├── ui/                          shadcn/ui primitives
│   ├── machines/
│   │   ├── MachineCard.tsx
│   │   └── MachineList.tsx
│   ├── jobs/
│   │   ├── JobTriggerButton.tsx
│   │   └── JobStatusTimeline.tsx
│   └── findings/
│       ├── FindingCard.tsx          Severity badge + AI summary + expand toggle
│       └── RawOutput.tsx            Collapsible Volatility3 JSON
├── lib/
│   ├── auth.ts                      Better Auth server instance
│   ├── auth-client.ts               Better Auth browser client
│   └── api.ts                       Typed fetch helpers for Fastify API
├── package.json
└── next.config.ts
```

---

## Auth setup

```typescript
// lib/auth.ts
import { betterAuth } from 'better-auth'
import { organization, apiKey, access } from 'better-auth/plugins'

export const auth = betterAuth({
  database: { url: process.env.DATABASE_URL! },
  secret: process.env.BETTER_AUTH_SECRET!,
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    }
  },
  plugins: [
    organization(),
    apiKey(),
    access({
      roles: {
        admin: ['org:manage', 'namespace:manage', 'apikey:manage', 'machine:trigger', 'schedule:edit', 'machine:view', 'results:view'],
        write: ['machine:trigger', 'schedule:edit', 'machine:view', 'results:view'],
        read:  ['machine:view', 'results:view'],
      }
    })
  ]
})
```

```typescript
// lib/auth-client.ts
import { createAuthClient } from 'better-auth/client'
import { organizationClient, apiKeyClient } from 'better-auth/client/plugins'

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  plugins: [organizationClient(), apiKeyClient()]
})
```

---

## Pages spec

### Login / Register
- Email + password form with validation
- "Continue with GitHub" button
- Error states: invalid credentials, email already taken, unverified email

### Onboarding (shown once after signup)
- Step 1: "Create an org" (name + slug) or "Join an existing org" (enter org slug → request pending)
- Redirect to dashboard after org creation
- Pending join request: show waiting screen with "You'll be notified when an admin approves"

### Dashboard (`/org/[orgId]/dashboard`)
- Summary cards: total machines, machines online (last_seen < 10 min), open CRITICAL findings, jobs last 24h
- Last 5 jobs table: machine, status, timestamp, link to results
- Machines by namespace: collapsible list, status dot per machine

### Machines (`/org/[orgId]/machines`)
- Grouped by namespace with namespace header + machine count
- Per machine: hostname, OS, arch, RAM, agent version, last seen, last job status
- "Trigger dump" button per machine (Write+ role) → opens job create modal
- "Deregister" button (Write+ role) → confirms, calls DELETE /machines/:id

### Jobs (`/org/[orgId]/jobs`)
- Filter by: all / running / done / failed / namespace / machine
- Status timeline per job: PENDING → DUMPING → UPLOADING → QUEUED → ANALYZING → DONE / FAILED
- "Trigger new job" button → machine picker → calls POST /jobs
- Poll job status every 5s while any job is in-flight

### Results (`/org/[orgId]/results/[jobId]`)
- Header: machine name, job timestamp, analysis duration, worker size
- Findings sorted by severity (CRITICAL first)
- Per finding card:
  - Severity badge (colour-coded: CRITICAL=red, HIGH=orange, MEDIUM=yellow, LOW=blue, INFO=grey)
  - Plugin name
  - AI summary (shown by default)
  - "Show raw output" toggle → JSON code block
- Empty state: "No findings — this machine looks clean"
- Failed job state: error code + message + "Trigger new job" button

### Settings — Namespaces
- List namespaces with machine count
- "New namespace" form: name + slug
- Delete namespace: disabled if machines exist (tooltip: "Deregister all machines first")

### Settings — Members
- Table: email, role, status (active / pending), invited by, joined at
- "Invite member" form: email → creates pending invitation
- Pending members: "Approve" button → sets status = active
- Role dropdown per member: admin / write / read
- "Remove" button with confirmation

### Settings — API Keys
- List: key prefix (first 8 chars), name, last used, created at
- "Create key" form: name → shows full key once in a copy modal ("Copy this key now — it won't be shown again")
- "Revoke" button with confirmation

---

## Data fetching pattern

Server Components fetch initial data. Client Components use SWR for polling:

```typescript
// jobs/page.tsx — poll in-flight jobs every 5s
const { data: jobs } = useSWR(
  `/api/jobs?orgId=${orgId}&status=running`,
  fetcher,
  { refreshInterval: 5000 }
)
```

All Fastify API calls go through `lib/api.ts` typed helpers. The session cookie is forwarded via `cookies()` in Server Components.

---

## Environment variables

```
DATABASE_URL              Shared with api/
BETTER_AUTH_SECRET        Shared with api/
GITHUB_CLIENT_ID          GitHub OAuth app client ID
GITHUB_CLIENT_SECRET      GitHub OAuth app client secret
NEXT_PUBLIC_API_URL       https://api.memoryinsight.io (Fastify base URL)
```

---

## Acceptance criteria (Phase 1)

- [ ] New user can sign up, create an org, create a namespace, generate an API key, and copy the install command — all without leaving the UI
- [ ] Pending join request shows correct waiting state; approved by admin → user lands on dashboard
- [ ] Machine appears in Machines page within 1 poll cycle after `meminsight-agent register`
- [ ] Manual job trigger → status timeline updates to DONE within the 15-min SLA
- [ ] FAILED job shows error code + human-readable message
- [ ] Results page: AI summary shown by default, raw JSON expandable on click
- [ ] CRITICAL finding has red severity badge
- [ ] Read-role user sees "Trigger dump" button as disabled (not hidden)
- [ ] API key shown once in copy modal, not retrievable afterwards
- [ ] Namespace delete blocked with tooltip when machines exist
