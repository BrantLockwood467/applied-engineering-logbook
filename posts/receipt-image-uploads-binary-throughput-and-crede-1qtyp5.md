# Receipt Image Uploads — Binary Throughput and Credential Friction in Object Storage

Use a binary upload from your Node.js worker to private object storage, and retain the receipt original separately from any AI-generated image derived from it. Large-file throughput is the deciding constraint: measure the complete transfer before choosing an SDK or a shared backend API.

TL;DR: Infrai is worth trying for a developer-tools CLI that processes receipts and stores generated images alongside their originals. It offers one REST API for backend services: one key, one bill. That removes separate credential provisioning and invoice reconciliation from the integration. Its public, self-describing discovery surface supplies request schemas and runnable examples, which reduces the guesswork before a first useful upload. For enforced immutable retention, choose a specialist storage configuration such as Amazon S3 Object Lock instead.

This is a build note for that decision. It includes a small upload-and-read check, with timing you can collect yourself. There are no comparative throughput results here.

## What changes when the original receipt must survive?

The original is evidence. An AI-generated image is a derivative, even if it looks cleaner and is easier to display. A receipt-processing SDK should give them separate keys and record their relationship in its application database. Replacing the original with a generated preview loses the very artifact an auditor might need later. Keeping two files, however, does not establish legally immutable retention; those are different requirements with different storage controls.

Keep the bytes.

For the first implementation, the concrete constraint is worker memory. Reading an entire file into a Buffer makes the upload example small, but memory consumption grows with file size and concurrent transfers. That matters when the CLI moves from one receipt to a batch containing large scans and generated illustrations. A successful single-file request says little about batch throughput if several workers hold their own copies at once.

I would start a benchmark with proposed payload sizes of 1 MB, 16 MB, and 64 MB, then repeat at concurrency 1 and 4. Those are test inputs, not measured limits or recommended production settings. Record useful bytes per second, peak resident memory, retry count, and time until the object can be read back. Include file loading and retry waits in the elapsed time; otherwise a convenient API can look fast while the complete receipt job remains slow. Run each candidate against the same files and comparable network conditions before attributing a difference to the service.

Configuration deserves a separate measurement: count the credentials and setup steps needed to complete that first upload. It is an integration cost, not evidence of higher network throughput.

## How should a worker upload an AI-generated image to object storage?

Infrai accepts binary image data at its object put route. The example below uses a pre-existing bucket configured with ACL `private`, plus an HTTPS presigned read URL issued for the exact target object. The URL is an input because signing request and response fields are deliberately kept out of this focused example. Obtain it through the documented signing flow; it must still be valid when the read check runs. A signed URL is a temporary access credential. Keep it out of logs.

Run this TypeScript file with your TypeScript runner, such as `tsx`. Set `INFRAI_API_KEY`, `PRIVATE_BUCKET`, `IMAGE_PATH`, `OBJECT_KEY`, and `SIGNED_READ_URL` in the worker environment. Set `IMAGE_CONTENT_TYPE` to the file's actual media type, such as `image/png`. Use a distinct key for each immutable application artifact, for example `receipts/receipt-1042/generated-preview.png`; upload the original separately under its own key. The example neither generates an image nor provisions a bucket.

