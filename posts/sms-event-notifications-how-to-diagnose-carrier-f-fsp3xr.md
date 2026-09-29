# SMS Event Notifications: How to Diagnose Carrier Filtering and Registration Failures

Short answer: do not resend a telehealth login code until you know whether the first attempt was rejected, still progressing, or accepted by the carrier. Build one notification timeline from delivery events, suppress invalid destinations, and allow a bounded resend only for a new attempt. That choice favors delivery reliability over raw send volume.

| Signal | Pick this action when | Why |
| --- | --- | --- |
| Permanent recipient failure | The normalized event says the number or address is invalid | Suppress that channel destination and request a correction |
| Registration or signature rejection | The sender identity or required signature fails a destination rule | Stop retries and route the configuration error to operations |
| Temporary delivery failure | The event is explicitly retryable and the code remains useful | Retry with backoff inside a strict attempt budget |
| Accepted or unknown state | No terminal event has arrived | Wait and expose another approved verification path |

This field guide follows a telehealth appointment login. The same account may receive email reminders, so suppression belongs to a channel-specific destination record rather than the whole patient profile. A bounced email must not silently disable a valid phone number.

## Implement an evidence contract before retries

Before tuning retry behavior, assign ownership for each statement the system can make. The authentication service owns challenge creation, expiry, and consumption. A channel adapter owns translation from a source event into the small delivery vocabulary used below. The workflow owns suppression and retry decisions. Operations owns registration and signature configuration. This division is governance, not ceremony: without it, a transport callback can accidentally extend an authentication secret or a user-interface click can masquerade as carrier evidence.

Write the contract down as invariants. One challenge may have several attempts, but one successful verification consumes it. An attempt belongs to exactly one channel and destination key. A permanent email result cannot suppress SMS. Raw destinations and codes stay out of metrics and ordinary logs. Every normalized terminal state points back to a unique source event that an authorized operator can inspect under the applicable retention policy.

There is a cost. Keeping raw and normalized records requires access controls, retention decisions, and adapter reviews. The benefit is a diagnostic trail that separates what your software decided from what the transport reported. For a health-related login flow, that boundary is worth making explicit.

## Evaluate one failed challenge before changing policy

Start with a single challenge ID and reconstruct time order across the `2` channel types. Confirm when the challenge was created, which attempt IDs were submitted, when each source event arrived, which normalized state it produced, and whether verification eventually succeeded. Next, compare that trace with route-level counts. One invalid destination is account work; many sender-policy failures on one route are configuration work; rising unknown states after an adapter deployment point back to parsing. This audit is intentionally narrow. It prevents a fleet-wide retry change from being based on one impatient click.

The correction often happens here: submission success was treated as delivery, or callback absence was treated as rejection. Neither inference is supported. Record the gap, leave the state pending or unknown, and test the proposed classification against duplicate and out-of-order events before changing production policy.

## How should SMS event notifications classify carrier failures?

Use the most specific terminal event you have, not the patient's tap on “send again.” That click proves the code was not useful to that person at that moment. It does not prove that a carrier rejected the SMS. The first message may still be queued, delayed, delivered to an offline device, or sent to a number the patient entered incorrectly.

Authentication codes are security material. OWASP recommends consistent responses, a side channel, rate limiting, single-use tokens, and expiration. Those controls still apply during degraded delivery. Do not reveal whether a destination exists, and do not extend a token's life merely because another transport attempt occurred.

The diagram in words is simple: a login challenge enters on the left; policy creates one code; SMS and email attempts branch in the middle; delivery events flow back into one timeline; verification consumes the challenge on the right. Resends create attempts, not new identities. Keep those nouns separate in storage.

Fast retries hurt.

They can produce several messages that arrive out of order and add traffic while a registration problem remains unchanged. A `5`-second client timer is therefore interface state, not delivery policy. The explicit trade-off is a little more waiting in exchange for fewer duplicate codes and cleaner evidence.

A permanent recipient error should update a suppression record keyed by channel and normalized destination. Store the source event, classification, and decision time. Then stop automated sends to that destination until the user supplies and verifies a correction.

Do not turn every failure into suppression. Timeouts and missing callbacks are unknowns, not evidence that a phone number is invalid. This is the sharp edge: an overbroad rule can lock a patient out just as effectively as a broken sender.

For email reminders, a hard bounce supports suppression; a transient event takes another branch. Yahoo's sender guidance calls for authentication, valid DNS, low complaint rates, and prompt handling of invalid recipients. Those are email controls. Do not reuse an email bounce label as an SMS carrier diagnosis.

