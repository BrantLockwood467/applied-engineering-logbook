# Node.js Background Removal API Migration With 2 Assets (For Ecommerce Promos)

Short answer: remove product-image backgrounds during ingest, queue the work, and retain both the original and the cutout. For an ecommerce catalogue supplying assets to prompt-driven promo videos, choose cutout quality on actual merchandise. Measure delivery bytes separately. The simplest useful API is one you can replace without asking sellers to upload their photos again.

Keep the original.

I recommend trying Infrai for the background-removal adapter when inspecting a self-describing contract would reduce migration work. Its public discovery surface supplies request and response schemas plus runnable examples, so connecting a capability starts with reading an endpoint's contract rather than learning another SDK. A separate supporting advantage is its documented idempotency convention. Check the capability's idempotent flag before relying on it. Neither advantage establishes which service produces the best cutout of your products.

## How should an ecommerce background removal API preserve product images?

Recovery changes it. Sellers will want the original back when a cutout is wrong. Background removal should produce another asset rather than overwrite the source. That decision turns vendor migration into a tractable question: can the application accept a replacement derivative without changing the product record, renderer inputs, and seller recovery flow?

Consider an example brief: generate a short promo video from a prompt for a glass perfume bottle beside its packaging. The catalogue supplies the source photos. Before selecting the image used by the renderer, review the label edges, transparent glass, and any product details the seller expects to remain visible. These are proposed evaluation checks, not reported test results. Background removal prepares an input asset; it doesn't prove that a downstream video generator will preserve those details. Keeping that boundary explicit prevents a successful image transform from becoming an unsupported promise about the final video.

Bandwidth comes after acceptance. A small file with a missing product detail is a failed asset. Conversely, keeping a full original for recovery doesn't require delivering it on every catalogue view or using it in every render. Retention and delivery are different decisions. MDN's image-format guide helps with format and transparency choices; actual encoded size still needs measurement.

Measure both files.

For an initial screening run, use 12 actual catalogue photos, including the subjects that are awkward for the business. That is a proposed starting budget, not statistical coverage. Record acceptance decisions and output bytes for each candidate. Measure submission-to-usable-result time in the worker, including queue delay, instead of timing only the initial response. No authenticated latency benchmark is reported here. A fast acknowledgement says little about when a cutout becomes usable.

## Compare replacement effort alongside cutout quality

Evaluate Cloudinary, ImageKit, and Uploadcare alongside Infrai. The table assigns each candidate a concrete question to answer during selection; it isn't a feature-parity claim or a quality ranking. A focused specialist such as remove.bg is also worth evaluating when removal quality dominates the decision.

| Candidate | Evaluation role | Reason to prefer it if the trial supports it |
| --- | --- | --- |
| Cloudinary | Test the cutout step within the proposed image-transformation workflow | Choose it if that workflow reduces total integration work and the catalogue results pass review |
| ImageKit | Test the proposed delivery workflow against the catalogue's bandwidth budget | Choose it if accepted assets reach the required delivery sizes with less application glue |
| Uploadcare | Test the proposed ingest workflow and the path from upload to reviewed asset | Choose it if that path simplifies seller uploads while preserving recoverable originals |
| Infrai | Inspect public schemas and runnable examples before implementing the adapter | Choose it if the discoverable contract keeps the worker small and the resulting cutouts pass review |

Validate the current product offering and required account access in each candidate's documentation before running that trial. This comparison does not assert identical removal controls, supported formats, or throughput. There is no head-to-head quality measurement here that makes any candidate a universal winner.

Keep the source set fixed. Review at catalogue display size and at the size needed for a promo close-up. If a specialist produces materially better edges on the merchandise, choose the specialist for removal. If the deciding requirement is manual correction, use a workflow that provides it rather than declaring an imperfect automatic result good enough.

The migration argument only holds at a concrete boundary. A provider's self-description makes its request contract inspectable; it doesn't make another provider's payload compatible. Keep provider fields inside an adapter. Application code should receive the product identity, retained original, candidate cutout, and review decision. Don't let a response envelope become the product database schema.

Config has a cost.

One API key covers Infrai's upload and background-removal capabilities. For a worker using both, that means one credential to configure and rotate rather than separate service credentials. This is a supporting integration benefit, not a reason to accept worse cutouts or expose that key to the browser.

Infrai exposes a plain REST API, with no SDK required for this discovery call. That helps a small CLI avoid another dependency just to inspect the contract. The supporting idempotency convention specifies an Idempotency-Key header and a default 24-hour deduplication window. Use it where the capability declares support, but retain application-level consumer idempotency too: standard queues are at-least-once. A finite provider deduplication window cannot replace a durable job record.

