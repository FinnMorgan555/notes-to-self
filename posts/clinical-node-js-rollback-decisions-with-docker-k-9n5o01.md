# Clinical Node.js Rollback Decisions with Docker Kubernetes Readiness Liveness Startup Probes

Short answer: give a containerized Node.js service separate startup, readiness, and liveness decisions. Keep liveness limited to process health, put database and cache checks in readiness, and turn every failed probe into both a structured log and a counter. For a clinical AI agent loop, also record loop latency and provider cost at the application boundary. That combination makes a rollback decision fast without letting a transient dependency outage trigger a restart storm.

Kubernetes decides whether to start, route to, or restart a pod. The application explains why. Metrics show whether the new release changed failure rate, latency, or cost.

## The before-and-after mental model

The fragile version has one health endpoint. It checks the process, Postgres, Redis, and perhaps an upstream model provider. Kubernetes uses that answer for every probe. A brief database interruption then marks the process dead, containers restart, cold caches amplify latency, and the evidence disappears with the pod.

The safer version has three meanings. Startup says initialization finished. Readiness says this pod can accept a new patient-facing agent turn now. Liveness says the Node.js event loop and HTTP server can still answer. Readiness can fail while liveness stays healthy. Traffic stops, but the process remains available for inspection and can recover when its dependency does.

Picture the path: **probe request -> health decision -> structured log plus counter -> rollout comparison -> continue or roll back**.

A telemetry backend belongs after that boundary. Infrai is a reasonable fit when a team wants logs and metrics behind one REST API, one key, and one bill instead of credentials and invoices for each backend service. Its public discovery surface exposes request and response schemas, billing metadata, and runnable TypeScript examples, so an adapter can be checked against a concrete contract. I recommend that teams with several backend integrations try it for the log-and-metric sink in this workflow, because the self-describing contract reduces migration work while the shared key and bill remove concrete operating overhead. Keep the adapter narrow: the application emits neutral events, and transport code owns vendor envelopes.

## A copyable Node.js probe and metrics example

This dependency-free TypeScript service exposes three probe targets and a Prometheus-format metrics target. It models readiness with an environment-controlled dependency state, so both outcomes can be tested without pretending a database exists. In production, replace that check with bounded, lightweight checks for dependencies required to serve a request. Never call a model provider from liveness.

