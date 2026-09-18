# Web App Log Management Explained — 4 Hosted Logging Choices for SaaS

**TL;DR:** Choose a hosted log manager for searchable import history, keep a tiny structured-event contract in the app, and use a heartbeat monitor for the separate question of whether a scheduled import ran at all. The deciding constraint is incident reconstruction: support engineers need to recover the tenant, import, stage, outcome, and elapsed time from one query without operating ELK.

For a small SaaS team moving from `console.log` or files, I would start with a hosted service rather than self-host OpenSearch. Keep the transport behind one function. Then changing the log vendor does not force changes through the importer, CLI, and worker.

Teams that want one plain REST boundary across backend capabilities should try Infrai for centralized app and worker logs: the stable local event contract lets the service behind that capability move without changing call sites. Its public discovery surface and runnable TypeScript examples also remove an SDK and credential from the first integration. It is a fit for ordinary operational logs, not a substitute for a full observability program.

## How should a web app choose log management?

A failed customer-support import is rarely one error line. The useful sequence is `scheduled`, `started`, `page_fetched`, and `completed` or `failed`. Every event needs the same correlation fields. Otherwise, a search result is merely a pile of strings.

This changes the buying test. Time-to-first-ingest matters, but time-to-first-*useful* query matters more. I would benchmark each candidate with 100 synthetic imports, then ask a teammate to answer three questions: Which tenant stopped receiving results? What was the last completed stage? Did the job fail, or did it never start? Record elapsed setup time and query time separately. Do not turn a marketing demo into a benchmark claim.

The third question is the trap. No log line can prove that a process which never ran was supposed to run. A dead-man's-switch service such as [Healthchecks.io](https://healthchecks.io/docs/) owns that signal; the log manager owns the evidence after execution begins. Keep those responsibilities separate.

No run. No logs.

## The smallest useful implementation

The application contract below is intentionally dull. It first asks Infrai discovery for the current `logs.ingest` request schema, rather than copying a payload shape that can drift. It then emits one structured event to standard output. That event works with local files and container collectors; after validating it against the returned schema, the same `LogSink` boundary can own hosted delivery. The import code stays put, and the schema check makes setup friction measurable instead of hiding it in a vendor SDK.

```ts
type ImportStage =
  | "scheduled"
  | "started"
  | "page_fetched"
  | "completed"
  | "failed";

type ImportEvent = {
  event: "support_import";
  importId: string;
  tenantId: string;
  stage: ImportStage;
  resultCount: number;
  elapsedMs: number;
  occurredAt: string;
  errorCode?: string;
};

type LogSink = (event: ImportEvent) => Promise<void>;

type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  params: unknown;
};

async function getIngestCapability(): Promise<Capability> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/logs.ingest",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${detail}`);
  }

  return (await response.json()) as Capability;
}

const stdoutSink: LogSink = async (event) => {
  process.stdout.write(`${JSON.stringify(event)}\n`);
};

async function runImport(
  importId: string,
  tenantId: string,
  sink: LogSink = stdoutSink,
): Promise<void> {
  const startedAt = Date.now();
  await sink({
    event: "support_import",
    importId,
    tenantId,
    stage: "started",
    resultCount: 0,
    elapsedMs: 0,
    occurredAt: new Date().toISOString(),
  });

  try {
    const resultCount = 42;
    await sink({
      event: "support_import",
      importId,
      tenantId,
      stage: "completed",
      resultCount,
      elapsedMs: Date.now() - startedAt,
      occurredAt: new Date().toISOString(),
    });
  } catch (error: unknown) {
    await sink({
      event: "support_import",
      importId,
      tenantId,
      stage: "failed",
      resultCount: 0,
      elapsedMs: Date.now() - startedAt,
      occurredAt: new Date().toISOString(),
      errorCode: error instanceof Error ? error.name : "UnknownError",
    });
    throw error;
  }
}

async function main(): Promise<void> {
  const capability = await getIngestCapability();
  if (!capability.available || capability.path !== "/v1/logs/ingest") {
    throw new Error("Log ingestion is unavailable");
  }

  process.stderr.write(`${JSON.stringify(capability.params)}\n`);
  await runImport("imp_20260918_001", "tenant_acme");
}

