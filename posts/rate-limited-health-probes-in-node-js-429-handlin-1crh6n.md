# Rate-Limited Health Probes in Node.js: 429 Handling and Free Status Dashboards

If you just want to be able to explain a late shipment to a customer three days after it happened, the smallest thing that works is one Node.js polling worker that writes every health check it makes — including the attempts that came back 429 — as a line in an append-only log, plus a static status page rendered from that log. No agent. No hosted uptime product. A cron entry, a fetch call, and a file.

The rate limit isn't the hard part. Keeping the evidence while the rate limit is hitting you is.

That distinction is the whole article. In a logistics stack, the thing you get paged about ("the carrier tracking API is flaky") and the thing you get asked about a week later ("why did these 40 shipments show a stale ETA between 14:00 and 15:30?") are answered by completely different artifacts. Alerting needs the current value. Incident reconstruction needs the full sequence of observations, gaps included. Most uptime tutorials only build the first one, then quietly throw away the second by collapsing a retry loop into a single boolean.

Here is the decision I'd write on a whiteboard before touching code, framed by the question you'll be asked later rather than by what you want to alert on now.

| What you must reconstruct later | Smallest thing that works | Evidence it leaves | Where it stops |
| --- | --- | --- | --- |
| "Was the carrier API answering at 14:20?" | One polling worker writing NDJSON check records | Timestamp, status, latency, response headers per attempt | Only sees what it polled; a gap is invisible unless the gap itself is recorded |
| "Did our checker even run?" | A dead-man's switch the worker pings after each window | A missing ping is the signal | Says nothing about the target's health |
| "Was it them, or our egress?" | Probes from two networks or regions | Divergent results per vantage point | Doubles request volume against a rate-limited API |
| "Why did those 40 shipments go stale?" | Check log joined to shipment ids by time window | Correlation between throttled windows and business rows | Retention cost grows with volume |
| "Is it down right now?" | Static page rendered from the last record | Public answer in one HTTP request, no infrastructure | Presentation only; proves nothing about the alert path |

Four of those five rows are cheap. The expensive one is the fourth, and it's the one that ends arguments with a customer.

## How should a Node.js polling worker handle 429 rate limits without losing uptime evidence?

Draw the loop in words before you write it: scheduler fires, worker claims a window id, worker spends a bounded request budget against the target, each attempt is classified and written, the window closes with one outcome, the outcome updates the dashboard and — separately — feeds the alert rule.

Two rules make that loop trustworthy.

First, a 429 is not "down". It's a statement about your relationship with the API, not about the health of the service behind it. RFC 6585 defines the status, and RFC 9110 defines `Retry-After` as either a delay in seconds or an HTTP-date, so a correct client parses both forms. If the server also emits the `RateLimit` header fields now going through the IETF HTTP API working group, honor them and stop guessing entirely. Treating a throttled response as an outage produces a false red on your status dashboard, and a false red teaches the on-call engineer to ignore it — which is a far more expensive failure than the one you were monitoring for.

Second, log the attempts, not just the verdict.

This is the part people skip. A retry loop that tries four times over 30 seconds and records only the last result has quietly deleted the most interesting data in the whole system: how long the target was refusing you, whether the refusals were throttles or timeouts, and whether the backoff was even respected. When the reconstruction question arrives, the difference between `{"outcome":"down"}` and four timestamped records showing `429, 429, 429, up` is the difference between "we think there was an issue" and "the carrier API throttled us from 14:07:12 to 14:07:51, we retried within the advertised delay, and the first successful read after that returned a tracking event stamped 13:52." One of those is a support answer. The other is a shrug.

Backoff itself is boring and solved. Exponential delay with full jitter — pick a random value between zero and the exponential ceiling — prevents a fleet of workers from synchronizing their retries into a thundering herd after a shared outage. Cap the ceiling at something you can defend, usually well under the polling interval, so a struggling window can't overlap the next scheduled one. Two workers polling the same target on overlapping windows will double your request rate against an API that just told you to slow down, and they'll also write interleaved records that make the timeline ambiguous.

Idempotency is easy here, which is a nice change. Reading a health endpoint has no side effect, so the only cost of a retry is quota. Writing the check record does have a side effect, so give each record a `checkId` derived from the scheduled window rather than from the attempt, and make the log append-only so a replay can't rewrite history.

