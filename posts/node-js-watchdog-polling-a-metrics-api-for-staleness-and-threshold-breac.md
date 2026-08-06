# Node.js Watchdog: Polling a Metrics API for Staleness and Threshold Breaches

If you just want the recommendation: run one small Node.js watchdog on a fixed schedule, validate metric freshness before evaluating the failure threshold, and alert only after two consecutive bad samples. Keep delivery behind an interface so a paging outage can't hide the original failure.

Freshness comes first.

A `200` response can carry an old sample. A failed request can also erase the distinction between “the application is healthy” and “the observer knows nothing.” I teach this as three separate states: fresh and below threshold, fresh and above threshold, or unknown. Only the middle state is evidence of an application breach. The last state is an observability failure and deserves its own alert path.

## How should Node.js polling turn metrics API failures into a threshold alert?

Start with the operating decision, not the timer syntax. This table is the field guide I use before anyone writes a cron expression.

| Situation | Pick | Why | Main catch |
| --- | --- | --- | --- |
| One small service, a stable metrics endpoint, and minute-level detection | A Node.js polling watchdog | Few moving parts; the threshold logic stays visible | The watchdog needs independent supervision |
| Producers can emit a failure event at the source | Event-driven evaluation | No polling delay; each event has context | Delivery, deduplication, and replay become part of the design |
| Many services, labels, and alert owners | A standards-based collector and rule engine | Central rules and consistent routing | More configuration and a larger operational surface |
| A batch job already runs under cron | A post-run check | The check aligns with job completion | A missing job run must be detected separately |

For the simple case, define the contract in plain language: poll every 60 seconds, reject a sample older than 120 seconds, calculate `failures / total`, and open an alert after two fresh samples breach the configured ratio. Those numbers are examples, not universal defaults. Your mileage may vary. The useful part is that each value names a different concern: cadence, freshness, threshold, and persistence. Don't collapse them into one “failed” boolean. If the API can't be reached, record `poll_error`. If its sample is old, record `stale_sample`. If the data is current and the ratio is high, record `threshold_breach`. That vocabulary makes logs searchable and keeps an on-call engineer from chasing an application regression when the real issue is an expired credential or a stopped exporter. This is also where “free and simple” needs an honest definition. The watchdog can use the runtime, an existing scheduler, and an existing notification channel without adding a commercial service. It still costs attention: someone owns the process, the credential, the alert destination, and the runbook.

Small is good.

Ownerless isn't.

## Pick the evaluation path that matches the failure source

Use scheduled polling when the metrics API is the authoritative place to ask, “How many operations failed in the last window?” It fits low-cardinality checks and teams that can tolerate detection latency of roughly one polling interval plus the confirmation window. I like it for a single batch pipeline or internal integration because I can show the entire decision path on one screen.

Use event-driven evaluation when the source already knows about each failure and losing per-event context would hurt diagnosis. The diagram in words is: producer emits, durable transport retains, consumer groups by window, evaluator applies policy, notifier routes. That path can react sooner, but it introduces retry and deduplication policy. You need an event identity and an answer for late arrivals. “Send an alert in the catch block” isn't that answer.

Use a collector and rule engine when one watchdog starts multiplying. Ten copies usually mean ten subtly different timeout rules, secret-loading conventions, and labels. Central evaluation gives the team one place to review thresholds and ownership, while application code remains focused on producing signals. The catch is operational weight. A tiny team with one endpoint may spend more time maintaining the alerting stack than improving the check.

Cron is a launch mechanism, not memory. If cron starts a fresh Node.js process for every poll, a two-sample streak can't live only in a variable. Persist the streak in a small durable store, or keep one supervised process alive and let its scheduler retain state. Also detect the absence of runs. A perfect threshold evaluator that silently stopped yesterday is decorative monitoring.

I use feature flags for alert-policy rollout when a new threshold could page too aggressively. Start in observe-only mode, compare decisions with real incidents, then enable notification. A kill switch should disable delivery, not measurement, so the decision log remains available for tuning. This follows the same operational separation described in Martin Fowler's feature-toggle guidance: changing runtime behavior is safer when the control is explicit and owned.

## Implement a freshness-first watchdog

Here is the focused TypeScript example. The endpoint and response shape are an internal contract for this example, not a public vendor API. It expects cumulative counts for one completed window and treats missing, malformed, or stale data as unknown rather than healthy.

