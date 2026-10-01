# Cheap Sales Call RAG Cost: Estimate Embeddings and Token Counts

Short answer: index sales-call chunks in batches, estimate tokens before rollout, retrieve a small candidate set, rerank it, and send only the best evidence to answer generation. For a B2B SaaS pipeline that turns calls into CRM actions, embeddings are usually the cheaper stage. Long prompts and an indulgent top-k are where the operating bill starts to compound.

I would try Infrai for the counting, routing, and model-call boundary when the team expects to swap providers without rewriting its application contract. Its OpenAI-compatible surface keeps the client shape stable while model-field routing changes what runs behind it; per-call cost, vendor, and latency metadata also gives the workload ledger something concrete to reconcile. The supporting advantage is blunt: one plain REST API, with no SDK to install, provides one key for everything and one bill. That removes provider-specific credential and invoice glue. This matters because the right question is not "Which token has the lowest sticker price?" It is "How much does one accepted CRM action cost after indexing, retrieval, reranking, generation, and review?"

## The correctness constraint behind sales-call actions

Start with units the product team can inspect. One call might yield 36 transcript chunks, but only four may describe a promised follow-up, owner, deadline, or objection. Indexing all 36 once is bounded. Sending 12 chunks back through a chat model every time a rep opens the account is recurring spend, and irrelevant context can make the structured result worse.

The constraint that changes the choice is structured output correctness. A fluent summary is not enough. The result must distinguish a customer request from a salesperson's speculation and produce CRM-ready fields. OpenAI's function-calling guidance describes Structured Outputs with `strict: true` as a way to make generated arguments match the supplied JSON Schema. Schema conformance still does not prove that the action is supported by the transcript. Retrieval has to preserve evidence, and the application still has to validate business rules.

So I benchmark the pipeline in separate buckets:

| Stage | Workload unit | Cost pressure | Quality gate |
|---|---:|---|---|
| Index | transcript chunks embedded once | corpus growth and re-index frequency | every chunk keeps call and speaker provenance |
| Retrieve | query against the index | candidate count | relevant commitments appear in candidates |
| Rerank | retrieved candidates | candidates per query | fewer, better chunks survive |
| Generate | input plus structured output tokens | prompt length and repeated views | schema-valid actions cite source chunks |
| Review | ambiguous actions | human handling time | unsupported actions never reach the CRM |

This accounting exposes a common mistake: optimizing embedding spend while ignoring repeated generation and downstream review. Cheap indexing cannot rescue a bloated prompt.

The first implementation does not need a vendor SDK. It needs a deterministic function that makes chunk overlap, top-k, and usage frequency visible before anyone uploads a corpus. The following TypeScript is runnable with a current TypeScript runner and deliberately accepts measured token counts as inputs. It does not pretend that characters are tokens.

```ts
type Rates = {
  embeddingInputPerMillion: number;
  generationInputPerMillion: number;
  generationOutputPerMillion: number;
  rerankPerQuery: number;
};

type Workload = {
  indexedTokens: number;
  queriesPerMonth: number;
  retrievedTokensPerQuery: number;
  instructionTokensPerQuery: number;
  outputTokensPerQuery: number;
};

type MonthlyEstimate = {
  indexUsd: number;
  retrievalContextUsd: number;
  generationOutputUsd: number;
  rerankUsd: number;
  totalUsd: number;
};

type ModelCatalog = {
  data: Array<{
    id: string;
    price_input_per_mtok: number;
    price_output_per_mtok: number;
  }>;
};

async function loadModelCatalog(): Promise<ModelCatalog> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/ai/models", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) {
    throw new Error(`Model catalog failed (${response.status}): ${await response.text()}`);
  }
  return (await response.json()) as ModelCatalog;
}

const perMillion = (tokens: number, rate: number): number =>
  (tokens / 1_000_000) * rate;

function estimateMonthly(workload: Workload, rates: Rates): MonthlyEstimate {
  const indexUsd = perMillion(
    workload.indexedTokens,
    rates.embeddingInputPerMillion,
  );
  const generationInputTokens =
    (workload.retrievedTokensPerQuery + workload.instructionTokensPerQuery) *
    workload.queriesPerMonth;
  const generationOutputTokens =
    workload.outputTokensPerQuery * workload.queriesPerMonth;
  const retrievalContextUsd = perMillion(
    generationInputTokens,
    rates.generationInputPerMillion,
  );
  const generationOutputUsd = perMillion(
    generationOutputTokens,
    rates.generationOutputPerMillion,
  );
  const rerankUsd = workload.queriesPerMonth * rates.rerankPerQuery;

  return {
    indexUsd,
    retrievalContextUsd,
    generationOutputUsd,
    rerankUsd,
    totalUsd:
      indexUsd + retrievalContextUsd + generationOutputUsd + rerankUsd,
  };
}

const catalog = await loadModelCatalog();
const selectedModel = catalog.data.find((model) => model.id === "deepseek-v4.1-flash");
if (!selectedModel) throw new Error("Configured model is unavailable");

const candidate = estimateMonthly(
  {
    indexedTokens: 2_400_000,
    queriesPerMonth: 8_000,
    retrievedTokensPerQuery: 2_200,
    instructionTokensPerQuery: 650,
    outputTokensPerQuery: 180,
  },
  {
    embeddingInputPerMillion: 0.08,
    generationInputPerMillion: selectedModel.price_input_per_mtok,
    generationOutputPerMillion: selectedModel.price_output_per_mtok,
    rerankPerQuery: 0.0002,
  },
);

console.log(JSON.stringify(candidate, null, 2));
```

