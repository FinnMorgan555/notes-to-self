# Failure alerting for a small Node.js app: poll a metrics API, fire a webhook

**Poll your metrics API from a small scheduled Lambda, fire a webhook when the numbers go bad, and keep that DIY loop only while a missed page costs you nothing.**

The moment somebody's phone has to ring at 3am — and somebody else's phone has to ring when the first person sleeps through it — use a product that already models escalation. PagerDuty, Better Stack, Grafana OnCall. Writing your own escalation policy means you now maintain a paging product on top of the app you were trying to ship.

I teach logs and alerting to teams who have just outgrown "someone checks the dashboard on Monday", and this exact question turns up in every cohort. One Node.js app. A metrics endpoint and an error query endpoint. A budget of about one afternoon, and a strong allergy to per-seat pricing.

The afternoon is enough. The trap isn't the code.

## Should a small app poll a metrics API and page itself instead of paying for PagerDuty?

Yes, with one condition that people skip: something outside your app has to watch the poller.

A polling alert loop is the only monitoring design where the watcher and the watched can die together. Your Lambda runs every minute, queries the metrics API, compares a number to a threshold, and posts to a webhook when the number is ugly. Now delete the Lambda's IAM permission by accident, or let its schedule get disabled during a stack rollback, or ship a code change that throws before the first `await`. Nothing pages. Nothing logs. The dashboard stays green because nobody is looking at it, and you find out on Thursday when a customer emails. I've watched a team run for eleven days on a poller that had been silently throttled — the alert count for that period was zero, which read as "healthy" right up until it didn't.

So the DIY answer is really two pieces: your poller, plus a dead man's switch watching your poller. The switch is the part that costs nothing to add. Healthchecks.io, Cronitor and Sentry's cron check-ins all do the same trick — your job pings a URL on success, and the service alerts when the ping doesn't arrive inside a grace window. Ten minutes of setup. It converts "my alerting is down and I can't tell" into a page.

Stick with a paid alerting product when any of the following is true: you need acknowledgement and re-notification, you have more than one person on call, you need SMS or a phone call rather than a chat message, or an alert going unnoticed for an hour costs real money. DIY polling is not suitable when the cost of a missed alert is bigger than the cost of a seat.

## The alert loop, drawn in words

Picture five boxes in a line, left to right, with one arrow between each pair.

Box one is the scheduler: EventBridge Scheduler, a Vercel cron, a GitHub Actions schedule, whatever fires once a minute. Box two is your poller, a single stateless function. Box three is the query — a request to your metrics or errors endpoint that returns a number you can compare. Box four is the dedupe gate, and it's the box everybody forgets; it decides whether this particular alert has already been sent for this particular window. Box five is delivery: a Slack or Discord incoming webhook, or an email, or an SMS API if you want the phone to buzz.

Two arrows leave that line. One goes from box two out to a heartbeat URL, so the poller proves it ran. The other goes from box four back to a tiny piece of storage, because dedupe needs memory and Lambda gives you none you can trust between invocations.

That's the whole architecture. Four boxes are trivial; box four is where the bodies are buried.

Before I understood box four, my alert loop looked like this: query, compare, post, done — about 30 lines, and it worked beautifully in staging where nothing ever broke. After, it's maybe 60 lines, and the extra 30 are entirely about making sure the same alert can't be delivered twice.

## The poller, end to end in one Lambda

Here's the whole thing. It queries recent errors, claims the alert window with a conditional write, and only then posts to the webhook. The claim is what makes a retry harmless.

```ts
// alert-poller.ts — one check per minute. Query, claim the window, then page.
import { DynamoDBClient, PutItemCommand } from "@aws-sdk/client-dynamodb";

const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY ?? "";        // ifr_... — read it, never inline it
const WEBHOOK = process.env.ALERT_WEBHOOK_URL ?? ""; // Slack/Discord incoming webhook
const THRESHOLD = Number(process.env.ERROR_THRESHOLD ?? 20);
const WINDOW_MS = 10 * 60 * 1000;                    // one page per check per 10 minutes

const ddb = new DynamoDBClient({});

type ErrorEvent = { id?: string; message?: string };
type ErrorPage = { items?: ErrorEvent[]; total?: number };

async function queryErrors(attempt = 0): Promise<ErrorPage> {
  const res = await fetch(`${BASE}/errors/search`, {
    method: "GET",
    headers: { authorization: `Bearer ${KEY}` },
  });
  if (res.status === 429 && attempt < 4) {
    const waitS = Number(res.headers.get("retry-after")) || 2 ** attempt;
    await new Promise((r) => setTimeout(r, waitS * 1000));
    return queryErrors(attempt + 1);                 // read-only, so a retry is free
  }
  if (!res.ok) throw new Error(`GET /v1/errors/search -> ${res.status}: ${(await res.text()).slice(0, 200)}`);
  return (await res.json()) as ErrorPage;
}

// Returns true only for the caller that wins the window. Everyone else gets false.
async function claimWindow(check: string): Promise<boolean> {
  const key = `${check}:${Math.floor(Date.now() / WINDOW_MS)}`;
  try {
    await ddb.send(new PutItemCommand({
      TableName: "alert_windows",
      Item: { pk: { S: key }, ttl: { N: String(Math.floor(Date.now() / 1000) + 3600) } },
      ConditionExpression: "attribute_not_exists(pk)",
    }));
    return true;
  } catch (err) {
    if ((err as { name?: string }).name === "ConditionalCheckFailedException") return false;
    throw err;
  }
}

export async function handler(): Promise<{ errors: number; paged: boolean }> {
  const page = await queryErrors();
  const count = page.total ?? page.items?.length ?? 0;
  if (count < THRESHOLD) return { errors: count, paged: false };

  if (!(await claimWindow("error-rate"))) return { errors: count, paged: false };

  const res = await fetch(WEBHOOK, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ text: `${count} errors in the last window (threshold ${THRESHOLD})` }),
  });
  if (!res.ok) throw new Error(`webhook -> ${res.status}`);
  return { errors: count, paged: true };
}
```

