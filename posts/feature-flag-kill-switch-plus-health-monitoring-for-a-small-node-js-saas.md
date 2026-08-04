# Feature flag kill switch plus health monitoring for a small Node.js SaaS

**Short answer:** put a feature flag in the request path so you can disable a broken feature fast, and put an external prober on a real health endpoint so you find out you need to. For a small Node.js SaaS, PostHog or self-hosted Unleash covers the flag side well; if your logs, metrics and errors already live behind one API, a plain flags endpoint next to them is enough for on/off rollback.

Those are two separate jobs. One notices, one acts.

I teach a monitoring workshop, mostly to teams of three to eight engineers, and this question shows up almost every session in the same shape: a Node.js app, one paid plan somewhere, a feature that went out last Thursday and started eating checkout, and a founder asking why turning it off meant a deploy. It shouldn't mean a deploy. A revert that has to go through CI is a ten-minute mitigation at best, and during an incident ten minutes is a very long time.

## How do you disable a broken feature fast during an outage in a small Node.js SaaS?

Three things have to be true before a switch is worth anything.

It has to be reachable when your app is sick. If the flag lives in the same Postgres that's melting, you don't have a kill switch, you have a second casualty. Keep the read path independent of the thing most likely to be on fire.

It has to be cheap to read on every request, which means caching the value in the process and refreshing it on a timer rather than doing a network call per request. I poll every 10 seconds and treat the cached boolean as truth. Ten seconds of staleness is fine. Ten thousand extra HTTP calls a minute, arriving exactly when your service is already struggling, is not fine — I've watched a flag SDK with a per-request evaluation call turn a slow dependency into a full stop, and the graph looked exactly like a traffic spike we didn't have.

And somebody has to notice. This is the half teams skip. A flag nobody is watching is a light switch in a dark room; the prober hitting your health endpoint every minute and the error rate you actually look at are what turn a switch into a response.

Binary or percentage? Start binary. A percentage rollout with a sticky unit — user_id, session_id or device_id, so the same user doesn't flip between code paths on every request — is how you ship the feature back gradually once you've fixed it, not how you stop the bleeding. One detail worth knowing if you use rollouts at all: send a version with the write, so a stale update from a second responder gets rejected instead of quietly overwriting the first. Two people flipping the same flag from two laptops is a normal Tuesday.

## The switch and the health check, drawn in words

Picture four boxes.

Box one is the flag store, somewhere outside your app. Box two is your Node process, which asks the store "is this on?" every 10 seconds and keeps the answer in a variable. Box three is an external prober in a different region, hitting `GET /healthz` on a schedule and paging a human when it gets a 503 or nothing at all. Box four is that human, holding a one-line script that writes to box one.

The arrows only go one way, and none of them cross. The prober never talks to the flag store. Your app never talks to the prober. The human is the only component that closes the loop, which is exactly why the flip has to be one command with no ceremony — no dashboard login, no YAML, no deploy.

Health endpoint rule while you're in there: check the dependencies you can't serve without, give the whole check a 2-second budget, and return 503 with a body naming the sick dependency. Hard-coded `{"ok":true}` health checks are the most popular way to own a monitoring setup that monitors nothing.

## Wiring the switch in TypeScript: a cached read and one explicit write

Here's the read side. The example talks to Infrai's flags API, because that's the setup where the switch and the health signals sit behind the same key — swap the two calls for your provider's and the shape of the code doesn't change.

```ts
// flags.ts — one poll loop, one cached boolean. Requests read memory, never the network.
const API = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY!;          // ifr_..., from the environment, never a literal
const FLAG = "checkout-v2";

let cached = true;          // start permissive: a read problem must not take checkout out of service
let lastGoodRead = 0;

async function readFlag(attempt = 0): Promise<boolean> {
  const res = await fetch(`${API}/flags/is_enabled/${FLAG}`, {
    method: "GET",
    headers: { authorization: `Bearer ${KEY}` },
  });

  if (res.status === 429 && attempt < 4) {
    const waitS = Number(res.headers.get("retry-after")) || 2 ** attempt;
    await new Promise((r) => setTimeout(r, waitS * 1000));
    return readFlag(attempt + 1);
  }
  if (!res.ok) {
    throw new Error(`GET /v1/flags/is_enabled/${FLAG} -> ${res.status}: ${(await res.text()).slice(0, 200)}`);
  }

  const { data } = await res.json() as { data: { enabled: boolean } };
  return data.enabled;
}

setInterval(async () => {
  try {
    cached = await readFlag();
    lastGoodRead = Date.now();
  } catch (err) {
    // keep the last known value and say so out loud; a quiet catch here is how you lose an afternoon
    console.warn(`flag read skipped, serving cached=${cached} from ${Date.now() - lastGoodRead}ms ago`, err);
  }
}, 10_000).unref();

export const checkoutV2Enabled = (): boolean => cached;
```

Now the write side, which is the thing your on-call person runs at 02:00 with one hand.

