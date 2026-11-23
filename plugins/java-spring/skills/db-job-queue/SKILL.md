---
name: db-job-queue
description: Database-backed job queues on PostgreSQL in Spring — status columns, claiming with FOR UPDATE SKIP LOCKED, stale-row recovery, release limits, @Scheduled fixed-delay workers and Spring Integration pollers, and transaction boundaries that survive failures. Use when adding or changing a queue table, a worker or poller that processes rows, or retry/recovery behaviour. Never let an exception escape a @Transactional call inside a transaction you mean to commit.
---

# Database Job Queues

A table is the queue: producers insert rows, workers claim and process them. It is durable,
transactional with the business write that enqueues it, and needs no broker. These rules keep it
correct with more than one worker and after crashes.

## 1. Table shape

```sql
CREATE TABLE invoice_delivery_queue (
    id                     BIGSERIAL PRIMARY KEY,
    invoice_id             BIGINT      NOT NULL REFERENCES invoice (id),
    status                 VARCHAR(32) NOT NULL DEFAULT 'PENDING',
    attempts               INT         NOT NULL DEFAULT 0,
    infra_release_count    INT         NOT NULL DEFAULT 0,
    error_message          TEXT,
    due_at                 TIMESTAMP   NOT NULL,
    processing_started_at  TIMESTAMP,
    created_at             TIMESTAMP   NOT NULL
);
CREATE INDEX idx_invoice_delivery_queue_claim ON invoice_delivery_queue (status, due_at, id);
```

- Statuses: `PENDING → PROCESSING → COMPLETED | FAILED`. Add stage-specific statuses only when a
  worker must resume mid-way. Keep them in a Java `enum` stored as `@Enumerated(EnumType.STRING)`.
- `processing_started_at` is what stale recovery measures. Set it on claim, clear it on release.
- Index the claim predicate and its `ORDER BY`.

## 2. Claiming

Claim in its own short transaction, so the lock is released before slow work (HTTP, browser, SMTP)
begins:

```sql
UPDATE invoice_delivery_queue
SET status = 'PROCESSING', processing_started_at = :now, attempts = attempts + 1
WHERE id IN (
  SELECT id FROM invoice_delivery_queue
  WHERE status = 'PENDING' AND due_at <= :now
  ORDER BY due_at, id
  LIMIT :size
  FOR UPDATE SKIP LOCKED)
RETURNING id, invoice_id
```

- `SKIP LOCKED` is what makes two workers take disjoint rows. Prove it with a two-connection
  integration test.
- When the claim joins other tables, lock only the queue row: `FOR UPDATE OF q SKIP LOCKED`.
  Otherwise the worker blocks the business table (a user withdrawing a request) for the whole claim.
- Filter out rows whose owner is already final (withdrawn, rejected, approved) **in the claim
  query**, so they are never picked up, rather than claiming and discarding.
- `RETURNING` does not preserve order; sort in Java if order matters.
- Offer a cheap `existsClaimable()` (`SELECT EXISTS (...)`) so a worker can skip expensive setup
  (starting a browser, logging in) when there is nothing to do.

## 3. Outcomes

Distinguish **why** a row did not complete, because each needs a different response:

| Outcome | Meaning | Action |
| --- | --- | --- |
| Completed | work done | `COMPLETED` |
| Business failure | the work itself failed (bad data, not found) | `FAILED` with a reason; no automatic retry |
| Infrastructure loss | worker lost its browser, session, connection | back to `PENDING`, increment `infra_release_count`; on the *N*-th release in a row, `FAILED` |
| Abandoned | worker died mid-row | stale recovery (section 4) |

```java
@Transactional
public QueueRow release(QueueRow row, int releaseLimit) {
    var releases = row.getInfraReleaseCount() + 1;
    if (releases >= releaseLimit) {
        log.warn("Queue row id={} lost its worker {} times - failing it", row.getId(), releases);
        row.markFailed(RELEASE_LIMIT_ERROR.formatted(releases));
    } else {
        row.markReleased();
    }
    return repository.save(row);
}
```

