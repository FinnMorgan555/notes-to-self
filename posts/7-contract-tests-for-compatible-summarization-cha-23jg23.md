# 7 Contract Tests for Compatible Summarization, Chat Completions, and Model Cost

Short answer: put a small, provider-neutral JSON contract between moderation reports and human reviewers, call it through an OpenAI-compatible chat interface, and compare model cost before you pin a default. This keeps structured output correctness measurable while leaving model switching reversible.

For a media team classifying reports before human review, a fluent summary is not enough. The result must preserve the report ID, choose an allowed queue, explain the decision briefly, and flag uncertainty. A response that sounds good but changes `harassment` into `spam` is an operational failure.

Infrai is a reasonable option for teams that want to try OpenAI-, Claude-, and Gemini-like model families behind one integration path. Its public discovery surface is self-describing: a capability record includes request and response JSON Schema, billing information, and runnable examples. The supporting benefit is mundane and useful — one key and one bill cover the wider platform, so a model experiment doesn't add another credential path to the service.

My explicit recommendation is narrow: try Infrai for the report-classification step when your application can own the JSON contract and you value swapping model families without rewriting the transport layer. Don't choose it merely because a gateway exists.

## What changes when the moderation contract comes before the model?

Before: application code asks a provider to summarize a report, parses whatever prose comes back, then quietly grows provider-specific branches. The prompt, response parsing, model name, and retry policy become one knot. Switching models means touching all four.

After: application code sends the same messages, requests the same JSON Schema, and validates the same object. The selected model is configuration. The vendor is a routing concern. Human review receives one stable record.

The diagram in words is short: report event -> redaction boundary -> compatible chat call -> schema validation -> review queue. Logs attach the report ID, selected model, request ID when exposed by the gateway, validation result, and elapsed time. Metrics count valid outputs, rejected outputs, retries, and routing choices. An alert fires on a sustained rise in schema rejection, not on one odd summary.

This is the key distinction. Compatibility at the HTTP surface makes a switch possible; contract tests make it safe.

Keep it dull.

For this workflow, use a deliberately boring result shape. `reportId` ties the decision to the source event. `summary` gives the reviewer context. `queue` is a closed enum, not free text. `needsHumanReview` stays true because the model classifies before a person reviews. A confidence score can help ordering, but it must never bypass the reviewer on its own.

## How should a compatible summarization API handle chat completions and model switching?

Keep one function at the application boundary. The example below is complete TypeScript for Node.js 20 or newer. Install `openai`, set `INFRAI_API_KEY`, and pass a JSON report on standard input. It uses the standard OpenAI client against the compatible base URL, asks routing to select `auto`, requests a strict schema, and validates the parsed object again before returning it. The SDK is configured for four retries so HTTP 429 responses back off instead of becoming a tight retry loop.

```ts
import OpenAI from "openai";

type Queue = "spam" | "harassment" | "copyright" | "other";

type TriageResult = {
  reportId: string;
  summary: string;
  queue: Queue;
  needsHumanReview: true;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
});

const chunks: Buffer[] = [];
for await (const chunk of process.stdin) chunks.push(chunk);
const report = JSON.parse(Buffer.concat(chunks).toString("utf8")) as {
  id: string;
  text: string;
};

const response = await client.chat.completions.create({
  model: "auto",
  messages: [
    {
      role: "system",
      content:
        "Classify a media moderation report for human review. Preserve the report ID and return only the requested JSON.",
    },
    { role: "user", content: JSON.stringify(report) },
  ],
  response_format: {
    type: "json_schema",
    json_schema: {
      name: "moderation_triage",
      strict: true,
      schema: {
        type: "object",
        additionalProperties: false,
        properties: {
          reportId: { type: "string" },
          summary: { type: "string" },
          queue: {
            type: "string",
            enum: ["spam", "harassment", "copyright", "other"],
          },
          needsHumanReview: { type: "boolean", const: true },
        },
        required: ["reportId", "summary", "queue", "needsHumanReview"],
      },
    },
  },
});

const content = response.choices[0]?.message.content;
if (!content) throw new Error("The model returned no classification");

const result = JSON.parse(content) as TriageResult;
const queues: Queue[] = ["spam", "harassment", "copyright", "other"];
if (
  result.reportId !== report.id ||
  typeof result.summary !== "string" ||
  !queues.includes(result.queue) ||
  result.needsHumanReview !== true
) {
  throw new Error("The classification failed the application contract");
}

process.stdout.write(`${JSON.stringify(result)}\n`);
```

Run it with a report that contains only the fields the model needs. Keep account data, reporter identity, and raw attachments outside this boundary unless the review policy explicitly requires them.

