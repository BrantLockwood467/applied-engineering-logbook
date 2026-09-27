# 3 E-commerce Tenant Boundaries Across Public CDN and Signed Storage URLs

TL;DR: Use a stable public CDN URL for an avatar only when the merchant intends that image to be public. Keep signed agreements private and issue short-lived signed URLs after authorization. Do not let those delivery choices define tenant isolation. Enforce isolation in the object key, the application authorization check, and the deletion job, then test all three boundaries.

For an e-commerce account system, “profile image” hides two different assets. A storefront avatar is published. A seller agreement happens to be an image or PDF, but it has an explicit deletion deadline and must remain tenant-scoped. Putting both behind signed URLs adds churn to public content. Putting both on public URLs leaks the document. One storage rule cannot express both jobs.

My decision rule is blunt: classify by disclosure intent first, retention second, and file extension never. The interesting question is not which URL looks cleaner in a database. It is which component can prove that tenant A cannot read, overwrite, or postpone deletion of tenant B's object.

## Should public avatars use a CDN or private storage signed URLs?

The deletion deadline changes the choice. A signed URL is a temporary capability, not a deletion mechanism. The Amazon S3 documentation says a presigned URL grants time-limited access using the permissions of the principal that created it. It can also be used multiple times until it expires. Imagine an agreement due for deletion at midnight: a worker deletes its database row, an earlier signed link remains unexpired, and the object delete then fails. The UI says the document is gone while the capability and bytes may remain. This is why one timestamp and one happy-path call do not satisfy the job. The object deletion must finish before completion is recorded, and an outstanding link window must be part of the policy decision.

URLs expire. Obligations don't.

That distinction produces three boundaries:

1. The namespace binds every private object to one tenant and one immutable object ID.
2. The read path authorizes the current principal before it creates a short-lived URL.
3. The retention worker selects due records by tenant, deletes the object, and records the outcome.

The public avatar takes a different path. Its URL may be stable and cacheable because public delivery is the product requirement. Never overwrite the same public key. Publish `avatar/<opaque-id>/<version>.<ext>` and change the profile pointer when a new version is accepted. The old version then has a separate cleanup date. This avoids asking caches to guess whether bytes at an unchanged address have changed. It also keeps rollback understandable: the profile record points to a known version instead of relying on an overwrite becoming visible everywhere at once.

Private documents use opaque keys too, but obscurity is not authorization. A key such as `tenant/t_8f2/document/d_91c/original` is useful because every internal operation carries the tenant boundary. It is still private, and the application still checks access before signing a read.

Short-lived links reduce the window in which a copied capability works. Shorter is not automatically better. A five-second link that expires during a large download is broken DX; a link lasting several hours enlarges the exposure window. Set the duration from a measured download budget, then add a small margin. Measure it. Do not cargo-cult a number from somebody else's app.

## The smallest working implementation

I want one narrow storage interface. No provider configuration in request handlers. No bucket policy encoded in a UI component. The domain layer passes tenant-qualified keys and receives either an object operation result or a signed URL.

```ts
type AssetClass = "public-avatar" | "private-agreement";

type Asset = {
  id: string;
  tenantId: string;
  ownerId: string;
  class: AssetClass;
  objectKey: string;
  deleteAfter: Date | null;
  deletedAt: Date | null;
};

interface ObjectStore {
  putPrivate(key: string, body: Uint8Array, contentType: string): Promise<void>;
  publish(key: string, body: Uint8Array, contentType: string): Promise<string>;
  signRead(key: string, expiresInSeconds: number): Promise<string>;
  delete(key: string): Promise<void>;
}

interface AssetRepository {
  insert(asset: Asset): Promise<void>;
  findForTenant(assetId: string, tenantId: string): Promise<Asset | null>;
  findDueForDeletion(before: Date, limit: number): Promise<Asset[]>;
  markDeleted(assetId: string, deletedAt: Date): Promise<void>;
}
```

The read method accepts the authenticated tenant from server-side context. It does not trust a tenant ID from a query string. The repository lookup includes both `assetId` and `tenantId`, so a valid object ID from another shop returns the same absence as an unknown ID.

```ts
async function getAgreementDownload(
  assetId: string,
  auth: { tenantId: string; userId: string },
  assets: AssetRepository,
  store: ObjectStore,
): Promise<{ url: string; expiresInSeconds: number }> {
  const asset = await assets.findForTenant(assetId, auth.tenantId);

  if (!asset || asset.class !== "private-agreement" || asset.deletedAt) {
    throw new Error("asset_not_found");
  }

  const expiresInSeconds = 15 * 60;
  const url = await store.signRead(asset.objectKey, expiresInSeconds);
  return { url, expiresInSeconds };
}
```

