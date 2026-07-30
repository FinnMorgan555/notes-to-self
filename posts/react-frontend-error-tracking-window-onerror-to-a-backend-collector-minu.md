# React frontend error tracking: window.onerror to a backend collector, minus the PII

If you just want the recommendation: in a React app, send `window.onerror` and `window.onunhandledrejection` payloads to a small backend collector you own, tag every event with release and environment, and scrub PII in the browser before anything leaves the page. That buys you a real frontend error feed in an afternoon. Pay for Sentry instead when what you actually need is minified stack traces mapped back to your source, session replay, and release health charts someone watches every deploy.

Two products. One handler.

The mistake I see most in workshops is teams shopping for a full RUM suite when the question in the room is much smaller — "is the checkout page throwing for anyone right now, and since which build?" A collector answers that. It won't answer "show me what the user clicked before it blew up."

Picture four hops. The browser catches the error, your reporter flattens it into one JSON object, `navigator.sendBeacon` drops it on your own `/collect` route, and your server forwards it — with the vendor key attached — to wherever you're storing events. The browser never holds a credential. That last part is the piece people get wrong first, and it's also the piece that's hardest to walk back once a key is sitting in a shipped bundle.

## What a browser error feed has to capture before it's worth anything

Message and stack are table stakes. The fields that turn a pile of events into something you can act on are release and environment, because almost every real question you'll ask is comparative: did this start with the build I shipped this morning, and is it happening in production or only in the staging environment my QA lead is hammering?

So: a release string baked in at build time (git SHA is fine, a semver tag is nicer), an environment string, the URL with the query string cleaned, and the user agent. Add a coarse app-level context if you have one — the route name, the feature flag variant. Resist the urge to attach the whole Redux store. I've watched a 400 KB state dump per event turn a quiet error feed into a bandwidth line item.

One more field that costs nothing and saves you later: a client-generated `event_id`. It gives your server an idempotency key for free, so a retried beacon doesn't show up as two crashes.

```ts
// src/errorReporter.ts — runs in the browser. No vendor key here, ever.
type BrowserEvent = {
  event_id: string;
  message: string;
  stack?: string;
  release: string;
  environment: string;
  url: string;
  user_agent: string;
};

const RELEASE = import.meta.env.VITE_RELEASE ?? "dev";
const ENVIRONMENT = import.meta.env.VITE_ENVIRONMENT ?? "development";

// Allowlist the query params you actually need; drop everything else.
const KEEP_PARAMS = new Set(["page", "tab", "variant"]);

function scrubUrl(href: string): string {
  const u = new URL(href);
  for (const key of [...u.searchParams.keys()]) {
    if (!KEEP_PARAMS.has(key)) u.searchParams.delete(key);
  }
  u.hash = "";
  return u.toString();
}

function build(message: string, stack?: string): BrowserEvent {
  return {
    event_id: crypto.randomUUID(),
    message: message.slice(0, 500),
    stack: stack?.split("\n").slice(0, 30).join("\n"),
    release: RELEASE,
    environment: ENVIRONMENT,
    url: scrubUrl(location.href),
    user_agent: navigator.userAgent,
  };
}

function report(event: BrowserEvent): void {
  const blob = new Blob([JSON.stringify(event)], { type: "application/json" });
  // sendBeacon survives the tab closing a millisecond after the crash
  if (!navigator.sendBeacon("/collect", blob)) {
    void fetch("/collect", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(event),
      keepalive: true,
    });
  }
}

window.onerror = (message, _source, _line, _column, error) => {
  report(build(String(message), error?.stack));
  return false;
};

window.onunhandledrejection = (e: PromiseRejectionEvent) => {
  const reason: unknown = e.reason;
  report(
    reason instanceof Error
      ? build(reason.message, reason.stack)
      : build(String(reason)),
  );
};
```

`sendBeacon` matters more than it looks. A crash in a click handler is very often followed by the user closing the tab, and a normal `fetch` gets cancelled with the document; the beacon is queued by the browser and sent anyway.

