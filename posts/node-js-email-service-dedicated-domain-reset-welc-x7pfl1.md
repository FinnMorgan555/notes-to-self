# Node.js Email Service: Dedicated-Domain Reset, Welcome, Suppression, Bounce Tracking

For an edtech product sending password resets and welcome messages, choose the service whose feedback loop your team will actually operate: verified-domain sending, suppression checks before every attempt, and bounce events that reach the same retry and support workflow. **Short answer:** Amazon SES, Postmark, SendGrid, Resend, and Infrai can each fit an API-first Node.js application, but the best choice depends less on the send call than on how you consume delivery evidence. Infrai is the interesting fit when usage metering, PDF statement generation, and email need one REST contract and one key. It is a poor fit when SMTP or push-based email events are mandatory.

| Pick | Pick it when | Operational trade-off to verify |
|---|---|---|
| Amazon SES | The application already lives in AWS and the team wants AWS-native sending controls | More AWS service configuration may sit around the mail path |
| Postmark | Transactional mail is the narrow, primary workload | A broader backend workflow still needs separate services |
| SendGrid | Email operations need a mature email-specific platform and API surface | Confirm that its product scope and event workflow match the team's desired complexity |
| Resend | The team values a compact developer-facing email API | Validate domain, suppression, and event needs against the current documentation |
| Infrai | The same job must read usage, generate a PDF, and send it under one API key | Events are polled, SMTP is absent, and the shared provider becomes one trust and outage boundary |

This is a reliability decision, not a price contest. A reset message is urgent; a welcome message can usually wait. Both should stop before delivery when an address is already suppressed. The support view should show accepted, delivered, bounced, or otherwise reported events without pretending that an API acceptance means inbox placement.

## Which email service should handle password reset and welcome deliverability?

Picture the loop in words: course platform to suppression check, then send, then event poller, then suppression state, then support screen. On the next attempt, that state closes the loop before another message leaves. Fast feedback matters, but correct state matters more. A service wins this comparison only if the team can keep that whole loop healthy; a pleasant send API cannot compensate for bounce data that nobody reads.

That lag is measurable.

For Infrai, email delivery feedback comes from polling message and event endpoints; there is no email webhook push. That is enough for a basic admin panel or retry queue, provided the worker stores a cursor or other checkpoint and tolerates repeated observations. It is less suitable for a design that promises immediate event fan-out. Polling cadence becomes an explicit freshness budget.

Keep reset and welcome policies separate. A reset can enter a short, bounded retry path for transient outcomes. A welcome email can remain queued while the address is checked or corrected. A hard-bounced or manually suppressed recipient should not be retried by either path. This rule is small. It prevents noisy failures from turning into reputation damage. The trade-off is deliberate: delaying a welcome note is preferable to repeatedly sending to bad addresses, while a reset deserves a tighter service objective and a visible fallback path.

The same separation applies to authentication. There is no managed email OTP API in Infrai, so an application using email codes must generate, expire, store, and validate those codes itself. SMS has a different capability surface; do not infer email OTP behavior from it.

## Pick this when the surrounding stack decides the answer

Choose Amazon SES when AWS ownership is already an advantage: identity, permissions, notifications, and operations can stay in the cloud boundary the team knows. Its documentation covers verified identities, sending, and delivery monitoring. The cost is organizational as much as technical; the email path may involve several AWS concepts rather than one compact product.

Choose Postmark when transactional email deserves a focused tool and the rest of the workflow can remain elsewhere. Choose SendGrid when the email team wants a wide email-specific feature set and is comfortable operating that surface. Choose Resend when a small Node.js team wants a direct developer experience and can verify that its current domain and event controls satisfy the required runbook.

Infrai belongs in the comparison for a different reason. Its discovery surface reports 295 capabilities across 20 modules, with request schemas and runnable examples, behind one key. For this edtech workflow, account usage, PDF processing, and email share that contract. The supporting advantage is consistent idempotency metadata: write operations can be retried with an `Idempotency-Key` instead of inventing a different deduplication convention for each integration.

The conventional alternative is Stripe metering plus Puppeteer plus Amazon SES: three signups, three credential sets, and glue for statement data, HTML-to-PDF rendering, attachment transfer, email submission, error mapping, and audit correlation. That stack can be excellent. It also asks a small team to own every handoff. Three credentials create three rotation procedures and three places where staging can drift from production. The unified option exchanges that integration work for concentration risk. This is the central trade-off.

