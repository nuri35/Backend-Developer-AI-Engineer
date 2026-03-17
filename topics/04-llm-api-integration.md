# 04 — LLM API Integration

**Week:** 1–2
**Time to learn:** 4–6 hours

---

## What is this topic?

This is where backend engineering meets AI. You already know how to call external APIs, handle errors, and build reliable services. This topic shows you how to apply those skills specifically to LLM APIs.

The two main APIs you will use are:
- **OpenAI** (GPT-4o, GPT-4o mini)
- **Anthropic** (Claude 3.5 Sonnet, Claude 3.5 Haiku)

Both have TypeScript/Node.js SDKs and similar patterns.

---

## Basic setup

### OpenAI

```bash
npm install openai
```

```typescript
import OpenAI from "openai"

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
})

const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "What is a vector database?" }
  ],
  max_tokens: 500,
  temperature: 0.7
})

const text = response.choices[0].message.content
```

### Anthropic

```bash
npm install @anthropic-ai/sdk
```

```typescript
import Anthropic from "@anthropic-ai/sdk"

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY
})

const response = await anthropic.messages.create({
  model: "claude-3-5-sonnet-20241022",
  max_tokens: 500,
  system: "You are a helpful assistant.",
  messages: [
    { role: "user", content: "What is a vector database?" }
  ]
})

const text = response.content[0].text
```

---

## API structure: system, user, assistant roles

Every LLM API uses a messages array with roles:

```typescript
const messages = [
  // System: instructions that never change
  { role: "system", content: "You classify support tickets." },

  // User: the input
  { role: "user", content: "My payment failed." },

  // Assistant: the model's previous response (for conversation history)
  { role: "assistant", content: '{"category": "billing", "priority": "high"}' },

  // User: the next input
  { role: "user", content: "I tried again and it failed again." }
]
```

---

## Token counting and cost estimation

Every API call costs money based on the number of tokens. Always track this.

```typescript
// After an OpenAI call
const usage = response.usage
console.log({
  input_tokens: usage.prompt_tokens,
  output_tokens: usage.completion_tokens,
  total_tokens: usage.total_tokens
})

// Estimate cost (GPT-4o pricing as example)
const INPUT_COST_PER_1K = 0.0025   // $0.0025 per 1K input tokens
const OUTPUT_COST_PER_1K = 0.01    // $0.01 per 1K output tokens

const cost = 
  (usage.prompt_tokens / 1000) * INPUT_COST_PER_1K +
  (usage.completion_tokens / 1000) * OUTPUT_COST_PER_1K
```

---

## Structured output with JSON Schema

The most useful feature for backend systems: force the model to return a specific JSON structure.

### OpenAI structured outputs

```typescript
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  response_format: {
    type: "json_schema",
    json_schema: {
      name: "ticket_classification",
      schema: {
        type: "object",
        properties: {
          category: { type: "string", enum: ["billing", "shipping", "account", "technical"] },
          priority: { type: "string", enum: ["low", "medium", "high"] },
          summary: { type: "string" }
        },
        required: ["category", "priority", "summary"]
      }
    }
  },
  messages: [...]
})
```

### Validation with Zod

Always validate the output even when using JSON mode:

```typescript
import { z } from "zod"

const TicketSchema = z.object({
  category: z.enum(["billing", "shipping", "account", "technical"]),
  priority: z.enum(["low", "medium", "high"]),
  summary: z.string().min(1).max(200)
})

try {
  const parsed = JSON.parse(response.choices[0].message.content)
  const ticket = TicketSchema.parse(parsed)
  // Now TypeScript knows the exact shape
} catch (error) {
  // Handle validation failure — retry or fallback
}
```

---

## Retry logic with exponential backoff

LLM APIs can fail temporarily (rate limits, timeouts, server errors). You need retry logic.

