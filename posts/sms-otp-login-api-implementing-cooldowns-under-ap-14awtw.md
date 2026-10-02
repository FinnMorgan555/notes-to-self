# SMS OTP Login API: Implementing Cooldowns Under Application-Owned Templates

TL;DR: An SMS OTP endpoint must refuse a resend before it contacts the delivery adapter, cap failed verification attempts, and suppress a recipient after an authoritative invalid-recipient result. Keep the application team responsible for the message template and event vocabulary. That ownership makes login behavior testable even when delivery infrastructure changes.

For an e-commerce account, the important boundary is not the SDK call. It is the transition from `eligible` to `sent`, `verified`, `expired`, or `suppressed`. Put that transition in application storage, expose it through low-cardinality metrics, and treat SMS as a restricted authenticator rather than proof that a person controls an identity forever.

Before: a login handler generates a code, sends a message, and hopes repeated requests behave. After: one state machine decides whether a code may be issued, while a narrow adapter only delivers an application-owned template.

Much clearer.

## Why should the application own the OTP template?

Template ownership fixes the contract at the point where product intent is known. The application chooses the purpose, locale, expiry wording, and stable template version. The delivery adapter receives rendered text plus an opaque attempt ID; it does not decide authentication policy. This is especially useful in commerce, where a buyer login and a seller payout login may look similar but carry different risk.

The split is a diagram in words: browser -> authentication service -> challenge store -> delivery adapter -> mobile network. Delivery status travels back to the authentication service, which updates suppression state and emits an event. Verification never asks the delivery adapter whether the code is correct.

There is a real trade-off. Central ownership adds translation and review work to the application team. In return, copy changes, telemetry names, and security behavior travel together. A provider-owned template may reduce initial wiring, but it moves a user-facing authentication artifact across the boundary and can make replacement tests less representative. For this system, consistent behavior wins.

## How should a Node.js SMS OTP login API enforce cooldowns?

Use random codes, store only a keyed digest, and make every decision against server time. The following TypeScript is a focused in-memory example. Replace the maps with a data store that supports atomic compare-and-set before running more than one process. The policy numbers are explicit engineering choices for this example, not universal security thresholds.

```ts
import { createHmac, randomInt, randomUUID, timingSafeEqual } from "node:crypto";

type DeliveryResult =
  | { kind: "accepted"; receiptId: string }
  | { kind: "invalid-recipient" };

interface SmsDelivery {
  send(input: { to: string; body: string; attemptId: string }): Promise<DeliveryResult>;
}

type Challenge = {
  id: string;
  digest: Buffer;
  expiresAt: number;
  nextSendAt: number;
  failures: number;
  consumed: boolean;
  templateVersion: "seller-login-v1";
};

const challenges = new Map<string, Challenge>();
const suppressed = new Set<string>();
const secret = Buffer.from(process.env.OTP_HMAC_SECRET ?? "", "utf8");
const CODE_TTL_MS = 5 * 60_000;
const RESEND_COOLDOWN_MS = 60_000;
const MAX_FAILURES = 5;

function digest(challengeId: string, code: string): Buffer {
  return createHmac("sha256", secret).update(`${challengeId}:${code}`).digest();
}

export async function issueSellerLoginCode(
  phone: string,
  delivery: SmsDelivery,
  now = Date.now(),
): Promise<{ challengeId: string; retryAfterMs: number }> {
  if (secret.length < 32) throw new Error("OTP_HMAC_SECRET must be at least 32 bytes");
  if (suppressed.has(phone)) throw new Error("recipient-suppressed");

  const previous = challenges.get(phone);
  if (previous && now < previous.nextSendAt) {
    return { challengeId: previous.id, retryAfterMs: previous.nextSendAt - now };
  }

  const challengeId = randomUUID();
  const code = randomInt(0, 1_000_000).toString().padStart(6, "0");
  const challenge: Challenge = {
    id: challengeId,
    digest: digest(challengeId, code),
    expiresAt: now + CODE_TTL_MS,
    nextSendAt: now + RESEND_COOLDOWN_MS,
    failures: 0,
    consumed: false,
    templateVersion: "seller-login-v1",
  };

  // Persist this transition atomically before calling the delivery adapter.
  challenges.set(phone, challenge);
  const result = await delivery.send({
    to: phone,
    body: `Your seller sign-in code is ${code}. It expires in 5 minutes.`,
    attemptId: challengeId,
  });

  if (result.kind === "invalid-recipient") {
    suppressed.add(phone);
    challenges.delete(phone);
    throw new Error("recipient-suppressed");
  }
  return { challengeId, retryAfterMs: RESEND_COOLDOWN_MS };
}

export function verifySellerLoginCode(
  phone: string,
  challengeId: string,
  candidate: string,
  now = Date.now(),
): boolean {
  const challenge = challenges.get(phone);
  if (!challenge || challenge.id !== challengeId) return false;
  if (challenge.consumed || now >= challenge.expiresAt) return false;
  if (challenge.failures >= MAX_FAILURES) return false;

  const supplied = digest(challengeId, candidate);
  if (!timingSafeEqual(challenge.digest, supplied)) {
    challenge.failures += 1;
    return false;
  }

  challenge.consumed = true;
  return true;
}
```

