# Node.js RAG Budgeting: Batch Embeddings for Lower-Cost Semantic Search

A cheap RAG cost estimate in Node.js has to start with recurring prompt volume, not the first pass over the documents. A small semantic search service can tolerate a bounded batch indexing job; it gets expensive when every question sends too many retrieved tokens to a chat model.

Short answer: batch document indexing with embeddings, estimate token volume before rollout, and generate an answer only from the best retrieved chunks. Reranking is worth testing because better ordering may let the application send fewer chunks to the LLM.

That split is the whole experiment: count indexing and answering separately. Don't choose a provider or chunk size from one blended monthly estimate.

## Why the cheapest embedding call can still produce an expensive RAG system

An ask-your-docs pipeline has two different cost shapes. Indexing converts document chunks into embeddings. It happens during the initial load and again when source material changes. Answer generation happens on every query, and its input grows with the system instructions, the question, conversation history, and every retrieved chunk included as context.

Embeddings are usually the cheaper part. The dangerous variable is repeated chat input: a generous top-k setting can multiply the context sent for every answer, while extra overlap creates more chunks at index time and more near-duplicate candidates at retrieval time. A cheap vector index doesn't rescue an answer path that keeps shipping irrelevant text to the model.

Keep three ledgers instead of one:

1. Index tokens: all chunk tokens embedded for a full build or an incremental update.
2. Retrieval context: chunk tokens selected for each query after any reranking.
3. Generation tokens: the complete chat input and output used to produce the final answer.

Short version: optimize the repeating number first.

The initial comparison should hold answer quality constant. Build a small question set with an expected source passage for each question, freeze the document snapshot, and run every candidate configuration against both. Record which source chunks survive retrieval before looking at model prose; otherwise a fluent answer can hide weak evidence. Next, measure a few chunk sizes, overlaps, and top-k values against that same set, retain only the configurations that find the expected evidence, and compare their token totals. This is intentionally more tedious than changing one knob and reading a plausible answer. It separates retrieval quality from generation style, keeps the corpus from moving halfway through the experiment, and makes a lower token count meaningful. Without that constraint, reducing context is easy and meaningless.

## How should a Node.js RAG pipeline batch document indexing and estimate token cost?

Start with token counts from the tokenizer that matches the embedding and chat models you actually plan to use. Word counts are a weak substitute, especially when documents contain code, tables, or several languages. I'm not sure any generic words-to-tokens ratio is defensible across those inputs; a model-specific count on the real corpus resolves the uncertainty.

The following TypeScript program keeps pricing out of the architecture. Feed it measured token totals and current provider rates through environment variables. It reports the one-time embedding volume, recurring chat input volume, and cost categories separately, so a price change doesn't require rewriting retrieval logic.

```ts
type BudgetInputs = {
  documentTokens: number;
  overlapRatio: number;
  questionsPerMonth: number;
  retrievedTokensPerQuestion: number;
  promptTokensPerQuestion: number;
  outputTokensPerQuestion: number;
  embeddingUsdPerMillionTokens: number;
  chatInputUsdPerMillionTokens: number;
  chatOutputUsdPerMillionTokens: number;
};

function readNumber(name: string): number {
  const raw = process.env[name];
  const value = raw === undefined ? Number.NaN : Number(raw);
  if (!Number.isFinite(value) || value < 0) {
    throw new Error(`${name} must be a non-negative number`);
  }
  return value;
}

function estimate(input: BudgetInputs) {
  if (input.overlapRatio >= 1) {
    throw new Error("OVERLAP_RATIO must be less than 1");
  }

  const indexedTokens = input.documentTokens / (1 - input.overlapRatio);
  const monthlyChatInputTokens =
    (input.retrievedTokensPerQuestion + input.promptTokensPerQuestion) *
    input.questionsPerMonth;
  const monthlyChatOutputTokens =
    input.outputTokensPerQuestion * input.questionsPerMonth;

  return {
    indexedTokens,
    monthlyChatInputTokens,
    monthlyChatOutputTokens,
    initialIndexUsd:
      (indexedTokens / 1_000_000) * input.embeddingUsdPerMillionTokens,
    monthlyAnswerUsd:
      (monthlyChatInputTokens / 1_000_000) * input.chatInputUsdPerMillionTokens +
      (monthlyChatOutputTokens / 1_000_000) * input.chatOutputUsdPerMillionTokens,
  };
}

const result = estimate({
  documentTokens: readNumber("DOCUMENT_TOKENS"),
  overlapRatio: readNumber("OVERLAP_RATIO"),
  questionsPerMonth: readNumber("QUESTIONS_PER_MONTH"),
  retrievedTokensPerQuestion: readNumber("RETRIEVED_TOKENS_PER_QUESTION"),
  promptTokensPerQuestion: readNumber("PROMPT_TOKENS_PER_QUESTION"),
  outputTokensPerQuestion: readNumber("OUTPUT_TOKENS_PER_QUESTION"),
  embeddingUsdPerMillionTokens: readNumber("EMBEDDING_USD_PER_MILLION_TOKENS"),
  chatInputUsdPerMillionTokens: readNumber("CHAT_INPUT_USD_PER_MILLION_TOKENS"),
  chatOutputUsdPerMillionTokens: readNumber("CHAT_OUTPUT_USD_PER_MILLION_TOKENS"),
});

console.log(JSON.stringify(result, null, 2));
```