```ts
type Sample = {
  failures: number;
  total: number;
  windowEnd: string;
};

type Alert = {
  kind: "threshold_breach" | "stale_sample" | "poll_error";
  summary: string;
  details: Record<string, number | string>;
};

type WatchdogConfig = {
  endpoint: string;
  token: string;
  intervalMs: number;
  timeoutMs: number;
  maxAgeMs: number;
  failureRatio: number;
  consecutiveBreaches: number;
};

const config: WatchdogConfig = {
  endpoint: "https://metrics.internal.example/v1/metrics/query",
  token: process.env.METRICS_TOKEN ?? "",
  intervalMs: 60_000,
  timeoutMs: 5_000,
  maxAgeMs: 120_000,
  failureRatio: 0.05,
  consecutiveBreaches: 2,
};

let breachStreak = 0;
let running = false;

async function notify(alert: Alert): Promise<void> {
  // Replace this boundary with the team's existing notification adapter.
  console.error(JSON.stringify({ at: new Date().toISOString(), ...alert }));
}

async function pollOnce(now = Date.now()): Promise<void> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), config.timeoutMs);

  try {
    if (!config.token) throw new Error("METRICS_TOKEN is empty");

    const response = await fetch(config.endpoint, {
      headers: { authorization: `Bearer ${config.token}` },
      signal: controller.signal,
    });
    if (!response.ok) throw new Error(`metrics status ${response.status}`);

    const sample = (await response.json()) as Sample;
    if (!Number.isFinite(sample.failures) || !Number.isFinite(sample.total)) {
      throw new Error("metrics payload has non-numeric counts");
    }
    if (sample.failures < 0 || sample.total <= 0 || sample.failures > sample.total) {
      throw new Error("metrics counts violate the sample contract");
    }

    const ageMs = now - Date.parse(sample.windowEnd);
    if (!Number.isFinite(ageMs) || ageMs < 0 || ageMs > config.maxAgeMs) {
      breachStreak = 0;
      await notify({
        kind: "stale_sample",
        summary: "Failure metric is missing or stale",
        details: { ageMs: Number.isFinite(ageMs) ? ageMs : "invalid" },
      });
      return;
    }

    const ratio = sample.failures / sample.total;
    breachStreak = ratio >= config.failureRatio ? breachStreak + 1 : 0;

    if (breachStreak === config.consecutiveBreaches) {
      await notify({
        kind: "threshold_breach",
        summary: "Failure ratio crossed the configured threshold",
        details: { ratio, failures: sample.failures, total: sample.total },
      });
    }
  } catch (error) {
    breachStreak = 0;
    await notify({
      kind: "poll_error",
      summary: "Metrics poll could not produce a trustworthy sample",
      details: { error: error instanceof Error ? error.message : String(error) },
    });
  } finally {
    clearTimeout(timeout);
  }
}

async function tick(): Promise<void> {
  if (running) return;
  running = true;
  try {
    await pollOnce();
  } finally {
    running = false;
  }
}

void tick();
setInterval(() => void tick(), config.intervalMs);
```

Notice the guard against overlapping runs — a slow poll doesn't create a second evaluator with competing state. Notification fires when the streak first reaches two, not on every later sample, which avoids repeated pages while preserving decision logs. A recovery notification can be added by tracking the previous state, but I leave it out here because recovery semantics should match the team's incident workflow.

## Test the states, then deploy the scheduler

Test the transitions.

Inject the clock and the fetch function in production code, then cover a fresh sample below threshold, the first breach, the second breach, recovery, stale data, an invalid timestamp, impossible counts, timeout, and an authorization failure. Assert the structured `kind`, not the prose summary. The text can improve without breaking every test.

One config footgun shaped this advice. I once deployed a poller with `METRICS_REGION=us-east-1` while its credential belonged to `us-east-2`; it returned an authorization-looking response, and I spent 47 minutes rotating a perfectly valid token before comparing the region values character by character. I'm not sure why I checked the secret before the endpoint context — habit, probably — but the lasting fix was to log non-secret configuration at startup and make region part of the health check.

Deploy the watchdog away from the workload it observes. Separate failure domains matter — if the application and its only checker disappear together, no threshold logic runs. Give the process a restart policy, a liveness signal, and a dead-man check based on its last successful execution. Log one structured decision per poll with sample time, age, counts, ratio, threshold, streak, duration, and outcome. Never log the token.

Alert delivery needs its own boundary and tests. The custom-appender pattern in the Logback manual is useful even outside Java: application or evaluator code emits a structured event, while an adapter owns transport-specific behavior. In Node.js, keep `notify()` replaceable, cap its duration, and report delivery failures through an independent route. Don't let a slow chat webhook block future metric polls.

Roll out in observe-only mode for several real windows. Compare would-have-alerted decisions with incident records, adjust the threshold with the service owner, and document who receives the page. Then enable delivery behind the explicit flag. Crisp before, crisp after.

## Know the limits before calling it done

This watchdog is not suitable when metrics arrive late by design, labels create thousands of independent series, or alert policy needs joins across services. Use a collector and rule engine in those cases. Stick with event-driven evaluation when every individual failure carries business context that a rolled-up ratio would discard.

The two-breach rule trades speed for noise reduction. It can miss a short, severe spike between polls, and a ratio can hide low-volume pain: one failure out of one request is `100%`, while ten failures in a million may be below policy despite affecting ten people. Pair the ratio with a minimum failure count or a latency objective when the service contract requires it. Choose those conditions with service owners; don't copy mine blindly.

There is also a boot-time gap. The in-memory streak resets whenever this long-running process restarts. That is acceptable only if losing one interval of confirmation matches the risk model. If it doesn't, persist the last sample time and streak with an expiry, or move evaluation into infrastructure designed for durable alert state. A cron-launched process needs that persistence from day one.

Finally, monitor the monitor. Page separately for stale samples and repeated poll errors, keep those routes lower-noise than a confirmed application breach, and make the runbook explain the three states. The goal isn't a clever script. It's an honest signal whose unknown state stays visible.

## References

- Martin Fowler, “Feature Toggles”: https://martinfowler.com/articles/feature-toggles.html
- Logback Manual, “Appenders”: https://logback.qos.ch/manual/appenders.html
