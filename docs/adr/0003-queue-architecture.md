# 0003 Queue Architecture: Redis pub/sub + BullMQ + SQS Separation

**Date**: 2026-03-29
**Status**: Accepted
**Deciders**: Vivek Agarwal
**Issue**: N/A (established during architecture design)

## Context

The platform has three distinct async coordination needs:

1. **Agent wakeup**: The agent daemon long-polls `GET /jobs/pending`. When a new job is created, the API must wake the specific agent holding the pending HTTP connection. This is pure pub/sub — no persistence needed, just a signal.

2. **Scheduled scans**: Users configure recurring scan schedules (e.g., "every 6 hours"). This requires a durable, repeating job scheduler with a backing store.

3. **Fargate dispatch**: When an agent completes a memory dump upload, the API must trigger a Volatility3 analysis container on AWS Fargate. This involves ECS task launching and requires durable, at-least-once delivery with retries.

## Decision

We will use **three separate queue mechanisms**, each serving exactly one purpose:

| Queue | Purpose | Implementation |
|---|---|---|
| Redis pub/sub | Wake long-poll connections | `PUBLISH machine:{id}:jobs {job_id}` |
| BullMQ | Repeating scheduled jobs | Creates `jobs` record, publishes to Redis |
| SQS | Trigger Fargate via Lambda | Published in `POST /jobs/:id/complete` handler |

**The BullMQ and SQS paths are completely independent.** A scheduled job creates a `jobs` record and publishes to Redis to wake the agent. The SQS path is only triggered when the agent calls `POST /jobs/:id/complete` after uploading the dump.

## Consequences

### Positive
- Each mechanism is optimized for its specific use case
- Redis pub/sub is low-latency for agent wakeup — no polling delay
- BullMQ provides repeating job semantics with Redis persistence (survives API restarts)
- SQS provides durable at-least-once delivery for Fargate dispatch with visibility timeout retries
- Clear separation prevents accidental coupling between scheduled jobs and analysis dispatch

### Negative
- Three queue systems to operate and monitor
- Redis is a shared dependency for both pub/sub and BullMQ
- More complex local development setup (Redis + SQS local emulation)

### Neutral
- SQS triggers Lambda which starts Fargate — this is the standard AWS serverless dispatch pattern
- Lambda acts as a thin dispatcher only — no analysis logic

## Alternatives Considered

### Option A: BullMQ for everything
Rejected: BullMQ doesn't naturally bridge to Fargate dispatch. Using it for agent wakeup adds unnecessary persistence and polling overhead for what is fundamentally a fire-and-forget pub/sub signal.

### Option B: SQS for everything
Rejected: SQS for agent long-poll wakeup would require the agent to poll SQS directly, adding AWS credentials to every agent machine and significant complexity. Redis pub/sub is far simpler for this use case.

### Option C: Polling instead of long-poll
Rejected: polling intervals create latency between job creation and agent pickup. Long-poll with pub/sub wakeup gives near-instant job delivery without the API hammering from repeated short-interval polls.