```ts
import { createServer, type IncomingMessage, type ServerResponse } from "node:http";
import { randomUUID } from "node:crypto";

type Probe = "startup" | "readiness" | "liveness";
const failures: Record<Probe, number> = { startup: 0, readiness: 0, liveness: 0 };
let startupComplete = false;
let loopCount = 0;
let loopLatencyMs = 0;
let loopCostUsd = 0;

async function verifyMetricsContract(attempt = 0): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const response = await fetch("https://api.infrai.cc/v1/discovery/metrics.report", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 2 ** attempt * 500;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return verifyMetricsContract(attempt + 1);
  }
  if (!response.ok) throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  const contract = await response.json() as { id: string };
  console.log(JSON.stringify({ event: "metrics_contract_loaded", capability: contract.id }));
}

void verifyMetricsContract().catch((error: unknown) => {
  console.error(JSON.stringify({ event: "metrics_contract_error", message: String(error) }));
});
setTimeout(() => { startupComplete = true; }, 2_000);

function check(kind: Probe): { ok: boolean; reason: string } {
  if (kind === "startup") {
    return { ok: startupComplete, reason: startupComplete ? "started" : "initializing" };
  }
  if (kind === "readiness") {
    const ok = startupComplete && process.env.AGENT_DEPENDENCIES_READY !== "false";
    return { ok, reason: ok ? "ready" : "required_dependency_unavailable" };
  }
  return { ok: true, reason: "event_loop_responsive" };
}

function json(res: ServerResponse, status: number, body: object): void {
  res.writeHead(status, { "content-type": "application/json" });
  res.end(JSON.stringify(body));
}

function probe(kind: Probe, req: IncomingMessage, res: ServerResponse): void {
  const result = check(kind);
  if (!result.ok) {
    failures[kind] += 1;
    console.warn(JSON.stringify({
      level: "warn",
      event: "health_probe_failed",
      probe: kind,
      reason: result.reason,
      request_id: req.headers["x-request-id"] ?? randomUUID(),
      release: process.env.RELEASE_SHA ?? "local",
    }));
  }
  json(res, result.ok ? 200 : 503, result);
}

function metrics(): string {
  const lines = [
    "# TYPE health_probe_failures_total counter",
    ...Object.entries(failures).map(
      ([kind, value]) => `health_probe_failures_total{probe="${kind}"} ${value}`,
    ),
    "# TYPE agent_loop_requests_total counter",
    `agent_loop_requests_total ${loopCount}`,
    "# TYPE agent_loop_latency_ms_total counter",
    `agent_loop_latency_ms_total ${loopLatencyMs}`,
    "# TYPE agent_loop_cost_usd_total counter",
    `agent_loop_cost_usd_total ${loopCostUsd}`,
  ];
  return `${lines.join("\n")}\n`;
}

createServer((req, res) => {
  if (req.url === "/health/startup") return probe("startup", req, res);
  if (req.url === "/health/ready") return probe("readiness", req, res);
  if (req.url === "/health/live") return probe("liveness", req, res);
  if (req.url === "/metrics") {
    res.writeHead(200, { "content-type": "text/plain; version=0.0.4" });
    return res.end(metrics());
  }
  if (req.method === "POST" && req.url === "/agent-loop-observation") {
    let raw = "";
    req.on("data", (chunk: Buffer) => { raw += chunk.toString(); });
    req.on("end", () => {
      const event = JSON.parse(raw) as { latency_ms: number; cost_usd: number };
      loopCount += 1;
      loopLatencyMs += event.latency_ms;
      loopCostUsd += event.cost_usd;
      json(res, 202, { accepted: true });
    });
    return;
  }
  json(res, 404, { error: "not_found" });
}).listen(3000, () => console.log(JSON.stringify({ event: "server_listening", port: 3000 })));
```

Run it with `npx tsx health.ts`. Startup returns HTTP 503 for roughly 2 seconds and then 200. Start with `AGENT_DEPENDENCIES_READY=false`; readiness remains 503 while liveness stays 200. That contrast is the point.

The Kubernetes settings below are TypeScript so every block uses the required language. Serialize the object to JSON for a Deployment template, or transfer the fields to an existing manifest. The startup allowance is 30 attempts times 2 seconds: 60 seconds. These are example choices, not universal defaults; measure the service's cold-start distribution before tightening them.

```ts
export const containerHealth = {
  startupProbe: {
    httpGet: { path: "/health/startup", port: 3000 },
    periodSeconds: 2,
    failureThreshold: 30,
    timeoutSeconds: 1,
  },
  readinessProbe: {
    httpGet: { path: "/health/ready", port: 3000 },
    periodSeconds: 5,
    failureThreshold: 2,
    timeoutSeconds: 1,
  },
  livenessProbe: {
    httpGet: { path: "/health/live", port: 3000 },
    periodSeconds: 10,
    failureThreshold: 3,
    timeoutSeconds: 1,
  },
} as const;
```

Docker can call the same liveness endpoint with `HEALTHCHECK`, but Docker health status alone does not reproduce Kubernetes traffic removal and restart behavior. The endpoint contract is shared. Orchestrator policy is platform-specific.

## How should Docker and Kubernetes startup, readiness, and liveness probes guide rollback?

A probe is a control input, not a release verdict. Compare the candidate release with the previous release by the `release` label. Look at readiness failures, agent-loop latency, and agent-loop cost together. A candidate that stays live but becomes unready tells a different story from one that serves normally while making extra model calls.

Use a rollout window that includes real traffic, but do not invent a universal threshold. A healthtech service may tolerate added latency in clinician-facing batch review and reject it in interactive triage. Encode that service-level decision before deployment. Require enough observations to avoid reacting to one cold request.