```typescript
async function callWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error: any) {
      const isRetryable = 
        error.status === 429 ||  // rate limit
        error.status === 500 ||  // server error
        error.status === 503     // service unavailable

      if (!isRetryable || attempt === maxRetries - 1) {
        throw error
      }

      // Exponential backoff with jitter
      const baseDelay = Math.pow(2, attempt) * 1000  // 1s, 2s, 4s
      const jitter = Math.random() * 1000
      await new Promise(resolve => setTimeout(resolve, baseDelay + jitter))
    }
  }
  throw new Error("Max retries reached")
}

// Usage
const response = await callWithRetry(() =>
  openai.chat.completions.create({ model: "gpt-4o", messages })
)
```

---

## Fallback model chain

When your primary model fails or is too slow, fall back to a cheaper or different model:

```typescript
async function callWithFallback(
  messages: any[],
  primaryModel = "gpt-4o",
  fallbackModel = "gpt-4o-mini"
): Promise<string> {
  try {
    const response = await openai.chat.completions.create({
      model: primaryModel,
      messages,
      timeout: 10000  // 10 second timeout
    })
    return response.choices[0].message.content
  } catch (error) {
    console.warn(`Primary model failed, using fallback: ${error.message}`)

    const fallbackResponse = await openai.chat.completions.create({
      model: fallbackModel,
      messages
    })
    return fallbackResponse.choices[0].message.content
  }
}
```

---

## Multi-provider routing

For cost optimization, route different types of requests to different models:

```typescript
type TaskComplexity = "simple" | "medium" | "complex"

function selectModel(complexity: TaskComplexity): string {
  switch (complexity) {
    case "simple":
      return "gpt-4o-mini"          // $0.15/1M tokens — cheap
    case "medium":
      return "claude-3-5-haiku-20241022"  // fast and capable
    case "complex":
      return "claude-3-5-sonnet-20241022" // best quality
  }
}

// Example: classify the task complexity first, then pick the model
async function smartCall(task: string): Promise<string> {
  const complexity = await classifyComplexity(task)
  const model = selectModel(complexity)
  return callLLM(model, task)
}
```

---

## Building an LLM service wrapper in NestJS

Wrap all LLM calls in a single service. This makes it easy to:
- Switch providers
- Add logging
- Track costs
- Add retries in one place

```typescript
// src/ai/llm.service.ts
@Injectable()
export class LlmService {
  private readonly openai: OpenAI
  private readonly anthropic: Anthropic

  constructor(private readonly configService: ConfigService) {
    this.openai = new OpenAI({ apiKey: configService.get("OPENAI_API_KEY") })
    this.anthropic = new Anthropic({ apiKey: configService.get("ANTHROPIC_API_KEY") })
  }

  async complete(params: {
    model: string
    systemPrompt: string
    userMessage: string
    maxTokens?: number
    temperature?: number
  }): Promise<{ text: string; inputTokens: number; outputTokens: number }> {
    const response = await callWithRetry(() =>
      this.anthropic.messages.create({
        model: params.model,
        max_tokens: params.maxTokens ?? 1000,
        system: params.systemPrompt,
        messages: [{ role: "user", content: params.userMessage }]
      })
    )

    return {
      text: response.content[0].text,
      inputTokens: response.usage.input_tokens,
      outputTokens: response.usage.output_tokens
    }
  }
}
```

---

## Key terms

| Term | Meaning |
|------|---------|
| Chat Completions | The main API endpoint for conversational LLM calls |
| JSON mode | Forces the model to return valid JSON |
| JSON Schema | Defines the exact structure of the expected JSON output |
| Rate limit | Maximum number of requests or tokens per minute the API allows |
| Retry logic | Automatically retrying failed requests after a delay |
| Exponential backoff | Waiting longer between each retry attempt |
| Fallback model | A cheaper or different model used when the primary fails |
| Token usage | The count of input and output tokens in a request |

---

## What to do next

- Build the `LlmService` wrapper in NestJS with retry logic and cost tracking
- Test structured output with Zod validation
- Read [02 — Prompt Engineering](./02-prompt-engineering.md) to write better prompts
- Read [08 — Streaming](./08-streaming.md) to stream responses to your users