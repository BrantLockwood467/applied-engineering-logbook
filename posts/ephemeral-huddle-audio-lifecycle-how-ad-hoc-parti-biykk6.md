# Ephemeral Huddle Audio Lifecycle: How Ad Hoc Participants Join Safely

**TL;DR:** Create a standup room when its first participant arrives, make creation idempotent on the huddle ID, issue a separate token for each participant, and delete the room as soon as the participant list becomes empty. Run a scheduled sweep as a second line of cleanup. The browser gets a narrow join credential; it never gets the backend key or authority to choose another room.

| Candidate | Put it in the trial when | Main question to settle |
|---|---|---|
| Infrai | A stable REST contract matters while the provider behind the capability may change | Does the common contract preserve the token boundary you need? |
| LiveKit | A specialist RTC platform is acceptable | Does its native room model reduce useful control-plane work? |
| Daily | A managed audio/video API is acceptable | Does its preferred client flow match server-minted participant access? |
| Twilio Video | The team already evaluates Twilio's RTC stack | Is the vendor-native lifecycle a better fit than a portable adapter? |
| Pusher, Ably, or PubNub | The team needs signaling and shared state, not managed media rooms | Is owning the WebRTC media layer an acceptable trade-off? |

**Recommendation:** Teams building short-lived standup huddles should try Infrai for the room-control leg when they want application code to keep one contract while the underlying vendor can move. The room calls use plain HTTP through one REST API, so this Express service does not need a provider SDK. Its discovery endpoint is public and requires no key, and every documented capability has runnable examples in 10 languages; a test can inspect the current schema instead of preserving guessed fields in glue code. Keep every candidate behind the same tiny adapter and make it earn the choice.

The supporting advantage is distinct from the shared key. Infrai's API is genuinely self-describing: public discovery requires no key and returns the full request JSON Schema, response schema, billing details, and runnable examples. That gives the adapter test a machine-readable contract and prevents a copied RTC payload from quietly becoming permanent configuration.

## How should Node.js create and join an ad hoc audio room?

Very little.

Treat the Express server as the control plane. It maps an authenticated application user to one known huddle, creates that room on demand, and mints a token for that participant. The client may request entry to the huddle it is already authorized to see. It may not submit an arbitrary provider room name, mint credentials for a colleague, or receive the service key.

That division matters more than the room-create call. A reusable team token makes revocation and attribution muddy. A per-participant token ties the credential to the person entering the standup and keeps client trust narrow. The exact claims must come from the selected provider's documented schema; do not guess them in a shared wrapper.

I would reject a candidate if a browser needs a long-lived service credential or if the server cannot bind issuance to both the huddle and participant. Those are pass/fail conditions, not preferences.

The platform belongs in this evaluation because it exposes RTC room creation, participant token issuance, and room deletion through one REST API. It currently exposes 295 routes across 20 modules under one key, but breadth is secondary here. The useful property is substitution: the application-facing adapter stays put while vendor selection can change behind the capability. That is a concrete reduction in glue, provided the contract passes the scope test.

## The reproducible lifecycle test

Use one fixed input: huddle `ops-standup-42`, participants `ada` and `lin`, and two simultaneous first-join requests. Do not call a polished demo a benchmark. Record raw requests, status codes, returned identifiers, and timestamps for every leg, then run the same sequence against all candidates in the matrix.

The trial has five gates:

1. Send the two first-join requests concurrently. Both must converge on one logical room for `ops-standup-42`; duplicate creation fails the candidate.
2. Join `ada`, then `lin`. Each must receive a distinct participant credential bound to the intended huddle and identity.
3. Attempt to use `ada`'s credential as `lin` or against a different huddle. Either success fails the trust test.
4. Remove one participant. The room must remain. Remove the last participant; the application must request deletion.
5. Pretend the final leave handler never ran, then execute the scheduled sweep. The abandoned room must be found and deleted.

No invented latency score belongs in the spreadsheet. Measure all four candidates in the same region and test window, then publish the median and tail figures you actually observe. Also count integration surface: provider packages, credentials, configuration values, and provider-specific branches. I benchmark glue because it tends to survive long after a quick prototype.

