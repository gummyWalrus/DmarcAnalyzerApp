# ADR 0002: Ingestion and Worker Execution Strategy

- Status: accepted, partially superseded by ADR 0008
- Date: 2026-03-31
- Superseded in part by ADR 0008: the database-backed job queue and the
  Kubernetes CronJob were never built. Same image, worker mode, the 60-minute
  polling cadence and checkpointed backfill still hold.

## Context

DMARC ingestion includes periodic mailbox polling, retries, backfill, and long-running tasks (exports, digests, alerts). We need reliability without heavy platform dependencies.

## Decision

- ~~Use a lightweight database-backed job queue.~~ **Superseded.** There is no job
  entity or table. `QueueWorkerService` reads every active polled source each
  pass and iterates in-process; ADR 0008 records that there is no claim path.
- Use a hosted worker mode in the same application image.
- Polling cadence is global every 60 minutes, running 24/7.
- Backfill is unlimited, processed oldest-to-newest with checkpoints.
- ~~Kubernetes runs worker mode via CronJob-triggered executions.~~ **Superseded.**
  The Helm worker is a `Deployment` pinned to one replica, and
  `WorkerSingleInstanceLock` enforces that limit in the database (ADR 0008).

## Consequences

### Positive

- Reliable retries and resumability with minimal external components.
- Same code/image for API and worker simplifies delivery.
- ~~K8s CronJob aligns with periodic schedule and resource control.~~ Not built.

### Negative

- ~~DB queue requires careful locking semantics and index tuning.~~ Not built, so
  the cost never arose; the cost of having no queue is that only one worker may
  run per database (ADR 0008).
- ~~Cron-triggered workers may add slight delay versus always-on worker.~~ Not
  built. The worker is always on.

### Follow-up

- ~~Define job schema, locking, retry, and dead-letter policies.~~ Not built.
  Retry is in-process exponential backoff (`Worker__MaxRetryAttempts`,
  `Worker__RetryBaseDelaySeconds`); there is no dead-letter state.
- ~~Add operational views for queue depth and failed jobs.~~ Not built. Sync
  outcomes are visible through `GET /api/v1/mailbox-sync-runs` and
  `GET /api/v1/mailbox-health`.
