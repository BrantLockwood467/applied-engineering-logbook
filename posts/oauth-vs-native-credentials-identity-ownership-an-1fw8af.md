# OAuth vs Native Credentials: Identity Ownership and Session Lifecycle in Node.js

Short answer: for a B2B SaaS app adding phone one-time-code login, keep identity ownership with an OAuth provider when account recovery is the hard part, and keep a native credential path as a deliberately separate fallback. The least complex design is a short-lived OTP verification flow that creates your own session only after the identity decision is complete.

| Choice | Identity owner | Session lifecycle | Recovery posture |
| --- | --- | --- | --- |
| OAuth | External identity provider | Your app maps an authorization result to its session | Provider recovery, plus your account-linking rules |
| Native credentials | Your app and its database | You issue, rotate, revoke, and audit sessions | You own phone-number changes, backup factors, and support resets |

The table is the decision note. It is not a ranking. Ownership changes the failure mode, and recovery is where those failures become support tickets.

## What does identity ownership change in an OAuth versus native credentials flow?

OAuth moves proof of identity to an authorization server. Your Node.js service receives a redirect result, validates the state and issuer, then finds or creates a local account. The provider owns its login ceremony and much of its recovery surface. Your system still owns authorization, account linking, and the session that protects your product.

Native credentials keep the whole chain inside your boundary. A phone OTP is a credential event, not a durable identity by itself: numbers get recycled, users lose SIM access, and a support agent can be tricked. Store a salted challenge record with a 15-minute expiry and attempt counter. Bind it to a purpose such as `login` or `change_phone`, and reject a code submitted for a different purpose.

I keep the identity record and the session record separate. That small split makes it possible to disable sessions after a phone change without deleting the account. It also makes audits readable: `otp_verified`, `session_created`, and `recovery_started` are different events.

## How should OAuth and native credentials shape the session lifecycle?

Treat both options as inputs to one session policy. A verified OAuth callback and a verified OTP should produce the same internal subject, the same idle timeout, and the same revocation checks. Mixing policies is a quiet privilege bug: users who enter through the “fallback” path can accidentally receive a longer-lived session.

Here is a small TypeScript boundary. It intentionally has no vendor SDK; the adapter can call an OAuth library or your SMS gateway behind the same application contract.

```ts
type Identity = { subject: string; assurance: "oauth" | "phone_otp" };

type SessionStore = {
  create(input: { subject: string; expiresAt: Date }): Promise<string>;
  revokeForSubject(subject: string): Promise<void>;
};

export async function establishSession(
  identity: Identity,
  sessions: SessionStore,
  now = new Date(),
): Promise<string> {
  const expiresAt = new Date(now.getTime() + 8 * 60 * 60 * 1000);
  return sessions.create({ subject: identity.subject, expiresAt });
}
```

The number eight is a policy example, not a universal setting. I benchmark the complete path, including SMS delivery and callback latency, because a fast token exchange does not make a slow recovery flow usable. Your mileage may vary by carrier and region. Don't use a provider's p95 claim as your own SLO.

A practical test matrix includes an expired code, a reused code, five wrong attempts, a revoked session, a changed phone number, and an OAuth account whose email matches an existing native account. The last case must require an explicit linking decision; silently merging identities is how a plausible login becomes an account takeover.

## Where is the native path the better trade-off?

Choose native credentials when you need a self-contained recovery policy, must support an identity provider outage without changing the login contract, or have a regulated process for phone-number ownership. The price is operational work: abuse throttling, message delivery monitoring, secure support resets, and key rotation for session signing or storage.

Choose OAuth when your team cannot staff that recovery surface and the provider's account-recovery controls meet your threat model. The catch is dependency: a provider can change claims, consent behavior, or recovery requirements. Keep an immutable provider subject, record the issuer, and make unlinking require a second verified factor.

Three failure patterns deserve explicit runbooks. A recycled number can authenticate the wrong person; a lost device can strand the right person; and a support override can bypass both. Consider a user who changes a phone number, then immediately loses the old device: if the change revokes every session before the new factor is confirmed, the user needs a separate recovery route; if it leaves old sessions alive, an attacker with that device keeps access. Add a short hold for high-risk changes, require an independent factor for support recovery, and write an audit event for every approval. None is solved by adding another button. Recovery needs an independent factor, a delay for high-risk changes, and an audit trail that on-call staff can inspect.

Exactly.

## A rollout method that keeps recovery testable

Start with a feature flag for the native path and log decisions without logging OTP values. Use synthetic numbers in staging, deterministic clock injection for expiry tests, and a redacted correlation ID across the SMS provider, callback handler, and session store. Monitor completion rate, resend rate, lockouts, and recovery escalations separately; one blended success metric hides the exact path that is failing.

I once treated a `401` as proof that the credential was wrong. It turned out to be an expired session after a phone change. That distinction matters: the user needs a recovery prompt, not another OTP. Short logs. Clear states.

The final decision rule is plain: own the identity ceremony only when you are ready to own recovery for every edge case. Otherwise, delegate authentication to OAuth, normalize the result into your own subject model, and apply one session lifecycle to every entry point.

## References

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OAuth 2.0 Security Best Current Practice (RFC 9700): https://www.rfc-editor.org/rfc/rfc9700
- OAuth 2.0 Authorization Framework (RFC 6749): https://www.rfc-editor.org/rfc/rfc6749

## Further reading

- OpenID Connect Core 1.0: https://openid.net/specs/openid-connect-core-1_0.html
- NIST Digital Identity Guidelines, Authentication (SP 800-63B): https://pages.nist.gov/800-63-3/sp800-63b.html