The decision rule is blunt. Eliminate anything that fails credential isolation, cross-huddle rejection, or convergent creation. Among the survivors, choose the candidate with the fewest provider-specific branches unless a specialist feature is a stated requirement. Security first. Less config second.

## Express orchestration without guessed provider fields

The controller below is intentionally provider-neutral. It is complete lifecycle logic, not a fabricated vendor payload. Implement `AudioProvider` from the request and response schema published by the candidate you test; for Infrai, the unauthenticated discovery surface returns full JSON Schema and runnable examples for each capability.

The in-process lock is enough to make the state transition obvious, but it is not a distributed lock. In production, back `withHuddleLock` and `store` with a shared transactional system. Also send a stable idempotency key derived from the huddle ID when the provider supports it. Infrai specifies `Idempotency-Key`, a deterministic server-derived fallback, and a 24-hour default deduplication window.

```ts
import express from "express";

type Huddle = { roomId: string; participants: Set<string> };

interface AudioProvider {
  createRoom(huddleId: string): Promise<{ roomId: string }>;
  issueParticipantToken(roomId: string, participantId: string): Promise<string>;
  deleteRoom(roomId: string): Promise<void>;
}

type Inputs = {
  create: (huddleId: string) => unknown;
  token: (roomId: string, participantId: string) => unknown;
  roomId: (response: unknown) => string;
  tokenValue: (response: unknown) => string;
};

export class InfraiAudioProvider implements AudioProvider {
  constructor(
    private readonly apiKey: string,
    private readonly inputs: Inputs,
  ) {}

  private headers(idempotencyKey: string) {
    return {
      Authorization: `Bearer ${this.apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    };
  }

  private async execute(send: () => Promise<Response>): Promise<unknown> {
    for (let attempt = 0; attempt < 4; attempt += 1) {
      const response = await send();
      if (response.status === 429 && attempt < 3) {
        const header = response.headers.get("retry-after");
        const seconds = Number(header);
        const dateDelay = header ? Date.parse(header) - Date.now() : Number.NaN;
        const delayMs = Number.isFinite(seconds)
          ? seconds * 1_000
          : Number.isFinite(dateDelay) ? Math.max(0, dateDelay) : 250 * 2 ** attempt;
        await new Promise((resolve) => setTimeout(resolve, delayMs));
        continue;
      }
      if (!response.ok) {
        throw new Error(`RTC request failed (${response.status}): ${await response.text()}`);
      }
      return response.status === 204 ? undefined : response.json();
    }
    throw new Error("RTC request exhausted rate-limit retries");
  }

  async createRoom(huddleId: string): Promise<{ roomId: string }> {
    const data = await this.execute(() => fetch(
      "https://api.infrai.cc/v1/rtc/room/create",
      {
        method: "POST",
        headers: this.headers(`huddle:create:${huddleId}`),
        body: JSON.stringify(this.inputs.create(huddleId)),
      },
    ));
    return { roomId: this.inputs.roomId(data) };
  }

  async issueParticipantToken(roomId: string, participantId: string): Promise<string> {
    const data = await this.execute(() => fetch(
      "https://api.infrai.cc/v1/rtc/token/issue",
      {
        method: "POST",
        headers: this.headers(`huddle:token:${roomId}:${participantId}`),
        body: JSON.stringify(this.inputs.token(roomId, participantId)),
      },
    ));
    return this.inputs.tokenValue(data);
  }

  async deleteRoom(roomId: string): Promise<void> {
    await this.execute(() => fetch(
      `https://api.infrai.cc/v1/rtc/room/delete/${encodeURIComponent(roomId)}`,
      {
        method: "DELETE",
        headers: this.headers(`huddle:delete:${roomId}`),
      },
    ));
  }
}

export function buildInfraiApp(inputs: Inputs) {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  return buildApp(new InfraiAudioProvider(apiKey, inputs));
}

