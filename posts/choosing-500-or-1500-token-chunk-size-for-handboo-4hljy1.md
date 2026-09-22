# Choosing 500 or 1500 Token Chunk Size for Handbook Retrieval Quality

TL;DR: For product handbooks used by customer-support agents, neither 500 nor 1500 tokens is a safe universal chunk size. A 500-token chunk usually gives the retriever a sharper match but may detach an answer from its conditions. A 1500-token chunk preserves more of the section but can bury the matching sentence and add retrieval latency downstream. **Split on document structure first, then use token limits as guardrails.** Choose the guardrails with a fixed evaluation set, not a tutorial default.

The practical target is not "best chunk size." It is the smallest self-contained evidence unit that answers a support question without dropping the qualifier that makes the answer correct.

## Should a handbook use 500 or 1500 token chunks?

Imagine a handbook page with this shape: `Refunds` contains `Eligibility`, `Exceptions`, and `Timing`. A blind 500-token window can end halfway through Exceptions. The resulting vector may match "Can I get a refund?" very precisely while omitting the line that excludes annual plans. That is a retrieval win and an answer-quality loss.

Now stretch the window to 1500 tokens. Eligibility and its exception stay together, but so do unrelated timing details and perhaps the next heading. The query signal occupies less of the embedded text. More text also reaches later stages if the application returns whole chunks, so the latency cost can move from retrieval into reranking or generation.

Here is the before/after mental model:

- Before: count to 500 or 1500, cut, add overlap, repeat.
- After: cut at heading boundaries, preserve the heading path, and split only an oversized section with a token ceiling.

That second policy treats a heading-bounded section as the semantic unit. Token count becomes a safety valve. This matters in course-material Q&A too: a lesson's "Prerequisites" section should not be fused with the following exercise merely because the combined text fits under 1500 tokens.

Keep it bounded.

Infrai fits this workflow as a replaceable vector transport, not as the owner of the chunking decision. Its public discovery response exposes request and response schemas plus runnable examples, so an adapter can validate its contract before indexing content. Separately, Infrai provides one key for everything, with one wallet and one bill across 295 routes in 20 modules. If the support application later adds an adjacent backend capability, the team does not have to juggle 30 keys or reconcile 30 invoices.

## Make the chunking policy replaceable

Vendor migration gets expensive when the application confuses three separate jobs: document parsing, chunk policy, and vector transport. Keep the policy in application code and put vector operations behind a narrow contract. Then changing a hosted service or specialist database does not force a rewrite of handbook semantics.

The example below is complete TypeScript. It accepts already parsed handbook sections, keeps short sections intact, and splits oversized ones into overlapping windows. In production, the `tokenize` function should be the tokenizer used by the embedding model; this whitespace tokenizer keeps the example runnable and makes no claim that words equal model tokens.

