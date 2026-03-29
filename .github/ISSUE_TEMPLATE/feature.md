---
name: Feature / Implementation
about: A new feature or implementation task (linked to DAG.md)
title: '[FEAT] '
labels: ['feature']
---

## Summary

<!-- 1-3 sentences describing what this implements and why. -->

## DAG Reference

<!-- Issue number from DAG.md. List dependencies. -->

**DAG Issue**: #
**Blocked by**: #
**Blocks**: #

## Service(s) Affected

<!-- Check all that apply -->
- [ ] agent/ (Go daemon)
- [ ] api/ (Fastify)
- [ ] worker/ (Python Volatility3)
- [ ] lambda/ (Python dispatcher)
- [ ] web/ (Next.js UI)
- [ ] infra/ (docker-compose, AWS)
- [ ] docs/

## Acceptance Criteria

<!-- Verifiable, testable statements. All must be met before the PR is merged. -->

- [ ] Given [context], when [action], then [outcome]
- [ ] Given [context], when [action], then [outcome]

## Technical Notes

<!-- Architecture constraints, ADRs to reference, known gotchas. -->

- Read: `docs/prd-[service].md`
- Read: `docs/adr/[relevant ADR]`
- Relevant section in `ARCHITECTURE.md`:

## Spec

<!-- Link to spec file once created. -->

**Spec**: `docs/specs/[ISSUE-NUMBER]-[feature-name].md`
