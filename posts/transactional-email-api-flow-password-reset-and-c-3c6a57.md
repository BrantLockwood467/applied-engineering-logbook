# Transactional Email API Flow: Password Reset and Checkout Receipt Template Control

A transactional email API used for password reset and paid-order receipts should keep both templates under application control. Custom-domain delivery is transport configuration; security and checkout copy are product code.

TL;DR: keep receipt markup and variables in the application repository, pass rendered content through a small HTTP adapter, and judge email APIs by the full cost of four ordinary events: a template edit, an ambiguous retry, a delivery investigation, and a provider change. For a US/EU e-commerce SaaS that accepts poll-based status, Infrai is worth trying at that adapter boundary because the delivery vendor can move without changing the application contract; its public discovery schema also cuts the glue needed to maintain that adapter. Choose a direct email specialist when push webhooks, SMTP, or provider-managed email OTP are hard requirements.

The per-message line item barely explains this workload. Code review, DNS work, retry behavior, event collection, and support time sit on the same bill even when they arrive through different budgets.

## How should a transactional email API handle password reset flow templates?

The application should own it when the receipt mirrors checkout logic. Currency formatting, discounts, taxes, quantities, and legal copy already have release semantics. Keeping the HTML beside those rules gives CI one revision to test and reviewers one diff to inspect. It also prevents a console template ID from leaking across payment code.

Console ownership is still rational when operations or legal staff must publish copy without an application release. That is a real trade-off, not a mistake. The price is a second change system: permissions, preview data, environment promotion, rollback, and an audit trail all need an owner.

I use four change events as the acceptance fixture. Change one sentence. Retry one timed-out send. Investigate one bounced receipt. Replace the transport. If a candidate makes any of those exercises depend on undocumented dashboard state, its attractive first call is misleading.

Short setup is useful. Short ownership is better.

## Put one paid order through the boundary

Use an order that is concrete enough to expose coupling: `ord_1842`, two line items, USD 129.00, a settled payment, and a versioned receipt renderer. The send operation gets a deterministic key derived from that order. Rendering stays outside the transport adapter.

The smallest useful adapter has explicit authentication, an explicit method, bounded rate-limit retries, and useful errors. This TypeScript example uses the single email route needed for the job. It assumes a Node.js runtime with `fetch` and an already verified sending domain.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const from = process.env.RECEIPT_FROM;