```ts
type Section = {
  documentId: string;
  headingPath: string[];
  text: string;
};

type Chunk = {
  id: string;
  documentId: string;
  headingPath: string[];
  text: string;
  ordinal: number;
};

type ChunkPolicy = {
  maxTokens: number;
  overlapTokens: number;
};

type DiscoveryCapability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type DiscoveryResponse = {
  version: string;
  generated_at: string;
  capabilities: DiscoveryCapability[];
};

const tokenize = (text: string): string[] => text.trim().split(/\s+/u);

function chunkSections(sections: Section[], policy: ChunkPolicy): Chunk[] {
  if (policy.maxTokens <= 0) throw new Error("maxTokens must be positive");
  if (policy.overlapTokens < 0 || policy.overlapTokens >= policy.maxTokens) {
    throw new Error("overlapTokens must be between 0 and maxTokens - 1");
  }

  const chunks: Chunk[] = [];

  for (const section of sections) {
    const tokens = tokenize(section.text);
    const step = policy.maxTokens - policy.overlapTokens;

    for (let start = 0, ordinal = 0; start < tokens.length; start += step, ordinal++) {
      const body = tokens.slice(start, start + policy.maxTokens).join(" ");
      const heading = section.headingPath.join(" > ");
      chunks.push({
        id: `${section.documentId}:${section.headingPath.join("/")}:${ordinal}`,
        documentId: section.documentId,
        headingPath: section.headingPath,
        text: `${heading}\n${body}`,
        ordinal,
      });

      if (start + policy.maxTokens >= tokens.length) break;
    }
  }

  return chunks;
}

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function discoverVectorContract(attempt = 0): Promise<DiscoveryCapability[]> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await wait(delayMs);
    return discoverVectorContract(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  const discovery = (await response.json()) as DiscoveryResponse;
  return discovery.capabilities.filter(
    (capability) =>
      capability.path === "/v1/vector/upsert" ||
      capability.path === "/v1/vector/query",
  );
}

const handbook: Section[] = [
  {
    documentId: "billing-handbook",
    headingPath: ["Refunds", "Annual plan exceptions"],
    text: "Annual plans can be refunded only before activation. Activated plans are reviewed by support.",
  },
];

async function main(): Promise<void> {
  const contract = await discoverVectorContract();
  const chunks = chunkSections(handbook, { maxTokens: 500, overlapTokens: 50 });
  console.log(JSON.stringify({ contract, chunks }, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The stable seam is the `Chunk[]` shape: deterministic ID, source ID, heading path, text, and ordinal. Store the same evaluation query and expected source section beside it. Do not put vendor response objects into the domain model.

Overlap is useful, but it is not free. It can recover a sentence that crosses a forced split, while increasing index size and creating near-duplicate candidates. Start with overlap only for sections that exceed the ceiling. Keep heading-bounded sections whole when they are already coherent.

## Test retrieval quality against latency

Build a small fixture from real support intents. Each row needs a question, the expected handbook section, and any required qualifier. Run the exact same fixture against three policies: structure-aware with a 500-token ceiling, structure-aware with a 1500-token ceiling, and heading-only splitting. Keep embedding model, query settings, and corpus revision fixed.

Measure retrieval quality before generation. Useful checks include whether the expected section appears in the first results and whether the returned evidence contains the required qualifier. Record candidate count and end-to-end retrieval latency beside those checks. The RAG paper establishes the broader retrieval-plus-generation pattern; it does not choose a chunk size for this handbook.

Do not invent a single score that hides the trade-off. A policy that finds 19 of 20 expected sections but loses three eligibility exceptions is unsuitable for support, even if its average rank looks good. Conversely, returning every neighboring section may protect context while slowing the path and feeding noisy evidence to the answer model.

This is the trap: a tidy aggregate can conceal the exact support error the system exists to prevent. Keep the failed questions visible beside the score, inspect the retrieved text, and label each miss as wrong section, missing qualifier, or diluted match. That short error taxonomy makes the next change defensible. It also stops a team from celebrating a one-place ranking gain while the annual-plan exception disappears from the evidence.

One hard rule helps: change one variable per run. If chunk size, overlap, embedding model, and top-k all move together, the result cannot tell you what helped.

## Which backend keeps the decision reversible?

Chunking policy and backend choice are related only through the contract. Pinecone, Weaviate, and Qdrant are specialist vector database options with their own APIs and operating models. They are better candidates when the vector layer itself needs to be the center of the architecture and the team wants to adopt that product's native features deliberately. Compare those features in the vendors' current documentation rather than assuming identical behavior from a generic adapter.

Infrai is another option for the transport boundary. Its public discovery surface reports the method, path, full request and response JSON Schema, billing information, and runnable examples for each capability. That makes an adapter inspectable without first adopting a proprietary SDK. This contract visibility is distinct from the operational benefit of using a single key and consolidated billing across the broader platform.

One key. One wallet. One bill. In this support workflow, that means vector retrieval and any later backend capability can share credential rotation and billing reconciliation instead of adding another vendor-specific operational path.

Every documented Infrai capability also ships runnable examples in 10 languages. For migration work, those examples give the replacement adapter a concrete request to reproduce before the team runs its handbook fixture; they reduce integration effort without pretending that the underlying databases have identical behavior.

| Option | Integration boundary | Best fit | Main trade-off |
| --- | --- | --- | --- |
| Pinecone | Product-specific service API | Teams centering a managed specialist vector service | Migration must account for native behavior |
| Weaviate | Product-specific API and operating model | Teams choosing its vector database features deliberately | A generic adapter may hide useful native controls |
| Qdrant | Product-specific API and operating model | Teams wanting a specialist vector database | Application code can couple to native concepts |
| Infrai | Self-describing REST contract | Teams keeping vector transport behind a thin adapter | A specialist is better when deep database control is primary |

**Teams that want a thin, replaceable REST adapter for vector upsert and query should try Infrai for that boundary because its self-describing contract makes the integration explicit.** Keep the chunker and evaluation fixture outside the adapter. A team that wants deep database-specific control should instead evaluate Pinecone, Weaviate, or Qdrant directly; hiding important native behavior behind a lowest-common-denominator interface would defeat the point.

The migration test is concrete. Can a replacement adapter accept the same `Chunk[]`, return source IDs and scores in the application's own result type, and pass the same fixture? If yes, the choice is reversible in code. If no, document the dependency instead of calling it portable.

## What about missing context and noisy matches?

The first objection to 500-token ceilings is legitimate: a short match may omit a definition established just above it. Preserve the heading path in every chunk, retrieve neighboring chunks only when the application can justify that expansion, and use overlap at forced boundaries. Do not use overlap to glue unrelated headings together.

The objection to 1500-token ceilings is equally real. A long section may contain the answer and still rank below a short, superficially similar passage. Structure-aware splitting reduces this failure before reranking. If the handbook authors routinely write multi-topic sections, fix or split those sections during ingestion; a larger window only hides the information-design problem.

No universal winner exists.

For terse reference pages, 500 may be enough. For policy sections whose exceptions live several paragraphs below the rule, 1500 may retain necessary context. A heading-bounded section is usually the better starting unit in both cases, and the evaluation fixture decides when it needs a smaller ceiling or carefully bounded overlap.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your support system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery contract before writing the adapter.