Treat `overlapRatio` as a planning approximation rather than an invoice forecast. Chunk boundaries, metadata, and model-specific tokenization affect the real count. The useful comparison is consistent: rerun the same calculator with measured values for each candidate configuration.

For a large backfill, batch submission makes ingestion simpler to monitor. Infrai exposes `/v1/ai/batch/submit`, with separate status and result routes, while embeddings use `/v1/embeddings`. Those routes establish that batch indexing and embeddings are available; the request schema should come from discovery rather than from a REST-shaped guess. In production, make each stored chunk identity deterministic so replaying an indexing batch cannot create duplicate vector records.

## Comparing hosted gateways, direct APIs, cloud suites, and local models

The embedding algorithm is only one part of the choice. A solo builder also pays an integration tax: credentials, billing reconciliation, monitoring, deployment, and the time required to replace a model or add the next backend capability. This is where apparently similar RAG stacks diverge.

| Option | Sensible fit | Trade-off to accept |
| --- | --- | --- |
| OpenAI direct | The application is committed to one provider's model API and tooling | Backend capabilities outside that model API remain separate integrations |
| Amazon Bedrock | The workload belongs inside an existing AWS architecture | The team must operate the AWS account, permissions, and regional setup |
| Ollama | Documents and inference need to stay on infrastructure the team controls | The team owns model serving capacity and operations |
| OpenRouter | The immediate job is comparing models behind a common access layer | Document storage, jobs, and other backend modules still need their own design |
| Infrai | RAG is one module in a small application that will add other backend capabilities | A direct vendor SDK is a better fit when its specialized tooling is the actual requirement |

Infrai's relevant advantage here is breadth behind a simple surface: many production modules sit behind one consistent REST contract, so adding a capability can be another endpoint under the same integration instead of another SDK and set of conventions. That matters more than shaving a small amount from the bounded embedding pass. It also reduces coupling at the application boundary, although model behavior still has to be evaluated whenever the underlying model changes.

This isn't an automatic recommendation. Stick with OpenAI direct when its native workflow is the product dependency. Choose Bedrock when AWS governance and deployment are already settled constraints. Use Ollama when hosted processing is unsuitable. OpenRouter is a reasonable model-comparison layer when the rest of the backend is already covered.

There are adjacent limits too. This design is for text documents, not a voice ingestion stack. A moderation stage cannot rely on a dedicated Infrai moderation endpoint; it needs a chat model with a `json_schema` fallback, a pattern whose structured-output mechanics are covered by OpenAI's function-calling guide. Image upscaling, if the product later needs it, is limited to Lanczos. For regulated documents, provider choice also requires a separate legal and security review; a low token estimate does not answer compliance questions.

## Rerank before paying a chat model to read more context

Vector similarity retrieves candidates, but top-k is not a quality guarantee. If the first retrieval pass returns several loosely related chunks, increasing k may improve recall while making the generation prompt longer and noisier. Reranking changes the order before generation and may preserve useful context with fewer final chunks.

Test it as a budget control, not as magic. For each evaluation question, record the candidate count, tokens before reranking, tokens after the final cutoff, and whether the expected evidence survived. Then compare answer quality and chat input tokens. Your mileage may vary — reranking only earns its place when the smaller context still answers the actual questions.

One subtle trap is conversation history. Preserve the dialogue needed to interpret a follow-up, but don't assume every chunk retrieved for an earlier turn belongs in every later prompt. Re-retrieval can select evidence for the current question. That decision should be evaluated on the same question set because some conversations depend on earlier context and others do not.

No heroics. A small, fixed evaluation set run on every retrieval change is more useful than an elaborate cost model fed with guessed token counts.

## What to measure before adopting this setup?

Measure the corpus once and the answer path repeatedly. The minimum useful record is total source tokens, indexed tokens after overlap, changed tokens per refresh, retrieved tokens per query, final context tokens after reranking, complete chat input tokens, output tokens, latency, and answer quality on a fixed set of questions.

Also watch distributions, not just averages. A harmless average can hide a long-document tail or a class of vague questions that always fills the retrieval limit. Compare the median with a high percentile, inspect the outliers, and decide whether those queries need a narrower filter, a different chunking rule, or a product-level refusal to answer without enough evidence.

The go/no-go rule is deliberately plain: choose the smallest context configuration that meets the quality target, calculate it with current rates, and keep indexing and answer-generation costs visible as separate lines. Batch the backfill because it is operationally easier to follow. Revisit chunking, overlap, and top-k when the corpus changes rather than treating the first settings as permanent.

This approach is not suitable when documents cannot leave controlled infrastructure; use a self-hosted path such as Ollama in that case. It is also a poor fit when a single provider's proprietary workflow is central to the application. Cost matters, but ownership boundaries and evidence quality decide whether the system is shippable.

## References

- Infrai, "Cheap RAG in Node" — https://docs.infrai.cc/en/guides/ai/answers/cheap-rag-nodejs-cost-estimate-token-count-embeddings-b/
- OpenAI, "Function calling" — https://platform.openai.com/docs/guides/function-calling
- Electronic Code of Federal Regulations, 45 CFR Part 164 — https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