The copy is intentionally produced beside `templateVersion`. That gives reviewers one place to inspect the promise made to a seller. In production, normalize and validate the phone number before using a stable recipient key, encrypt sensitive recipient data, and ensure the challenge ID presented at verification is the one issued to the browser. Those concerns are omitted from the small example so the state transitions stay visible.

Reject early.

One pitfall deserves emphasis: a cooldown checked only in the browser is decoration. Two tabs or a direct API client bypass it. The server must reserve the send window atomically. Otherwise concurrent requests can both observe eligibility and both send. Consider the exact race: request A reads no active cooldown; request B does the same a few milliseconds later; both create a challenge; both hand a different code to the delivery adapter. The seller may receive the second message first and enter its code after request A overwrites the stored digest. The UI then reports a wrong code even though the person copied the newest message on screen. A transaction, conditional write, or data-store lock must cover the eligibility check and reservation. The network call should remain outside that transaction, with its attempt ID already fixed. If delivery times out, retry that attempt idempotently rather than opening another cooldown window. This design accepts a less convenient implementation in exchange for one observable send decision per user action. It also makes the expected test precise: launch two issue operations against one recipient and assert one reservation, one adapter attempt, and one blocked result.

## Observe decisions, not secrets

Start with five events: `otp.issue_allowed`, `otp.issue_blocked`, `otp.delivery_accepted`, `otp.recipient_suppressed`, and `otp.verify_result`. Each event needs a timestamp, anonymous attempt ID, template version, purpose, and bounded reason code. Do not attach the code or raw phone number.

From those events, publish counters for issue decisions, delivery outcomes, verification outcomes, and suppressions. A histogram can measure time from issue to successful verification. Keep phone numbers, attempt IDs, and free-form errors out of metric labels; their cardinality grows with traffic and they expose data that operators do not need for aggregation. Logs may retain an opaque correlation ID under the system's access and retention policy.

A useful alert compares stages instead of watching raw sends. If accepted deliveries remain steady while successful verifications fall, investigate expiry wording, latency, and the verification path. If invalid-recipient results rise, inspect input normalization and acquisition quality. Page only on a symptom tied to customer access or abuse controls; route slow trend changes to a dashboard or ticket.

The suppression record also needs provenance. Record the bounded reason, source, and time, then define the reviewed process that can clear an incorrect suppression after ownership is re-established. Permanent, context-free suppression creates a different failure: reassigned or corrected numbers can never recover.

## What about retries and duplicate callbacks?

A retry must reuse the same delivery attempt ID. The adapter can then make repeated calls idempotent, while a deliberate resend creates a new challenge and invalidates the prior one. Do not generate a fresh code inside a generic network retry. That turns one user action into several valid secrets with uncertain delivery order.

Callback processing follows the same rule. Deduplicate by callback identity, permit only valid state transitions, and retain the raw delivery status outside metrics if audit policy permits. Translate transport-specific statuses into a small internal vocabulary. `accepted` means the delivery system accepted work; it does not mean the user received or read the message.

For deploys, test the boundary with a fake adapter that can return `accepted` or `invalid-recipient`. Then run concurrency tests against the real data store: two simultaneous issue requests should produce one allowed transition. A clock-injected expiry test should fail at the exact boundary, and five wrong codes in this example should block the sixth verification attempt. Crisp tests beat hopeful comments.

## Does suppression weaken account recovery?

It can, if suppression is mistaken for proof of fraud. Suppression is a delivery decision: stop sending to a destination that the authoritative delivery result identifies as invalid. Account recovery is an identity decision and needs a separate, reviewed path. Do not silently switch to email merely because SMS failed; an attacker may control one channel, and the channels have different properties.

NIST describes PSTN-based out-of-band authentication as restricted and requires verifiers to consider risks such as device swap and number porting. So SMS OTP should sit inside a broader risk decision, especially for seller payout access. Rate limits reduce guessing and message abuse. They do not turn possession of a phone number into durable identity proof.

Email notifications have a related but distinct trust layer. If the commerce system sends recovery or security mail, DMARC provides domain-level policy and reporting built on SPF and DKIM alignment. It does not validate an SMS recipient, and an email bounce should not suppress a phone number. Keep channel suppression lists and reason vocabularies separate even when one authentication service coordinates both.

The final operational rule is compact: the authentication service owns policy, state, templates, and evidence; adapters own protocol translation and delivery. Measure every state change. Suppress only from defined evidence. Give recovery its own threat model.

## Sources

- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