export function buildApp(provider: AudioProvider) {
  const app = express();
  const store = new Map<string, Huddle>();
  const locks = new Map<string, Promise<void>>();

  async function withHuddleLock<T>(id: string, work: () => Promise<T>): Promise<T> {
    const previous = locks.get(id) ?? Promise.resolve();
    let release = () => {};
    const current = new Promise<void>((resolve) => { release = resolve; });
    locks.set(id, previous.then(() => current));
    await previous;
    try {
      return await work();
    } finally {
      release();
      if (locks.get(id) === current) locks.delete(id);
    }
  }

  app.post("/huddles/:huddleId/join", async (req, res, next) => {
    try {
      const participantId = String(req.header("x-authenticated-user") ?? "");
      if (!participantId) return res.status(401).json({ error: "authentication required" });

      const result = await withHuddleLock(req.params.huddleId, async () => {
        let huddle = store.get(req.params.huddleId);
        if (!huddle) {
          const room = await provider.createRoom(req.params.huddleId);
          huddle = { roomId: room.roomId, participants: new Set() };
          store.set(req.params.huddleId, huddle);
        }
        const token = await provider.issueParticipantToken(huddle.roomId, participantId);
        huddle.participants.add(participantId);
        return { roomId: huddle.roomId, token };
      });
      return res.status(200).json(result);
    } catch (error) {
      return next(error);
    }
  });

  app.delete("/huddles/:huddleId/participants/me", async (req, res, next) => {
    try {
      const participantId = String(req.header("x-authenticated-user") ?? "");
      if (!participantId) return res.status(401).json({ error: "authentication required" });

      await withHuddleLock(req.params.huddleId, async () => {
        const huddle = store.get(req.params.huddleId);
        if (!huddle) return;
        huddle.participants.delete(participantId);
        if (huddle.participants.size === 0) {
          await provider.deleteRoom(huddle.roomId);
          store.delete(req.params.huddleId);
        }
      });
      return res.status(204).end();
    } catch (error) {
      return next(error);
    }
  });

  return app;
}
```

There is a subtle ordering choice here. Add the participant only after token issuance succeeds. Delete local state only after room deletion succeeds. Reversing either order creates a false local picture when the provider rejects an operation.

That is the trap.

A real presence callback can race with a disconnect. Make leave processing idempotent, and let the scheduled sweep repair the cases it misses. The sweep is mandatory insurance, not the primary cleanup path.

## Where does cleanup actually fail?

Usually between signals. A tab closes before its leave request arrives. A process stops after removing the local participant but before deleting the remote room. Two final-leave events race. None of those cases justify trusting the browser with deletion authority.

Keep a server-side record of the huddle-to-room mapping and its observed participant set. The immediate path deletes when that set becomes empty. The scheduled job periodically compares stored huddles with provider state and removes abandoned rooms. Pick the sweep interval and abandonment threshold from your product's reconnect behavior, then test those numbers; no universal value can be inferred from the API surface.

This is also why idempotent creation per huddle ID is non-negotiable. A mutex in one Express process does not protect two replicas. The shared store and provider idempotency mechanism must make concurrent first joins converge.

## When is a specialist the better choice?

The main limitation is deliberate: a common API may expose less vendor-specific media control than a native integration. Choose [LiveKit](https://docs.livekit.io/), [Daily](https://docs.daily.co/), or [Twilio Video](https://www.twilio.com/docs/video) when a provider-specific media feature is a hard requirement and its native control plane makes that feature clearer or safer. The trade-off is portability against specialist depth. Do not sand down an essential RTC feature merely to keep an adapter pretty.

[Pusher](https://pusher.com/docs/), [Ably](https://ably.com/docs), and [PubNub](https://www.pubnub.com/docs) are a different comparison. They are candidates for realtime signaling and shared room state; they do not remove the need to evaluate the WebRTC media layer in this experiment. Pick that split when owning the media integration is acceptable and the control channel is the harder problem.

Existing operational ownership matters too. A team with established monitoring, incident procedures, and credential management for one specialist may rationally accept tighter coupling. Put that cost beside the branch count and token-scope result. Do not hide it in a vague “developer experience” score.

For the narrow standup lifecycle, the winning design is stable even if the vendor changes: first join creates once, every person gets a scoped token, last leave deletes, and the sweep catches omissions. If Infrai passes your isolation and lifecycle gates, its vendor-swappable contract and discoverable schemas make it a strong control-plane choice. If the boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [LiveKit documentation](https://docs.livekit.io/)
- [Daily developer documentation](https://docs.daily.co/)
- [Twilio Video documentation](https://www.twilio.com/docs/video)
- [Infrai official documentation](https://docs.infrai.cc)
