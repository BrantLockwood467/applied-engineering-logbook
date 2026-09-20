# Node.js Welcome Email API: Gaming Onboarding Deliverability and Bounce Suppression

A player can retry a signup in seconds, but a mailbox can remain bad for months. That constraint changes the design: keep welcome-email templates in the application repository, treat the email API as a transport, and put bounce-driven suppression in front of every send path.

TL;DR: choose an API only after a thin Node.js adapter proves four things: it accepts your rendered message, authenticates your domain with DKIM and SPF, delivers signed event callbacks, and gives you enough bounce detail to maintain one application-level suppression list. Template ownership is the deciding axis. It determines how easily you can review copy, test rendering, switch transports, and keep game-specific state out of a provider dashboard.

## How should an email API handle onboarding welcome message deliverability?

A gaming welcome message is application behavior. It can depend on locale, age gate, region, platform, and whether the new account was created directly or linked from a console identity. Those rules already live beside code. Moving the final template and its variables into a remote dashboard creates a second deployment system, a second permission model, and an easy way for code and copy to disagree.

So I would start with repository-owned templates. The application renders a complete subject, HTML body, and plain-text body; the transport gets an envelope plus that rendered content. A provider-hosted template can still be reasonable when non-developers must publish copy independently. The price is coupling: template identifiers, variable semantics, preview behavior, and rollout history become part of the integration. That is not automatically bad. It should be a conscious ownership decision.

This keeps the selection exercise honest. A polished editor is irrelevant if the API obscures permanent failures or makes event verification awkward. A sparse API can still fit when the repository already has review, localization, and rendering tests. I care about the path from `player.created` to the first accepted send, but I care more about the path from `recipient.invalid` to every future send being blocked.

That failure is durable.

Consider the full retry path. A player submits an address, the account service commits the player record, and the welcome job renders the current repository template. The transport accepts it. Later, a signed callback reports a permanent mailbox failure, so the event consumer records a suppression. The player presses resend in the game client, then a support tool triggers another welcome message after reviewing the account. Both attempts must stop at the same suppression lookup before either transport call. If the suppression lives only in the first job, the support path bypasses it. If it lives only in the transport dashboard, a later transport migration forgets it. This sequence is why template ownership and recipient state should be separate decisions: the repository owns what to say, while an application-level store owns whether this address may be contacted. The adapter joins those decisions at send time and nowhere else.

Email authentication is separate from template ownership. SPF publishes which systems may send for a domain. DKIM signs selected message content so a receiver can validate it. DMARC lets a domain publish policy and reporting instructions based on identifier alignment. Passing one does not imply that the others are configured correctly; inspect receiver-visible authentication results during testing instead of treating a dashboard checkmark as proof.

## The smallest useful boundary

The adapter is deliberately boring. There is one send method, one event shape, and no provider template ID.

This is the whole contract.

```ts
type WelcomeMessage = {
  recipient: string;
  locale: string;
  playerId: string;
};

type DeliveryEvent = {
  id: string;
  recipient: string;
  kind: "delivered" | "transient_bounce" | "permanent_bounce" | "complaint";
  occurredAt: string;
};

interface MailTransport {
  send(input: {
    to: string;
    subject: string;
    html: string;
    text: string;
    metadata: Record<string, string>;
  }): Promise<{ messageId: string }>;

  verifyEvent(rawBody: Uint8Array, signature: string): DeliveryEvent;
}

interface SuppressionStore {
  has(recipient: string): Promise<boolean>;
  add(input: { recipient: string; reason: string; eventId: string }): Promise<void>;
}
```

That interface is the benchmark harness. Time how long it takes to implement against a candidate API, but record the work, not a vanity requests-per-second number. Count required configuration keys, provider-specific types that escape the adapter, manual dashboard steps, and fixtures needed to test a bounce. Fewer moving parts usually mean a cleaner first call. They do not excuse weak event handling.

Rendering stays local and deterministic. The example escapes player-controlled text because welcome data should never become markup by accident. In a real codebase, use a maintained escaping or templating library rather than expanding a home-grown renderer.

```ts
const escapeHtml = (value: string): string =>
  value.replace(/[&<>"]/g, (character) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    '"': "&quot;"
  })[character] as string);

function renderWelcome(input: WelcomeMessage) {
  const player = escapeHtml(input.playerId);
  return {
    subject: "Your game account is ready",
    text: `Account ${input.playerId} is ready. Open the game to continue.`,
    html: `<p>Account <strong>${player}</strong> is ready.</p><p>Open the game to continue.</p>`
  };
}

async function sendWelcome(
  input: WelcomeMessage,
  mail: MailTransport,
  suppressions: SuppressionStore
): Promise<"sent" | "suppressed"> {
  const recipient = input.recipient.trim().toLowerCase();
  if (await suppressions.has(recipient)) return "suppressed";

  const rendered = renderWelcome(input);
  await mail.send({
    to: recipient,
    ...rendered,
    metadata: { messageType: "welcome", playerId: input.playerId }
  });
  return "sent";
}
```

