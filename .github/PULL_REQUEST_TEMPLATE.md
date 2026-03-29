## Summary

<!-- 1-3 sentences describing what this PR does. -->

## Related Issue

Closes #[ISSUE-NUMBER]

## Spec

**Spec**: `docs/specs/[ISSUE-NUMBER]-[feature-name].md` (N/A for bug fixes / dependency updates)

## ADR

**ADR**: `docs/adr/[NNNN]-[title].md` (N/A if no architectural decision)

---

## Checklist

### Before Submitting
- [ ] Spec file exists and is linked above (or N/A for bug fix / chore)
- [ ] ADR created and linked (if architectural decision made)
- [ ] All commits follow `type(scope): description` format (scope: agent|api|worker|lambda|web|infra|docs)
- [ ] Branch name follows `feat/[ISSUE-NUMBER]-[desc]` or `fix/[ISSUE-NUMBER]-[desc]`
- [ ] DAG.md dependencies for this issue are all merged

### Security
- [ ] All new Fastify routes protected by `sessionAuth` or `apiKeyAuth` (no unauthenticated endpoints without approval)
- [ ] No tokens, secrets, or API keys logged or returned in response bodies
- [ ] Claude API key not added to env vars — SSM Parameter Store only
- [ ] Better Auth used for all auth operations — no custom auth code

### Database (if schema changed)
- [ ] New tables/columns defined in `api/src/db/schema.ts` first
- [ ] Migration generated with `pnpm drizzle-kit generate` and committed
- [ ] `ON DELETE RESTRICT` on `machines.namespace_id` preserved
- [ ] No manual creation of Better Auth tables (user, session, account, organization, member, invitation, apikey)

### Worker (if worker/ or lambda/ changed)
- [ ] Volatility3 subprocess still uses `unshare --net`
- [ ] No network access added to the subprocess
- [ ] ISF symbol files not pulled at runtime — bundled in image
- [ ] `ephemeralStorageGiB` matches task definition if Dockerfile changed

### Tests
- [ ] Unit tests written for new domain functions
- [ ] Integration/E2E tests written for new API endpoints / queue handlers
- [ ] Test coverage not reduced

### Code Quality
- [ ] No `any` types (TypeScript), no bare `except` (Python), no `panic` (Go)
- [ ] No hardcoded secrets or credentials
- [ ] No `console.log` / `print()` / `fmt.Println` left in production code
- [ ] No TODOs — GitHub issues opened for follow-up work instead

### Service Changes
- [ ] `AGENTS.md` updated if service behavior or architecture changed
- [ ] `.env.example` updated if new environment variables added
- [ ] `ARCHITECTURE.md` updated if domain model changed (new entity, renamed field)
- [ ] `docker compose up` still works after changes

---

## Testing Instructions

<!-- How should reviewers test this? Step-by-step. -->

1.
2.

## Screenshots / Recordings

<!-- For UI changes. Optional. -->

## Notes for Reviewer

<!-- Anything the reviewer should pay special attention to, known limitations, or follow-up issues. -->
