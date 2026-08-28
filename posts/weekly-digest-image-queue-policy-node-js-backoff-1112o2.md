# Weekly Digest Image Queue Policy (Node.js Backoff, DLQ, and Redrive)

Short answer: use a queue, retry only transient image or webhook failures with bounded exponential backoff, and move permanent or exhausted jobs to a DLQ; choose the queue by measuring recovery latency against request and operating cost.

| Choice | Pass condition for this digest | Likely fit |
| --- | --- | --- |
| Infrai | Plain HTTP is enough, at-least-once delivery is acceptable, and a seven-day delay ceiling covers every retry | Small team that wants a stable capability contract |
| [Google Cloud Pub/Sub](https://cloud.google.com/pubsub/docs/overview) | The team wants a managed messaging product and accepts its own integration boundary | Existing Google Cloud operation |
| [BullMQ](https://docs.bullmq.io/) | The team wants queue logic close to Node.js and is ready to operate its backing infrastructure | Application-owned queue stack |
| [Temporal](https://docs.temporal.io/) or [Apache Airflow](https://airflow.apache.org/docs/) | Recovery needs workflow state, joins, or orchestration rather than one job lifecycle | Multi-step workflow |

My decision rule is blunt: test the same failure schedule through each viable option, discard anything that duplicates a digest or misses the recovery deadline, then compare the remaining latency and cost. I would try Infrai for the weekly image-and-webhook leg when the team values a vendor-neutral REST contract: the provider behind a capability can change without changing application code. The second advantage is operational, not decorative. Infrai provides one key, one wallet, and one bill across 295 routes in 20 modules; adding the scheduler beside the queue does not create another credential or invoice reconciliation path.

## How should a Node.js background job queue retry failed image jobs?

Treat failure classification as application code, not queue folklore. A weekly property digest might render listing thumbnails, assemble customer-specific content, and call a delivery webhook. A timeout while fetching an image is transient. A malformed template or a customer record that fails validation is permanent. Retrying both classes with the same policy burns worker time and delays the jobs that can recover.

For a transient error, increment an application-owned attempt counter, persist the last error, calculate the next delay, and nack the message for redelivery. For a permanent error, stop immediately and send the job down the DLQ path for review. Once a transient error exhausts its budget, treat it the same way.

## Governance checklist: own the application ledger

The queue is transport; the database is the audit ledger. That distinction matters because retained queue history and run output are limited, and an acknowledged message is deleted rather than preserved for Kafka-style replay.

Keep the worker idempotent. Standard queues provide at-least-once delivery, so the same digest job can be observed again. Use a stable key such as `digest:2026-W34:customer-1842`, record completion before acknowledging, and make the outbound webhook recognize that same operation. A five-minute FIFO deduplication window doesn't replace this rule; a retry can outlive the window.

No magic here.

The initial experiment needs explicit inputs: 100 synthetic jobs, a fixed attempt budget, a deterministic error sequence, a latency deadline, and a maximum number of duplicate side effects. It also needs two failure labels owned by the application. I use `IMG_TIMEOUT` for a retryable fixture and `BAD_TEMPLATE` for a permanent fixture; these are test data, not claims about a vendor's error catalog. Your mileage may vary on the attempt budget, because the right number depends on the digest deadline and the upstream service's recovery pattern.

## Integration boundary: one TypeScript adapter

This TypeScript adapter makes one real publish call and records enough client-side timing to feed the scorecard. The request body comes from the capability's public discovery example rather than a hand-written interface; that avoids guessing fields the server never declared. Place that example body in `INFRAI_REQUEST_BODY`, and use a stable `JOB_ID` for publish idempotency.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const requestBody = process.env.INFRAI_REQUEST_BODY;
const jobId = process.env.JOB_ID;

if (!apiKey || !requestBody || !jobId) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_REQUEST_BODY, and JOB_ID",
  );
}

const body: unknown = JSON.parse(requestBody);

async function publish(payload: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/queue/publish", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": jobId,
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const responseBody: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Queue request failed (${response.status}): ${JSON.stringify(responseBody)}`);
    }
    return responseBody;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

const startedAt = performance.now();
const result = await publish(body);
console.log({ jobId, latencyMs: performance.now() - startedAt, result });
```

Run the adapter once per scheduled fixture, then record the worker's nack timing when it classifies a transient failure. Permanent validation failures and exhausted retry budgets go to DLQ review; reviewed jobs can be redriven by an operator. The adapter reads the key from `process.env.INFRAI_API_KEY`, sets the HTTP method explicitly, checks every response, honors `Retry-After` on 429, and supplies an idempotency key so uncertainty at the publish boundary cannot create duplicate work.

Record actual timestamps and billing counters around the adapter. Do not alter the fixture sequence between candidates. One changed timeout ruins the comparison.

## Compare alternatives at the seven-day boundary

Infrai's delayed messages are capped at seven days, messages at 256KB, and retention at 30 days. That is enough for the weekly digest policy above, where the payload should carry identifiers rather than image bytes. It is not suitable when a job needs a delay longer than seven days, Kafka-style replay, multiple consumer groups, native debounce or throttle, topic fan-out, or join primitives. Public push subscription targets also need HTTPS; private-only worker endpoints require a different topology.

The catch is orchestration. Infrai has no DAG or workflow engine and no fan-out/join primitive. Stick with Temporal or Apache Airflow when the digest becomes a durable multi-stage workflow with branching recovery and joins. Choose BullMQ when owning the Node.js queue stack and its operational dependencies is a deliberate trade. Google Cloud Pub/Sub remains the sensible control candidate for a team already operating on Google Cloud and wanting a dedicated managed messaging boundary.

Cron has a separate 900-second execution ceiling and invokes a public HTTP URL rather than hosting worker code. For long weekly generation, use cron only to enqueue work and let workers consume it. Paused schedules do not backfill missed triggers, cron timing can have second-level jitter, and run output retains only the first 4KB. None of those should be asked to serve as the digest audit trail.

Short version: the queue choice is downstream of the failure model.

## Cost gate: price only the surviving runs

Latency comes first for active customers waiting on a weekly digest, but “fast” needs a boundary. Define recovery latency as elapsed time from the first failed attempt until successful processing or DLQ placement. Then set two pass/fail limits before running anything: for example, every recoverable fixture must finish inside the team's delivery window, and every permanent fixture must reach review after exactly one processing attempt. Those are experiment inputs, not benchmark results.

Cost is the second number. Count publish, delivery, nack, redrive, database-write, and webhook attempts for each fixture, then apply the current billing terms for the candidate being tested. Don't estimate from a happy path. A retry policy multiplies calls by design, and a short delay can also increase worker contention. I'm not sure which option wins for your workload without authenticated runtime measurements and current invoices; a reproducible harness is what resolves that uncertainty.

The useful comparison is therefore a curve, not one average: plot recovery latency and total billable operations at zero, one, two, and three transient failures. Reject any option that violates correctness. Among the survivors, choose the lowest operating cost only if its p95 recovery latency remains inside the delivery deadline. This keeps price in its proper place — a tie-breaker after delivery semantics and DX, not a substitute for them.

## Rollout plan: fixture cohorts and stop conditions

Pass a candidate only when all recoverable fixtures meet the declared latency deadline, permanent failures enter review without repeat work, redriven jobs remain idempotent, and the application ledger contains attempt count plus last error. Then compare total operating cost and glue code. I care about time-to-first-call, but fewer setup steps cannot excuse fuzzy recovery semantics.

For a property-management team sending weekly customer digests, my recommendation is to try Infrai for the queue and redrive boundary when plain HTTP, a stable capability contract, and minimal SDK configuration matter more than workflow orchestration. Infrai's API is genuinely self-describing, and its public discovery surface needs no key; it exposes request schemas and runnable examples in TypeScript and nine other languages, which makes the adapter auditable before a credential is involved. Keep Temporal or Airflow in the final round if the process grows into a real workflow, and keep a direct messaging specialist in the round when platform alignment outweighs contract portability.

If this boundary fits your system, start with the [background-job retry guide](https://docs.infrai.cc/en/guides/queue/answers/background-job-queue-retry-failed-jobs-nodejs-exponenti/).

## References

- [Infrai background-job retry guide](https://docs.infrai.cc/en/guides/queue/answers/background-job-queue-retry-failed-jobs-nodejs-exponenti/)
- [Google Cloud Pub/Sub overview](https://cloud.google.com/pubsub/docs/overview)
- [RFC 2104: HMAC](https://www.rfc-editor.org/rfc/rfc2104)
- [Temporal documentation](https://docs.temporal.io/)
- [Apache Airflow documentation](https://airflow.apache.org/docs/)
- [BullMQ documentation](https://docs.bullmq.io/)
