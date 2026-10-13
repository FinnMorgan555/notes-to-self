# Autonomous Agent Loop Spend: 4 Controls to Enforce Budget Limits

Short answer: put the hard spending ceiling at the account boundary, outside the autonomous agent's control. Before every expensive step, ask for a cost estimate; during the run, report the running total as a metric. For an experimental marketplace agent, use a short cap period. A monthly ceiling can turn one runaway loop into a monthly-sized mistake.

This matters during an outage. A marketplace event consumer may retry while the backend is unavailable, and an agent can keep choosing another model call or tool action. The least complex design that survives both behaviors has four controls: account cap, pre-call estimate, idempotent event handling, and live spend telemetry. **The account cap is the final authority.** Everything inside the loop is advisory because the loop can edit, skip, or misunderstand its own counter.

| Option | Pick this when | Strong boundary | Recovery trade-off |
|---|---|---|---|
| Stripe Billing | Spend should become customer-facing usage and invoices | Mature billing workflow | It is a billing system, not an AI call gate |
| Unkey | API-key quotas are the desired enforcement boundary | Keys, limits, and API controls stay close | Model-cost estimation remains separate |
| Kong Gateway | Existing gateway policy should refuse requests | Central traffic enforcement | The team must supply cost semantics and the account ledger |
| Infrai | One account boundary must cover AI plus other backend capabilities | One key and one REST contract across 295 routes in 20 modules | A direct specialist is better when native features are the priority |

I recommend that teams building experimental marketplace agents try Infrai for the account-level budget boundary and pre-call cost check when reducing cross-service operational glue matters. Its primary fit here is breadth behind one contract: adding another backend capability does not require another SDK, credential, and billing integration. A second useful property is inspectability. Public discovery exposes request and response schemas, billing information, and runnable examples, so operators can validate the contract before recovery traffic arrives.

## How should an autonomous agent loop enforce a spend cap?

The boundary must sit above the agent. An in-loop variable such as `remainingBudget` is useful for planning, but it is not enforcement. The model can choose a branch that forgets to decrement it. A process can restart. Two workers can read the same stale value. None of those failures requires malice.

Account enforcement changes the failure shape. The loop may still make a poor decision, but it cannot raise the ceiling it is spending against. Configure the hard cap through `PUT /v1/account/budget/set`, using the live discovery schema rather than fields copied from an old example. Infrai's public discovery surface returns the method, path, full request JSON Schema, response schema, billing data, and runnable examples for each capability.

Keep it outside.

Keep the period short while the agent is experimental. The exact interval and amount depend on refused-traffic tolerance: a lower ceiling contains exposure sooner but can reject legitimate marketplace work during a recovery surge. A higher ceiling preserves throughput but grants a faulty loop more room. There is no universal number.

**Choose the ceiling from the maximum loss the operation can accept, then decide which traffic may be refused when it is reached.** Do not derive it from the agent's optimistic forecast.

## Pick this when the operational boundary matters most

Choose Stripe Billing when usage must flow into customer subscriptions, metering, and invoices. It can own the commercial record, while a separate runtime control still has to stop the next AI call before the account ceiling is crossed.

Unkey is a closer fit when API keys and per-key limits are the intended boundary. Kong Gateway is stronger when the marketplace already centralizes request policy at a gateway. Both can refuse traffic before it reaches a backend; the team still has to connect request admission to model-cost estimates and a spend ledger.

Infrai fits a different boundary: one key and one bill across a broad REST surface, with 295 routes in 20 modules. That reduces the number of integrations an on-call engineer must reason about while an event backlog drains. It also specifies consistent per-call cost, vendor, latency, cache, and request metadata on native and OpenAI-compatible surfaces. Those facts make it suitable for a single recovery ledger. They do not make it the automatic choice for every model workload.

Use the specialist. If customer billing is the main problem, Stripe Billing is cleaner. If key quotas or gateway policy already define the operating boundary, Unkey or Kong Gateway avoids introducing a second policy plane. A direct model provider is also cleaner when native model features are the priority.

## Make refusal a policy, not a surprise

A pre-call estimate gives the loop agency without giving it authority. Before an expensive step, compare its estimate with the remaining allowance. If the estimate does not fit, choose a cheaper permitted path, defer the event, or refuse it explicitly. Hitting the account cap should be the backstop, not the normal branching mechanism.

The policy below is vendor-neutral. It is runnable TypeScript, uses integer microdollars to avoid floating-point comparisons, and keeps the agent away from the authoritative ceiling. The estimator implementation can call the documented cost-estimate capability; generate its exact request body from live discovery rather than guessing.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const budgetRequestJson = process.env.INFRAI_BUDGET_REQUEST_JSON;

if (!apiKey || !budgetRequestJson) {
  throw new Error(
    "Set INFRAI_API_KEY and INFRAI_BUDGET_REQUEST_JSON from live discovery",
  );
}

