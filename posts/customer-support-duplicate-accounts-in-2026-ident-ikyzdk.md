# Customer Support Duplicate Accounts in 2026: Identity Resolution Through Email Lookup

To trace duplicate accounts through identity resolution and email lookup in customer support, contain the stolen session before anyone tries to tidy up customer records.

Short answer: trace duplicate accounts by using a verified email to produce opaque candidates, revoke the stolen session, rotate its refresh-token family after reauthentication, and resolve identity only when evidence beyond an email match supports the link.

Email lookup is discovery. It isn't identity proof. Mixing those jobs creates an account-enumeration surface for bots and gives a rushed support agent too much authority in one click. The practical design is a small investigation ledger with separate commands for containment, candidate lookup, and identity linking. Measure each command independently, because a 40 ms lookup means nothing if a stolen session stays active.

## How should customer support trace duplicate accounts through identity resolution and email lookup?

Treat the workflow as three decisions with different proof requirements. First: which login subject may be compromised? Second: which records are plausible duplicates? Third: does the evidence authorize linking those records? Only the second question is answered by an email lookup, and even there the result is a candidate set rather than a verdict.

The distinction matters because an email address is both useful and dangerous. An exact match can narrow the search, but a public response that changes when an account exists can disclose membership. OWASP recommends generic authentication responses so differences in wording, status, or processing behavior don't become an enumeration signal. Apply the same discipline to duplicate-account discovery: untrusted callers receive one fixed response shape, while authorized support tooling receives opaque case references under a separate permission boundary.

Here is the decision table I would put in the design review. It is deliberately stingy with authority.

| Stage | Input | Output | What it must not authorize |
| --- | --- | --- | --- |
| Contain | Authenticated compromise report, subject reference | Revoked session set, recorded security event | Account linking or data movement |
| Discover | Canonical value from a verified email | Opaque duplicate candidates | Ownership transfer or token issuance |
| Prove | Reauthentication or an organization-approved recovery decision | Evidence decision tied to a case | Broad searches for other customers |
| Resolve | Approved evidence decision and candidate references | Reversible identity link | Destructive record merging by default |

Order is the control. Contain, discover, prove, resolve.

Don't let the support UI collapse those verbs into a single “merge account” action. A bot with a stolen operator session should encounter scoped permissions, generic outward responses, and a lookup rate policy before it can turn a list of addresses into a customer directory. A legitimate agent should see why a candidate was produced, which evidence rule is still unmet, and which security action has already completed. That extra state is useful config. A dozen unrelated feature flags are not.

## Build the investigation ledger before the lookup

The smallest useful data model preserves facts without pretending they settle identity. A login subject, a customer identity, a verified-email lookup key, and a resolution case are separate records. Two subjects can remain attached to two customer identities while a case is open. That temporary ambiguity is healthy; forcing a one-to-one model too early hides duplicates or encourages irreversible merges.

Store a keyed digest for lookup rather than using a plain email as an index exposed to support code. The source value should come from the same canonicalization policy used when the address was verified. Don't quietly lowercase, strip punctuation, or apply provider-specific mailbox rules inside the investigation handler. Those transformations can change equivalence, and I'm not sure a universal normalization rule exists that is correct for every mail system. The uncertainty is resolved locally: document one verification-time policy, version it, and use its canonical result for both index writes and reads.

The ledger also needs provenance. A candidate record can say that lookup-key version `2` matched subjects `sub_41` and `sub_93`; it cannot say those subjects have the same owner. An evidence decision records the rule used, the authorized actor, and the time. A security event records containment separately. This is more paperwork than one mutable `merged: true` field — and that is exactly why an investigator can later reconstruct what happened without reading secrets from logs.

Keep credentials out of it.

Refresh tokens, raw email addresses, and recovery answers do not belong in the case timeline. The timeline needs references and decisions, not reusable access material. Log enough to answer who authorized a link and whether revocation completed, while keeping the actual replacement token on the authenticated delivery path.

## Walk one stolen-session report through the commands

Suppose `sub_41` reports a stolen browser session, and a protected lookup later produces `sub_93` as a possible duplicate. The first durable write is the compromise event. Session revocation follows against the subject named by that report. Reauthentication then gates issuance of a replacement refresh-token family. The duplicate investigation can wait; containment cannot.

Why not link first? Because linking changes the identity graph while security code is deciding which sessions belong inside the revocation set. The safer invariant is narrower: all sessions selected by the compromise command stop being accepted before a replacement family becomes usable. Make that command idempotent so a support-tool retry does not create another active family. A dropped connection should lead to the same recorded result for `cmd_7f3`, not a second credential.

This TypeScript sketch shows the boundary. It is an internal interface, not a public API route.