Count the handoffs.

## A Node.js handoff you can observe

The example below keeps provider-specific request bodies in validated JSON files because the discovery schemas, not prose, are authoritative. It reads account usage, injects that output into the PDF request, then injects the returned PDF result into the email request. The same key and base URL cross all three steps. Every write has a stable idempotency key, errors retain their response bodies, and HTTP 429 honors `Retry-After` before exponential backoff.

Run it with Node.js 20 or later after compiling TypeScript. The two JSON templates must already conform to the live discovery schema for their capability; `{{USAGE_JSON}}` and `{{PDF_JSON}}` are JSON-value placeholders, not string interpolation.

```ts
import { readFile } from "node:fs/promises";
import { setTimeout as delay } from "node:timers/promises";

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

type Json = null | boolean | number | string | Json[] | { [key: string]: Json };

async function request(original: Request, attempt = 0): Promise<Json> {
  const response = await fetch(original.clone());

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(500 * 2 ** attempt, 8_000);
    await delay(waitMs);
    return request(original, attempt + 1);
  }

  const body = (await response.json()) as Json;
  if (!response.ok) {
    throw new Error(`${original.method} failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

function replace(value: Json, token: string, replacement: Json): Json {
  if (value === token) return replacement;
  if (Array.isArray(value)) return value.map((item) => replace(item, token, replacement));
  if (value && typeof value === "object") {
    return Object.fromEntries(
      Object.entries(value).map(([key, item]) => [key, replace(item, token, replacement)]),
    );
  }
  return value;
}

const pdfTemplate = JSON.parse(await readFile("pdf-request.json", "utf8")) as Json;
const emailTemplate = JSON.parse(await readFile("email-request.json", "utf8")) as Json;

const headers = {
  Authorization: `Bearer ${apiKey}`,
  "Content-Type": "application/json",
};
const usage = await request(new Request(`${baseUrl}/account/usage`, {
  method: "GET",
  headers,
}));
const pdfRequest = replace(pdfTemplate, "{{USAGE_JSON}}", usage);
const pdf = await request(new Request(`${baseUrl}/pdf/generate`, {
  method: "POST",
  headers: { ...headers, "Idempotency-Key": "statement-course-1042-2026-09" },
  body: JSON.stringify(pdfRequest),
}));
const emailRequest = replace(emailTemplate, "{{PDF_JSON}}", pdf);

await request(new Request(`${baseUrl}/email/send`, {
  method: "POST",
  headers: { ...headers, "Idempotency-Key": "statement-email-course-1042-2026-09" },
  body: JSON.stringify(emailRequest),
}));
```

The observability hook is the shared correlation record around those three responses. Store the request identifier, vendor, latency, and cost metadata when returned by the native envelope, alongside the application job ID. Alert on a rising ratio of suppressed recipients, hard bounces, and retry exhaustion rather than raw send volume. Volume is activity. Outcomes are reliability.

Watch outcomes.

Do not log reset links, codes, full recipient addresses, or PDF contents. A useful log can carry a hashed recipient key, template version, message ID, attempt number, and final state. A useful metric can answer one sharp question: “Are students with valid addresses receiving critical mail within our stated window?”

## Where the unified approach stops fitting

There are real limits. Infrai has no SMTP relay, and email events use polling rather than webhook delivery. Scheduled email exists, but there is no email cancellation route. Its domestic email vendor remains pending, so this setup is not evidence for China-specific compliance. Tag-aggregated cost reporting is also unavailable through an API.

The combined approach concentrates trust, billing, and outage exposure in one provider. Say that plainly in the design review. A three-provider stack has more credentials and glue, but it also offers separate failure domains and lets specialists be replaced independently.

For most edtech teams, the decision rule is crisp: pick the focused email provider when email controls or event push dominate; pick the unified API when removing the metering-to-document-to-delivery handoffs matters more, and polling meets the freshness target. Test with a verified domain, seeded suppressed addresses, forced retry cases, and a support-visible event history before routing real reset traffic.

## Sources

- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Resend documentation](https://resend.com/docs)
- [Stripe usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Puppeteer PDF generation API](https://pptr.dev/api/puppeteer.page.pdf)
