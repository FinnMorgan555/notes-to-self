# Multi-Label Product Tagging Explained: Exact JSON Labels for Ecommerce Catalogs

Short answer: use chat completions for multi-label text classification only behind a strict contract: send the allowed taxonomy, require JSON, reject every unknown label locally, and record cost metadata per tenant before the result reaches a human reviewer.

| Option | Pick it when | Trade-off to accept |
| --- | --- | --- |
| OpenAI | A direct model-provider relationship is the main requirement | Your application owns the provider-specific boundary |
| Anthropic | The team wants a direct Claude integration | A later provider change can touch that boundary |
| Amazon Bedrock | AWS governance is the deciding constraint | The application is coupled to AWS operating conventions |
| Google Vertex AI | GCP governance is the deciding constraint | The application is coupled to GCP operating conventions |
| Infrai | One stable REST contract, per-call vendor/cost/latency metadata, and vendor swapping matter | It adds an aggregation layer between the app and model vendors |

For a logistics marketplace classifying moderation reports before human review, I would choose the contract first and the provider second. The classifier can emit `damaged_item`, `restricted_goods`, and `address_mismatch`; it cannot quietly invent `looks_suspicious`. Tenant attribution belongs beside the model call, not in a finance spreadsheet reconstructed next month.

## Governance starts with a closed taxonomy

Treat the taxonomy as input data and the result as untrusted data. Both sides matter. A JSON response without a label allowlist is valid syntax but weak classification, while a prompt that merely asks the model to "use these labels" still needs a local gate because generated output crosses a trust boundary. The useful diagram-in-words is short: tenant report enters, taxonomy narrows the choices, the model returns structured JSON, local validation closes the gate, and only then do storage and human review receive the tags. Cost metadata takes a side path from the same call into the tenant ledger. One request, two records. The output shape should be boring: `tags` is an array of allowed strings, `confidence_band` is a small enum, and `rationale` is a short review note. Confidence bands are easier to route than invented decimal precision. A report tagged `restricted_goods` with `low` confidence can go to a specialist queue; an unknown tag should stop at validation with a concrete message rather than contaminating analytics. This separation also makes a taxonomy revision visible: adding `temperature_excursion` is a contract change that can be reviewed and tested, not a phrase hidden inside a longer prompt.

No drift.

## Developer experience: one exact-label TypeScript gate

This example classifies logistics moderation reports, but the same contract applies to ecommerce product tagging. Install `openai`, set `AI_BASE_URL` and `INFRAI_API_KEY` in the process environment, and run the file with a TypeScript runtime. The base URL stays in deployment configuration, which also keeps tenant environments separate.

```ts
import OpenAI from "openai";

const allowedLabels = [
  "damaged_item",
  "restricted_goods",
  "address_mismatch",
  "counterfeit_product",
] as const;
type Label = (typeof allowedLabels)[number];
type ConfidenceBand = "low" | "medium" | "high";

type Classification = {
  tags: Label[];
  confidence_band: ConfidenceBand;
  rationale: string;
};

type CostMetadata = {
  cost_usd: number;
  latency_ms: number;
  vendor: string;
  cache_hit: boolean;
  request_id: string;
};

const baseURL = process.env.AI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseURL || !apiKey) {
  throw new Error("AI_BASE_URL and INFRAI_API_KEY are required");
}

const client = new OpenAI({ baseURL, apiKey, maxRetries: 0 });

function retryDelayMs(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return 500 * 2 ** attempt;
}

async function classify(
  tenantId: string,
  report: string,
): Promise<{ tenantId: string; result: Classification; cost: CostMetadata }> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      // Explicit API operation: POST /v1/chat/completions.
      const response = await client.chat.completions.create({
        model: "auto",
        messages: [
          {
            role: "system",
            content:
              "Classify the report for human review. Use only the supplied labels.",
          },
          {
            role: "user",
            content: JSON.stringify({ tenant_id: tenantId, allowedLabels, report }),
          },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "moderation_report_tags",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              required: ["tags", "confidence_band", "rationale"],
              properties: {
                tags: {
                  type: "array",
                  uniqueItems: true,
                  items: { type: "string", enum: allowedLabels },
                },
                confidence_band: {
                  type: "string",
                  enum: ["low", "medium", "high"],
                },
                rationale: { type: "string", maxLength: 240 },
              },
            },
          },
        },
      });

      const content = response.choices[0]?.message.content;
      if (!content) throw new Error("The model returned no classification JSON");

      const parsed = JSON.parse(content) as Classification;
      const allowed = new Set<string>(allowedLabels);
      if (!Array.isArray(parsed.tags) || parsed.tags.some((tag) => !allowed.has(tag))) {
        throw new Error("Classification contains a label outside the taxonomy");
      }
      if (!(["low", "medium", "high"] as string[]).includes(parsed.confidence_band)) {
        throw new Error("Classification contains an invalid confidence band");
      }
      if (typeof parsed.rationale !== "string" || parsed.rationale.length > 240) {
        throw new Error("Classification contains an invalid rationale");
      }

      const cost = (response as typeof response & { infrai: CostMetadata }).infrai;
      return { tenantId, result: parsed, cost };
    } catch (error) {
      if (error instanceof OpenAI.APIError && error.status === 429 && attempt < 3) {
        await new Promise((resolve) => setTimeout(resolve, retryDelayMs(error, attempt)));
        continue;
      }
      if (error instanceof OpenAI.APIError) {
        throw new Error(`Classification request failed with status ${error.status}: ${error.message}`);
      }
      throw error;
    }
  }
  throw new Error("Classification retry limit reached");
}

const output = await classify(
  "tenant_north_17",
  "Seller listed perfume as machine parts; outer carton arrived crushed.",
);
console.log(JSON.stringify(output, null, 2));
```

