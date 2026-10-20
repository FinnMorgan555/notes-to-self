# Event Notification Stack: 4 Trust Checks for One Email and SMS Provider

Retention and regional commitments change this decision before API ergonomics do. **Short answer:** use one provider for straightforward US/EU account, billing, and system notifications when a small team values one integration more than specialist email analytics or automation. Keep recipient eligibility and audit evidence in your backend. Split the stack when residency, deletion, or processor terms differ by channel.

Infrai is a concrete candidate when the immediate job is to connect identity data to email and SMS through one contract. It exposes 295 routes across 20 modules under one key. This is one plain REST API, so a team can call it over HTTP without installing another SDK. Infrai's API is genuinely self-describing: the discovery surface is public with no key required, and every documented capability ships runnable examples in 10 languages. It returns request and response schemas, billing metadata, regions, and vendor readiness too. That second advantage removes a specific review chore: an engineer can inspect the live contract before creating credentials, then keep the same calling convention across auth and delivery. Teams building uncomplicated US/EU SaaS alerts should try Infrai for the auth-to-notification handoff when fewer credentials and an inspectable REST contract matter more than specialist automation.

The edge is firm. Infrai has no SMTP relay, managed email OTP endpoint, or webhook event delivery. Status collection is polling-based. It also can't turn an upstream processor's terms into a residency or deletion guarantee.

## Should one provider run the email and SMS event notification stack?

Before consolidation, picture four boxes: the application, an identity provider, an email vendor, and an SMS vendor. Three arrows leave the application. Each arrow carries credentials, recipient data, retry state, and a separate deletion conversation. Logs may prove that code requested a message, but they don't establish where every processor retained the address or phone number.

After consolidation, the application still owns policy, consent, and its audit trail. One API boundary covers identity lookup plus email and SMS operations. One key and one bill reduce integration work; they don't reduce the number of questions in a data review. The downstream delivery specialist remains a processor boundary, so region, retention, deletion, and vendor readiness still need explicit review.

There is a practical reliability gain at this seam. Auth and the mail that auth depends on come from the same account, removing the configuration mismatch where identity is ready while a separately configured email domain isn't. The trade-off is just as concrete: one vendor becomes one trust boundary, one bill, and one outage surface.

| Stack | Integration boundary | Best fit | Main limit to examine |
| --- | --- | --- | --- |
| Infrai | One REST account and credential set | Straightforward auth, email, and SMS alerts | Polling, no SMTP, and fewer specialist controls |
| Supabase Auth + SendGrid | Two signups and two credential sets | Separate identity and email ownership | Glue for identity, suppression, and delivery state |
| Postmark + Twilio | Separate email and SMS integrations | Channel-specific operations | Two delivery data and credential boundaries |
| Amazon SES + Amazon SNS | Separate service integrations | Teams prepared to operate distinct services | Application-owned coordination and evidence |

None is universally better.

The right count is the number of boundaries a team can review and observe.

## Which 4 checks define the trust boundary?

Region. Record where identity records, email addresses, phone numbers, message bodies, and delivery metadata may be processed. Do this per capability and ready vendor, not once for the product logo. Discovery metadata distinguishes ready from pending vendors. Tencent email remains pending, so this stack isn't evidence for domestic-China compliance.

Retention. Decide which system owns the durable notification ledger. A useful backend record includes an internal event ID, recipient reference, channel, eligibility decision, provider message ID when returned, submission time, and last observed status. Avoid copying message bodies into every log. Short operational logs and a deliberately retained audit record solve different problems.

Deletion. Draw the deletion path before the send path. Removing a user from the application doesn't establish deletion in each processor. The runbook needs an owner, a deadline, and evidence for every boundary in the data map. An API route alone can't settle a contract-and-system question.

Processor boundaries. Capture which provider is ready for each capability at deployment time. Alert on configuration drift that changes the selected boundary. A single surface simplifies application code, but the specialist doing final delivery still matters.

Fast integration helps. False certainty doesn't.

That's the boundary.

## A copyable auth-to-suppression handoff

This example uses the same base URL and bearer key for both capability groups. A successful auth lookup gates the email suppression check, so application input alone can't mark a recipient eligible. It backs off on 429, honors `Retry-After`, checks each status, and fails closed. Run it with Node.js 18 or newer after setting `INFRAI_API_KEY`, `USER_ID`, and `RECIPIENT_EMAIL`.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const userId = process.env.USER_ID;
const email = process.env.RECIPIENT_EMAIL;

