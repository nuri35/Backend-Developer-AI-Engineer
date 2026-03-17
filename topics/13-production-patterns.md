# 13 — Production Patterns

**Week:** 8
**Time to learn:** 5–6 hours

---

## What is this topic?

This is where your backend experience gives you a big advantage. Making AI systems production-ready is fundamentally a backend engineering problem. Rate limiting, caching, queues, circuit breakers — you know all of these. You just need to apply them to AI.

---

## The AI Gateway pattern

All AI calls should go through a single service. This is the AI equivalent of an API Gateway.

```
Client → Your NestJS Services → AI Gateway → LLM APIs
```

The AI Gateway handles:
- Authentication and rate limiting
- Request logging and cost tracking
- Caching
- Provider routing (which model to use)
- Circuit breaking
- Fallback logic

```typescript
// src/ai/ai-gateway.service.ts
@Injectable()
export class AiGatewayService {
  constructor(
    private readonly rateLimiter: RateLimiterService,
    private readonly cache: CacheService,
    private readonly costTracker: CostTrackingService,
    private readonly llmService: LlmService
  ) {}

  async complete(params: GatewayRequest): Promise<GatewayResponse> {
    // 1. Rate limit check
    await this.rateLimiter.check(params.userId)

    // 2. Check cache
    const cached = await this.cache.get(params)
    if (cached) return { ...cached, fromCache: true }

    // 3. Select model based on complexity
    const model = this.selectModel(params.complexity)

    // 4. Call LLM with circuit breaker
    const response = await this.circuitBreaker.execute(() =>
      this.llmService.complete({ ...params, model })
    )

    // 5. Track cost
    await this.costTracker.record(params.userId, response.usage)

    // 6. Cache the result
    await this.cache.set(params, response)

    return response
  }
}
```

---

## Caching

LLM calls are expensive. Cache responses whenever the same question is asked again.

### Exact match caching

```typescript
@Injectable()
export class CacheService {
  constructor(private readonly redis: Redis) {}

  private buildCacheKey(params: GatewayRequest): string {
    const hash = createHash("md5")
      .update(JSON.stringify({
        model: params.model,
        systemPrompt: params.systemPrompt,
        userMessage: params.userMessage
      }))
      .digest("hex")
    return `llm:cache:${hash}`
  }

  async get(params: GatewayRequest): Promise<LlmResponse | null> {
    const key = this.buildCacheKey(params)
    const cached = await this.redis.get(key)
    return cached ? JSON.parse(cached) : null
  }

  async set(params: GatewayRequest, response: LlmResponse): Promise<void> {
    const key = this.buildCacheKey(params)
    const ttl = 3600  // 1 hour
    await this.redis.setex(key, ttl, JSON.stringify(response))
  }
}
```

### Semantic caching

Similar questions should get the same answer without calling the API again:

```typescript
async function semanticCacheGet(
  question: string,
  threshold = 0.95
): Promise<LlmResponse | null> {
  const questionEmbedding = await embedText(question)

  // Search the cache for a semantically similar question
  const cached = await vectorDb.search("response-cache", {
    vector: questionEmbedding,
    limit: 1,
    score_threshold: threshold
  })

  if (cached.length > 0) {
    return cached[0].payload.response as LlmResponse
  }

  return null
}

async function semanticCacheSet(
  question: string,
  response: LlmResponse
): Promise<void> {
  const embedding = await embedText(question)
  await vectorDb.upsert("response-cache", {
    points: [{
      id: createHash("md5").update(question).digest("hex"),
      vector: embedding,
      payload: { question, response, cachedAt: new Date() }
    }]
  })
}
```

---

## Rate limiting

You need to control both your own users and the LLM provider limits.

### Provider rate limits (tokens per minute)

```typescript
@Injectable()
export class ProviderRateLimiter {
  private readonly TPM_LIMIT = 100000  // tokens per minute for your account

  async checkAndConsume(estimatedTokens: number): Promise<void> {
    const key = `tpm:${new Date().getMinutes()}`
    const current = await this.redis.incrby(key, estimatedTokens)
    await this.redis.expire(key, 60)

    if (current > this.TPM_LIMIT) {
      const retryAfter = 60 - new Date().getSeconds()
      throw new TooManyRequestsException(`Rate limit exceeded. Retry after ${retryAfter}s`)
    }
  }
}
```

### Per-user rate limiting (Redis sliding window)

```typescript
async function checkUserRateLimit(
  userId: string,
  maxRequestsPerMinute = 10
): Promise<void> {
  const key = `ratelimit:${userId}`
  const now = Date.now()
  const windowStart = now - 60000  // 1 minute window

  // Remove old entries
  await redis.zremrangebyscore(key, 0, windowStart)

  // Count recent requests
  const count = await redis.zcard(key)

  if (count >= maxRequestsPerMinute) {
    throw new TooManyRequestsException("Rate limit exceeded")
  }

  // Add current request
  await redis.zadd(key, now, `${now}`)
  await redis.expire(key, 60)
}
```

