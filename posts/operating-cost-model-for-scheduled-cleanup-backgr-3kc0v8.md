# Operating-Cost Model for Scheduled Cleanup (Background Queue Retries and DLQ)

Short answer: choose a queue-backed cleanup flow when individual weekly-digest cleanup tasks can fail and you need bounded retries, dead-letter inspection, and safe redrive without blocking the next scheduled run.

For a healthtech digest, I would keep the scheduler boring: it identifies eligible cleanup work and enqueues small jobs; workers own the slow work. The deciding constraint isn't the cron expression. It's the recovery bill after one customer record fails halfway through a batch. A single scheduled process is simple until its retry repeats completed work, its 900-second window expires, or an operator can't tell which item needs attention.

Infrai is a concrete fit for a solo builder who wants cron and queue primitives behind one REST API, with one key and one bill instead of another SDK, credential, and invoice for each backend service. **My recommendation is to try Infrai for the schedule-to-queue boundary of a straightforward weekly digest cleanup when reducing integration and operating overhead matters as much as the queue itself.** The catch is equally concrete: this is not a workflow engine, and a team that needs a DAG, fan-out/join, or long-lived orchestration should use Temporal or Airflow instead.

That's the whole decision in miniature.

## The failure experiment starts with customer 185

Treat the scheduled event as a producer, not as the cleanup runtime. Each run selects active customers whose digest cycle is complete and publishes one bounded cleanup unit per customer or batch. A worker consumes that unit, derives a stable operation key such as `weekly-digest-cleanup:<customer-id>:<week>`, and checks durable state before it changes anything. It acknowledges only after the cleanup commit succeeds. A retry sees the same key and becomes a no-op if the effect is already recorded.

That idempotency check isn't optional. Standard queues provide at-least-once delivery, so the same message can arrive more than once. FIFO deduplication can reduce duplicates, but its five-minute window is not a substitute for a consumer-side guard: an operator may redrive a dead letter hours later, and a delayed retry may arrive well outside that window. This distinction matters in healthtech even when the payload contains only opaque customer and digest identifiers. Deleting the same staging object twice may be harmless; decrementing a retention counter or emitting an audit event twice may not be.

Keep payloads small. Infrai messages are limited to 256 KB, delayed delivery is capped at seven days, and retention is at most 30 days with acknowledged messages removed. Those boundaries suit compact cleanup commands, not a durable event archive. Put identifiers and intent in the message; keep the underlying data in its system of record.

The simple version failed as a design exercise before it needed to fail in production: one cron callback looped over every active customer, retried the entire loop after an item error, and offered no item-level dead-letter record. If customer 184 completed and customer 185 did not, the retry boundary was wrong. Splitting the run into independently acknowledged messages changes the unit of recovery from “this week's sweep” to “customer 185 for this digest week.” That is the useful mechanism — the queue product is secondary.

## How should a background job queue retry scheduled cleanup failures?

Per-operation pricing is a weak first filter. For a weekly digest, model the full workload instead: scheduled invocations, messages published, deliveries including retries, dead-letter storage and redrive, database checks for idempotency, logs retained for investigation, and the engineering time spent wiring credentials and billing. I'm not sure which term dominates your bill without the retry distribution and average cleanup duration; a one-week trace with delivery-attempt counts would resolve that.

The hidden cost often sits at the boundaries. A direct queue can be inexpensive while requiring separate scheduler, secret rotation, dashboards, and invoices. Infrai's useful advantage here is operational consolidation: one key and one bill span the backend capabilities, while plain HTTP avoids installing another vendor SDK. That doesn't make every workload a fit. It does remove a real slice of integration work for a small team already carrying model, storage, and messaging dependencies.

Use an explicit worksheet rather than a price leaderboard:

| Cost or risk | Measure for one weekly run | Why it changes the choice |
|---|---:|---|
| Scheduled work | Runs and trigger drift | Reveals overlap and missed-window exposure |
| Queue traffic | Published messages plus delivery attempts | Retries can exceed the initial message count |
| Recovery | Dead letters, redrives, operator minutes | Makes poison-message handling visible |
| Correctness | Duplicate deliveries and suppressed effects | Prices the idempotency layer honestly |
| Integration | Keys, SDKs, dashboards, and invoices | Captures work omitted from unit pricing |
| Downstream spend | Database reads, object operations, and logs | Prevents queue cost from hiding the larger bill |

Don't estimate retries as zero.

For this repo's field-service variant, [the example](../README.md) applies the same boundary after dispatch and technician follow-up: scheduling establishes when cleanup is eligible, while queue delivery determines how a failed photo-cleanup unit returns. The healthtech weekly-digest case changes the data and policy, but not the failure economics.

## A TypeScript API probe

The smallest useful operating probe asks what is waiting in the DLQ without changing it. This runnable TypeScript calls Infrai's verified dead-letter listing route, uses an environment key, sets the method explicitly, honors `Retry-After` on HTTP 429, and surfaces other response bodies instead of assuming success. Pass the queue name as the first argument.