## Pick the checker that matches the question you'll be asked

A single in-process poller is the right call when one team owns the target, the polling interval is measured in minutes, and the reconstruction question will be about one dependency. It's roughly 80 lines of code and it costs nothing beyond the box it runs on.

A dead-man's switch belongs next to it when "the job silently stopped" is a real risk — and in a logistics stack it usually is, because the checker tends to run on the same host as the importer that died. The pattern inverts the direction: the worker pings a receiver after each window, and the receiver alarms when the ping doesn't arrive. Nothing your worker can observe about a target will ever tell you that your worker isn't running.

Multi-vantage probing earns its keep when you need to distinguish "the carrier is down" from "our NAT gateway's IP got rate-limited". The catch is that every vantage point multiplies your request volume against exactly the API that's already throttling you, so budget it deliberately instead of adding regions because the dropdown offers them.

Fingerprinting is worth stealing from the error-tracking world. Sentry's grouping model derives a fingerprint from the shape of an event so that thousands of repeated exceptions collapse into one issue with a count and a first-seen timestamp, and repeated probe failures want the same treatment: one incident record with a window, a count, and the distinct status codes observed, rather than 400 identical alert lines.

Cost shows up on the retention side, not the polling side. Ingest-priced log products bill by volume — Amazon CloudWatch publishes per-GB log ingestion pricing, and it's the line item that surprises teams who decided to log every attempt — so decide up front how long the raw check records live and what gets rolled up. A defensible split is 30 days of full-fidelity attempt records and a much longer retention for the per-window summaries, because the reconstruction questions that arrive months later are almost always about windows, not attempts.

## One bounded polling window, in TypeScript

Node's global `fetch` (added in 18.0, stable since 21.0) is enough; there's no client library in this example on purpose, because the whole point is that the pattern outlives whatever HTTP wrapper you're fond of this year.

```ts
type Outcome = "up" | "down" | "throttled" | "error";

interface CheckRecord {
  checkId: string;
  target: string;
  attempt: number;
  outcome: Outcome;
  status?: number;
  latencyMs: number;
  retryAfterMs?: number;
  detail?: string;
  observedAt: string;
}

const TARGET = process.env.HEALTH_TARGET_URL ?? "https://carrier.example.net/healthz";
const MAX_ATTEMPTS = 4;
const WINDOW_MS = 45_000;
const REQUEST_TIMEOUT_MS = 5_000;

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

// Retry-After is either delay-seconds or an HTTP-date (RFC 9110). Parse both,
// otherwise fall back to exponential backoff with full jitter.
function backoffMs(retryAfter: string | null, attempt: number): number {
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const at = Date.parse(retryAfter);
    if (!Number.isNaN(at)) return Math.max(0, at - Date.now());
  }
  return Math.floor(Math.random() * Math.min(30_000, 500 * 2 ** attempt));
}

// One JSON object per line: append-only, greppable, and cheap to ship anywhere.
function record(entry: CheckRecord): void {
  process.stdout.write(JSON.stringify(entry) + "\n");
}

export async function runCheckWindow(checkId: string): Promise<Outcome> {
  const deadline = Date.now() + WINDOW_MS;

  for (let attempt = 1; attempt <= MAX_ATTEMPTS; attempt += 1) {
    const startedAt = Date.now();
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), REQUEST_TIMEOUT_MS);

    try {
      const res = await fetch(TARGET, { method: "GET", signal: controller.signal });
      const latencyMs = Date.now() - startedAt;
      const observedAt = new Date().toISOString();

      if (res.status === 429) {
        const retryAfterMs = backoffMs(res.headers.get("retry-after"), attempt);
        record({ checkId, target: TARGET, attempt, outcome: "throttled", status: 429, latencyMs, retryAfterMs, observedAt });
        if (attempt === MAX_ATTEMPTS || Date.now() + retryAfterMs > deadline) return "throttled";
        await sleep(retryAfterMs);
        continue;
      }

      const outcome: Outcome = res.ok ? "up" : "down";
      record({ checkId, target: TARGET, attempt, outcome, status: res.status, latencyMs, observedAt });
      return outcome;
    } catch (cause) {
      record({
        checkId, target: TARGET, attempt, outcome: "error",
        latencyMs: Date.now() - startedAt,
        detail: cause instanceof Error ? cause.name : "unknown",
        observedAt: new Date().toISOString(),
      });
      if (attempt === MAX_ATTEMPTS) return "error";
      await sleep(backoffMs(null, attempt));
    } finally {
      clearTimeout(timer);
    }
  }

  return "throttled";
}
```