Do not mistake `lowercase()` for full mailbox canonicalization. The local part of an address is defined as case-sensitive in SMTP, even though many deployed systems behave otherwise. Choose and document an identity rule for your account system; do not silently remove dots or plus tags based on assumptions about one mailbox operator.

## Bounce handling is a state transition

An accepted API request is not delivery. SMTP can produce a temporary failure, a permanent failure, or an eventual delivery after retries. Enhanced status codes make those outcomes more precise, but the application still needs a conservative mapping from transport events to recipient state.

The useful rule is small: a verified permanent bounce or complaint suppresses future sends; a transient bounce updates observability but does not immediately mark the address invalid. Exact classifications vary by transport, so normalize them inside the adapter and retain the original event for diagnosis. Never let transport-specific event names leak into game logic.

Events can be duplicated or arrive out of order. Use the event ID as an idempotency key, store the event timestamp, and make suppression monotonic for permanent failures. A later `delivered` event for an older message must not casually clear a newer complaint. Recovery needs an explicit path, such as a player changing and verifying the address.

```ts
async function applyDeliveryEvent(
  rawBody: Uint8Array,
  signature: string,
  mail: MailTransport,
  suppressions: SuppressionStore,
  seen: { claim(eventId: string): Promise<boolean> }
): Promise<void> {
  const event = mail.verifyEvent(rawBody, signature);
  if (!(await seen.claim(event.id))) return;

  if (event.kind === "permanent_bounce" || event.kind === "complaint") {
    await suppressions.add({
      recipient: event.recipient.trim().toLowerCase(),
      reason: event.kind,
      eventId: event.id
    });
  }
}
```

Verify the callback signature against the exact request bytes before parsing. Apply a timestamp or replay window if the transport's signing scheme defines one. Return success only after the event is durably claimed or queued; otherwise retries can create gaps that are hard to reconstruct later.

One list should guard all workers and all message types. A suppression table hidden inside the welcome-email job is a trap because a separate retention campaign may continue sending to the same invalid mailbox. The transport may also maintain its own list. Keep that protection enabled, then mirror actionable events into the application-level list so the rule survives a migration.

## Tests that expose a bad fit

I would not begin with an inbox screenshot. Start with deterministic tests around the boundary. A candidate API is usable only if its sandbox or fixtures let the adapter prove these cases without editing production data by hand.

Measure the boundary.

- A suppressed address causes zero transport calls.
- The same permanent-bounce event received twice creates one durable transition.
- A transient bounce does not enter permanent suppression.
- An invalid signature changes no state.
- An older delivery event cannot clear a newer suppression.
- HTML and text render from the same reviewed template inputs.

Then run a controlled end-to-end check with a domain dedicated to the test environment. Inspect the received message for DKIM validation, SPF evaluation, and DMARC alignment. Record the message identifier and correlate it with the callback. Do this for each deployment environment because DNS, return-path configuration, and signing selectors can differ even when application code is identical.

Keep promotional consent out of the welcome transaction. If a message mixes onboarding with marketing, classification and unsubscribe obligations become harder to reason about. RFC 8058 defines the one-click mechanism using `List-Unsubscribe` and `List-Unsubscribe-Post`; it also requires DKIM coverage for those headers. Add it to eligible list mail, not as a cargo-cult header on every transactional message.

The gaming context may also include SMS, but email consent does not grant messaging consent. CTIA's messaging principles emphasize consumer choice, consent, and opt-out handling. Model channel consent separately, suppress independently, and do not fall back from a bounced email to SMS unless the player authorized that channel for that purpose.

## What I would change at scale

The synchronous lookup is fine for the first implementation. At higher volume, I would move event ingestion onto a durable queue, partition workers by recipient hash, and make the suppression write idempotent at the database constraint. The send path still performs a current suppression read. Queueing improves burst handling; it does not remove the consistency requirement.

I would split reputation domains or subdomains by message class only when the operational team can own the extra DNS, monitoring, and key rotation. Configuration multiplies fast. Every additional stream needs explicit SPF coverage, DKIM selector management, DMARC observation, alerting, and a tested migration plan. Config bloat is operational debt, even when each checkbox looks harmless.

Watch rates rather than isolated events: attempted sends, locally suppressed sends, accepted requests, deliveries, transient bounces, permanent bounces, complaints, event verification failures, and callback lag. Break them down by message type and deployment, while keeping recipient addresses out of metric labels. Alerts should point to a stage in the pipeline. `Delivery dropped` is vague; `verified permanent-bounce events stopped updating suppression state` is actionable.

The trade-off remains straightforward. Repository-owned templates optimize code review, deterministic tests, and transport portability. Dashboard-owned templates optimize independent copy publishing. Pick the owner first, then reject any API that cannot support authenticated sending, verifiable events, and exportable suppression state around that choice. No editor can repair a broken feedback loop.

## Sources

- https://www.rfc-editor.org/rfc/rfc5321
- https://www.rfc-editor.org/rfc/rfc3463
- https://www.rfc-editor.org/rfc/rfc7208
- https://www.rfc-editor.org/rfc/rfc6376
- https://www.rfc-editor.org/rfc/rfc7489
- https://datatracker.ietf.org/doc/html/rfc8058
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