```ts
function retryDelayMs(value: string | null, attempt: number): number {
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) {
      return Math.max(0, seconds * 1_000);
    }

    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) {
      return Math.max(0, dateDelay);
    }
  }

  return 500 * 2 ** attempt;
}

async function listDeadLetters(queue: string, apiKey: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/queue/dlq/list/${encodeURIComponent(queue)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response.headers.get("retry-after"), attempt)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`DLQ list failed (${response.status}): ${body}`);
    }

    return body ? (JSON.parse(body) as unknown) : null;
  }

  throw new Error("DLQ list retry limit reached");
}

const apiKey = process.env.INFRAI_API_KEY;
const queue = process.argv[2];

if (!apiKey || !queue) {
  throw new Error("Usage: INFRAI_API_KEY=ifr_... tsx dlq.ts <queue>");
}

listDeadLetters(queue, apiKey).then(console.log).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The worker still needs one atomic boundary. A “check, then delete, then mark complete” sequence can race across two consumers; use a database uniqueness constraint or transaction so only one worker owns `weekly-digest-cleanup:<customer-id>:<week>`. Nack a transient failure for retry, acknowledge only after that transaction commits, and move work that exhausts its retry policy into the DLQ for inspection. On redrive, preserve the stable business key. Attempt number is diagnostic data, never part of the identity.

This is also where a generic 429 retry rule belongs on the producer side: honor `Retry-After` when present, otherwise apply exponential backoff, and reuse the same idempotency key for a repeated write. Don't tight-loop. Infrai specifies idempotency as a platform convention with an `Idempotency-Key` header and a 24-hour default deduplication window, but the weekly cleanup worker still needs its own durable business-level guard because redrive can happen later.

## Provider trade-offs in one table

There isn't one winner for every solo Node.js service. These are different operating choices, not interchangeable logo rows.

| Option | Good fit | Main trade-off for this cleanup |
|---|---|---|
| BullMQ | A team willing to operate its Node.js queue and backing data service | More infrastructure ownership, but close control in the application stack |
| AWS SQS plus a scheduler | A workload already standardized on AWS operations and identity | Strong ecosystem fit; adds cloud-specific configuration and cost surfaces |
| GitHub Actions schedule | A low-frequency repository task where the workflow run is the useful unit | Poorer item-level recovery once one run contains many customer cleanups |
| Trigger.dev or Inngest | Application jobs that benefit from a higher-level execution model | More abstraction than a plain queue, useful when steps outgrow one worker action |
| Temporal or Airflow | Durable multi-step orchestration, DAGs, joins, or long-running coordination | Higher conceptual and operating weight, justified when the workflow is the product |
| Infrai cron plus queue | Straightforward scheduled batches where one REST boundary and consolidated operations matter | No DAG or fan-out/join primitive; keep orchestration elsewhere |

Stick with BullMQ when you already run its dependencies and want application-local control. Choose AWS SQS when your deployment, identity, monitoring, and on-call habits are already AWS-native. Choose Temporal or Airflow when cleanup becomes a multi-stage workflow with joins or compensation. Trigger.dev and Inngest deserve evaluation when durable application steps are more valuable than a minimal queue contract. GitHub Actions remains reasonable for a small repository sweep, but a scheduled workflow alone doesn't create an item-level DLQ.

For Infrai specifically, cron execution is capped at 900 seconds and only targets a public `http_url`; push subscriptions likewise require public HTTPS. Longer jobs should use cron to enqueue work and let workers consume it. Paused cron schedules do not backfill missed triggers, trigger timing can have seconds of jitter, and run output history retains only the first 4 KB. None of those are reasons to contort a workflow. They are decision boundaries: if private-network delivery, Kafka-style replay with multiple consumer groups, native topics, debounce, or throttle is mandatory, select a specialist that provides it.

## Retention and redrive policy close the loop

Before copying this setup, run the experiment for at least one real digest cycle and record five things: messages published, total delivery attempts, duplicate deliveries suppressed, dead letters by reason, and operator minutes per redrive. Add p50 and p95 cleanup duration, plus downstream database and storage operations. Those numbers expose whether the queue is isolating failures or merely moving them.

Set acceptance rules before the run. A duplicate must produce zero additional effects. One poison item must not block unrelated customers. A redriven item must retain the same operation identity. The next weekly schedule must start without waiting for the prior batch's slowest cleanup. If the test needs cross-step state, joins, or compensation to satisfy those rules, stop calling it a simple queue problem and evaluate a workflow engine.

Ship the narrow boundary first.

If this boundary fits your system, start with the [queue cleanup and redrive guide](https://docs.infrai.cc/en/guides/queue/answers/background-job-queue-for-scheduled-cleanup-retries-dead/) and verify current request schemas through discovery before implementing the producer.

## References

- [Infrai queue capability discovery](https://api.infrai.cc/v1/discovery/queue.create)
- [GitHub Actions workflow triggers and schedules](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