Run that on a fixed schedule with a `checkId` derived from the scheduled minute, and one throttled window leaves this behind:

```text
{"checkId":"2026-03-11T14:07:00Z","attempt":1,"outcome":"throttled","status":429,"latencyMs":112,"retryAfterMs":20000}
{"checkId":"2026-03-11T14:07:00Z","attempt":2,"outcome":"throttled","status":429,"latencyMs":98,"retryAfterMs":20000}
{"checkId":"2026-03-11T14:07:00Z","attempt":3,"outcome":"up","status":200,"latencyMs":244}
```

Before: a timer paints a green dot. After: a window with an identity, three timestamped observations, a documented delay you actually waited, and a latency series you can chart. Same number of requests. Wildly different forensic value.

## What a free status dashboard actually proves

Free is easy to get here, and it's worth being precise about what you're getting. A dashboard that reads the last check record and renders a static HTML file — published from any static host, or committed to a repository and served from its pages — costs nothing to run and answers "is it down right now?" in a single request. That is genuinely useful, and it's also the extent of it.

The failure mode is a green dot with no timestamp.

A status page renders the last thing it knew, so a worker that stopped running 90 minutes ago will happily display a healthy service forever. Render the age of the most recent successful check next to the state, and let the page go grey (not green, not red) once that age exceeds two polling intervals. Grey means "we don't know", which is honest, and honest beats reassuring. I'd also keep the alert rule out of the dashboard entirely: the page is a reader of state, and a reader can never prove the notification path works. Send yourself a deliberate test alert on a schedule if you want that proof.

One more separation worth making. Metrics answer the present tense, logs explain the past tense, and a check record is a log entry that happens to contain a number. If you only keep the rolled-up metric, you keep the shape of the incident and lose the reason for it.

## Where this stops working

This pattern is not suitable when you need managed escalation — on-call rotations, phone and SMS delivery, acknowledgement tracking, incident timelines that other humans subscribe to. Building that yourself is a product, not a script, and you'll do it worse. Stick with a hosted alerting or incident-management platform for the delivery layer and let it consume the outcome your worker already computes.

It's also the wrong layer for tracing. A check record tells you the health endpoint returned 503 at 14:07; it can't tell you which downstream call inside that request was slow. If your reconstruction questions are consistently "which service in the chain", you want distributed tracing with propagated context, and OpenTelemetry's log and trace specifications are the vendor-neutral place to start rather than a richer polling loop.

Retention and privacy can force the decision too. If your check records carry customer-identifying payload fragments, you inherit deletion and export obligations that a plain append-only file on a disk does not satisfy, and a logging platform with per-record deletion and lifecycle rules is the honest answer. I'm not sure there's a clean rule for how much response body to capture — capturing everything is what makes reconstruction possible and also what creates the obligation. Capture the status line, the timing, the rate-limit headers, and a hash of the body; go further only when you've decided who's allowed to read it. Your mileage may vary, and it depends more on your legal posture than your architecture.

Everything above is roughly a day of work. The dashboard is the part people build first and the part that matters least.

## References and further reading

- RFC 6585, Section 4 — 429 Too Many Requests: https://www.rfc-editor.org/rfc/rfc6585#section-4
- RFC 9110 — the `Retry-After` header field: https://www.rfc-editor.org/rfc/rfc9110#field.retry-after
- MDN — HTTP 429 status reference: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
- IETF HTTP API WG — RateLimit header fields for HTTP: https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/
- Exponential backoff and jitter: https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
- Node.js global `fetch`: https://nodejs.org/api/globals.html#fetch
- Sentry — event grouping and fingerprint mechanics: https://docs.sentry.io/concepts/data-management/event-grouping/
- Amazon CloudWatch pricing, including per-GB log ingestion: https://aws.amazon.com/cloudwatch/pricing/
- OpenTelemetry logs specification: https://opentelemetry.io/docs/specs/otel/logs/