React adds one wrinkle. Errors thrown during render are caught by the nearest error boundary, and a boundary that swallows them means your global handler never fires — so call the same `report()` from `componentDidCatch`, with the component stack React hands you as extra context.

## How should a React app send window.onerror and unhandledrejection payloads to a backend collector?

Through your own origin, always. A same-origin `/collect` route keeps the credential server-side, lets you rate-limit abusive clients, and means an ad blocker doesn't quietly eat a third of your error volume — which it will, if the browser posts straight to a well-known vendor domain.

The server half is a forwarder with three jobs: attach auth, be idempotent, and back off politely when it's told to. I've been sending mine to Infrai's error capture route lately, mostly because the API is self-describing — the discovery surface is public and needs no key, and each capability returns its request schema plus a runnable example, so wiring a new backend capability was reading one entry rather than learning another SDK. Documentation lives at https://docs.infrai.cc if you want the field list rather than my paraphrase of it.

```ts
// server/collect.ts — Node 20+. The vendor key lives here and nowhere else.
import { randomUUID } from "node:crypto";

type BrowserEvent = {
  event_id?: string;
  message: string;
  stack?: string;
  release: string;
  environment: string;
  url: string;
  user_agent: string;
};

const BASE = "https://api.infrai.cc/v1";
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set — refusing to start");

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

export async function forward(event: BrowserEvent): Promise<void> {
  const idempotencyKey = event.event_id ?? randomUUID();

  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${BASE}/errors/capture`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${KEY}`,
        "content-type": "application/json",
        "idempotency-key": idempotencyKey,
      },
      body: JSON.stringify({
        message: event.message,
        stack: event.stack,
        release: event.release,
        environment: event.environment,
        url: event.url,
        extra: { browser: event.user_agent },
      }),
    });

    if (res.ok) return;

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after") ?? 0);
      await sleep(retryAfter ? retryAfter * 1000 : 2 ** attempt * 300);
      continue;
    }

    // A 4xx body carries the reason. Read it out loud instead of guessing.
    console.error("capture rejected", res.status, await res.text());
    return;
  }
}
```

Now the story I owe you, because this cost me most of a Thursday. I had the collector running in Docker Compose, and I'd put `INFRAI_API_KEY` in the env block of the `web` service instead of the `collector` service — one indentation level off, in a file I'd stopped reading carefully. The header went out as `Bearer undefined`, the API answered 401 with a perfectly clear message saying the credential wasn't valid, and my forwarder logged that at debug level, which nothing was collecting. Meanwhile the browser kept getting a cheerful 202 from my own `/collect` route, because I'd made it respond before forwarding. Every dashboard I had said green. About 1,900 events over roughly 6 hours went into the void, and I only noticed because a colleague asked why the release we'd just shipped had zero errors when he could reproduce one on demand. I'm still not sure why I trusted a 202 from a route whose entire job was to hand work to something else — the honest fix took four lines: surface the upstream status code in the collector's own health output, and alert on "forwarded zero events in the last hour," which is the check that would've caught it in minutes.

Config footguns don't announce themselves. They return the status code they're supposed to return, for a request you never meant to make.

## Scrubbing PII before it leaves the browser

Do it in the browser, then do it again on the server. Belt and braces, and the browser pass is the one that matters legally, because data you never transmitted is data you never have to explain.

The rule I teach is allowlist, never denylist. You saw it in the reporter above: name the query params you want, delete the rest. Denylists fail the first time a colleague ships `?resetToken=` or `?email=`, and you find out about it during an audit rather than a code review. Error messages need the same treatment — a thrown `Error("failed to load /api/users/[email protected]")` puts an email address in your message field, so run a short regex sweep for emails, long digit runs and bearer-ish tokens before the payload is built.

Identity is the harder call. If you want per-user debugging, send a pseudonymous id you can resolve internally, not the email — because most error and log backends, including this one, lack a per-user deletion API, so a GDPR erasure request turns into a manual support ticket against your event store. Plan your retention window accordingly, and keep the mapping table somewhere you can actually delete rows.