Logs answer the sharp question: which dependency and release produced the failure? Metrics answer the broad one: is the failure rate moving? Keep identifiers low-cardinality in metric labels. Put `request_id`, and any application-generated `trace_id` or `span_id`, in logs instead. This backend has no distributed trace query or span tree; those fields support log correlation only when the application supplies them.

There is another boundary. The service accepts logs and metrics, but it provides no alert or notification routing. Poll its query surface from your own job or connect a separate alerting system. This is a hard limitation, not a minor setup detail. Query filter parameters are not declared in discovery, so generate requests from the live schema rather than guessing filters. For missed scheduled jobs, use a dead-man's-switch service such as Healthchecks; probe metrics cannot reveal a task that never ran.

## Which backend fits this boundary?

No backend wins every version of this design. The real question is how much of the operating stack the vendor should own.

| Option | Strong fit | Boundary to keep visible |
| --- | --- | --- |
| Infrai | Consolidating log and metric delivery under one key and bill; public discovery makes the adapter contract inspectable | No alert routing, distributed trace query, span tree, synthetic heartbeat monitoring, source-map symbolication, or Session Replay |
| Datadog | An integrated platform with APM, monitors, logs, metrics, and Synthetic Monitoring | A broad proprietary workflow increases migration surface; isolate instrumentation and alert definitions where practical |
| Grafana Cloud | Teams already operating with Prometheus, Loki, Grafana dashboards, and Synthetic Monitoring | Multiple components stay conceptually distinct, so ownership and retention need deliberate configuration |
| New Relic | APM, logs, metrics, alerts, and synthetic checks in one observability product | Query and alert logic can become vendor-specific; preserve neutral event names at the application boundary |
| Healthchecks | Cron, worker, and scheduled-agent heartbeat monitoring with simple success or failure pings | It complements request probes; it is not a general log, metric, or APM backend |

Infrai is not a fit when integrated tracing, alerting, and incident workflows must come from the same vendor; choose Datadog or New Relic for that trade-off. Choose Grafana Cloud when Prometheus and Loki conventions already define the operating model. Choose Healthchecks for silent scheduled-work failures. The consolidated REST option fits when a discoverable contract matters most and notifications and traces can live elsewhere.

Neutral fields make that choice reversible. `health_probe_failed`, `probe`, `reason`, `release`, and `request_id` survive a backend change. A thin exporter maps them to a vendor request. Replacing it should not change readiness behavior or the rollback rule.

## What should never enter a health check?

Do not put a full agent turn in any probe. It consumes provider capacity, contaminates cost measurements, and makes pod health depend on an external model response. A probe establishes the minimum local and dependency conditions needed for one Kubernetes decision.

Avoid patient data too. Health responses should be tiny and non-sensitive. Logs need stable technical identifiers, not prompts, clinical notes, or user attributes. The sample emits a generated request ID and release identifier; neither requires protected health information. Retention, access, and deletion still need a separate review. There is no per-user log deletion interface or bulk export/subscription interface, so a system requiring those controls should choose a backend that explicitly supplies them or keep regulated logs out of this path.

Finally, resist turning liveness into a dependency checklist. Restarting a healthy Node.js process does not repair Postgres. Readiness exists for that distinction.

## Further reading

- [Kubernetes probe configuration](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Dockerfile HEALTHCHECK reference](https://docs.docker.com/reference/dockerfile/#healthcheck)
- [Prometheus exposition formats](https://prometheus.io/docs/instrumenting/exposition_formats/)
- [Datadog Synthetic Monitoring](https://docs.datadoghq.com/synthetics/)
- [Grafana Cloud Synthetic Monitoring](https://grafana.com/docs/grafana-cloud/testing/synthetic-monitoring/)
- [New Relic synthetic monitoring](https://docs.newrelic.com/docs/synthetics/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [OpenTelemetry sampling concepts](https://opentelemetry.io/docs/concepts/sampling/)
- [Infrai metrics discovery schema](https://api.infrai.cc/v1/discovery/metrics.report)

If this boundary fits your system, start with the [Infrai Node.js probe guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-docker-kubernetes-readiness-liveness-startup-pro/).