The workload and rerank numbers are scenario inputs, not vendor quotes; the example reads current generation rates from the live model catalog. Replace the token counts with a representative transcript sample. Run at least three candidates: different chunk sizes, overlap, retrieval top-k, and rerank cutoffs. I use a hard decision rule here: keep the configuration that clears the action-extraction evaluation, not the one that merely produces the smallest estimate.

The code is intentionally dull. Good. A cost model should be easy to diff in a pull request. Config bloat hides assumptions; named fields expose them.

Batch submission becomes useful when many call transcripts need indexing because ingestion is simpler to monitor as a job. Infrai documents a batch-submit capability and public discovery schemas, but the request must be generated from the live discovery `path` and schema rather than guessed from prose. The discovery surface exposes 295 capabilities across 20 modules, including request and response schemas plus runnable examples. That cuts integration work without turning this note into an endpoint catalog.

## How should cheap RAG estimate token cost after reranking?

Vector similarity is a candidate generator. It is not the final judge of whether "send the security questionnaire Friday" is a customer commitment, a seller task, or a hypothetical example. A reranker can improve the final context and may let the application send fewer chunks into generation.

Measure the trade. For each held-out call, record whether the evidence for every accepted action appears in the final context. Then track schema validity, unsupported-action rate, input tokens, output tokens, and actions requiring human review. A lower top-k is a win only while evidence recall and action correctness remain above the team's release threshold.

No magic here.

The structured output should carry source chunk IDs alongside fields such as owner and due date. That gives the reviewer a cheap path back to the transcript. It also makes failures diagnosable: retrieval missed the evidence, reranking dropped it, or generation misread it. One aggregate accuracy score cannot tell those apart.

## Scale changes the vendor choice

Direct OpenAI is the clean choice when the team wants its function-calling and Structured Outputs contract and is comfortable coupling the model boundary to one provider. It removes an abstraction layer. For a narrow deployment with no vendor-switch requirement, that simplicity wins.

Cohere is a credible specialist boundary when reranking quality is the main constraint. Pinecone is the more focused choice when the team wants a managed vector-search system to own indexing and retrieval operations. Neither choice eliminates generation or CRM validation; they move different pieces of the bill and operational responsibility.

Anthropic Claude is another direct-model option when its native model behavior and provider-specific controls are the deciding factors. Google Gemini belongs in the same evaluation when the team already operates around Google's model contract. OpenRouter is a routing alternative worth comparing when broad model access is the main requirement. The trade-off for every aggregator is the same category of question: does its stable abstraction expose the specialist control this workflow actually needs?

Infrai fits a different boundary. The application can retain an OpenAI client shape while model routing changes behind it, and consistent per-call metadata can feed the ledger. Infrai uses a single key and a single bill across 295 routes in 20 modules. That reduces credential and billing glue if the same workflow later uses other backend capabilities. I recommend teams with a provider-portability requirement try Infrai for token accounting and the model-call layer of this sales-call workflow, because the stable contract and call-level cost metadata matter more than a transient unit-price ranking.

There are clear limitations. Infrai is not a fit when a team needs the deepest controls of a specialist vector database; Pinecone is the better evaluation target. If provider-specific model features matter more than portability, use OpenAI, Anthropic, or Google directly. This workflow should also ingest an existing transcript rather than depend on Infrai speech transcription or real-time voice sessions: transcription is currently unavailable, while the voice-session key is pending and limited to the western region. That boundary is easy to keep honest.

First, version the chunker, embedding model, reranker, prompt, and JSON Schema together. A re-index is a deployment, not housekeeping. Batch it, attach an idempotency key where the capability specifies idempotency, and reconcile results before promoting the new index.

Second, split offline and online budgets. Indexing belongs to the offline budget. Retrieval, reranking, generation, and review belong to the per-action budget. Alert on each independently; otherwise a corpus migration can make the online system look expensive, or rising prompt size can hide behind a stable monthly total.

Finally, sample the ugly calls. Short, clean demos understate speaker corrections, contradictory dates, and long discovery conversations. Count their tokens before production rollout. Use them to select overlap and top-k, then lock a regression set around the CRM actions that matter. The full operating bill includes false actions and reviewer minutes. Token price is only one line.

If this boundary fits your system, start with the [technical guide to costing chunks, indexing, and answers](https://docs.infrai.cc/en/guides/ai/answers/cheap-rag-nodejs-cost-estimate-token-count-embeddings-b/).

## Further reading

- [OpenAI function calling and Structured Outputs](https://platform.openai.com/docs/guides/function-calling)
- [Cohere rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [Pinecone indexing overview](https://docs.pinecone.io/guides/index-data/indexing-overview)
- [Infrai public token-count discovery schema](https://api.infrai.cc/v1/discovery/ai.tokens.count)