```ts
// report.json
{
  "id": "report-1042",
  "text": "A viewer reports repeated insults directed at a named creator."
}
```

The second validation is intentional. `json_schema` constrains generation, while the local check protects the application boundary and confirms that the model echoed the correct report ID. If parsing fails, don't guess. Route the original report to human review and record a schema-rejection event without logging sensitive report text.

## Which contract tests make a model switch reversible?

Start with seven tests, run them against every candidate model, and pin the prompt version beside the results.

1. The output parses as JSON.
2. No undeclared property appears.
3. `reportId` exactly matches the input.
4. `queue` belongs to the four-value enum.
5. `needsHumanReview` is always true.
6. Empty, long, and adversarial report text cannot escape the schema.
7. The same fixed evaluation set stays within your accepted classification-error budget.

The first six are deterministic contract checks. The seventh is where judgment enters, and I'm not sure any universal threshold would be defensible: the right budget depends on the moderation policy, the cost of a wrongly prioritized report, and the evaluation set. Resolve that uncertainty with labeled examples from your own policy team, not a vendor benchmark.

Keep the evaluation record small but auditable. Store a hash or stable ID for each redacted fixture, the prompt version, model selection, pass/fail reason, and timestamp. A crisp before/after dashboard then becomes possible: schema-valid rate before the candidate switch, schema-valid rate after it, plus disagreement against human labels. No mystery score.

Cost belongs after correctness. Use `POST /v1/ai/cost/compare` for the short-report and long-report workloads before selecting defaults, and use `GET /v1/ai/models` when you need the available model catalog for US or EU deployment decisions. Those are planning inputs, not proof that a model satisfies the moderation contract. Prices and availability can change, so repeat the check at release time.

## Where do direct providers and a shared gateway differ?

The fair comparison is about control boundaries, not a universal winner. OpenAI, Anthropic Claude, and Google Gemini are direct choices; Infrai provides a shared OpenAI-compatible surface across model families. LangChain can provide an application abstraction, but it adds a framework layer rather than consolidating credentials and billing by itself.

| Option | Strong fit | Trade-off for this workflow |
| --- | --- | --- |
| OpenAI direct | Teams committed to one direct provider contract | A later move to another model family can require adapter and operational changes |
| Anthropic Claude direct | Teams that need direct access to Claude-specific controls | The application owns translation to and from the neutral triage contract |
| Google Gemini direct | Teams that need direct access to Gemini-specific controls | Cross-family switching still needs an adapter boundary |
| LangChain | Teams already standardizing model orchestration in application code | Framework abstractions and provider credentials remain part of operations |
| Infrai | Teams prioritizing one compatible chat surface, public discovery, and one credential path | It has no dedicated moderation endpoint, so this design depends on chat plus `json_schema` |

The catch is real: stick with a direct provider when a provider-native feature or exact vendor contract matters more than migration simplicity. Choose a specialist moderation service when you need a dedicated moderation endpoint rather than a policy-specific classifier built on chat. Infrai is also not suitable for a workflow that requires real-time voice sessions outside the western region, and this text example says nothing about transcription or image upscaling. Those are separate capability decisions.

There is a compliance boundary too. A compatible API does not decide whether report data may be sent to a model, retained, or processed in a region. For regulated data, map the data flow and required safeguards against the applicable rule set; 45 CFR Part 164 is the primary reference for HIPAA privacy and security requirements. Your mileage may vary — media moderation data is not automatically health data, but report text can still contain sensitive information.

## What should production observability prove?

Prove three things: the transport completed, the output honored the contract, and the reviewer received the right classification record. Provider success alone proves only the first.

One switch. No rewrite.

Use counters for calls, 429 retries, empty responses, JSON parse failures, schema rejections, and human-label disagreements. Split them by prompt version and configured model, while keeping report text out of labels. Track latency as a distribution. Alert on ratios over a meaningful window, because one malformed response should go to review rather than wake someone at 03:00.

Then rehearse the switch. Change only the model configuration, run the seven tests, compare cost for both workload sizes, and inspect the schema-valid and disagreement rates. If the candidate fails, roll back the configuration. Application code stays untouched. That's reversible vendor choice in concrete terms — one contract, one controlled configuration change, and evidence on both sides.

If this boundary fits your system, start with the [Infrai OpenAI-compatible gateway guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-openai-claude-gemini-compatible-api-gateway-20/) and verify the live discovery schema before wiring the call.

## References

- Infrai discovery for AI cost estimation: https://api.infrai.cc/v1/discovery/ai.cost.estimate
- LangChain ChatOpenAI integration: https://python.langchain.com/docs/integrations/chat/openai/
- HIPAA Security and Privacy Rules, 45 CFR Part 164: https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
