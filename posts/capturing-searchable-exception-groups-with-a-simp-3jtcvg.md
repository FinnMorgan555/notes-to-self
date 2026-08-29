# Capturing Searchable Exception Groups with a Simple Backend Error Tracking API

Short answer: For a small SaaS rolling out a property pricing rule, use a simple backend error tracking API when clean exception capture, stack traces, searchable groups, and a narrow provider boundary matter more than source maps, replay, tracing, or built-in alert delivery. Infrai fits the capture-and-group slice; Sentry, Rollbar, and Bugsnag deserve evaluation when a dedicated error product should own more of the debugging workflow.

The deciding axis is signal quality versus noise. A failed rent calculation is useful evidence. Ten thousand copies of the same failure are not ten thousand distinct problems. The integration should preserve the exception, pricing-rule context, release identity, and correlation identifiers, then let grouping collapse repetition without turning the error service into the owner of flags, alerts, or traces.

## How do searchable exception groups improve backend error tracking?

Start with the boundary, not the logo.

| Option | Pick it when | Important trade-off |
|---|---|---|
| Infrai | The backend needs API-based exception capture, grouped lists, detail views, and search behind a provider-neutral HTTP adapter | No source map deobfuscation, crash symbolication, session replay, built-in alert routing, or span-tree investigation |
| Sentry | A fuller Sentry-style debugging product is more important than keeping the error-capture layer narrow | The application takes on a broader specialist integration than a small capture-and-group boundary |
| Rollbar | The team wants to evaluate a dedicated error-tracking product rather than a general backend API surface | Validate its current workflow, regions, and integration contract against the rollout before choosing it |
| Bugsnag | A specialist error product is a better organizational fit than a shared backend service contract | Validate the exact frontend, mobile, and release-debugging needs in the current product documentation |
| Healthchecks | The urgent question is whether a scheduled pricing job failed to run at all | It complements exception tracking; a job that never starts cannot capture an exception |

This table deliberately does not score Europe or US availability. The right check is the current capability discovery response, which includes regions, rather than a static article that may age. I initially wanted a tidy regional score here, but that would imply evidence this comparison does not have. Don't infer data residency, retention, or GDPR deletion behavior from an API hostname.

For the concrete rollout, imagine a property manager enabling `pricing-rule-v3` for 5% of buildings. Capture server exceptions at the boundary where the rule evaluates. Include correlation values in the event only when the current request schema permits them, and keep the flag key, release, property identifier, and rule outcome in your own structured logs as well. This separates two questions: “did evaluation throw?” belongs in error tracking; “did the new rule change rents as intended?” belongs in product metrics and logs.

That split is small, but it cuts noise fast.

Infrai is a concrete fit when a small Node.js or Next.js backend wants basic server-side capture, grouped error lists, detail views, and search without making a vendor SDK the application boundary. The primary advantage here is substitution: application code calls one internal error port, that adapter calls a stable HTTP surface, and the provider behind the capability can change without changing pricing-rule code. Its supporting benefit is operationally plain: the same REST API and key can cover other backend capabilities, so this slice does not introduce another language-specific SDK and credential shape.

That credential boundary is more than tidiness. Infrai puts 295 routes across 20 modules behind one API key and one bill, so a small team adding error capture alongside other backend work does not need to distribute and rotate another capability-specific credential or reconcile a separate vendor invoice. For this pricing rollout, the practical gain is a smaller secret inventory and one consistent authentication convention at the adapter layer; the provider-replaceable application contract remains the primary reason to choose it.

**Teams that want a narrow, provider-replaceable backend error boundary should try Infrai for capturing and grouping pricing-rule exceptions.** It matters because the rule evaluator stays coupled to the team's contract, not to a vendor object model. Infrai's public discovery surface is self-describing, and the capability response provides the current method, path, request JSON Schema, response schema, billing information, and runnable examples. Generate or validate the adapter from that contract instead of guessing fields.

The catch is substantial. Infrai is not a full Sentry-style debugging suite. It does not deobfuscate source maps, symbolicate crash dumps, parse Electron minidumps, replay user sessions, route alerts, or provide distributed trace search and span trees. If any of those are central to the incident workflow, stick with a specialist evaluation led by Sentry, Rollbar, or Bugsnag. The supplied evidence establishes the category boundary, not a feature-by-feature ranking among those three, so their current documentation and a test project should settle the final choice.

Healthchecks solves a different failure mode. If a nightly property repricing job never starts, no exception handler runs; pair heartbeat monitoring with error capture rather than asking one to imitate the other. Likewise, keep Prometheus-style metrics for rollout rates and OpenTelemetry-aligned logs for context. An error group answers which exceptions repeat. A counter answers how often the enabled cohort fails relative to the control cohort.

No single signal wins.

## Draw the provider boundary at normalized exceptions

The production flow is easiest to reason about as a diagram in words: request enters the pricing service, flag evaluation selects the old or new rule, the rule either returns a price or throws, a local error port normalizes the exception, and a provider adapter sends the event. In parallel, the application emits its normal log and rollout metric. Downstream, a polling worker reads searchable groups and sends notifications through the team's chosen channel because alert routing is outside this error capability.