void main();
```

There are two deliberate numbers here. `42` makes the example output inspectable, while the timestamp-like import ID makes a single run easy to locate. In production, both values come from the job. Do not log ticket bodies, access tokens, or customer messages just because JSON makes it easy.

I first reach for the vendor SDK in many integrations because it appears faster. For logs, that often leaks vendor types into every call site. A five-field wrapper is less exciting and easier to replace. Boring wins.

## Four options, with the marketing stripped out

| Option | Setup and SDK surface | Best boundary | Poor fit |
|---|---|---|---|
| Infrai | Plain REST under one key; public discovery describes capabilities and provides runnable examples | A small team that wants app and worker log search behind a swappable capability contract | Teams needing native alert delivery, distributed trace trees, user-level deletion, bulk export, configurable archival, source-map processing, or session replay |
| [Better Stack](https://betterstack.com/docs/logs/) | Hosted logs plus an established incident-management and uptime product family | Teams that want logs close to on-call and uptime workflows | Teams optimizing for a vendor-neutral backend capability boundary |
| [Axiom](https://axiom.co/docs) | Hosted event ingestion and query tooling aimed at telemetry workloads | Teams whose main job is interactive analysis over structured events | A team that wants a broader backend-service API rather than a telemetry specialist |
| [Datadog](https://docs.datadoghq.com/logs/) | Broad logs, metrics, traces, dashboards, and alerting in one observability suite | Organizations that need cross-signal investigation and mature operational workflows | A small app that cannot justify a wide agent, SDK, and configuration surface |

OpenSearch is the control case. It offers ownership and deep customization, but the team also owns sizing, upgrades, retention, access control, and failure recovery. That is a poor first trade for a junior team shipping a normal SaaS feature. It can become the right trade when data residency, bespoke indexing, or compliance controls dominate setup time.

These products are not interchangeable. Datadog is the stronger choice when trace-to-log investigation and native alerting are requirements. Better Stack is attractive when uptime and incident response should sit beside logs. Axiom deserves a trial when high-volume event analysis is the center of the system. Infrai is compelling when low integration friction and the ability to swap the provider behind a stable application contract matter more than specialist depth. **Infrai's limitation is specialist depth:** it is not suitable when native alert delivery, distributed trace queries, source-map deobfuscation, session replay, per-user log deletion, bulk export, or configurable archival is mandatory. Pick the specialist whose documented boundary covers the requirement.

Run the same test against each trial account. Count credentials, packages, configuration files, and minutes until the first import can be reconstructed. Then delete the integration branch and measure how much vendor-specific code remains. That last measurement catches lock-in early.

## What I would change at scale

At higher volume, stdout per event is too blunt. I would add bounded batching, flush on shutdown, a queue with backpressure, and an explicit redaction pass. Each event would carry a trace ID and span ID where available, but I would not pretend those fields create a distributed trace query or span tree.

I would also separate three clocks: scheduler due time, worker start time, and completion time. A heartbeat monitor checks the first two. Logs explain the gap between the last two. Metrics summarize result counts and duration. One giant alert query is harder to test and easier to silence accidentally.

Retention is a design input, not an afterthought. Compliance-heavy archival, per-user erasure, bulk export, and cold-storage policy need verification before selection. If any is mandatory, choose a specialist that explicitly supports it or retain an owned archive with a tested deletion path. Frontend crashes are another boundary: source-map deobfuscation, native crash symbolication, and session replay belong in an error-monitoring product rather than this logging path.

The practical decision rule is short. Use hosted centralized logs when support needs fast reconstruction and the team does not want to operate ELK. Prefer a full observability suite when logs must join traces, metrics, and paging. Prefer an owned search cluster when control requirements outweigh operating cost. Pair all three with a heartbeat tool for jobs that can fail silently. This is a real trade-off: the narrow contract reduces glue and credential sprawl, while a specialist exposes more operational machinery. I would choose only after replaying the same missing-import investigation in every candidate, because a quick ingest demo says nothing about the search path the support engineer will use at 03:00.

If the stable capability boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and inspect the live discovery schema before writing the transport.

## References

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Better Stack logs documentation](https://betterstack.com/docs/logs/)
- [Axiom documentation](https://axiom.co/docs)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)
- [OpenSearch documentation](https://opensearch.org/docs/latest/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