Sender registration and message signatures are configuration boundaries. If an event identifies that failure, another copy with the same sender and content will meet the same policy. Mark the attempt terminal, alert the owning team, and preserve the original reason beside your normalized category. Do not show the raw reason to the patient; it may expose internal routing details.

Country and route context belong in the diagnostic record. US and EU traffic can traverse different policy regimes, so a global “SMS is down” alert discards a dimension operators need. Still, avoid guessing from geography. Classification must come from an attributable event or a documented route rule.

A registration rejection is not permission to swap identities dynamically. That can break user recognition and complicate audit trails. Retain the generic login response, offer an already-approved alternate verification method, and send the failure to the operational queue. A tempting assumption is that another submission is harmless because the user asked for it; the event timeline shows why that assumption fails. The new attempt can race the old one, preserve the same configuration defect, and make the resulting alert count look like several incidents instead of one.

## Govern state transitions with one attempt ledger

The ledger accepts generic events. Map provider payloads at the ingress boundary, retain raw data under normal privacy controls, and pass only the normalized event into this function.

```ts
type DeliveryClass =
  | "accepted"
  | "delivered"
  | "temporary_failure"
  | "invalid_recipient"
  | "sender_policy_failure"
  | "unknown";

type Attempt = {
  id: string;
  challengeId: string;
  channel: "sms" | "email";
  destinationKey: string;
  attemptNumber: number;
  expiresAt: string;
  state: "pending" | "delivered" | "retryable" | "terminal";
  lastEventAt?: string;
};

type DeliveryEvent = {
  attemptId: string;
  occurredAt: string;
  classification: DeliveryClass;
  sourceEventId: string;
};

type Decision = {
  attempt: Attempt;
  suppressDestination: boolean;
  alertReason?: "sender_policy_failure";
};

export function applyDeliveryEvent(
  attempt: Attempt,
  event: DeliveryEvent,
): Decision {
  if (event.attemptId !== attempt.id) {
    throw new Error("delivery event does not match attempt");
  }
  if (attempt.lastEventAt && event.occurredAt < attempt.lastEventAt) {
    return { attempt, suppressDestination: false };
  }

  const next = { ...attempt, lastEventAt: event.occurredAt };
  switch (event.classification) {
    case "delivered":
      return { attempt: { ...next, state: "delivered" }, suppressDestination: false };
    case "temporary_failure":
      return { attempt: { ...next, state: "retryable" }, suppressDestination: false };
    case "invalid_recipient":
      return { attempt: { ...next, state: "terminal" }, suppressDestination: true };
    case "sender_policy_failure":
      return {
        attempt: { ...next, state: "terminal" },
        suppressDestination: false,
        alertReason: "sender_policy_failure",
      };
    case "accepted":
    case "unknown":
      return { attempt: { ...next, state: "pending" }, suppressDestination: false };
  }
}
```

The timestamp comparison assumes ingress timestamps are comparable. If they are not, assign a monotonic sequence during ingestion. Enforce uniqueness on `sourceEventId` too; a duplicate callback must not spend another retry or produce another alert.

A resend worker should read this state, challenge expiry, and attempt budget in one transaction. It may enqueue a new attempt only when policy permits. Use exponential backoff with jitter for explicitly temporary failures, capped before the challenge expires. For pending or unknown states, waiting and an alternate path are safer than blind duplication.

Instrument state changes, not message contents. Count attempts by channel and classification, suppressions by reason, retry-budget exhaustion, and verification success after each attempt number. Event-delay histograms separate slow feedback from failures. Logs need challenge ID, attempt ID, route region, normalized class, and source event ID, while excluding the code and full destination. Alert on sustained sender-policy failures by route; one invalid number belongs in a workflow, not a pager.

Test the ugly orderings: a duplicate event, an older event after a newer one, a temporary failure followed by delivery, and a permanent email failure beside a healthy SMS destination. Then canary parser changes by route. A mapping error that turns unknown events into permanent failures can otherwise create a suppression wave.

## Limits and trade-offs

Delivery receipts do not prove that a person saw a message, and a resend request does not prove non-delivery. Keep authentication success as its own event. The system can then answer two distinct questions: “Did transport report delivery?” and “Did the patient complete verification?”

No generic classifier defines every carrier reason or registration rule. This approach does not fit a system that lacks attributable delivery events; there, `unknown` must remain unknown and an approved alternate authentication path carries more weight. Maintaining adapters also costs engineering time, while aggressive suppression reduces traffic at the risk of blocking a corrected or reassigned destination. Review mappings when routes change and preserve the unknown category.

Unknown is honest.

The operating rule is concise: suppress only on evidence of an invalid destination, retry only an explicitly retryable attempt within the challenge limits, and escalate sender-policy failures instead of multiplying them.

## Sources and References

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://senders.yahooinc.com/best-practices/
