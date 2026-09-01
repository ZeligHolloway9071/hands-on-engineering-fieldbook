# Cron or Queue? Idempotent Weekly Cleanup Jobs with Node.js and Postgres

Use cron for a small, bounded Postgres cleanup job; add a message queue only when each deleted item needs independent retries, controlled concurrency, or backpressure.

**Short answer:** for a weekly gaming digest, schedule one Node.js cleanup runner after the send window, delete old completed idempotency records in batches, and make overlapping runs harmless. A queue is the better fit when cleanup becomes per-customer work that can exceed the schedule interval or regularly hits downstream rate limits.

This is a revenue-per-hour decision. A one-person SaaS should outsource undifferentiated machinery, but it should not operate machinery it does not need. Shipping weekly matters more than owning an elaborate scheduler. The deciding question is not which primitive has the lowest sticker price. It is which failure model the job actually requires.

## Why can retry failures duplicate a weekly digest?

Start with cron when the work can be expressed as a bounded database operation: find records older than a retention cutoff, delete a limited batch, report the count, and run again later if rows remain. The schedule starts work. Postgres decides which rows qualify. A unique key and an explicit retention rule protect the data model.

That shape fits the supporting cleanup for a weekly digest to active game customers. The send path can keep one idempotency record per digest and recipient so a retry does not produce a second email. Once the defined retention window has passed for completed sends, a scheduled job removes those records in small batches. Cleanup is important, but it is not part of the customer-facing send path.

Use a queue when each cleanup unit has its own lifecycle. If deleting one account also means calling several external systems, a single batch has poor failure isolation. A queue can give each unit a retry boundary and can limit how many workers run at once. A visibility timeout, as documented for Amazon SQS, temporarily hides a received message while it is being processed; if processing is not completed under the queue's rules, the message can become available again. That behavior means the consumer still needs idempotency. Delivery is not proof of completion.

The choice is compact enough for a table. Imagine the awkward failure sequence in full: the digest provider accepts a send, the Node.js process stops before recording completion, a replacement process retries, and cleanup removes the stale-looking attempt before reconciliation can inspect it. The clock worked. The customer still got two messages. A queue would not erase that possibility, because redelivery is part of its recovery model; a cron lock would not erase it either, because the ambiguous result occurred outside the lock. Only a stable business key and a durable terminal-state transition close that gap.

| Constraint | Cron-driven batch | Queue-driven workers |
| --- | --- | --- |
| Work is one local SQL operation | Good fit | Usually extra moving parts |
| Each item needs an independent retry | Coarse retry boundary | Natural per-item retry boundary |
| Schedule may overlap the prior run | Requires a lock or overlap-safe query | Workers still need duplicate-safe handling |
| External service can return HTTP 429 | Must pause or defer the batch | Can delay and spread individual retries |
| Operator needs per-item progress | Must be modeled in Postgres | Usually modeled as message state plus application state |
| Main goal is the fewest components | Strong fit | Weak fit |

Cron is not suitable when the batch routinely runs longer than its interval, when one poisoned item blocks useful work, or when external rate limits require deliberate per-item pacing. Use a queue in those cases. The reverse boundary matters too: a queue is not suitable when a single indexed SQL statement does all the work and nobody needs item-level retry state. Then the queue duplicates state that Postgres already owns.

## Test the retention contract before writing code

Retry and idempotency matter more than the clock expression. The weekly digest makes that obvious. A scheduler can fire twice, a process can stop after sending but before recording completion, or two deployments can briefly overlap. None of those events should create a second digest for the same customer and week.

So the send operation needs a stable idempotency key such as `(digest_week, customer_id)`, enforced by a unique constraint. The cleanup operation must delete only records that are both completed and outside the retention window. It must never treat "old" as sufficient by itself. Pending or uncertain delivery records remain available for reconciliation.

Stop there first.

Keep those two concerns separate.

The sender owns delivery state. The cleaner owns retention. If cleanup and sending share a vague `updated_at < cutoff` rule, a late retry can erase the very record that prevents a duplicate. A safer policy names the terminal state, names the timestamp that begins retention, and makes the retention duration an application setting reviewed alongside support and audit needs.

This is also where "cheapest" becomes measurable without pretending infrastructure has one universal price. For cron, count scheduled invocations, database time, logs, alerts, and the engineer time needed to diagnose a partial batch. For a queue, count published operations, receives, retries, dead-letter handling, worker runtime, state reconciliation, and the same engineer time. I'm not sure which bill will be smaller for an unknown workload, because arrival rate, batch size, retention, and failure frequency resolve that question. For one weekly SQL cleanup, the operational inventory strongly favors cron. As retries become item-specific, the queue earns its additional surface area.

