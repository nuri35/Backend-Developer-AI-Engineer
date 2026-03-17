# 14 — Cost Management

**Week:** 8
**Time to learn:** 2–3 hours

---

## What is this topic?

LLM API costs can grow very fast. Unlike traditional infrastructure where you pay for capacity, LLM APIs charge per token — and every user request generates tokens. Without careful management, a successful product can quickly become an expensive one.

---

## How LLM pricing works

You pay for input tokens (what you send) and output tokens (what you receive). They have different prices, with output tokens usually costing more.

Example pricing (approximate, check current rates):

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| GPT-4o | $2.50 | $10.00 |
| GPT-4o mini | $0.15 | $0.60 |
| Claude 3.5 Sonnet | $3.00 | $15.00 |
| Claude 3.5 Haiku | $0.80 | $4.00 |

A typical RAG response might use:
- 1,500 input tokens (system prompt + context + question)
- 300 output tokens (the answer)

With Claude 3.5 Haiku: `(1500/1M × $0.80) + (300/1M × $4.00) = $0.0012 + $0.0012 = $0.0024`

That is less than a cent per request. But at 100,000 requests per day, it adds up to $240/day.

---

## Tracking cost per request

Always log token usage and calculate cost for every request:

```typescript
interface CostRecord {
  userId: string
  feature: string
  model: string
  inputTokens: number
  outputTokens: number
  costUsd: number
  timestamp: Date
}

const MODEL_PRICING: Record<string, { input: number; output: number }> = {
  "claude-3-5-sonnet-20241022": { input: 3.00, output: 15.00 },
  "claude-3-5-haiku-20241022": { input: 0.80, output: 4.00 },
  "gpt-4o": { input: 2.50, output: 10.00 },
  "gpt-4o-mini": { input: 0.15, output: 0.60 }
}

function calculateCost(model: string, inputTokens: number, outputTokens: number): number {
  const pricing = MODEL_PRICING[model]
  if (!pricing) return 0

  return (
    (inputTokens / 1_000_000) * pricing.input +
    (outputTokens / 1_000_000) * pricing.output
  )
}

async function recordCost(
  userId: string,
  feature: string,
  model: string,
  usage: TokenUsage
): Promise<void> {
  const costUsd = calculateCost(model, usage.inputTokens, usage.outputTokens)

  await db.insert("cost_records", {
    userId,
    feature,
    model,
    inputTokens: usage.inputTokens,
    outputTokens: usage.outputTokens,
    costUsd,
    timestamp: new Date()
  })

  // Also track in Redis for real-time budget checking
  const today = new Date().toISOString().split("T")[0]
  await redis.incrbyfloat(`cost:user:${userId}:${today}`, costUsd)
}
```

---

## Budget alerts and spending limits

```typescript
@Injectable()
export class BudgetService {
  async checkUserBudget(userId: string): Promise<void> {
    const today = new Date().toISOString().split("T")[0]
    const todaySpend = parseFloat(
      await this.redis.get(`cost:user:${userId}:${today}`) ?? "0"
    )

    // Alert at 80% of daily budget
    const DAILY_BUDGET = 5.00  // $5 per user per day

    if (todaySpend >= DAILY_BUDGET) {
      throw new ForbiddenException("Daily AI budget exceeded. Resets at midnight.")
    }

    if (todaySpend >= DAILY_BUDGET * 0.8) {
      // Log warning but don't block
      this.logger.warn(`User ${userId} is at 80% of daily budget`, { todaySpend })
    }
  }

  async getMonthlySpend(userId: string): Promise<number> {
    const result = await this.db.query(
      `SELECT SUM(cost_usd) as total 
       FROM cost_records 
       WHERE user_id = $1 
       AND timestamp >= date_trunc('month', NOW())`,
      [userId]
    )
    return result.rows[0].total ?? 0
  }
}
```

---

## Model routing for cost optimization

Not every request needs the most powerful (and most expensive) model. Route based on complexity:

```typescript
type RequestComplexity = "simple" | "medium" | "complex"

function classifyComplexity(request: string): RequestComplexity {
  const wordCount = request.split(" ").length

  if (wordCount < 20) return "simple"
  if (wordCount < 100) return "medium"
  return "complex"
}

function selectModel(complexity: RequestComplexity): string {
  switch (complexity) {
    case "simple":
      return "claude-3-5-haiku-20241022"    // $0.80/M input — cheapest
    case "medium":
      return "gpt-4o-mini"                   // $0.15/M input — very cheap
    case "complex":
      return "claude-3-5-sonnet-20241022"    // $3.00/M input — best quality
  }
}

// In your AI Gateway
async function complete(request: AiRequest): Promise<AiResponse> {
  const complexity = classifyComplexity(request.userMessage)
  const model = selectModel(complexity)

  return this.llmService.complete({ ...request, model })
}
```

---

## Reducing costs through prompt optimization

Shorter prompts cost less. Review your prompts regularly:

```typescript
// Bad: Verbose system prompt (250 tokens)
const verbosePrompt = `
You are a very helpful and friendly assistant who works for our company.
Your job is to help customers with their questions about products.
You should always be polite and professional.
If you don't know the answer, please say so honestly.
Never make up information that you are not sure about.
Always try to provide complete and accurate answers.
`

// Better: Concise system prompt (60 tokens)
const concisePrompt = `
Customer support assistant. Answer product questions accurately.
Say "I don't know" if unsure. Never fabricate information.
`
```

Reducing from 250 to 60 tokens in the system prompt saves:
- 190 tokens × 100,000 daily requests × $3/1M = **$57/day saved**

---

## Prompt caching

Anthropic and OpenAI support prompt caching. If you send the same system prompt in many requests, you pay less for the repeated part:

```typescript
// Anthropic prompt caching — mark long system prompts for caching
const response = await anthropic.messages.create({
  model: "claude-3-5-sonnet-20241022",
  max_tokens: 1024,
  system: [
    {
      type: "text",
      text: longSystemPrompt,  // This gets cached after the first call
      cache_control: { type: "ephemeral" }
    }
  ],
  messages: [{ role: "user", content: userMessage }]
})

// Check if the cache was used
const cacheHit = response.usage.cache_read_input_tokens > 0
```

Cached tokens cost ~10x less than regular input tokens. For a 500-token system prompt used 10,000 times per day, this can save $13.50/day.

---

## Response caching to avoid duplicate API calls

If the same question is asked repeatedly, return the cached answer:

```typescript
async function getWithCache(
  question: string,
  ttlSeconds = 3600
): Promise<string> {
  const cacheKey = `response:${createHash("md5").update(question).digest("hex")}`

  const cached = await redis.get(cacheKey)
  if (cached) {
    // Free! No API call needed
    return cached
  }

  // Not cached — call the API (costs money)
  const answer = await callLlmApi(question)

  await redis.setex(cacheKey, ttlSeconds, answer)
  return answer
}
```

---

## Cost dashboard queries

```sql
-- Total cost today
SELECT SUM(cost_usd) as today_total
FROM cost_records
WHERE DATE(timestamp) = CURRENT_DATE;

-- Cost by feature
SELECT feature, SUM(cost_usd) as total, COUNT(*) as requests
FROM cost_records
WHERE timestamp >= NOW() - INTERVAL '7 days'
GROUP BY feature
ORDER BY total DESC;

-- Most expensive users
SELECT user_id, SUM(cost_usd) as total
FROM cost_records
WHERE timestamp >= DATE_TRUNC('month', NOW())
GROUP BY user_id
ORDER BY total DESC
LIMIT 10;

-- Average cost per request by model
SELECT model, AVG(cost_usd) as avg_cost, COUNT(*) as requests
FROM cost_records
GROUP BY model;
```

---

## Key terms

| Term | Meaning |
|------|---------|
| Input tokens | The tokens you send to the model (prompt, context, history) |
| Output tokens | The tokens the model generates in response |
| Prompt caching | Reusing cached tokens for repeated system prompts to reduce cost |
| Model routing | Sending different types of requests to cheaper or more expensive models |
| Budget ceiling | A hard limit on how much a user can spend on AI per day or month |
| Token budget | Setting a maximum number of tokens for a request to control cost |

---

## What to do next

- Add cost tracking to every LLM call in your system
- Build a simple daily cost report
- Implement model routing for simple vs complex requests
- Set up a budget alert that triggers when daily spend exceeds a threshold