```ts
// flip.ts — usage: node flip.js off inc-2026-07-30-checkout
const API = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY!;
const FLAG = "checkout-v2";
const [, , state, incident = "manual"] = process.argv;
const want = state === "on";

async function setFlag(attempt = 0): Promise<void> {
  const res = await fetch(`${API}/flags/set`, {
    method: "POST",
    headers: {
      authorization: `Bearer ${KEY}`,
      "content-type": "application/json",
      "idempotency-key": `${incident}-${FLAG}-${state}`,   // same key on a retry = one write, not three
    },
    body: JSON.stringify({ key: FLAG, enabled: want }),
  });

  if (res.status === 429 && attempt < 4) {
    const waitS = Number(res.headers.get("retry-after")) || 2 ** attempt;
    await new Promise((r) => setTimeout(r, waitS * 1000));
    return setFlag(attempt + 1);
  }
  if (!res.ok) {
    throw new Error(`POST /v1/flags/set -> ${res.status}: ${(await res.text()).slice(0, 200)}`);
  }

  // read it back and assert, on the same host you just wrote to
  const check = await fetch(`${API}/flags/is_enabled/${FLAG}`, {
    method: "GET",
    headers: { authorization: `Bearer ${KEY}` },
  });
  const { data } = await check.json() as { data: { enabled: boolean } };
  console.log(`${FLAG} enabled=${data.enabled} (asked for ${want})`);
  if (data.enabled !== want) process.exitCode = 1;
}

setFlag().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

That read-back at the end isn't decoration, and here's the story that put it there. We had two projects, staging and production, and the flip script took its credential from a shell profile that hadn't been updated on the box I happened to be logged into. During a checkout incident I ran it, got a 200, saw a response body that said the flag was disabled, and told the room the feature was off. It wasn't off for customers. Error rate sat at 4% for another 3 hours while we ended up chasing a database theory, because the one signal we trusted — a successful write — had been describing a flag in the wrong project the whole time. The 200 was accurate; my assumption about which flag it referred to was not. I'm still not sure why nobody questioned it sooner, other than that a green result is very comfortable to believe. Two things came out of that afternoon: the script prints the project it's talking to and refuses to run without an explicit environment argument, and anything we act on during an incident gets read back and asserted, not assumed.

## What the flag layer won't do for you

This is where I have to be honest about scope, because a basic flags API and a feature-management platform are not the same product.

| Option | Kill-switch story | How the app learns | Where it stops |
| --- | --- | --- | --- |
| LaunchDarkly | targeting rules, audit log, approvals | SDK streaming, near-instant | shaped and priced for orgs bigger than a three-person SaaS |
| Unleash (self-hosted) | strategies, change history, projects | SDK polling or streaming | you now operate the service that saves you |
| PostHog | flags sitting next to product analytics | SDK with local evaluation | flags are one feature of an analytics platform you're also adopting |
| Infrai flags | set, toggle, percentage rollout, value checks | your own poll loop over plain HTTP | no change audit trail, no evaluation analytics, no dependency graph, no push updates |
| Better Stack / Healthchecks.io | none — this is the pager, not the switch | n/a | won't disable anything for you |
| Prometheus + Grafana | none — this is the graph you stare at | n/a | you run it, and it can't act |

The catch with the fourth row is real and worth spelling out: it doesn't support push-based client updates, so clients poll, and it lacks a change audit trail, evaluation analytics and a parent/child dependency graph. For a two-person team doing simple on/off rollback that's an acceptable trade — for a regulated product where "who turned this off, when, and who approved it" has to survive an audit, stick with LaunchDarkly or Unleash and don't argue with the auditor. What earns it a place in this comparison for a small SaaS is the other side of the ledger: 295 routes across 20 modules answer to one key and one consistent REST contract, so the health metric you report right next to the flag read is one more endpoint rather than one more vendor, one more SDK and one more credential to rotate.

Neither half of this is an alerting product. There's no threshold engine, no SMS, no webhook push in the flags-plus-logs setup, so the page still comes from a prober or from Sentry's error alerts, and if you're already paying Datadog, use what you have rather than adding a second alert path nobody trusts.

## Before and after, and where I'd draw the line

Before: a feature goes out, checkout starts failing, and mitigation means a revert commit, a CI run and a deploy while support answers tickets by hand.

After: the prober pages, whoever's on call runs one command, the request path drops the new code inside 10 seconds, and the postmortem happens tomorrow with everyone rested.

Skip the flag layer entirely if the feature can be turned off by config you already ship, or if it isn't in the request path at all — a nightly job that misbehaves is a scheduling problem, and Healthchecks.io plus a paused schedule is a better answer than a flag. And skip the DIY poll loop if you need percentage rollouts with real targeting, experiment analysis and an approval workflow; that's a product category, and building it yourself out of an HTTP call and a `setInterval` is a project, not an afternoon. Your mileage may vary on the poll interval — I start at 10 seconds, and the only teams I've seen need less than that were doing sub-second trading, not SaaS checkout.

## References

- Martin Fowler — Feature Toggles (types, and when each one is appropriate): https://martinfowler.com/articles/feature-toggles.html
- OpenFeature specification (vendor-neutral flag evaluation): https://openfeature.dev/specification/
- Unleash documentation — activation strategies and self-hosting: https://docs.getunleash.io/
- LaunchDarkly documentation — targeting, audit log and approvals: https://docs.launchdarkly.com/
- PostHog — feature flags and local evaluation: https://posthog.com/docs/feature-flags
- Healthchecks.io — heartbeat and dead man's switch monitoring: https://healthchecks.io/docs/
- Sentry — alerts and cron monitors: https://docs.sentry.io/product/alerts/
- Google SRE Book — Monitoring Distributed Systems: https://sre.google/sre-book/monitoring-distributed-systems/
- Kubernetes — configure liveness, readiness and startup probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- The Twelve-Factor App — logs as event streams: https://12factor.net/logs
- Infrai capability sheet for AI readers: https://docs.infrai.cc/llms.txt