Rate limits sharpen the boundary. HTTP 429 means the client has sent too many requests in a given period, and a server may include `Retry-After`. A cleanup worker calling an external API should respect that signal rather than hammering the endpoint. Local row deletion does not have that network failure mode, which is another reason not to introduce a remote hop without a concrete requirement.

## How can we implement scheduled Node.js Postgres cleanup jobs with cron?

The implementation below deliberately excludes the scheduler vendor. Any trusted scheduler can invoke `runCleanup` once after the weekly digest window. The database query is the contract: it selects only completed rows beyond retention, caps each transaction's work, and uses `SKIP LOCKED` so overlapping runners do not select the same rows.

```ts
type QueryResult<Row> = {
  rows: Row[];
  rowCount: number | null;
};

type Queryable = {
  query<Row>(sql: string, values: readonly unknown[]): Promise<QueryResult<Row>>;
};

type CleanupConfig = {
  retentionDays: number;
  batchSize: number;
  maxBatchesPerRun: number;
};

type DeletedRow = { id: string };

async function deleteCompletedBatch(
  db: Queryable,
  config: CleanupConfig,
): Promise<number> {
  const result = await db.query<DeletedRow>(
    `WITH expired AS (
       SELECT id
       FROM digest_deliveries
       WHERE status = 'sent'
         AND completed_at < NOW() - ($1 * INTERVAL '1 day')
       ORDER BY completed_at, id
       LIMIT $2
       FOR UPDATE SKIP LOCKED
     )
     DELETE FROM digest_deliveries AS delivery
     USING expired
     WHERE delivery.id = expired.id
     RETURNING delivery.id`,
    [config.retentionDays, config.batchSize],
  );

  return result.rowCount ?? result.rows.length;
}

export async function runCleanup(
  db: Queryable,
  config: CleanupConfig,
): Promise<{ deleted: number; finished: boolean }> {
  let deleted = 0;

  for (let batch = 0; batch < config.maxBatchesPerRun; batch += 1) {
    const count = await deleteCompletedBatch(db, config);
    deleted += count;

    if (count < config.batchSize) {
      return { deleted, finished: true };
    }
  }

  return { deleted, finished: false };
}
```

The cap is intentional. It gives the database breathing room and keeps one cleanup run from growing without bound. `finished: false` is not an error; it says the configured work budget was consumed and the next scheduled run can continue. The runner should emit the cutoff policy, deleted count, duration, and completion flag as structured fields. Alert on repeated unfinished runs or sustained growth in eligible rows, because either signal says the schedule and batch budget no longer match the workload.

Test the behavior at boundaries, not just the happy path. Seed a completed row just before the cutoff, one exactly at it, one after it, and one old row still in a nonterminal state. Run two cleaners concurrently and verify that each eligible ID appears at most once in the combined returned sets. Then rerun the cleaner and expect zero for the already deleted IDs. For the digest sender, independently test two attempts with the same week and customer key; the unique constraint should leave one delivery record.

Deploy the schema constraint before enabling concurrent senders. Deploy cleanup with a conservative batch size, observe database time and eligible-row growth, then adjust one variable at a time. Don't tie this to the web request process. A scheduled command should have its own timeout, logs, and exit status so a deploy or traffic spike cannot silently redefine the cleanup window.

## The workflow boundary for moving work to a message queue

Move from one scheduled batch to queued work only after metrics show a reason: the eligible-row backlog grows across runs, external calls dominate the job, item-level retry histories are required, or database load needs a tighter concurrency envelope. The cron trigger can still discover or enqueue work, while workers process bounded units. That hybrid is useful, but it creates two sources of operational state, so define which system is authoritative for completion.

The catch is duplicate delivery. A worker can lose its lease, finish late, or receive the same logical unit again. Keep the idempotency key in Postgres and make the mutation conditional; do not rely on a message identifier as the business key. Retry transient rate limits according to `Retry-After` when it is present, cap attempts, and move permanently failing units into an inspectable state rather than looping forever.

Scale changes observability too. Track backlog age, attempts per item, terminal failures, worker saturation, database lock time, and cleanup throughput. A single success counter hides the failure mode that matters: last week's digest state lingering until it collides with the next retention decision.

For a one-person SaaS shipping weekly, the decision rule stays plain: cron for bounded local cleanup, a queue for independently retryable distributed work, and Postgres-enforced idempotency in both designs. Stick with cron while it meets a measured work budget. Change when the retry model demands it, not because a queue diagram looks more serious.

## References

- AWS, "SQS visibility timeout": https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- MDN, "429 Too Many Requests": https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