---

## Async queue for long-running AI tasks

Some AI tasks take too long for a synchronous HTTP response. Use a queue:

```typescript
// src/ai/jobs/ai-analysis.processor.ts
@Processor("ai-tasks")
export class AiTaskProcessor {
  @Process("analyze-document")
  async analyzeDocument(job: Job<AnalysisJobData>): Promise<void> {
    const { documentId, userId } = job.data

    // Update status: started
    await this.statusService.update(documentId, "processing")

    try {
      // Long-running AI work here
      const chunks = await this.documentService.loadAndChunk(documentId)
      const analysis = await this.aiService.analyzeChunks(chunks)

      await this.documentService.saveAnalysis(documentId, analysis)
      await this.statusService.update(documentId, "complete")

      // Notify user via WebSocket or email
      await this.notificationService.notify(userId, "analysis-complete", { documentId })
    } catch (error) {
      await this.statusService.update(documentId, "failed", error.message)
    }
  }
}

// src/documents/documents.controller.ts
@Post(":id/analyze")
async triggerAnalysis(@Param("id") documentId: string) {
  // Add to queue and return immediately
  const jobId = await this.analysisQueue.add("analyze-document", { documentId, userId })
  return { jobId, status: "queued", message: "Analysis started. You will be notified when complete." }
}
```

---

## Circuit breaker

When the LLM API is down or slow, fail fast instead of hanging:

```typescript
import * as CircuitBreaker from "opossum"

const options = {
  timeout: 15000,           // 15 seconds timeout
  errorThresholdPercentage: 50,  // Open after 50% failures
  resetTimeout: 30000       // Try again after 30 seconds
}

const breaker = new CircuitBreaker(callLlmApi, options)

breaker.fallback(async (params) => {
  // Return cached response or graceful degradation
  const cached = await cache.getLastKnownGoodResponse(params.userId)
  if (cached) return cached
  throw new ServiceUnavailableException("AI service is temporarily unavailable")
})

// Use the breaker instead of calling the API directly
const response = await breaker.fire(params)
```

---

## Session and conversation management

For chat applications, manage the conversation history carefully:

```typescript
@Injectable()
export class ConversationService {
  private readonly MAX_MESSAGES = 20
  private readonly MAX_TOKENS = 4000

  async getHistory(sessionId: string): Promise<Message[]> {
    const messages = await this.redis.lrange(`session:${sessionId}`, 0, -1)
    return messages.map(m => JSON.parse(m))
  }

  async addMessage(sessionId: string, message: Message): Promise<void> {
    await this.redis.rpush(`session:${sessionId}`, JSON.stringify(message))
    await this.redis.expire(`session:${sessionId}`, 3600)  // 1 hour TTL

    // Summarize if too long
    const history = await this.getHistory(sessionId)
    if (history.length > this.MAX_MESSAGES) {
      await this.summarizeHistory(sessionId, history)
    }
  }

  private async summarizeHistory(sessionId: string, messages: Message[]): Promise<void> {
    const summary = await this.aiService.summarize(messages.slice(0, 10))

    // Replace old messages with summary
    await this.redis.del(`session:${sessionId}`)
    await this.redis.rpush(`session:${sessionId}`, JSON.stringify({
      role: "system",
      content: `Previous conversation summary: ${summary}`
    }))
    // Keep the last 5 messages
    const recent = messages.slice(-5)
    for (const msg of recent) {
      await this.redis.rpush(`session:${sessionId}`, JSON.stringify(msg))
    }
  }
}
```

---

## Graceful degradation

When AI is unavailable, the system should still work — just without AI features:

```typescript
async function getProductRecommendations(userId: string): Promise<Product[]> {
  try {
    // Try AI-powered recommendations
    return await aiRecommendationService.getPersonalized(userId)
  } catch (error) {
    // Fallback to rule-based recommendations
    logger.warn("AI recommendations unavailable, using fallback", { error: error.message })
    return await productService.getPopularProducts({ limit: 10 })
  }
}
```

---

## Key terms

| Term | Meaning |
|------|---------|
| AI Gateway | A central service that all AI calls go through |
| Semantic caching | Caching responses for semantically similar (not just identical) queries |
| Circuit breaker | Stops calling a failing service to allow it to recover |
| Rate limiting | Controlling how many requests can be made per time period |
| Async queue | Processing long-running AI tasks in the background |
| Graceful degradation | Falling back to simpler behavior when AI is unavailable |

---

## What to do next

- Build an AI Gateway service in NestJS with rate limiting and caching
- Add a circuit breaker around your LLM API calls
- Move long document analysis to an async queue
- Read [14 — Cost Management](./14-cost-management.md) for controlling spending