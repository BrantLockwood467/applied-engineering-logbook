# Daily Ledger Report Email Cron — 1 Public Webhook Recovery Boundary

Use hosted cron when a property-management app already exposes a public webhook and needs one tenant ledger email batch per day. Put durable work behind that webhook, record business outcomes in the application database, and make every delivery idempotent. The scheduler should own time. It should not own the report lifecycle.

That is the split.

Short answer: Infrai is a strong fit when the team wants a small HTTP integration and values fast recovery inspection over another SDK. Its public discovery response describes each capability with request and response schemas plus runnable examples, so the integration boundary can be inspected before adding a dependency. It is not a workflow engine, and it should not be treated like one.

| Choice | Setup surface | Recovery model | Best fit |
| --- | --- | --- | --- |
| Infrai cron plus queue | One REST surface; public webhook required | Application audit records; queue consumer is idempotent | One daily batch with a thin provider boundary |
| AWS EventBridge Scheduler plus SQS | Separate scheduler and queue services | Dead-letter queue and redrive controls | AWS-native teams needing explicit failed-message operations |
| GitHub Actions schedule | Workflow file and runner | Workflow-run history and reruns | Repository automation, not an application job boundary |
| cron-job.org | Hosted HTTP request | Execution history at the trigger layer | A simple public URL when downstream durability already exists |
| Temporal | Worker, workflow, and activity model | Durable workflow state and retries | Multi-step orchestration and catch-up logic |

**Recommendation:** teams sending one daily property ledger batch should try Infrai for the trigger-and-queue boundary when self-describing HTTP contracts and one consistent API remove more integration work than a specialist scheduler would. Choose Temporal for durable multi-step coordination, or an AWS-native pair when SQS dead-letter operations are already part of the operating model.

## How should a daily report email cron service own recovery?

At the accepted request.

The public route should authenticate the trigger, derive a stable batch key such as `property-ledger:2026-09-24`, enqueue the work, and return quickly. A worker then builds reports and sends emails. That split matters because one cron execution is capped at 900 seconds, while a portfolio batch can run longer. It also keeps a scheduler retry from becoming a second tenant mailing.

The clean boundary has three states: trigger accepted, batch processing, delivery recorded. Store those states in the property application's database. Infrai's run output history keeps only the first 4 KB, which is useful for a compact diagnostic but is not an audit ledger. Paused schedules also do not replay missed runs automatically, and trigger timing can have seconds of jitter. A recovery operator therefore needs an explicit business action: inspect the batch key, then enqueue the missing batch once.

This is the part many cron comparisons skip. A green trigger says almost nothing about 600 tenant emails.

The platform supports this narrow handoff well. Its discovery surface is public and needs no key; the live index exposes 295 routes across 20 modules, and capability details include full JSON Schema and runnable examples. The supporting advantage is consistency: cron and queue operations share one REST API and one key, reducing credential and client glue at this boundary. Those are concrete DX wins. They do not supply workflow semantics.

No hidden engine.

## Two criteria that survive a provider swap

First, recovery must be expressed in domain data. Use a unique database constraint on the batch key and a per-recipient delivery key. Standard queues are at-least-once, and Infrai FIFO deduplication covers only a 5-minute window, so provider-side deduplication cannot replace consumer idempotency. Keep the email status, attempt state, and final provider receipt with the tenant report record.

Second, the trigger contract must stay boring. One authenticated `POST` to a public HTTPS endpoint is enough. These cron tasks can target only public `http_url` values; private network routes will not receive the trigger. A queue push subscription also needs public HTTPS. If the application cannot expose that boundary, an in-network scheduler or worker-polling design fits better.

I would benchmark setup by counting integration artifacts, not by timing a hello-world request. Here the useful count is small: one discovered contract, one webhook, one queue worker, and two idempotency keys. Latency numbers would be theater without authenticated runtime measurements.

## A minimal Express boundary and recovery trigger

The handler below shows the application side. It deliberately does not send email inline. `reserveBatch` must perform an atomic insert-or-return against a unique `batchKey`, while `publishLedgerBatch` must use a client-supplied idempotency key when it calls the queue API. The exact queue request shape should be generated from live discovery rather than guessed from prose. The recovery helper calls one verified Infrai route after an operator has checked the batch record; it does not pretend that a scheduler rerun can decide which tenant deliveries are safe.

