# Spec: [Feature Name]

> Copy this file to `docs/specs/[ISSUE-NUMBER]-[feature-name].md` and fill it in.
> The spec must be completed and linked in the GitHub Issue before implementation begins.

**Issue**: #[ISSUE-NUMBER]
**Status**: Draft | Review | Approved | Implemented
**Author**: [name]
**Date**: YYYY-MM-DD
**Services Affected**: [agent | api | worker | lambda | web — list all]

---

## Summary

[1-3 sentences describing what this feature does and why it exists.]

---

## Background and Motivation

[Why are we building this? What problem does it solve? What happens if we don't build it?
Include any relevant context from ARCHITECTURE.md or the relevant PRD.]

---

## Scope

### In Scope
- [Specific thing 1 that will be built]
- [Specific thing 2]

### Out of Scope
- [Thing that might seem related but is NOT part of this feature]
- [Explicitly list deferred work to prevent scope creep]

---

## Acceptance Criteria

> Each criterion must be verifiable. Write them as testable statements.

- [ ] Given [context], when [action], then [expected outcome]
- [ ] Given [context], when [action], then [expected outcome]
- [ ] [Edge case handled]
- [ ] [Error case handled]

---

## Technical Design

### Architecture Overview
[High-level description of the approach. Include a diagram if helpful (ASCII or Mermaid).]

```
[Optional: ASCII diagram of data flow or component interaction]
```

### API Changes (api/)

#### New Endpoints
```
POST /v1/[resource]
Auth: sessionAuth | apiKeyAuth
Request: { ... }
Response: { ... }
```

#### Modified Endpoints
[List any changes to existing endpoints]

### Database Changes (api/src/db/schema.ts)

#### New Tables
```typescript
export const [tableName] = pgTable('[table_name]', {
  id: text('id').primaryKey().$defaultFn(() => createId()),
  // ...
  createdAt: timestamp('created_at').notNull().defaultNow(),
})
```

#### Modified Tables
[Show schema diff — what columns are added/changed/removed]

### Queue Changes

| Queue | Change | Payload |
|---|---|---|
| Redis pub/sub | [new channel?] | `{ job_id }` |
| BullMQ | [new job type?] | `{ ... }` |
| SQS | [new message type?] | `{ ... }` |

### Drizzle Migration
```bash
cd api && pnpm drizzle-kit generate  # generates migration
# Migration file: api/drizzle/[timestamp]_[description].sql
```

---

## Security Considerations

- [ ] Auth middleware applied to all new routes (which: `sessionAuth` | `apiKeyAuth`)
- [ ] No tokens/secrets logged or returned in response bodies
- [ ] Worker subprocess isolation maintained if worker is affected
- [ ] Claude API key still fetched from SSM (not env vars) if worker is affected
- [ ] Rate limiting needed?
- [ ] Input validation with Zod on all request bodies

---

## Observability

- **Logs**: What should be logged? At what level? Which service?
- **Metrics**: Any new metrics to track?
- **Alerts**: Should any alert be created for failure conditions?

---

## Testing Plan

> Run `/plan-eng-review` before implementation begins. It generates a QA test plan.

**gstack test plan**: `~/.gstack/projects/{slug}/{user}-{branch}-test-plan-{date}.md`

### Unit Tests
- [ ] [Function/module that needs unit tests]
- [ ] [Error case coverage]

### Integration Tests
- [ ] [API endpoint integration test]
- [ ] [Queue consumer test]

### E2E Tests (if UI affected)
- [ ] [Playwright flow covering the critical path]

---

## Migration / Rollout Plan

- **Database migrations**: [yes/no — if yes, is it backward compatible?]
- **Breaking changes**: [yes/no — if yes, describe backward compatibility strategy]
- **Agent version impact**: [does this require a new agent binary? min version?]
- **Rollback plan**: How to revert if something goes wrong

---

## Dependencies on DAG

> Check DAG.md to confirm all blocking issues are resolved before starting.

- Blocked by issues: #[numbers]
- Blocks issues: #[numbers]

---

## Open Questions

| Question | Owner | Status |
|---|---|---|
| [Question about approach X] | [name] | Open |

---

## References

- Related ADR: [link if architectural decision was needed]
- Related PRD: `docs/prd-[service].md`
- Related issues: #[number]
- Architecture section: [section name in ARCHITECTURE.md]