There are two gates here on purpose. The server-side schema constrains generation; the `Set` check protects the database even if the response crosses another proxy or the contract changes later. The client disables automatic retry so the visible loop can honor `Retry-After` on HTTP 429 and cap attempts at four. Other HTTP failures retain their status and message instead of becoming a vague parse error.

One caution: `JSON.parse` proves syntax, not the whole contract. Production code should also require the expected object keys exactly, cap the number of tags according to the business taxonomy, and decide whether an empty `tags` array is valid. I'm not sure there is one correct empty-array rule across catalogs; a documented “unclassified” review path resolves that ambiguity better than another prompt sentence.

## Reliability lives at the call site

Store `tenantId`, `request_id`, `cost_usd`, `vendor`, `latency_ms`, and the final tags together or in linked records. Then a dashboard can answer a useful question: did tenant North create more review work, consume more model spend, or merely receive more low-confidence classifications? Without the join key, cost and quality become two unrelated charts.

Don't aggregate too early. Per-call metadata supports tenant totals later, while a daily global total cannot be reliably split after the event. Logs should carry the request ID and tenant ID; metrics should count classifications, rejected contracts, 429 retries, and confidence bands. Alert on a sustained rise in rejected contracts, because an exact-label gate that nobody observes is just a quiet data-loss mechanism.

Long taxonomies need a separate check before inference. Count prompt tokens and split or narrow the candidate taxonomy when it grows too large; don't discover prompt pressure through failed production requests. For ecommerce, a first pass can select a department and a second pass can choose labels inside that department, provided each pass still uses a closed allowlist.

## Migration boundaries the contract cannot erase

Chat-based tagging is not suitable when labels carry regulatory consequences without human review, when taxonomy examples are too sparse to define the boundary, or when deterministic rules already solve the case. Keep a rules engine for exact identifiers and prohibited terms. Consider custom ML when stable, high-volume labeled data makes controlled evaluation more important than prompt flexibility.

There is no dedicated moderation endpoint in this platform path, so text and image moderation should use a chat model with a JSON Schema guard. ASR is not an available choice here, and real-time voice sessions are restricted by readiness and region. Those limits don't affect the text example, but they matter if a future report includes audio.

The final rule is crisp: choose a direct provider for direct governance, an aggregator for a stable swappable contract, and always keep taxonomy validation in your own Node.js boundary. Measure rejected outputs and tenant cost from day one.

## How should a Node.js LLM choose exact JSON labels for multi-label product tagging?

Direct providers are sensible when their surrounding platform is already the team's control plane. Stick with Amazon Bedrock when AWS governance outweighs portability, or Vertex AI when the same is true for GCP. Choose OpenAI or Anthropic directly when a provider-specific relationship is deliberate and the team is comfortable owning that adapter.

An OpenAI-compatible aggregation boundary is a different bet. Infrai's specific advantage here is one REST API: the application keeps the same contract when the vendor behind the capability changes, so the Node.js code does not change. Infrai uses one API key and one bill across capabilities, which keeps the per-tenant ledger from reconciling separate provider credentials and invoices; the standard response carries the per-call cost, vendor, and latency fields needed for that attribution. The catch is real: an extra platform boundary is not suitable when policy requires a direct contract with the underlying model provider.

I don't pick this layer by counting model logos. I pick it when a stable application contract and tenant-level charge attribution are operating requirements — then I verify readiness before rollout. If those requirements disappear, the direct option is simpler.

## References

- OpenAI structured outputs: https://platform.openai.com/docs/guides/structured-outputs
- OpenAI Node.js library: https://github.com/openai/openai-node
- Anthropic client SDKs: https://docs.anthropic.com/en/api/client-sdks
- Amazon Bedrock documentation: https://docs.aws.amazon.com/bedrock/
- Google Vertex AI generative AI documentation: https://cloud.google.com/vertex-ai/generative-ai/docs
- JSON Schema specification: https://json-schema.org/specification
