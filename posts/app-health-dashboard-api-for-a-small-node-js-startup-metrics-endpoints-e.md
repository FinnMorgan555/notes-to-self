# App Health Dashboard API for a Small Node.js Startup: Metrics Endpoints, EU and US Hosting

## TL;DR

For a small Node.js startup, start with a boring `GET /health` endpoint plus an external uptime check, then add OpenTelemetry metrics only when the dashboard needs answers that a health check cannot provide. I would choose a managed monitoring service for the first alert path, compare its EU and US data-region options before sending it identifiers, and keep Prometheus for teams prepared to operate collection and retention themselves.

The endpoint is the contract. The dashboard is the view.

I teach logs, metrics, and alerting, so I tend to draw the incident path in words: request arrives -> app decides if its dependencies are usable -> monitor records the result -> an alert reaches a person. A green page that skips the dependency check is decoration, not health monitoring. OpenTelemetry defines metrics as measurements captured at runtime, which makes them useful for latency, error rate, and capacity questions after the basic check is trustworthy.

## What API should a small Node.js startup use for an app health dashboard without Prometheus?

Use a small HTTP health API first. `GET /health` should return success only after checking the dependencies that must work for a user request to succeed. For a service whose primary path needs its database, that normally means a cheap database readiness check with a strict timeout. Don't put a slow report query in this route; a health route gets called often, and its job is to expose a fast operational signal, not to rehearse the whole application.

I like two states in the response: `ok` for a ready service and `degraded` for a dependency that needs attention. The status code is what an uptime probe can judge quickly, while the JSON body gives a human enough context during triage. Keep the body free of customer data, access tokens, and raw exception messages. It's easy to turn a dashboard into an accidental data export.

```ts
import { createServer } from "node:http";

type Health = { status: "ok" | "degraded"; database: "ok" | "unavailable" };

async function databaseReady(): Promise<boolean> {
  // Replace this with the application's inexpensive readiness query.
  return true;
}

const server = createServer(async (request, response) => {
  if (request.method !== "GET" || request.url !== "/health") {
    response.writeHead(404).end();
    return;
  }

  const database = (await databaseReady()) ? "ok" : "unavailable";
  const health: Health = {
    status: database === "ok" ? "ok" : "degraded",
    database
  };

  response
    .writeHead(health.status === "ok" ? 200 : 503, { "content-type": "application/json" })
    .end(JSON.stringify(health));
});

server.listen(3000);
```

I have seen the other failure mode: a check that only proves the Node process exists. It stayed green while a required downstream dependency was unreachable, so the on-call engineer had to infer the real state from customer reports. Test this endpoint in CI with the dependency stubbed healthy and unavailable, then have the deployment platform hit it before routing production traffic. Small steps. Big payoff.

## Choosing the monitor and dashboard around that endpoint

There are three reasonable lanes. Prometheus gives you direct control of scraping, storage, alert rules, and query language, but the catch is that the team owns those pieces. Grafana Cloud, Datadog, and Better Uptime represent managed choices with different emphases: metrics and dashboards, broader application telemetry, and simple external checks respectively. I don't treat any one of them as a default; the useful question is who will investigate an alert at 03:00 and what evidence they need when they open it.

| Option | Good fit | Trade-off to accept |
| --- | --- | --- |
| Prometheus with Grafana | A team already running infrastructure and needing custom metric queries | You operate scraping, storage, upgrades, and alerting |
| Grafana Cloud | Teams that want managed dashboards and an OpenTelemetry-friendly path | Confirm the plan and region choices match your retention and data needs |
| Datadog | Teams correlating application metrics with logs and traces across several services | It can be more surface area than one health endpoint needs |
| Better Uptime | A simple external availability signal and clear incident notifications | It is not a substitute for in-process business metrics |

For the first week, I would wire the health URL to an external monitor and give the team a dashboard with availability, response time, deployment version, and the timestamp of the last successful dependency check. Add an error-rate metric when the app has a meaningful request volume. Add a latency histogram when a p95 answer will change a decision. OpenTelemetry is a sensible instrumentation vocabulary here because its metrics model describes counters, gauges, and histograms, but don't collect labels that explode into thousands of distinct time series.

This is where cost can surprise a team. On one project, I estimated $80 per month for telemetry and then saw a $612 bill after a release attached a request ID to a metric label; every request became a new series. I spent 47 minutes comparing the new dashboard dimensions with the release diff before I found that label. The chart looked ordinary because its request count was ordinary, while the bill reflected the number of distinct label combinations. We removed the request ID, kept only dimensions that an on-call engineer could actually group by, capped high-cardinality fields, and put a usage review beside the deployment checklist. I now ask one deliberately dull question in review: could this value be different for every request? If the answer is yes, it belongs in a trace or a log, not in a metric label. I still remember that invoice.

## EU and US hosting: decide what the monitor may receive

A monitor's region is separate from where your Node.js app runs. Start by writing down the locations of the app, its users, the probe locations you need, and the observability provider's storage region. Then decide what leaves the app. A public health endpoint can often reveal only a status and version; it does not need an email address, account ID, IP address, or stack trace. This reduction is practical security work, and it makes a later region decision less fraught.

For EU-facing products, have counsel or the privacy owner review the actual data-processing terms and transfer mechanism for the provider you select. GDPR Article 17 describes a right to erasure in defined circumstances, which is one reason retention, deletion workflow, and searchable log content belong in procurement instead of an afterthought. I'm not sure why these questions still get deferred until the dashboard is already full, but your mileage may vary with a very small team.

Be specific during vendor evaluation. Ask where metrics, logs, backups, and alert metadata live; whether an EU storage choice covers each of those; which probe regions can test your public endpoint; and how deletion requests are handled. Save the answers with the architecture decision. If you only need uptime, a monitor checking from an EU location and another from the US can show geographic reachability without pushing detailed application telemetry anywhere.

Keep private readiness checks private.

A useful split is a public `/health` route that returns a minimal result for uptime checks and an authenticated internal readiness route for operational detail. Rate-limit the public endpoint, document its meaning, and make the response stable enough that an alert rule won't flap whenever someone changes a message.

## Alerting, deployment, and the point where Prometheus earns its keep

Alert on symptoms a user would notice: sustained failed health checks, an error-rate threshold, or latency that stays above the service objective. A single failed probe is often noise; a window of failures with a named owner and a runbook link is actionable. The runbook should say how to confirm the current deployment version, inspect the dependency state, and roll back through the normal release process. Don't page a person for a graph that has no expected response.

Prometheus becomes attractive when the team needs to aggregate many service-level metrics, write custom recording and alerting rules, or keep a self-managed monitoring stack close to its workloads. Stick with a managed monitor when the system is one or two Node.js services and the real requirement is reliable external availability, not a new operations specialty. Neither path excuses missing tests: exercise the health route, alert configuration, and notification target in a staging environment after changes.

I also add an explicit ownership check to each release. Who receives the alert? Who can acknowledge it? Who is allowed to change the threshold? These questions feel administrative until an incident lands on a silent channel. The diagram-in-words is still the test — user request -> health signal and application metrics -> dashboard and alert -> a person with a documented next action. If one arrow is vague, fix that arrow before adding another chart.

## References

- https://opentelemetry.io/docs/concepts/signals/metrics/
- https://gdpr-info.eu/art-17-gdpr/
- https://prometheus.io/docs/introduction/overview/
- https://grafana.com/docs/grafana-cloud/
- https://docs.datadoghq.com/
- https://betterstack.com/docs/uptime/