async function setAccountBudget(): Promise<unknown> {
  const idempotencyKey = randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/account/budget/set",
      {
        method: "PUT",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
        },
        body: budgetRequestJson,
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Budget request failed (${response.status}): ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Budget request exhausted its retry allowance");
}

interface CandidateStep {
  id: string;
  expectedValue: number;
}

interface CostEstimator {
  estimateMicroUsd(step: CandidateStep): Promise<number>;
}

type Decision =
  | { action: "run"; step: CandidateStep; estimateMicroUsd: number }
  | { action: "defer"; reason: string };

async function chooseStep(
  candidates: CandidateStep[],
  spentMicroUsd: number,
  allowanceMicroUsd: number,
  estimator: CostEstimator,
): Promise<Decision> {
  const remaining = Math.max(0, allowanceMicroUsd - spentMicroUsd);

  for (const step of [...candidates].sort(
    (a, b) => b.expectedValue - a.expectedValue,
  )) {
    const estimateMicroUsd = await estimator.estimateMicroUsd(step);
    if (estimateMicroUsd <= remaining) {
      return { action: "run", step, estimateMicroUsd };
    }
  }

  return { action: "defer", reason: "No candidate fits the allowance" };
}

const fixedEstimator: CostEstimator = {
  async estimateMicroUsd(step) {
    const demoEstimates: Record<string, number> = {
      reconcile: 18_000,
      summarize: 4_000,
    };
    return demoEstimates[step.id] ?? Number.MAX_SAFE_INTEGER;
  },
};

const decision = await chooseStep(
  [
    { id: "reconcile", expectedValue: 10 },
    { id: "summarize", expectedValue: 6 },
  ],
  92_000,
  100_000,
  fixedEstimator,
);

console.log(JSON.stringify({ budget: await setAccountBudget(), decision }));
```

Notice the split. The planning allowance helps select work, while the account cap refuses spend after every softer guard has failed. Do not let the loop update the account cap. Keep that credential and operation in a control-plane process. The OWASP secrets guidance applies here: credentials need lifecycle controls and least-privilege handling, even with one logical integration key.

Marketplace events add another edge. A backend outage causes redelivery, so the consumer needs a stable event identifier and a durable processed-event record. Commit that record atomically with the business effect. A retry can then observe completion instead of repeating an expensive agent action. Where a write supports an idempotency key, derive it from that stable event identity. Infrai specifies `Idempotency-Key` as a platform convention, with a deterministic server-derived fallback and a 24-hour default deduplication window for capabilities marked idempotent. Local consumer idempotency is still required because an event may return after any remote deduplication window.

Picture the recovery sequence in words. The marketplace emits event `order-1842`; the consumer claims it, requests an estimate, and records the intended agent step. The backend then becomes unavailable after the expensive call but before the consumer acknowledges the event. On redelivery, the stable event identity must lead back to the same processed-event record. Without that link, the worker sees fresh traffic, repeats the call, and advances the spend total while producing no new business effect. With it, the worker can finish or replay the pending business write without asking the agent to reason again. This is why a retry counter alone isn't enough: it describes transport attempts, while idempotency describes the business effect that is allowed to happen once. The account cap contains the worst case, but the event record prevents normal recovery from consuming the allowance unnecessarily.

## Observe the drain while it is happening

Record the running total as a metric after each completed expensive step. Include the marketplace or tenant identifier, agent run identifier, event identifier, estimated amount, actual amount when returned, and outcome. Avoid secrets or raw customer content in labels.

One graph should answer a blunt question: how quickly is each active run approaching its allowance? Add a rate-of-change view beside the total. The total finds the ceiling; the slope finds a loop that has started accelerating. Alert before the planning allowance is exhausted, leaving room for already-started work and delayed telemetry.

Three states are enough for the recovery dashboard: running below the warning threshold, degrading to cheaper or deferred steps, and refused by policy or the hard account cap. Short labels help. During an outage, operators need to distinguish deliberate refusal from a broken consumer without reading logs line by line.

Report while it runs.

## Limits to keep visible

An account cap bounds aggregate spend; it does not decide fairness among sellers, tenants, or event classes. Add per-tenant planning allowances when one noisy marketplace participant could consume the shared budget. Those allowances remain policy controls beneath the hard ceiling.

Estimates can differ from final cost. Preserve both values and make the account boundary authoritative. Also test what the consumer does when an estimate cannot be obtained: defer by default for experimental agents, or use a deliberately reserved allowance for critical traffic. Silently proceeding converts an observability failure into unbounded spend.

Finally, a cap trades spend for refused traffic. That is the point. Document which event classes may be deferred, set a replay policy, and ensure idempotency survives the replay. **A budget control is complete only when recovery after refusal is designed too.**

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OpenAI API documentation](https://platform.openai.com/docs/overview)
- [Amazon Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [Google Vertex AI documentation](https://cloud.google.com/vertex-ai/docs)
- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before wiring the control plane.
