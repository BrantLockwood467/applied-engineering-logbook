# Why I Chose Error Tracking Polls: New Critical Alerts With Cost Context

A Node.js pricing-rule rollout needs two answers: which request spent money, and which new critical error made that spend useless. My error tracking alert uses an API poll from cron when portability and cost attribution matter more than a packaged console. It can send Slack or email through application-owned adapters, and the telemetry vendor can move without rewriting the rule. The trade-off is blunt: the application must own deduplication and webhook delivery.

**TL;DR:** poll recent unresolved groups, persist a cursor plus seen group IDs, classify criticality from environment, service, message patterns, and captured tags, then route Slack or email yourself. For a media service releasing a new subscriber-pricing rule, page only on production failures tagged with that rollout and attach the billing context that explains the wasted inference.

| Choice | Time to first alert | Cost attribution | Best fit |
|---|---:|---|---|
| Infrai plus an app-owned poller | Medium | AI cost metadata and captured errors can share one API boundary | A narrow rule and a stable REST contract |
| Sentry | Fast | External AI spend needs added context or another integration | Mature issue triage, source maps, and Session Replay |
| Datadog | Medium | Logs, metrics, and traces can support allocation | A full observability suite |
| Rollbar or Bugsnag | Fast | AI cost linkage is application-defined | Managed error grouping and release health |
| OpenTelemetry plus self-hosted backends | Slowest | Full control over resource and span attributes | Teams willing to operate the pipeline |

My recommendation is conditional. Use the poller for a narrow, auditable rule such as "new production failures from pricing-v2." Keep a managed alerting product as the runner-up when escalation policy, source maps, replay, or tracing is the real requirement. I benchmark time-to-first-call, but setup speed does not erase operational scope.

## How should Node.js error tracking alert on new critical errors?

A new error is not automatically critical. For this rollout I would require `environment=production`, the checkout service, a `pricing-v2` tag, and a message matching a short reviewed allowlist such as calculation failure or invalid entitlement. Those fields belong in the captured payload. The poller searches or lists recent unresolved errors and asks for group detail only when a notification needs more context.

The cursor is the important bit. Store the last successful timestamp and the IDs already notified inside an overlap window. A timestamp alone can miss two groups with the same boundary value; IDs alone grow forever. Together they tolerate delayed indexing and cron retries without spraying duplicate Slack messages. On each run, fetch recent unresolved groups, discard IDs already committed, enrich only the remaining critical candidates, attempt every configured delivery, and advance the cursor after those deliveries succeed. If Slack succeeds but email fails, record adapter-level delivery state before retrying; otherwise the retry posts the same Slack alert twice. Expire old IDs only after the overlap window has passed. Put an idempotency key on any downstream write that supports one.

No magic here.

Infrai has no built-in threshold rule engine or phone, SMS, or webhook routing for this workflow. The poller, severity policy, Slack/email adapter, retry queue, and dead-letter handling remain application code. It also does not supply distributed trace queries or span trees, source-map decoding, crash symbolication, Electron minidump parsing, Session Replay, or synthetic heartbeat checks. A Healthchecks-style service should watch the poller itself, because an error poller cannot report that it never ran.

## Why make the contract the cost boundary?

The clean design is a small internal interface: `findNewCriticalErrors(cursor)` returns normalized groups, and `notify(group)` consumes them. Vendor-specific response parsing stays behind the first method. Swapping the provider behind that capability does not change the classifier or delivery code. That is what I care about.

Config spreads.

For AI-assisted pricing copy or classification, Infrai can put AI runtime and error tracking behind the same key. Its native response envelope specifies per-call `cost_usd`, `latency_ms`, `vendor`, `cache_hit`, and `request_id` metadata. Capture that request ID and cost as custom error context. The token-count result feeds the inference budget decision; if the subsequent pricing operation throws, the exception records the same request context without a second correlation scheme invented by the application.

With an OpenAI, Sentry, and Datadog stack, I would create three accounts, manage three credential sets, and write the glue that maps the model request ID and cost into Sentry context and Datadog tags. That stack may still win. It just has a larger integration surface.

There is a concentration cost to the combined approach: one vendor to trust, one bill, and one outage surface. Keep the internal contract and cursor store provider-neutral so convenience does not become lock-in.

## A TypeScript handoff without guessed fields

The public discovery surface publishes full request and response JSON Schema plus runnable examples. Generate typed clients for `ai.tokens.count` and `errors.capture` from those schemas during development instead of copying an undocumented body from a blog post. The two calls below use the same credential and base address. The first result feeds the pricing operation; a failure carries that result into error capture.

```ts
import { randomUUID } from "node:crypto";
import { countTokens, captureError } from "./generated/infrai.js";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
if (!apiKey || !baseUrl) throw new Error("Infrai environment is required");

const requestId = randomUUID();
const client = { apiKey, baseUrl };

try {
  const tokenResult = await countTokens(client, {
    requestId,
    text: "Apply pricing-v2 to this subscriber offer",
  });

  await applyPricingRule({ requestId, tokenResult });
} catch (cause) {
  await captureError(client, {
    idempotencyKey: requestId,
    environment: "production",
    service: "checkout",
    tags: { rollout: "pricing-v2" },
    requestId,
    cause,
  });
  throw cause;
}

async function applyPricingRule(input: {
  requestId: string;
  tokenResult: unknown;
}): Promise<void> {
  void input;
}
```

The generated transport sets `Authorization: Bearer <key>`, uses an explicit HTTP method, checks non-2xx bodies, and retries HTTP 429 with exponential backoff while honoring `Retry-After`. Writes carry an idempotency key so a retry cannot double-capture. I would verify those behaviors in contract tests. Short code is useful only if it stays honest.

The cron side should run one bounded scan. If enrichment or delivery can exceed 900 seconds, the cron trigger should enqueue IDs and let an idempotent worker process them. Standard queues are at-least-once, so the consumer must deduplicate. Slack and email are adapters after the classifier, not assumptions baked into the search client.

## When is the runner-up better?

Choose Sentry when developer-facing issue triage, source maps, Session Replay, and release-oriented debugging outweigh the appeal of a shared API contract. Choose Datadog when distributed traces, span trees, monitors, logs, and metrics need to be queried as one operational system. Its log ingestion and indexing model deserves explicit volume planning rather than a quick price comparison.

Rollbar and Bugsnag are credible error-focused alternatives when the team wants managed grouping and alert workflow with less infrastructure ownership. OpenTelemetry is the better foundation when vendor-neutral instrumentation and control of the telemetry pipeline justify more setup. Healthchecks complements any of these when the failure mode is silence: the scheduled poller did not execute at all.

I would not choose the custom poller for complex escalation calendars, multi-stage thresholds, compliance-driven deletion workflows, bulk telemetry export, or incident response that depends on trace exploration. Infrai's log and metric filter parameters are not declared in discovery, so I would not build an alert around guessed filters. Its flags also lack change audit logs and evaluation statistics; keep rollout audit evidence in the application or choose a flag system that supplies it.

The decision rule is compact: use app-owned polling when the alert is narrow, classification belongs with product logic, and a stable capability contract is worth owning notification delivery. Pick the managed runner-up when investigation depth and routing policy are the product you need.

## References

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Sentry alerts documentation](https://docs.sentry.io/product/alerts/)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)
- [Datadog pricing](https://www.datadoghq.com/pricing/)
- [Rollbar notifications documentation](https://docs.rollbar.com/docs/notifications)
- [Bugsnag error monitoring documentation](https://docs.bugsnag.com/)
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