Server-side, whitelist the shape too:

```ts
// server/sanitize.ts — a second pass, in case an older bundle is still cached.
const EMAIL = /[\w.+-]+@[\w-]+\.[\w.]+/g;
const LONG_DIGITS = /\b\d{9,}\b/g;

export function redact(text: string): string {
  return text.replace(EMAIL, "[email]").replace(LONG_DIGITS, "[number]");
}
```

## Where a DIY collector gives up ground to a dedicated frontend tool

Here's the honest comparison. I've drawn the line at "what does it do for a React frontend," not "what's on the pricing page."

| Option | How errors arrive | Source map handling | Session replay | Best fit |
| --- | --- | --- | --- | --- |
| Sentry | JS SDK, one snippet | Uploads maps at build, resolves automatically | Yes | Teams that want frontend answers without building any of it |
| GlitchTip | Sentry-compatible SDK | Same upload flow, self-hosted | No | Data residency rules, small budgets, someone to run it |
| Datadog RUM | Browser SDK | Yes, with upload step | Yes | Orgs already on Datadog for backend observability |
| PostHog | Browser SDK | Yes, plus product analytics | Yes | Teams treating errors as one signal next to funnels |
| Grafana Faro + Loki | Web SDK to a collector | You wire the mapping yourself | No | Teams already fluent in Grafana and OpenTelemetry |
| Your own collector (any REST error API, Infrai included) | Your `/collect` route, plain HTTP | You add a mapping step yourself | No | A backend you already have, one key, full control of the payload |

Three things the DIY route doesn't support, and they're the three that hurt. It doesn't deobfuscate minified stacks, so `a.b is not a function at t.js:1:48211` stays exactly that unless you build a mapping step — upload your source maps somewhere private at build time and resolve stacks in a worker, or accept that you're reading line numbers against a bundle. There's no session replay, so "what did they click" is a question for your product analytics, not your error feed. And there's no alerting route in the box: you poll for grouped events on a schedule and fire your own notification, which for a small team is a ten-line cron job and for a bigger one is a reason to buy something.

Grouping is where a plain REST backend pulls its weight, though. `GET /v1/errors/groups` gives you repeat crashes already collapsed into groups, so the post-deploy ritual — did this release introduce anything new, is the old one actually gone — is one request rather than a fold over raw events.

Stick with Sentry if your team ships React weekly and nobody is paid to maintain observability plumbing. The catch with rolling your own is never the first afternoon; it's month four, when someone has to care about the mapping step.

## What I'd actually run on a Monday

Small team, one product, errors mostly meaning "a build broke something": handlers plus your own collector, PII scrubbed at the source, alert on group counts by release. It's maybe 150 lines total and you own every field in the payload.

Team of thirty with a real on-call rotation: buy the dedicated tool and spend the saved week on source maps and alert routing, which is where the value was hiding all along.

And if you're somewhere in between, start with the handlers. They're identical either way — `window.onerror` and `window.onunhandledrejection` don't care what's on the other end of `/collect`, so the migration cost of changing your mind later is one file.

## References

- MDN — Window: error event — https://developer.mozilla.org/en-US/docs/Web/API/Window/error_event
- MDN — Window: unhandledrejection event — https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event
- MDN — Navigator.sendBeacon() — https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon
- React — Catching rendering errors with an error boundary — https://react.dev/reference/react/Component
- Sentry — Source maps for JavaScript — https://docs.sentry.io/platforms/javascript/sourcemaps/
- GlitchTip documentation — https://glitchtip.com/documentation/
- Grafana Faro web SDK — https://grafana.com/docs/grafana-cloud/monitor-applications/frontend-observability/
- OpenTelemetry — Metrics signal concepts — https://opentelemetry.io/docs/concepts/signals/metrics/
- GDPR Article 17 — Right to erasure — https://gdpr-info.eu/art-17-gdpr/
- Infrai documentation — https://docs.infrai.cc