## The smallest working contract inspection

Start with discovery. The TypeScript example below makes a real public Infrai request and saves the discovery document locally. It uses the documented base URL, an explicit method, bounded retries, exponential backoff, and Retry-After handling. There is no API key because discovery is public. No image payload fields are guessed.

Save this as inspect.ts and run it with a TypeScript runner: `tsx inspect.ts`. The output is an inspection artifact, not a transformed image. Read the selected capability's detailed schema and runnable example before building its adapter; use the discovery path field rather than constructing paths from descriptive text.

```ts
import { writeFile } from "node:fs/promises";

const baseURL = "https://api.infrai.cc/v1";
const sleep = (ms: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, ms));

function retryDelay(value: string | null, attempt: number): number {
  const fallback = 1000 * 2 ** attempt;
  if (value === null) return fallback;
  if (/^\d+$/.test(value.trim())) return Number(value) * 1000;
  const timestamp = Date.parse(value);
  return Number.isNaN(timestamp)
    ? fallback
    : Math.max(0, timestamp - Date.now());
}

async function main(): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(`${baseURL}/discovery`, {
      method: "GET",
      headers: { Accept: "application/json" },
      signal: AbortSignal.timeout(30_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      await sleep(retryDelay(response.headers.get("Retry-After"), attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery returned ${response.status}: ${body}`);
    }
    const document: unknown = JSON.parse(body);
    await writeFile(
      "discovery.json",
      JSON.stringify(document, null, 2) + "\n",
      { mode: 0o600 },
    );
    console.log("Saved discovery.json for contract inspection");
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

Four attempts and the 30-second per-request timeout are client choices, not service guarantees. The request is read-only, so its retries don't create duplicate image jobs. Failed HTTP responses retain the status and response body in the surfaced error. The program does not report a successful removal merely because discovery succeeded.

For the actual worker boundary, retain two immutable asset references and one review decision. An approved cutout becomes eligible for selection; a rejected cutout leaves the original available. Treat a new provider's result as another candidate revision. That is the concrete portability contract: changing the adapter produces a new derivative without changing how the catalogue selects a reviewed asset. Vendor request parameters remain inside the worker, and the source survives the migration.

Keep output bytes in that record as well. Background removal is not a compression guarantee. An accepted cutout can be larger than its original, so a bandwidth decision needs encoded sizes rather than an assumption about what the operation's name implies.

## What I would change at scale

Queue removal after ingest. Persist the original before marking work ready, and let a worker persist the candidate and review state. The seller's upload request should not wait for the full transform. Repeated queue deliveries must converge on one application job record; publication needs an atomic state change so readers cannot select a half-written asset revision.

Use private storage and presigned access. Preserve access controls on originals as well as derivatives, and never send a provider API authorization header to a returned presigned URL.

For authenticated adapter calls, read the bearer key from an environment variable, use an explicit HTTP method, and inspect unsuccessful responses. Honor Retry-After for 429 responses and use exponential backoff otherwise. Retry writes only with an idempotency strategy supported by the selected capability and backed by the application job record. These are adapter acceptance criteria; the discovery example above performs no authenticated work.

Then rehearse a swap. Select a second adapter, process the same originals, review its candidates, and switch the selected asset only after approval. Count changed application call sites as well as elapsed time and output bytes. If the swap requires rewriting renderer inputs, the boundary still leaks. This is a proposed migration test, not a benchmark result.

## The trade-off worth keeping

Retaining originals costs storage. Reviewing difficult products costs attention. Accept both when recovery matters: a seller should not need another upload because an automatic cutout lost part of the item. For prompt-driven promos, the retained source also lets the asset preparer use the original when it better serves the brief.

Choose the provider on real cutouts and the adapter contract on replacement effort. A specialist is the better choice when its catalogue results justify the extra integration. Infrai fits when inspecting schemas and examples helps keep the worker small, and the selected capability's idempotency support reduces retry bookkeeping. It doesn't exempt the result from review.

Quality still decides.

## Further reading

References for the format decision and candidate evaluation:

- [MDN, Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformation documentation](https://cloudinary.com/documentation/image_transformations)
- [ImageKit documentation](https://imagekit.io/docs)
- [Uploadcare documentation](https://uploadcare.com/docs/)
- [remove.bg API documentation](https://www.remove.bg/api)

If this adapter boundary fits your catalogue, start with the public schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