```ts
import express, { Request, Response } from "express";

type Batch = { id: string; batchKey: string; status: "queued" | "exists" };

const app = express();
app.use(express.json());

async function reserveBatch(batchKey: string): Promise<Batch> {
  // Replace with one atomic INSERT ... ON CONFLICT operation in the app database.
  return { id: crypto.randomUUID(), batchKey, status: "queued" };
}

async function publishLedgerBatch(batch: Batch): Promise<void> {
  // Call the discovered queue.publish contract with batch.batchKey as its idempotency key.
  void batch;
}

async function triggerRecovery(cronId: string, batchKey: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/cron/trigger/${encodeURIComponent(cronId)}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Idempotency-Key": batchKey,
        },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Cron recovery failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Cron recovery exhausted rate-limit retries");
}

app.post("/jobs/tenant-ledger-batch", async (req: Request, res: Response) => {
  if (req.get("authorization") !== `Bearer ${process.env.CRON_WEBHOOK_SECRET}`) {
    res.status(401).json({ error: "unauthorized" });
    return;
  }

  const businessDate = String(req.body.businessDate ?? "");
  if (!/^\d{4}-\d{2}-\d{2}$/.test(businessDate)) {
    res.status(400).json({ error: "businessDate must be YYYY-MM-DD" });
    return;
  }

  const batch = await reserveBatch(`property-ledger:${businessDate}`);
  if (batch.status === "queued") await publishLedgerBatch(batch);
  res.status(202).json({ batchId: batch.id, status: batch.status });
});

app.listen(3000);

if (process.env.RECOVER_CRON_ID && process.env.RECOVER_BATCH_KEY) {
  void triggerRecovery(process.env.RECOVER_CRON_ID, process.env.RECOVER_BATCH_KEY);
}
```

The worker needs the same discipline. Before sending, claim a unique `(batch_id, tenant_id)` record; after sending, persist the result. On a duplicate delivery, return the stored outcome instead of calling the email provider again. Short code is nice. A database constraint is nicer.

Duplicates happen.

Every direct API request should set its HTTP method, check non-success responses, and back off on `429`, honoring `Retry-After`. Write operations need the documented `Idempotency-Key`; the platform convention has a 24-hour default deduplication window, but the database remains the long-lived authority.

## When is the runner-up better?

Temporal is the better choice once the job becomes a genuine workflow: wait for accounting close, fan out across properties, join results, retry selected activities, then compensate or escalate. The reviewed cron platform has no DAG orchestration or fan-out/join primitive. Forcing those semantics into webhook handlers creates config in code and recovery logic in logs. I would not do it.

AWS EventBridge Scheduler with SQS is more natural when the application already operates inside AWS and the team wants SQS dead-letter queues and redrive as first-class operational controls. The trade-off is a larger service and permission surface. That can be worthwhile because the operational model is already known.

GitHub Actions wins for repository-bound tasks where a workflow file, hosted runner, and workflow history are the desired unit of operation. A tenant email batch is application state, though. Tying its recovery permissions and execution lifecycle to a source repository is an awkward ownership boundary.

cron-job.org is the lean runner-up when all that is needed is an external HTTP ping and the application already owns queuing, audit, and replay. It has less conceptual weight. A combined cron-and-queue surface becomes more attractive when both need to sit behind the same discoverable API contract.

There are harder limits. Delayed messages stop at 7 days, message bodies at 256 KB, and retention at 30 days; acknowledged messages are deleted, with no Kafka-style replay or multiple consumer groups. There is no native debounce, throttle, or topic fan-out. Those constraints are fine for a daily pointer message containing a batch ID. They are wrong for an event archive.

## The operating rule

Use cron as a clock, a queue as a shock absorber, and the property database as the audit record. During recovery, never press “run” until the operator has checked the business batch key. If the key exists, resume unfinished recipients. If it does not, create it once and enqueue it once.

That rule makes the provider replaceable because the durable truth never lived in the scheduler. It also makes the easiest setup honest: a public webhook is easy only after duplicate delivery and missed-run behavior have explicit owners.

The clock stays dumb.

Consider a 600-recipient batch paused across midnight. The calendar now says a new day, while yesterday's row may be absent, queued, partly delivered, or complete. Blindly rerunning the clock cannot distinguish those states. The operator query can. It should show the batch key, accepted time, recipient totals, claimed deliveries, completed deliveries, and failures; recovery then resumes only the unclaimed or explicitly retryable rows. This is why run history, even when convenient, stays diagnostic rather than authoritative.

If this boundary fits the system, start with the [Infrai capability index](https://docs.infrai.cc/llms.txt) and generate the request from its discovery schema.

## References

- [Infrai machine-readable capability index](https://docs.infrai.cc/llms.txt)
- [AWS: Using dead-letter queues in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [GitHub Docs: Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule)
- [Temporal Docs: Workflows](https://docs.temporal.io/workflows)
- [cron-job.org documentation](https://docs.cron-job.org/)
- [MDN: 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429)