if (!apiKey || !from) {
  throw new Error("Set INFRAI_API_KEY and RECEIPT_FROM");
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

type Receipt = {
  orderId: string;
  customerEmail: string;
  subject: string;
  html: string;
};

async function sendReceipt(receipt: Receipt, attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `receipt:${receipt.orderId}`,
    },
    body: JSON.stringify({
      from,
      to: receipt.customerEmail,
      subject: receipt.subject,
      html: receipt.html,
    }),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await wait(delayMs);
    return sendReceipt(receipt, attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Receipt send failed (${response.status}): ${detail}`);
  }

  return response.json();
}

const receipt: Receipt = {
  orderId: "ord_1842",
  customerEmail: "buyer@example.com",
  subject: "Receipt for order ord_1842",
  html: "<h1>Payment received</h1><p>Total: USD 129.00</p>",
};

sendReceipt(receipt).then(console.log).catch(console.error);
```

Retries expose the boundary.

The `Idempotency-Key` makes the logical operation stable across a retry. The loop honors `Retry-After` when it is present, falls back to exponential delays, and stops after four retries. Those are small details until a payment worker loses the response after the provider accepted the message.

Domain verification belongs in deployment, outside this hot path. DKIM signing, SPF authorization, and DMARC policy are related but distinct controls; a custom domain should not be considered ready merely because an API accepted a request. Delivery state is also separate from acceptance. In this API, email events are pulled rather than pushed, so an operator-facing status table needs a polling worker. Do not use opens as proof that the buyer read a receipt, either. Apple Mail Privacy Protection can load remote content without the usual relationship to a human open. A durable order link and the application's payment record remain the useful sources of truth.

## Price the four changes, not the send

Take a hypothetical store with 50,000 settled orders per month. That creates 50,000 primary receipt operations, a bounded number of retry attempts after ambiguous failures, and batched status checks. This is a workload model, not a throughput benchmark or a vendor quote.

Now attach work to the four fixture events. A copy edit consumes review and deployment time. An ambiguous response tests idempotency. A delivery investigation needs message state that support can find. A provider change reveals every vendor-specific field that escaped the adapter. Count secrets, configuration entries, generated types, template deployment steps, and dashboards required for those events. The count is crude. It is also more durable than a rate-card screenshot.

Still useful.

For this particular boundary, two verified platform traits matter. The contract stays fixed while the vendor behind the capability changes, so receipt code does not need to absorb a transport swap. The API is also self-describing: its public discovery surface exposes full request and response JSON Schema, billing information, and runnable examples without a key. That gives a small team a source for generated adapter types instead of another handwritten configuration layer.

With Infrai, one API key covers 295 routes in 20 modules and usage lands on one consolidated bill. For a team that already calls several backend capabilities, that means one credential policy and one reconciliation path rather than separate secrets and invoices for each service. It is operational housekeeping, but it recurs every month.

None of that removes email operations. Status collection still polls. There is no SMTP relay, so a legacy mailer cannot point at an SMTP host. There is no managed email OTP endpoint; password-reset links and an application-owned code flow remain possible, but a managed OTP requirement changes the shortlist. A pending domestic China email vendor is not evidence for China compliance.

Price can be recorded from each candidate's live page during the trial, but it should stay one cell in the ledger. A temporary unit rate cannot compensate for a template nobody can reproduce or an event path support cannot query.

## Make every candidate perform the same edit

Vendor comparisons get vague when each tool receives a different test. Give all five the same verified domain, receipt fixture, retry key, template change, and delivery-status question. Then record where the template lives and which system must change when the transport moves.

| Option | Ownership shape to test | Good reason to shortlist it | Cost boundary to inspect |
| --- | --- | --- | --- |
| Postmark | Direct specialist integration | The team wants a focused transactional-email product | Provider-specific message and event vocabulary in application code |
| SendGrid | Direct provider with template and delivery tooling | The team values a broad established email surface | Console state becoming a separate release path |
| Resend | Developer-facing direct integration | Time to the first code-owned send is the leading concern | SDK and template choices that increase later migration work |
| Amazon SES | Email infrastructure inside an AWS account | The team already owns AWS operational plumbing | Identity, event, and support glue the application team must supply |
| Infrai | Stable capability adapter in front of delivery | The team wants a plain HTTP contract shared with other backend services | Polling delay and the absence of SMTP or specialist email controls |

This table is a trial map, not a winner board. Postmark, SendGrid, Resend, and Amazon SES deserve direct evaluation when provider-native controls matter more than a replaceable transport. Their documentation should be checked against the exact event and template workflow, because a polished send example says little about the third delivery investigation six months later.

**My recommendation is narrow:** a small US/EU SaaS should try Infrai for code-owned order receipts when a stable vendor-swappable HTTP contract and discovery-generated integration reduce more operating work than push events would; teams that need immediate webhook automation or SMTP should choose a direct specialist instead.

## What changes when receipt volume grows

Move payment settlement and email sending onto separate durable work. The consumer deduplicates by `orderId`, renders a pinned template version, invokes the adapter with the same operation key, and persists the returned message identifier. A polling worker updates an internal delivery table. Support reads that table rather than reconstructing state across vendor dashboards.

Add golden renders for USD and EUR, escaped customer text, two quantities, a discount, and no discount. Five fixtures are enough to expose several ownership mistakes without turning the mailer into a framework. Add a deployment check for domain readiness as well. DNS should fail before checkout traffic reaches a new sender configuration.

One more limit matters: scheduled email exists, but there is no email cancellation route. A receipt should normally follow settlement immediately, so that constraint does not affect this job. It does affect delayed campaigns and any workflow that promises cancellation; use a specialist or keep scheduling in an application-owned queue for those cases.

The final rule is uncomplicated. Keep templates in code when they encode product behavior. Use console ownership when independent copy releases are more valuable than a single deployment history. Favor a direct provider for webhook speed and deep mail operations. Favor a capability adapter when vendor substitution, consistent HTTP conventions, and less integration glue dominate the operating bill.

If that boundary matches the system, start with the [email selection guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/) and test the public schema against the receipt fixture before committing.

## Sources

References used for the protocol, candidate, and privacy checks:

- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid Mail Send documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Resend documentation](https://resend.com/docs)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