```ts
import { createHmac } from "node:crypto";

type SubjectId = string;
type CandidateId = string;
type TokenFamilyId = string;

type DuplicateCandidate = {
  candidateId: CandidateId;
  subjectId: SubjectId;
  lookupKeyVersion: number;
};

type InvestigationStore = {
  markCompromised(commandId: string, subjectId: SubjectId): Promise<void>;
  revokeSessions(commandId: string, subjectId: SubjectId): Promise<void>;
  issueReplacementFamily(
    commandId: string,
    subjectId: SubjectId,
    previousFamilyId: TokenFamilyId
  ): Promise<{ familyId: TokenFamilyId; refreshToken: string }>;
  findCandidates(keyVersion: number, digest: string): Promise<DuplicateCandidate[]>;
};

function makeLookupDigest(canonicalVerifiedEmail: string, secret: string): string {
  return createHmac("sha256", secret)
    .update(canonicalVerifiedEmail)
    .digest("hex");
}
```

There are no merge methods in that interface. Good. The lookup function has no session authority, and the containment methods have no email-search authority. A separate resolver can require an approved evidence decision before it creates a reversible link between subjects and a customer identity.

The command coordinator stays plain:

```ts
type ContainmentCommand = {
  commandId: string;
  subjectId: SubjectId;
  previousFamilyId: TokenFamilyId;
  reauthenticationApproved: boolean;
};

async function containAndRotate(
  store: InvestigationStore,
  command: ContainmentCommand
): Promise<{ familyId: TokenFamilyId; refreshToken: string }> {
  if (!command.reauthenticationApproved) {
    throw new Error("reauthentication_required");
  }

  await store.markCompromised(command.commandId, command.subjectId);
  await store.revokeSessions(command.commandId, command.subjectId);
  return store.issueReplacementFamily(
    command.commandId,
    command.subjectId,
    command.previousFamilyId
  );
}

async function discoverDuplicates(
  store: InvestigationStore,
  canonicalVerifiedEmail: string,
  lookupSecret: string
): Promise<DuplicateCandidate[]> {
  const keyVersion = 2;
  const digest = makeLookupDigest(canonicalVerifiedEmail, lookupSecret);
  return store.findCandidates(keyVersion, digest);
}
```

The `reauthentication_required` branch is internal behavior. A customer-facing authentication surface should still use a generic response where a more specific result would reveal account existence. Internally, operators need precise reason codes; externally, precision can become an oracle. The separation sounds fussy until bot traffic arrives. Then it is the design.

Linking should remain reversible. Keep ticket provenance and account events attached to their original subjects, and let the customer identity graph express that the subjects were resolved together. A destructive row merge erases context that support and security teams may need if the decision is disputed later.

## Test abuse resistance before enabling agent actions

Start deployment in observation mode. Compute candidates from verified addresses, record only opaque case data, and do not expose candidates to agents yet. Review collision patterns under the chosen canonicalization policy. Then enable read-only investigation. Linking comes last, after the team has a written evidence rule and can audit its use.

The tests should follow attacker behavior rather than just happy-path handlers:

- Send an unknown address and a known address through the untrusted entry point; verify the response status, shape, and visible wording are the same.
- Repeat `cmd_7f3`; verify it refers to one containment result and one replacement family.
- Attempt lookup with permission to revoke sessions but without permission to discover candidates; verify the identity boundary holds.
- Open two resolution attempts for the same candidates; verify neither can bypass the required evidence decision.
- Confirm that logs contain command IDs, actors, timestamps, and decision references, but no refresh token or raw recovery secret.

I benchmark four paths separately: time from report to revocation, candidate-lookup latency, age of unresolved cases, and active refresh-token families after rotation. These aren't invented target numbers. Each team needs thresholds derived from traffic, support staffing, and risk tolerance. The point is to avoid averaging unlike work into one cheerful dashboard. Median lookup latency cannot explain an old stolen session, and case age cannot prove that a credential stopped working.

Bot controls deserve separate budgets too. Public recovery, authenticated self-service, and an operator console have different callers and different consequences. Rate limiting only by IP address is weak against distributed requests; combine the signals your risk policy permits, and make reauthentication sensitive to the action being attempted. OWASP's authentication guidance supports generic errors and reauthentication after risk events. The exact threshold is a policy choice — your mileage may vary — so test false positives with real traffic before enforcing it broadly.

At scale, add an explicit case state machine: `candidate_found`, `contained`, `proof_pending`, `linked`, or `rejected`. Permit only named transitions, require an actor and evidence reference for each, and alert on impossible combinations. A compromise event with accepted old sessions deserves attention. So does more than one replacement family for one idempotent command. Invariants beat a wall of charts.

## Where should this design stop?

The catch is recall. Exact lookup by one verified email will miss duplicates created under different addresses. Adding phone numbers, device signals, fuzzy names, or behavioral features may surface more candidates, but it also increases collection, review load, and the chance of linking two customers incorrectly. This design is not suitable for automatic consolidation when one false link can expose another customer's support history. Keep authenticated self-service linking or an organization-controlled recovery process in that case.

Manual review has limits as well. It is slower and becomes inconsistent when the evidence policy is vague. Full automation is not the automatic answer. Automate containment and candidate generation first; keep identity resolution reviewable until measured outcomes justify a narrowly defined rule.

The stopping rule is intentionally dull. Once the stolen session is revoked and a fresh token family is issued after reauthentication, the urgent security job is complete. Link the duplicate accounts only when the documented evidence threshold is met. If it isn't, preserve both records and close the case unresolved.

No guess is better than the wrong merge.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
