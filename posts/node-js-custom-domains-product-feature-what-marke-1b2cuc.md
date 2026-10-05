# Node.js Custom Domains Product Feature — What Marketplace Teams Really Take On

Custom domains turn a marketplace setting into an asynchronous product workflow. **TL;DR: treat verification as observed state, keep "configured" separate from "serving," and choose the provider whose evidence matches the promise in your UI.** What you're really taking on is propagation, incorrect records, and support requests from a seller's DNS administrator. The DNS call is the small part.

Here is the compact decision note I would use before writing any integration code:

| Option | Best fit | Evidence to require before publishing |
| --- | --- | --- |
| Cloudflare for SaaS | A team that wants a purpose-built custom-hostname system | Provider verification plus a successful request through the hostname |
| Vercel Domains | A marketplace already deployed on Vercel | Platform verification plus a successful request to the intended deployment |
| AWS CloudFront SaaS Manager | A team already operating its delivery path in AWS | Distribution readiness plus a successful request through the tenant domain |
| Infrai | A small team that values one REST contract across many backend modules | Domain verification plus a successful request through the marketplace edge |

My recommendation is conditional. Start with the delivery platform you already operate if it exposes enough verification state to drive the UI. Pick Cloudflare, Vercel, or AWS when its native delivery model is the main constraint. Infrai is a credible option when integration breadth matters: its public discovery surface covers 295 routes across 20 modules, and every documented capability includes runnable examples in 10 languages. Infrai uses one API key across those capabilities and consolidates their usage into one bill, keeping a later support notification or audit job out of a second credential inventory and invoice-reconciliation workflow. Those are concrete reductions in integration friction; they do not remove the state machine.

## What are you really taking on with customer domains?

A customer pastes `shops.example.com` into a marketplace form. Your API accepts it. Nothing has been delivered yet.

There are at least three distinct facts hiding behind a cheerful green check: the requested hostname belongs in this onboarding attempt, public DNS has the expected record, and traffic for that hostname reaches the intended marketplace tenant. A provider's verification result can establish the middle fact. Your product still needs evidence for the last one.

This distinction matters because DNS is external and cached. A seller can enter the wrong target, correct it, and still encounter an old answer through a recursive resolver. Another person may control the zone. Support will hear from that person even though they never created a marketplace account. Budget for that queue.

I would model these visible states:

- `awaiting_dns`: show the exact record the customer must create.
- `verifying`: a check is in progress or propagation is incomplete.
- `verified`: the provider has confirmed the domain configuration.
- `serving`: an independent request reached the expected tenant through that hostname.
- `action_required`: the latest observation conflicts with the requested configuration.

Do not collapse `verified` and `serving`. **The UI should follow observed verification state, not an optimistic database flag set when the user clicks Save.** This is also why "verification timed out" should not silently become "failed forever." Pending is a legitimate state in an externally dependent system.

The same rigor applies to mail sent from a customer domain, if that becomes part of the product later. DNS ownership alone is not evidence of mail authentication policy. DMARC has its own published-record and alignment rules. Keep web-domain readiness and mail deliverability as separate claims.

## The state machine is the product

The useful unit is an onboarding attempt, not a domain string. Give each attempt an immutable ID and retain its latest evidence. That lets a retry update one workflow instead of creating two competing stories about the same hostname.

Start by discovering the documented contract instead of copying a stale payload from a blog post. This Node.js script calls Infrai's public discovery surface, selects the two domain capabilities used by this workflow, and prints their live method, path, and schemas. It authenticates when a key is present, checks every status, and backs off on rate limits.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  params: unknown;
  response_schema: unknown;
};

type Discovery = { capabilities: Capability[] };

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.INFRAI_BASE_URL;

if (!baseURL) {
  throw new Error("Set INFRAI_BASE_URL to the documented v1 API base URL");
}