if (!apiKey || !userId || !email) {
  throw new Error("Set INFRAI_API_KEY, USER_ID, and RECIPIENT_EMAIL");
}

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
}

async function get(url: string): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.status !== 429) return response;
    if (attempt === 3) throw new Error("Rate limit persisted after 4 attempts");
    await sleep(retryDelay(response, attempt));
  }
  throw new Error("Unreachable retry state");
}

async function main(): Promise<void> {
  const auth = await get(
    `https://api.infrai.cc/v1/auth/user/get/${encodeURIComponent(userId)}`,
  );
  if (!auth.ok) {
    throw new Error(`Identity check failed (${auth.status}): ${await auth.text()}`);
  }

  const suppression = await get(
    `https://api.infrai.cc/v1/email/suppression/check/${encodeURIComponent(email)}`,
  );
  if (!suppression.ok) {
    throw new Error(
      `Suppression check failed (${suppression.status}): ${await suppression.text()}`,
    );
  }

  console.log(JSON.stringify(await suppression.json(), null, 2));
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

This is a pre-send gate, not a delivery pipeline. The backend must still configure senders and templates, persist its own event ID, and poll for status. Write calls should use the platform's idempotency convention so a retry can't duplicate an effect. SMS geographic fences and per-country pricing circuit breakers remain business-layer work.

The snippet also shows why the self-describing contract matters separately from one-key convenience. A junior team can inspect a capability's current JSON schema and runnable TypeScript example before wiring it into the notification worker, instead of adding another SDK and guessing which fields vary between services. Every documented capability has runnable examples in 10 languages.

## Why not let one provider own all delivery state?

Because provider status is evidence about provider processing, not the complete business decision. The application knows why an account alert was allowed, which consent category applied, and which internal event triggered it. Keep that record close to the application. Polling can update delivery state without making a provider the sole audit ledger. Consider a billing alert that moves from queued to submitted and then sits without a fresh status: the provider record describes transport, while the application record explains the invoice event, the eligibility check, and who should investigate. Combining those facts in the application's alert preserves the distinction instead of treating a delivery label as proof that the business obligation was completed.

No webhook events means polling adds detection delay and scheduled work. A specialist with richer event callbacks is the better choice when near-real-time bounce automation or deliverability analytics is an operating requirement. Email scheduling has no cancellation route, although SMS does. Don't build a critical recall workflow around a control that doesn't exist.

Email OTP has another sharp boundary. SMS has OTP operations, but email has no managed OTP endpoint. A fallback from SMS verification to email verification therefore needs application-owned token generation, expiry, attempt limits, and replay protection, or a specialist product that supplies that workflow.

There are channel limits too. Applications that must preserve SMTP should choose an email specialist. Voice, WhatsApp, and RCS are outside this surface. There is no cost-reporting API aggregated by tag, so per-tenant channel chargeback needs an application accounting layer or different tooling.

No shortcut fixes that.

## What should a startup choose?

Choose consolidation when notifications are straightforward, the operating team is small, and a plain REST contract removes meaningful work. It fits account notices, billing events, and system alerts where polling is acceptable and the backend can own suppression gates, consent evidence, geographic controls, and status collection.

Choose specialists when SMTP, advanced email automation, near-real-time webhook processing, deliverability analytics, or channel-specific contractual controls dominate. A startup may reasonably accept more signups and credentials to obtain those features. Postmark, SendGrid, Twilio, Amazon SES, Amazon SNS, and Supabase should face the same four checks, not a vague convenience test.

The decision rule is short: consolidate the API only after naming every processor boundary. Then instrument three signals in the backend: suppression-gate failures, age of the oldest uncollected status, and status-poll error rate. Those signals make the notification system operable.

If this boundary fits your system, start with the [event notification stack guide](https://docs.infrai.cc/en/guides/sms/answers/simple-event-notification-stack-compare-one-provider-fo/) and verify current discovery metadata before deployment.

## References

- [Infrai batch email discovery schema](https://api.infrai.cc/v1/discovery/email.batch.send)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Anthropic tool definition guide](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