Fifteen minutes above is an example policy, not a universal optimum. The response includes the duration because SDK callers need to know when to request another link. The application stores the object key, never the generated URL. Persisting a signed URL creates a stale credential cache with no useful source of truth.

Uploads need the same tenant binding. Generate the destination key on the server after authentication. Validate the declared media type and the decoded file before publication, and keep an avatar private until that validation finishes. Only then publish the versioned object and update the profile pointer. This adds a state transition, but it prevents half-processed uploads from becoming the public record.

The deletion worker is intentionally boring:

```ts
async function deleteExpiredAssets(
  now: Date,
  assets: AssetRepository,
  store: ObjectStore,
): Promise<void> {
  const due = await assets.findDueForDeletion(now, 100);

  for (const asset of due) {
    if (!asset.deleteAfter || asset.deleteAfter > now || asset.deletedAt) continue;

    await store.delete(asset.objectKey);
    await assets.markDeleted(asset.id, now);
  }
}
```

The batch size of 100 is a starting control, not a benchmark result. Retry behavior matters more than clever concurrency here. A failed delete must leave the record eligible for another attempt. Marking the database row first creates the worst failure mode: the system reports compliance while the bytes remain.

## How do you prove tenant isolation?

Happy-path upload tests prove almost nothing. The useful suite is a tenant matrix. For every operation, create asset A under tenant A, authenticate as tenant B, and assert that read, replacement, and deletion all fail without revealing whether A exists.

I would keep these cases close to the storage adapter contract:

| Case | Expected invariant |
| --- | --- |
| Public avatar read | Works without a private storage credential |
| Public avatar replacement | Creates a new versioned key |
| Private agreement read by its tenant | Returns a newly signed, time-limited URL |
| Private agreement read by another tenant | Returns no URL and no existence signal |
| Due agreement deletion | Deletes the object before recording completion |
| Failed object deletion | Leaves the record retryable |

There is one more test people skip: inspect the generated key. Assert that user input cannot inject another tenant prefix, path separators, or a complete object key. The request may suggest a filename for display. It must not choose the authorization namespace.

Benchmark the paths separately. Public avatar delivery should be measured for cache hit behavior and bytes served. Private document access should measure authorization latency, signing latency, link refreshes, download completion before expiry, deletion queue age, and deletion failures. Combining them into one “storage latency” graph hides the decision you need to make.

Logging deserves restraint. Record asset ID, tenant ID, operation, policy decision, and result. Do not log the signed URL. It is usable access material until its expiration, and query strings have a habit of spreading through logs and tracing systems.

## What I would change at scale

First, I would put deletion work behind a durable queue and make the operation idempotent. The database record needs attempt count, last error category, and next-attempt time. An alert should fire on the age of the oldest overdue object, not merely on worker process health. A running worker can still delete nothing.

Second, I would separate public and private delivery domains and credentials. That makes an accidental policy crossover easier to detect and limits how much configuration each component carries. The upload service can write quarantined objects; the publisher can expose approved avatar versions; the download service can sign reads for authorized agreements; the retention worker can delete due objects. Each role stays small.

Third, I would add a reconciliation pass. It compares due metadata with object deletion outcomes and identifies objects with no live metadata owner. The cadence should follow the promised deletion window. If the contract says an agreement is deleted within a day of its deadline, a weekly reconciliation job cannot validate that promise.

This is also where cost enters, but only as an operational signal. Storage, request, retrieval, transfer, and management charges can vary by access pattern and storage class; the S3 pricing page lists those as separate dimensions. Model them with your own object sizes, replacement rate, cache hit rate, and deletion traffic. A copied “cost per gigabyte” number does not describe this workload.

## The trade-off I would ship

Use public, versioned delivery for storefront avatars that users knowingly publish. Use private objects plus post-authorization signed reads for seller agreements. Keep the explicit deletion deadline in application metadata and run deletion as a retryable, observable workflow.

This split costs a little glue: two asset states, two delivery paths, and a retention worker. It buys a much clearer security argument. Public images cache well because they are public by design. Private documents remain tenant-scoped because every access begins with application authorization. Deletion remains a lifecycle operation rather than a side effect of URL expiry.

The deciding test is simple. If a copied URL escapes, would its continued use violate the asset's disclosure policy? If yes, do not publish a permanent URL. If no, forcing every render through a signing call adds machinery without strengthening the intended boundary.

## Further reading

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://aws.amazon.com/s3/pricing/