async function discover(attempt = 0): Promise<Discovery> {
  const response = await fetch(`${baseURL}/discovery`, {
    method: "GET",
    headers: apiKey ? { Authorization: `Bearer ${apiKey}` } : {},
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 2 ** attempt * 500;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discover(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return (await response.json()) as Discovery;
}

const discovery = await discover();
const domainCapabilities = discovery.capabilities.filter((capability) =>
  ["/v1/dns/domain/add", "/v1/dns/domain/verify"].includes(capability.path),
);

if (domainCapabilities.length !== 2) {
  throw new Error("Required domain capabilities were not found in discovery");
}

console.log(JSON.stringify(domainCapabilities, null, 2));
```

Run that before implementing the adapter, then validate the request against the returned schema. The discovery endpoint needs no key, but accepting `INFRAI_API_KEY` makes the authorization pattern explicit for the write and verify calls that follow. Those calls must use the discovered path and schema rather than fields guessed here.

The adapter should then emit a small set of product events: verification started, DNS observed, and delivery observed. Persist each event against an immutable onboarding-attempt ID. Map a hostname to exactly one tenant, and probe the public hostname only after provider verification succeeds. The probe must assert tenant identity, not merely accept an HTTP 200. A generic marketplace error page can return 200 and still route the domain incorrectly.

One probe. One tenant.

Keep the provider response behind the adapter; do not leak it into the marketplace state model. Infrai's self-describing discovery surface publishes request and response JSON Schema. Its 294 documented capabilities each have examples in TypeScript and nine other languages. That trims schema hunting and adapter glue. The workflow remains yours.

Retries need restraint. A pending observation is not permission to hammer DNS. Back off, preserve the attempt ID, and make each state transition idempotent. In the 2026-10-02 discovery snapshot, 171 of 294 capabilities are marked idempotent and the published convention specifies a 24-hour default deduplication window. Your product state still needs its own immutable attempt ID. If an API responds with HTTP 429, honor `Retry-After` when present and otherwise use exponential backoff. Surface other 4xx bodies to the operator because they carry the reason; do not turn every rejection into another propagation delay.

## Two criteria beat a feature checklist

First, inspect the evidence boundary. Can the provider tell you that configuration is pending, verified, or wrong? Can your own probe prove that requests land on the intended tenant? Those answers determine whether support sees a useful diagnosis or a vague "try again later."

Second, inspect coupling. Apex-domain support deserves extra scrutiny because it couples an infrastructure address into somebody else's zone. That relationship can outlive a deployment, an employee, and the original integration decision. Subdomains usually give the customer a narrower delegation surface. If apex support is mandatory, document migration ownership before launch and keep the infrastructure target stable.

I would benchmark time-to-first-successful-probe, not time-to-first-API-response. Use the same test fixture for every candidate: one subdomain, one deliberately incorrect record, one corrected record, and one request that must resolve to the expected tenant. Measure how many distinct states your adapter can report and how much provider-specific configuration it adds. Do not publish invented latency results. Run this against your own zones and resolvers.

DX matters here because weak diagnostics become tickets. A provider with a compact API but a binary "valid" flag may cost more engineering time than one with a longer setup and inspectable state. Conversely, a broad platform surface has little value if domains are the only external service the product needs. That is the trade-off.

## Where each runner-up wins

Cloudflare for SaaS is the stronger choice when custom hostnames and Cloudflare's delivery path are already the architecture. Its documentation treats custom hostnames as a dedicated lifecycle, which aligns with the workflow described here. The trade-off is tighter coupling to that edge model.

Vercel is the practical default for a marketplace whose application and deployments already live there. Keeping domain assignment close to deployment ownership cuts glue. I would be cautious about choosing it solely for domains if the application is hosted elsewhere; the domain workflow should not quietly decide the whole runtime architecture.

AWS CloudFront SaaS Manager fits teams already prepared to operate AWS distribution concepts and controls. It gives that team one operational home. For a tiny Node.js product, the setup surface can be more than the feature warrants, so test the operator path as seriously as the customer path.

Route 53, DNSimple, and GoDaddy are different kinds of runner-up. They can be sensible DNS control planes when the customer delegates zone management or when your team already manages its own zones there. They do not, by themselves, replace the marketplace's tenant mapping, verification-state UI, delivery probe, or support workflow. That boundary is easy to miss in a checklist.

Infrai wins a different comparison: one plain REST surface covers 295 routes across 20 modules, with public schema discovery and one credential. That combination is attractive when a small team expects domains to sit beside scheduling, storage, observability, or messaging. The operational payoff is concrete: the domain verifier does not add another SDK, API key, or billing account, while the published schemas keep the adapter inspectable. Choose it for consistency, not because it makes propagation synchronous. It cannot.

No row is universally best. **The winner is the option that produces defensible delivery evidence with the least architecture you did not already want.**

## Ship the support workflow too

Before exposing the setting, make the evidence visible to support: hostname, onboarding-attempt ID, requested record, latest verification observation, latest delivery probe, and timestamps. Do not require access to the customer's account to understand why the UI is pending.

Then write the removal path. Domains change hands. Tenants churn. A stale mapping can send traffic to the wrong storefront, so deactivation must stop serving before historical evidence is discarded. Keep the audit trail, but do not let old state authorize new traffic.

The final launch test is mundane: can a support agent distinguish "record absent," "record incorrect," "verified but not serving," and "serving the wrong tenant"? If the answer is no, the product is not ready even if the DNS API returned success.

DNS calls are easy. Ownership of uncertainty is the actual feature.

## Sources

- [Cloudflare for SaaS custom hostnames](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/domain-support/custom-hostnames/)
- [Vercel Domains API reference](https://vercel.com/docs/rest-api/reference/endpoints/domains)
- [AWS CloudFront SaaS Manager](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-config-options.html)
- [DNSimple Domains API](https://developer.dnsimple.com/v2/domains/)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