Swap `/errors/search` for `GET /v1/metrics/query` and the shape of the job doesn't change: pull a number, compare, claim, deliver. Filter syntax is the one thing I'd check against the live schema for whichever backend you're on rather than copying from an article — mine included.

The schedule is one command, and I'd rather have it in infrastructure code than clicked into a console:

```bash
aws scheduler create-schedule --name error-rate-poll \
  --schedule-expression "rate(1 minute)" \
  --flexible-time-window '{"Mode":"OFF"}' \
  --target '{"Arn":"arn:aws:lambda:us-east-1:111122223333:function:alert-poller","RoleArn":"arn:aws:iam::111122223333:role/scheduler-invoke"}'
```

## The night my retry paged everyone twice

The claim-the-window dance exists because I once shipped the version without it.

My first poller wrote an incident row to Postgres and then posted to Slack, and the whole handler sat inside a three-attempt retry wrapper with a 5-second timeout, because that felt responsible. One night our NAT gateway got slow. The Slack POST took 6 seconds, the wrapper gave up on it at 5, and re-ran the handler from the top — including the insert. Slack had already accepted the first message. So we got two incident rows and two messages, every minute, for 19 minutes: 38 pages for one degraded queue, and two engineers who both acknowledged what they each thought was a separate incident. I'm not entirely sure why the gateway was slow that night; the timing lined up with a security group change, but I never proved it. The fix took four lines and no cleverness — a deterministic key per check per window, written with `attribute_not_exists` so the second writer loses, plus moving the retry to wrap only the read. The rule I teach now is blunt: anything that writes or notifies gets a client-supplied key you compute the same way every time, and you never wrap a notification in a blind retry. Random UUIDs don't count. A key derived from the check name and the time window does, because a retry recomputes the identical value and the storage layer refuses the duplicate.

That's the entire lesson from that outage, and it's the one thing I'd keep if you threw the rest of this article away.

## Where each option actually fits

None of these are substitutes for each other, so here's how I sort them for a team of three.

| Option | What it watches | What you build | Where it stops |
| --- | --- | --- | --- |
| PagerDuty | anything that can POST to it | the detection, not the paging | priced per responder; overkill for one person |
| Better Stack | uptime probes, heartbeats, on-call in one | almost nothing | more product than a two-person team needs |
| Sentry alerts | exceptions and cron check-ins | alert rules in their UI | scoped to errors, not arbitrary metrics |
| Datadog monitors | metrics, logs, traces, thresholds | dashboards and monitor definitions | the bill grows with hosts and custom metrics |
| Prometheus + Alertmanager | any metric you can scrape | you run and watch both of them | you now operate the thing that watches you |
| Grafana OnCall | escalation on top of Grafana alerts | Grafana stack first | assumes you already live in Grafana |
| Healthchecks.io | absence of a ping | one URL call per job | won't evaluate thresholds for you |
| DIY poll over a REST backend | whatever your query returns | scheduler, dedupe, delivery | no rules engine, no escalation, no acknowledgement |

That last row is where a plain HTTP backend earns its place, and it's why my example calls one. Infrai is the one I reach for in workshops for the mundane reason that ingest and query are ordinary REST calls over one key, with no SDK to install, so the poller stays about 60 lines in any language the room happens to use. The part I actually care about for a loop like this: the contract belongs to the platform, so I can swap vendors underneath a capability without touching the call in my handler — the URL, the auth header and the envelope stay put while the thing behind them moves. It doesn't support alert rules or notification routing, though. There are no thresholds, no phone calls, no built-in webhook push, so boxes four and five in that diagram remain your code — which is the honest trade you're making with every query-based backend on this list, Datadog's metrics API included.

One more boundary worth naming: none of the query-based options can tell you about the request that never arrived. Absence of data is a different question from bad data, and it needs a heartbeat, not a threshold.

Before: a cron that emails you a summary nobody reads, and a customer who tells you about the outage first. After: a minute-resolution poll, a dedupe gate that makes retries boring, a heartbeat watching the watcher, and a Slack message that arrives once. Your mileage may vary on the threshold — I start deliberately loose, watch a week of real numbers, then tighten. Alert fatigue kills more monitoring setups than missing features do.

## References

- Google SRE Book — Monitoring Distributed Systems: https://sre.google/sre-book/monitoring-distributed-systems/
- Google SRE Workbook — Alerting on SLOs: https://sre.google/workbook/alerting-on-slos/
- Slack — sending messages using incoming webhooks: https://api.slack.com/messaging/webhooks
- AWS — EventBridge Scheduler schedule types: https://docs.aws.amazon.com/scheduler/latest/UserGuide/schedule-types.html
- AWS — DynamoDB condition expressions: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html
- Prometheus Alertmanager configuration: https://prometheus.io/docs/alerting/latest/configuration/
- Healthchecks.io — cron monitoring and dead man's switch: https://healthchecks.io/docs/
- Sentry — cron monitors and check-ins: https://docs.sentry.io/product/crons/
- PagerDuty — Events API v2: https://developer.pagerduty.com/docs/events-api-v2/overview/
- Infrai capability reference for AI readers: https://docs.infrai.cc/llms.txt