The release limit is what stops a poison row from crashing the worker forever. Return a small
`enum` outcome (`CheckOutcome.COMPLETED | FAILED | RELEASED`) from the processing method instead of
using exceptions for control flow.

Retries of business failures are a human or scheduled decision (a Retry button, a reset by an
approver), not a loop in the worker. Record every reset in an audit table.

## 4. Stale recovery

At the **start** of each batch, on the worker's own thread, put back rows in progress longer than
`staleTimeout`:

```java
var threshold = LocalDateTime.now(clock).minus(staleTimeout);
var stale = repository.findAll(queue.status.in(IN_PROGRESS).and(queue.processingStartedAt.lt(threshold)));
stale.forEach(row -> release(row, releaseLimit));
```

Running it at batch start on a single scheduler thread guarantees it can never see a row that is
still being worked on. Recovery counts as a release, so it is bounded by the same limit.

## 5. Workers

**Scheduled batch** — for work with expensive setup shared across rows:

```java
@Scheduled(fixedDelayString = "${app.queue.batch.fixed-delay}")
public void runBatch() {
    recoverStale();
    if (!queueService.hasClaimableRow()) {
        return;
    }
    try (var session = startSession()) {          // setup failure: touch no row, back off
        for (var i = 0; i < batch.getSize(); i++) {
            var row = queueService.claimNext();
            if (row.isEmpty()) return;
            if (process(row.get(), session) == Outcome.RELEASED) {
                queueService.release(row.get(), batch.getReleaseLimit());
                return;                              // session is gone; end the batch
            }
        }
    }
}
```

- `fixedDelay`, not `fixedRate`: the interval runs from the end of the previous batch, so batches
  never overlap. Keep the default single scheduler thread.
- If setup fails, no row is claimed. Record the failure (stage, message, screenshot) in a health
  table and back off exponentially up to a cap; alert on N consecutive failures, once per outage.

**Spring Integration poller** — for steady per-row processing:

```java
@Component
public class InvoiceDeliverySource extends AbstractMessageSource<InvoiceDeliveryQueue> {

    @Override
    @Transactional
    protected @Nullable Object doReceive() {
        return queueService.claimNext().orElse(null);     // null = nothing this poll
    }

    @Override
    public String getComponentType() {
        return "postgres:invoice-delivery-poller";
    }
}
```

Configure `PollerMetadata` with a `PeriodicTrigger` (`setFixedRate(false)`) and
`maxMessagesPerPoll` from `@ConfigurationProperties`, feeding a `DirectChannel` consumed by a
`@ServiceActivator` that marks the row `COMPLETED` or `FAILED` in a `finally`.

## 6. Transactions — the rollback-only trap

When a method is `@Transactional` and calls another `@Transactional` bean method that throws, the
**shared** transaction is marked rollback-only — even if the caller catches the exception. The
commit then fails and undoes everything, including the status change that should have recorded
the failure.

- Decide per method what must commit together (the business result + the row's status) and put
  exactly that in one transaction.
- Inside that transaction, call methods that **return** an outcome rather than throw.
- Record failures in a separate transaction (`REQUIRES_NEW`, or after the main one ends), or let
  the whole thing roll back on purpose so the row stays claimable and the next poll retries it.
  Write a comment saying which you chose.
- `@Transactional` on a `protected` method works with CGLIB proxies (Spring 6+), but not on
  `private` methods or self-calls.

## 7. Time and housekeeping

- Use an injected `Clock` (`LocalDateTime.now(clock)`) for `due_at`, claims and stale thresholds,
  so tests can pin time — see `spring-boot-configuration`.
- Delete or archive `COMPLETED` rows past a retention period in a scheduled job.

## Checklist

- [ ] Claim uses `FOR UPDATE [OF q] SKIP LOCKED` in a short transaction, ordered, limited
- [ ] Final-state owners are excluded in the claim query
- [ ] Business failure, infrastructure release and abandonment are handled differently
- [ ] Release count is bounded; stale recovery runs at batch start on the worker thread
- [ ] No exception escapes a `@Transactional` call inside a transaction that must commit
- [ ] Concurrency and claim SQL covered by an integration test against real PostgreSQL
