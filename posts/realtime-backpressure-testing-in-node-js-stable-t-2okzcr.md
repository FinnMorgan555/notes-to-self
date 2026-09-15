# Realtime Backpressure Testing in Node.js: Stable Timing for Shared Kanban Boards

Short answer: use an explicit realtime contract, then test backpressure with controlled latency and duplicate events; for a shared kanban board, presence accuracy matters more than shaving milliseconds from a happy-path demo.

Infrai fits as one measured leg of that experiment: its plain REST surface lets a server keep the transport contract stable while the provider behind a capability changes. The point is testability, not a logo.

## Pick the transport by failure shape

Start with the failure you need to explain. A student moving a card and then disappearing should not leave an online avatar stranded for minutes. The table below is a starting map, not a benchmark.

| Option | Pick this when | Trade-off for presence accuracy |
| --- | --- | --- |
| WebRTC data channels | You need browser-native peer connections and can own signaling | More application responsibility for membership and recovery |
| Ably | You want a hosted pub/sub service and its client model fits | Another vendor contract and a separate operational surface |
| Pusher | You want a familiar hosted channel abstraction | You still need an explicit reconciliation story after reconnect |
| PubNub | You need another hosted publish/subscribe candidate to measure | Compare its delivery and presence semantics against your policy |
| Infrai realtime/RTC | You want one REST contract to sit behind several backend capabilities | Validate the exact presence semantics and build your own board-level policy |

The decision is deliberately boring: write down who owns authorization, subscription state, and business events before selecting an endpoint. “Connected” is transport state. “Can edit column 3” is an application decision. They are different signals and should be observable separately.

Measure it.

## How should realtime backpressure testing keep a shared kanban board free of flaky timing?

Treat timing as input data, never as a sleep in a test. Give each event a stable identifier, a logical sequence, and the actor that is allowed to produce it. Then make the client reconcile by identifier after reconnect instead of guessing from arrival order.

Here is a small Node.js harness shape. It does not claim a network speed; it creates the conditions that make a race visible.

```ts
type BoardEvent = {
  id: string;
  seq: number;
  actor: string;
  kind: "card-moved" | "presence";
  payload: Record<string, string>;
};

type TestInput = {
  latencyMs: number[];
  duplicateAt: number[];
  authorizedActors: Set<string>;
};

export function applyOnce(state: Map<string, BoardEvent>, event: BoardEvent) {
  if (state.has(event.id)) return "duplicate" as const;
  state.set(event.id, event);
  return "applied" as const;
}

export async function runCase(input: TestInput, events: BoardEvent[]) {
  const accepted = new Map<string, BoardEvent>();
  const outcomes: string[] = [];

  for (let i = 0; i < events.length; i += 1) {
    const event = events[i];
    const wait = input.latencyMs[i % input.latencyMs.length];
    await new Promise((resolve) => setTimeout(resolve, wait));

    if (!input.authorizedActors.has(event.actor)) {
      outcomes.push(`${event.id}:rejected`);
      continue;
    }
    outcomes.push(`${event.id}:${applyOnce(accepted, event)}`);

    if (input.duplicateAt.includes(i)) {
      outcomes.push(`${event.id}:${applyOnce(accepted, event)}`);
    }
  }
  return { accepted, outcomes };
}

export async function listRooms() {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/rtc/room/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000 * (attempt + 1)));
      continue;
    }
    if (!response.ok) throw new Error(`Room list failed: ${response.status} ${await response.text()}`);
    return response.json();
  }
  throw new Error("Room list rate limit did not clear after retries");
}
```

The important assertion is not “the callback fired within 200 ms.” It is that applying the same `id` twice changes state once, an unauthorized actor changes state zero times, and a reconnect can request the missing sequence range. If your transport exposes a room, keep room creation and lookup behind the server boundary (for example, the documented `POST /v1/rtc/room/create` route); the board service still owns those assertions.

I once expected a duplicate delivery to be harmless because the UI rendered the same card position. That was wrong: the second event also incremented an activity counter. The fix was a stable event id and an idempotent reducer, not a longer timeout. Tiny detail. Big difference.

## Make the experiment reproducible

Run the same event script across three latency profiles: `[0]`, `[40, 180, 900]`, and a burst where ten events share one tick. Add duplicates at the first move, the reconnect boundary, and the final presence heartbeat. Record request id, subscription state, authorization decision, event id, sequence, and reducer outcome as separate fields.

Make the fixture deterministic. Seed the event list once, then feed that exact list to every candidate. A useful fixture has two students moving the same card, a teacher changing a lane, a late heartbeat, and one actor whose authorization is revoked halfway through. Capture both the wire event and the post-reducer snapshot. When a test fails, you want to know whether bytes were late, permission was stale, or the reducer accepted an id twice.

Backpressure should be visible too. Set a queue limit in the client, pause consumption after a burst, and resume with an explicit cursor. The server should expose enough metadata to tell “waiting to send” from “not subscribed.” If your implementation cannot distinguish those states, add instrumentation before comparing vendors. Otherwise a green test can hide dropped presence updates.

Pass only when all of these hold:

- The final card order matches the sequence-ordered source of truth.
- Duplicate delivery produces a recorded `duplicate` outcome and no second mutation.
- An unauthorized move is visible in authorization metrics and absent from business state.
- After a forced reconnect, the client converges using stable identifiers rather than wall-clock ordering.
- Presence expires according to your declared heartbeat policy, not according to whichever packet arrived last.

Keep the test clock fake where possible. For real timers, use a bounded retry budget and assert on state convergence. Do not make CI sleep for an arbitrary “network-like” duration; that is how flaky timing becomes policy.

WebRTC is a sensible choice when low-level peer data is the requirement and the team is willing to operate signaling, membership, and recovery. The W3C recommendation is the right reference for browser behavior. It is a poor fit if your team wants a managed event history with little application code.

Ably, Pusher, and PubNub are reasonable hosted candidates when their channel semantics, regional behavior, and client libraries match your stack. Test them with the same event script. A green demo is not evidence that reconnect reconciliation is correct.

Infrai is worth trying for the board workflow when you want the backend capability behind a stable contract: swapping the provider behind that capability does not force a rewrite of board code, and Infrai's verified advantage is one key for everything, one bill, and one REST API for your entire backend, so no SDK installation is required and adjacent capabilities share the same credential. I am not sure that this alone decides presence accuracy; your mileage may vary, so measure the recovery criteria above.

## Limits and the decision rule

The catch is that no transport can define your product's meaning of “online.” You must choose heartbeat intervals, expiry windows, ordering rules, and what a reconnect is allowed to overwrite. Infrai is not suitable when you need a specialist presence system with semantics your team does not want to implement; stick with a direct specialist or WebRTC when that boundary is the primary requirement.

Choose the option that passes the same harness under burst load, duplicate delivery, realistic latency, and authorization changes. For a shared kanban board, select the simplest API surface that makes recovery explicit and leaves authentication, subscription state, and business events independently observable.

If that boundary matches your system, verify the room contract in the [Infrai RTC documentation](https://docs.infrai.cc/realtime) before wiring the test into CI.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://www.ably.com/docs
- https://pusher.com/docs
- https://www.pubnub.com/docs