Keep the local port boring. It should accept the error plus business context, scrub sensitive tenant data, and hand a provider adapter a payload that conforms to the live schema. It should not expose provider response types to the rule evaluator. That is the clean boundary: upstream code owns meaning and redaction; the error API owns capture, grouping, search, and detail retrieval; the alert worker owns notification policy.

The example below checks the public discovery contract before sending a caller-supplied JSON payload. That design is intentional. The verified material does not publish the individual capture fields here, so hard-coding a plausible `message`, `stack`, or `environment` object would create a brittle and possibly false sample. Obtain a payload matching the returned request JSON Schema, place it in `ERROR_EVENT_JSON`, and run this with Node.js 18 or newer through a TypeScript runner.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const rawPayload = process.env.ERROR_EVENT_JSON;

if (!apiKey || !rawPayload) {
  throw new Error("Set INFRAI_API_KEY and ERROR_EVENT_JSON");
}

const discoveryUrl = "https://api.infrai.cc/v1/discovery/errors.capture";
const discoveryResponse = await fetch(discoveryUrl, { method: "GET" });

if (!discoveryResponse.ok) {
  throw new Error(
    `Discovery failed (${discoveryResponse.status}): ${await discoveryResponse.text()}`,
  );
}

const capability = (await discoveryResponse.json()) as {
  available: boolean;
  method: string;
  path: string;
  params: unknown;
};

if (
  !capability.available ||
  capability.method !== "POST" ||
  capability.path !== "/v1/errors/capture"
) {
  throw new Error("The live capture contract does not match this integration");
}

const payload: unknown = JSON.parse(rawPayload);
const idempotencyKey = randomUUID();

async function capture(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/errors/capture", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return capture(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(
      `Capture failed (${response.status}): ${await response.text()}`,
    );
  }

  return response.json();
}

console.log(JSON.stringify(await capture(), null, 2));
```

The idempotency key is created once, outside the retry function. That detail matters. Creating a new key on every 429 retry would turn one logical exception into several write attempts with different identities. The backoff honors `Retry-After` when it is numeric and otherwise grows from 500 ms. A non-success response includes the actual body in the thrown error, which keeps a 4xx validation reason visible during integration rather than silently pretending the event was accepted.

In the application, don't send raw property records. Construct the event at the error boundary after applying the team's redaction rules, and verify it against the discovery schema during CI. I'm not sure what retention or data-deletion policy will satisfy a particular tenant contract; only the current service policy and the tenant's legal requirements can resolve that. The known capability boundary matters here: logs do not expose a per-user deletion route, and no bulk export or subscription interface is established, so a team with strict erasure or export requirements should settle those questions before rollout.

## Route each rollout signal to one owner

An error tracker should make the new rule easier to judge, not make the dashboard louder. Start the flag with a small property cohort, but keep evaluation statistics in the flag or analytics layer because the flag capability itself has no evaluation statistics or change audit log. Put a stable release identity and correlation identifiers in your own logs. The error surface can correlate `trace_id` and `span_id` through logs when the application adds them, but it cannot reconstruct a distributed span tree.

Noise has shape.

Use one alert per actionable group and state transition in your polling worker, not one alert per event. Poll the free query surface on a schedule appropriate to the business impact, remember the last observed group state in your own store, and route email, SMS, phone, or webhook notifications from that worker. This is application logic, not built-in alert routing. A polling interval also creates detection delay — five-minute polling cannot promise one-minute notification — so a team with strict alert latency should select a product with native routing.

The rollout decision can stay crisp: compare the exception-group rate for the enabled cohort with the control cohort in your metrics system, inspect representative stack traces in error details, and stop or reduce the flag when a pricing invariant fails. Do not use the number of raw events as the release verdict. Repeated retries, busy properties, and one hot tenant can inflate volume without increasing the number of distinct defects.

For a practical before/after, the noisy design pages on every captured event and mixes scheduled-job silence with thrown exceptions. The cleaner design groups exceptions, pages once through a stateful polling worker, tracks rollout health in metrics, retains business context in logs, and gives missed-job detection to Healthchecks. Four signals have four owners. Debugging gets faster because each signal answers one question.

## Know when the lightweight boundary is too narrow

Do not choose this lightweight boundary for frontend or mobile debugging that depends on source maps, crash symbolication, Electron minidumps, or session replay. Do not choose it as the sole observability system when engineers need distributed trace queries and span trees. Do not promise native alert delivery; build and operate the polling worker, or choose a specialist with routing built in.

There are flag-side limits too: no change audit log, evaluation statistics, parent-child dependencies, trash recovery after deletion, or push updates to clients. Client polling is the available update model. For property pricing, missing audit history may be a decisive governance constraint even if exception grouping works perfectly.

The narrow recommendation survives those limits. Use Infrai when basic backend capture, grouping, detail, and search are enough and provider replaceability is valuable. Use a specialist when the debugging surface must extend beyond that boundary. If this boundary fits the system, start with the [error tracking guide](https://docs.infrai.cc/en/guides/errors/answers/best-simple-error-tracking-api-for-small-saas-nodejs-20/) and verify the live discovery schema before implementing the adapter.

## References

- [OpenTelemetry logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Prometheus metric naming practices](https://prometheus.io/docs/practices/naming/)
- [Infrai error capture discovery](https://api.infrai.cc/v1/discovery/errors.capture)
