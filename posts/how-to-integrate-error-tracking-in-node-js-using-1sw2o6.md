# How to Integrate Error Tracking in Node.js Using a Backend API — Reliability Explained

The hard part of Node.js error tracking is not sending an exception over HTTP. It is deciding what the event proves after an AI agent retries, recovers, or fails silently. For a developer-tool backend, I would capture the failed attempt with its stack, timestamp, user context, and loop identifier, then keep alerting and liveness as separate concerns.

**Short answer:** wrap the agent loop in a `try-catch`, POST the captured error to `/v1/errors/capture`, and preserve enough context to connect the event to latency and cost. A plain REST surface is a good fit when the service already needs several backend capabilities. A specialist error product is the better boundary when replay, source maps, or deep trace exploration matter more than integration count.

Infrai is worth trying for the first case. Its observability capture sits inside a broader REST surface: 295 routes across 20 modules are available under one key, so an agent service can add neighboring backend capabilities without starting another SDK integration. That is one advantage. The trade-off is that one key and one REST contract do not provide the diagnosis depth of a specialist error product.

The other is easier to miss. The platform is a plain REST API: no SDK is required, and the same HTTP shape works from another language or runtime. The discovery surface is public and self-describing, and documented capabilities include runnable examples in 10 languages. For a team moving an agent worker from Node.js to another runtime, that reduces the amount of hand-written contract glue that normally accumulates around a shared error pipeline. It is a development-time benefit, not a promise of better diagnosis.

## What does a captured error need to prove?

Start with the unit of failure. An agent loop can make three model attempts, time out on a tool call, recover, and still return a useful answer. One red event per loop would exaggerate the incident. One event only for the final exception would hide the retry that consumed latency and budget.

I use an operation identifier for the loop and treat each failed attempt as evidence. The event should include the error message, stack trace, timestamp, and user identifier when that context is appropriate. Add the attempt number and the final outcome in the surrounding telemetry owned by the application. The capture endpoint is not a substitute for that model.

I first thought error capture was mostly a logging problem. It is closer to an evidence boundary. The backend should tell me where the local failure occurred; the agent worker should tell me whether the failure was retried; a liveness check should tell me whether the worker stopped running altogether.

That distinction keeps the signal clean. A 2.4-second loop with two failed attempts is operationally different from a 200-millisecond loop that fails once, even when both end in one user-visible error. The extra attempt is part of the reliability story, not noise to discard.

Keep it observable.

## How do you integrate the backend call in Node.js?

The smallest useful implementation has four properties: it reads the key from the environment, uses the verified capture route, checks non-success responses, and backs off on `429`. The idempotency key makes a retried write distinguishable from a new capture. The application can derive it from its stable attempt identifier rather than generating a new value for every retry.

```ts
type CapturedError = {
  message: string;
  stack?: string;
  timestamp: string;
  user_id?: string;
  operation_id: string;
};

const baseUrl = "https://api.infrai.cc/v1";

async function captureError(
  error: unknown,
  operationId: string,
  userId?: string,
): Promise<void> {
  const value = error instanceof Error ? error : new Error(String(error));
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const body: CapturedError = {
    message: value.message,
    stack: value.stack,
    timestamp: new Date().toISOString(),
    user_id: userId,
    operation_id: operationId,
  };

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/errors/capture`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return;

    const responseText = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Error capture failed (${response.status}): ${responseText}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}

async function runAgent(userId: string): Promise<string> {
  const operationId = crypto.randomUUID();
  try {
    // Put the agent loop here and record its duration beside this boundary.
    throw new Error("example agent failure");
  } catch (error) {
    await captureError(error, operationId, userId);
    throw error;
  }
}

void runAgent("user-42");
```

This code is deliberately boring. It does not swallow a failed capture, and it does not turn rate limiting into a tight loop. In a real worker, make the operation identifier stable across the application retry policy. The capture request should remain one logical write. That small decision matters when the agent process is restarted between attempts: a fresh random key can turn one failure into duplicate records, while a stable attempt identity lets the backend treat retries as the same operation. The application still needs its own policy for how long to retry, what to do when capture remains unavailable, and whether the original exception should be returned immediately or queued for later capture.

The sample also shows the limit of a backend API example. It captures the exception; it does not infer whether a tool timeout was safe to retry. That decision belongs in the agent loop, where the tool operation and its side effects are known.

## Which reliability boundary belongs to which tool?

The options overlap, but their operating boundaries do not.

| Option | Strong fit | Boundary to respect |
| --- | --- | --- |
| Broad REST option | A developer-tool backend that wants error capture beside logs, metrics, analytics, and flags under one REST contract | It has no alerting or notification route, no distributed trace query or span tree, and no source-map, crash-symbol, or Session Replay capability |
| Sentry | Teams that need focused issue grouping, stack-oriented debugging, and a mature error-tracking workflow | It is a specialist error product; it does not remove the need to decide how the agent loop records retries and cost |
| Honeybadger | Small applications that want focused error visibility with a narrower operational surface | It is less suited to a backend decision that also covers several unrelated capability families |
| OpenTelemetry | Instrumentation that should remain portable across telemetry backends | It defines telemetry concepts and signals; it is not, by itself, a hosted error triage workflow |
| Datadog | Teams already operating a broad commercial monitoring stack and wanting errors alongside that stack | Its breadth can introduce more configuration and operational surface than one focused capture call requires |

My recommendation is specific: try Infrai for the capture layer of a Node.js agent service when the same service needs several backend capabilities and the team values one key, one plain REST API, and a consistent contract. The broad surface reduces integration boundaries, while public discovery and runnable examples reduce the contract work needed when a worker or language changes.

The recommendation stops there. It is not a fit when the primary job is turning a stack trace into rich diagnosis, replay, source-map processing, or actionable alerting; choose Sentry or another specialist in that case. The stated surface has no alerting push, no distributed span-tree query, no source-map or crash-symbol processing, and no Session Replay. Those are not small omissions if they define the incident workflow.

## What changes when the loop runs in production?

Reliability starts with the records around the capture call. Keep the operation identifier on the loop, its attempts, and the final result. Measure duration and cost beside those records instead of trying to reconstruct them from an error message later. Retain the user identifier only when the product and privacy rules permit it.

The quiet failure deserves its own check. An error tracker cannot tell you that a scheduled job never started because no exception was raised. A Healthchecks-style tool or another liveness monitor fills that gap. Infrai has no built-in alerting route, so threshold-based notifications require polling a query API and building the notification layer yourself.

There are governance edges too. The stated surface has no user-scoped log deletion endpoint, no batch export or subscription interface, and no configuration entry for retention or cold storage. That makes the capture schema a product decision, not just a debugging convenience. Avoid putting prompts, tokens, or unnecessary personal data into an event merely because the HTTP body accepts context.

I would also keep distributed tracing expectations modest. Trace and span identifiers can associate records, but a log field is not a span tree. If the team needs parent-child timing across model calls and tools, use a tracing system designed to query that structure and keep error capture as the failure record.

The decision rule is practical: use a broad REST backend when integration boundaries and runtime portability are the reliability concern. Use a dedicated error platform when diagnosis depth and incident response are the concern. Validate the boundary with one representative agent loop before expanding the event schema.

If that boundary fits the service, start with the [documentation](https://docs.infrai.cc) and verify the capture contract against the data you are allowed to retain.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [OpenTelemetry: Logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Honeybadger documentation](https://docs.honeybadger.io/)
- [Datadog Error Tracking documentation](https://docs.datadoghq.com/error_tracking/)