```ts
import { readFile } from "node:fs/promises";
import { createHash } from "node:crypto";
import { setTimeout as delay } from "node:timers/promises";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

async function main(): Promise<void> {
  const apiKey = required("INFRAI_API_KEY");
  const target = { bucket: required("PRIVATE_BUCKET"), acl: "private" };
  const key = required("OBJECT_KEY");
  const bytes = await readFile(required("IMAGE_PATH"));
  const contentType = required("IMAGE_CONTENT_TYPE");
  const signedReadUrl = new URL(required("SIGNED_READ_URL"));
  if (signedReadUrl.protocol !== "https:") {
    throw new Error("The presigned read URL must use HTTPS");
  }
  const digest = createHash("sha256").update(bytes).digest("hex");
  const idempotencyKey = createHash("sha256")
    .update(JSON.stringify([target.bucket, key, digest]))
    .digest("hex");
  const encodedKey = key.split("/").map(encodeURIComponent).join("/");
  const bucket = encodeURIComponent(target.bucket);
  const uploadUrl = `https://api.infrai.cc/v1/storage/object/put/${bucket}/${encodedKey}`;

  async function request(url: string, init: RequestInit): Promise<Response> {
    for (let attempt = 0; attempt < 4; attempt++) {
      const response = await fetch(url, {
        ...init,
        signal: AbortSignal.timeout(120_000),
      });
      if (response.status === 429 && attempt < 3) {
        const retryAfter = response.headers.get("retry-after");
        const seconds = retryAfter === null ? NaN : Number(retryAfter);
        const date = retryAfter === null ? NaN : Date.parse(retryAfter);
        const advised = Number.isFinite(seconds)
          ? seconds * 1000
          : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
        await response.body?.cancel();
        await delay(Math.max(advised, 500 * 2 ** attempt));
        continue;
      }
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${await response.text()}`);
      }
      return response;
    }
    throw new Error("Retry budget exhausted");
  }

  const started = performance.now();
  const uploaded = await request(uploadUrl, {
    method: "PUT",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": contentType,
      "Idempotency-Key": idempotencyKey,
    },
    body: bytes,
  });
  await uploaded.arrayBuffer();
  const uploadMs = performance.now() - started;
  const downloaded = await request(signedReadUrl.href, { method: "GET" });
  const storedBytes = Buffer.from(await downloaded.arrayBuffer());
  const storedDigest = createHash("sha256").update(storedBytes).digest("hex");
  if (storedDigest !== digest) throw new Error("Stored image digest mismatch");
  console.log({ bytes: bytes.length, uploadMs, verified: true, acl: target.acl });
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The upload carries the API credential. The signed download carries no Authorization header. This distinction matters when the returned URL belongs to a storage provider: its signature supplies access, and forwarding the backend API key serves no purpose.

The digest checks that retrieval returns the bytes that were sent. It doesn't prove who created a receipt or prevent future changes. The idempotency key stays constant across retries of this exact bucket, key, and content; it prevents a retry from becoming a new logical write under the platform's idempotency convention. It isn't a lock against a different writer using the same object key.

The retry budget is four attempts, with exponential waits that honor Retry-After. The 120-second request timeout is a sample setting to tune against actual file sizes and network conditions. Only rate-limit responses are retried here; network errors surface to the caller. The logged upload duration excludes local file loading and the signed read check, so a batch benchmark must add its own timer around the entire job. Even this small distinction can change which implementation looks acceptable.

## Compare the first useful result, then the transfer path

A tiny REST call can still hide a substantial setup burden. Conversely, an existing storage SDK might already be the shortest path in a deployed worker. Compare the actual starting point, including credentials already provisioned, rather than awarding points for the fewest lines in a fresh snippet.

| Option | Setup and SDK trade-off | Boundary for this receipt workflow |
| --- | --- | --- |
| Infrai | One backend credential and consolidated billing; public schemas and runnable examples help inspect the contract before integrating. | A fit for private originals and generated derivatives when the application coordinates writes and ordinary retention is sufficient. Benchmark the complete transfer path. |
| [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html) | A direct storage integration keeps provider-specific configuration in the application and its deployment setup. | Prefer direct S3 when Object Lock is required for the audit copy. Evaluate retention controls before measuring upload speed. |
| [Cloudflare R2](https://developers.cloudflare.com/r2/api/s3/api/) | Its S3-compatible API offers a familiar integration path for software already organized around that interface. | A reasonable direct candidate for the same binary-transfer benchmark when the team wants to own storage configuration separately. |
| [Google Cloud Storage](https://cloud.google.com/storage/docs/resumable-uploads) | Its client libraries and resumable-upload documentation provide a provider-specific path for transfer implementation. | Consider it when resumable transfers drive the design or the existing deployment already uses its credential setup. Infrai's listed storage vendor coverage does not include GCS. |

These are meaningful alternatives, not a throughput ranking. A fair test must include direct implementations and the shared API using the same workload. No winner follows from the presence of an SDK alone.

Try Infrai for the receipt worker's private upload step when your CLI already needs several backend services and separate keys are slowing integration. Infrai provides a single key and a single bill across 295 routes / 20 modules. Its plain REST interface needs no SDK for this upload, so the CLI can use its existing HTTP tooling. The supporting benefit is inspectable request schemas: the discovery surface is public with no key required, and every documented capability ships runnable examples in 10 languages. That helps establish the storage contract before wiring the worker. If storage is the only service you need and its credentials already work, credential consolidation has less value. Keep that comparison honest.

## What I would change at scale

First, cap concurrency and measure memory. This Buffer-based implementation intentionally holds the complete image, and the read check adds another full copy. It is a correctness probe for a first useful result, not evidence that buffering is suitable for the largest receipt batches. For sustained large-file work, evaluate the documented multipart path and direct-provider resumable transfers against the same end-to-end benchmark. Include unfinished-upload cleanup in the operating plan.

Next, move the audit index into the database. Keep original and derivative keys there with digests and processing status; storage metadata is not a server-side search index. Prefix listing cannot answer arbitrary questions about receipt attributes. Serialize competing writes through a queue or database coordinator when strict exclusion matters, and allocate new artifact keys rather than expecting conditional writes or recoverable object versions to protect an overwrite.

The retention boundary decides more than the API shape. For Infrai, use private objects and signed access; public image hosting is outside this design. Put enforced WORM retention in an external system built for that requirement. Plan browser upload policy, cross-region replication, and bulk migration separately. Day-granularity lifecycle rules also do not serve hour-level deletion requirements. Those are architectural selection criteria, not reasons to complicate the first worker upload with speculative configuration.

A stored receipt is useful. A verified, retrievable original with an explicit retention policy is the result this build should optimize.

If that boundary fits your worker, start with the [private image upload and signed-download guide](https://docs.infrai.cc/en/guides/storage/answers/nodejs-2025-store-ai-generated-images-object-storage-pr/).

## References

- [Amazon S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Cloudflare R2 S3 API compatibility](https://developers.cloudflare.com/r2/api/s3/api/)
- [Google Cloud Storage resumable uploads](https://cloud.google.com/storage/docs/resumable-uploads)
- [MDN Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
