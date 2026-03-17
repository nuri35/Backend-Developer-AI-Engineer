# 12 — Observability

**Week:** 7
**Time to learn:** 3–4 hours

---

## What is this topic?

Observability in AI systems means being able to see exactly what your system is doing — every LLM call, every retrieval, every tool execution — including cost, latency, and quality.

If you already use OpenTelemetry for backend observability, this is familiar territory. You just need to add the AI-specific layer.

---

## Why AI observability is different

Regular backend observability tracks:
- HTTP request duration
- Database query time
- Error rates

AI observability adds:
- Token usage and cost per request
- Quality scores (faithfulness, relevance)
- Prompt versions
- Model versions
- Trace of the full reasoning chain

---

## Langfuse — the recommended tool

[Langfuse](https://langfuse.com) is an open-source LLM observability platform. It is self-hostable (important for privacy), has a TypeScript SDK, and integrates with OpenTelemetry.

```bash
npm install langfuse
```

### Basic setup

```typescript
import { Langfuse } from "langfuse"

const langfuse = new Langfuse({
  publicKey: process.env.LANGFUSE_PUBLIC_KEY,
  secretKey: process.env.LANGFUSE_SECRET_KEY,
  baseUrl: process.env.LANGFUSE_HOST  // your self-hosted instance
})
```

### Tracing a RAG request

```typescript
async function tracedRagQuery(
  userId: string,
  question: string
): Promise<RagResponse> {
  // Start a trace for this request
  const trace = langfuse.trace({
    name: "rag-query",
    userId,
    input: { question }
  })

  // Trace the retrieval step
  const retrievalSpan = trace.span({
    name: "vector-retrieval",
    input: { question }
  })

  const chunks = await retrieveRelevantChunks(question)

  retrievalSpan.end({
    output: { chunksRetrieved: chunks.length, topScore: chunks[0]?.score }
  })

  // Trace the LLM call
  const llmGeneration = trace.generation({
    name: "answer-generation",
    model: "claude-3-5-sonnet-20241022",
    input: buildRagPrompt(question, chunks)
  })

  const answer = await generateAnswer(question, chunks)

  llmGeneration.end({
    output: answer.text,
    usage: {
      input: answer.inputTokens,
      output: answer.outputTokens
    }
  })

  // End the trace
  trace.update({ output: { answer: answer.text } })
  await langfuse.flushAsync()

  return { answer: answer.text, sources: chunks.map(c => c.source) }
}
```

---

## What to track

### Per-request metrics

| Metric | Why it matters |
|--------|---------------|
| Input tokens | Directly affects cost |
| Output tokens | Directly affects cost |
| Time to first token | User experience |
| Total latency | User experience |
| Model version | Helps debug when a model update breaks things |
| Prompt version | Helps debug when a prompt change breaks things |
| Faithfulness score | Quality signal |

### Aggregate metrics

| Metric | Alert threshold |
|--------|----------------|
| Average faithfulness | Alert if < 0.75 |
| Error rate | Alert if > 1% |
| P95 latency | Alert if > 10s |
| Daily cost | Alert if > budget |
| Hallucination rate | Alert if > 0% |

---

## OpenTelemetry integration

If you already use OpenTelemetry for your backend, you can add LLM tracing to the same pipeline:

```typescript
import { trace, SpanKind } from "@opentelemetry/api"

const tracer = trace.getTracer("ai-service")

async function tracedLlmCall(params: LlmCallParams): Promise<LlmResponse> {
  return tracer.startActiveSpan("llm.chat", {
    kind: SpanKind.CLIENT,
    attributes: {
      "llm.model": params.model,
      "llm.prompt_version": params.promptVersion,
      "llm.max_tokens": params.maxTokens
    }
  }, async (span) => {
    try {
      const response = await callLlm(params)

      span.setAttributes({
        "llm.input_tokens": response.usage.inputTokens,
        "llm.output_tokens": response.usage.outputTokens,
        "llm.cost_usd": calculateCost(response.usage)
      })

      return response
    } catch (error) {
      span.recordException(error)
      throw error
    } finally {
      span.end()
    }
  })
}
```

---

## Prompt versioning in Langfuse

Store your prompts in Langfuse so you can track which version was used for each request:

```typescript
// Fetch prompt from Langfuse (it tracks which version is used)
const prompt = await langfuse.getPrompt("rag-system-prompt")

const response = await anthropic.messages.create({
  model: "claude-3-5-sonnet-20241022",
  system: prompt.compile({ context: retrievedContext }),
  messages: [{ role: "user", content: userQuestion }]
})
```

Now in Langfuse you can see: "This response used prompt version 3. The one before it used version 2." This makes debugging much easier.

---

## Alerting on quality degradation

```typescript
@Injectable()
export class QualityMonitorService {
  private recentScores: number[] = []

  async recordScore(faithfulness: number): Promise<void> {
    this.recentScores.push(faithfulness)

    // Keep last 100 scores
    if (this.recentScores.length > 100) {
      this.recentScores.shift()
    }

    // Alert if rolling average drops
    const average = this.recentScores.reduce((a, b) => a + b, 0) / this.recentScores.length

    if (average < 0.75 && this.recentScores.length >= 10) {
      await this.alertService.send({
        severity: "warning",
        message: `AI quality degradation: average faithfulness is ${average.toFixed(2)}`,
        context: { recentScores: this.recentScores.slice(-10) }
      })
    }
  }
}
```

---

## Key terms

| Term | Meaning |
|------|---------|
| Trace | A complete record of one end-to-end request through your AI system |
| Span | One step within a trace (e.g., one LLM call, one retrieval) |
| Generation | A specific span type for LLM calls that tracks token usage |
| Langfuse | An open-source LLM observability platform |
| LangSmith | Observability tool from LangChain |
| Drift | When output quality slowly gets worse over time |
| Prompt version | A labeled version of a prompt stored in a management system |

---

## What to do next

- Set up Langfuse locally with Docker
- Add tracing to your RAG system
- Create a dashboard showing daily cost and average faithfulness
- Read [13 — Production Patterns](./13-production-patterns.md) for reliability