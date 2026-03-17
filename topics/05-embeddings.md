# 05 — Embeddings

**Week:** 3
**Time to learn:** 3–4 hours

---

## What is this topic?

Embeddings are the foundation of semantic search and RAG systems. Without understanding embeddings, you cannot build a working RAG pipeline.

The good news: you do not need to understand the math. You need to understand what embeddings are, how to generate them, and when to use them.

---

## What is an embedding?

An embedding is a list of numbers that represents the **meaning** of a piece of text.

For example, the sentence "The dog ran through the park" might become:
```
[0.021, -0.143, 0.892, 0.034, -0.567, ...]  // 1536 numbers
```

These numbers capture the semantic meaning of the text. Two texts that mean similar things will have similar numbers (similar vectors). Two texts that mean different things will have very different numbers.

This is how semantic search works: instead of matching keywords, you match meaning.

---

## Semantic similarity

If two pieces of text have similar meaning, their embeddings will be close to each other in vector space. If they have different meaning, they will be far apart.

Examples:
- "The dog ran fast" and "The puppy sprinted quickly" → similar vectors (same meaning, different words)
- "The dog ran fast" and "Tax filing deadline is April" → very different vectors

This is powerful because it lets you find relevant content even when the exact words do not match.

---

## Cosine similarity

The most common way to measure how similar two embeddings are is **cosine similarity**. It returns a value between -1 and 1:

- **1.0**: identical meaning
- **0.8+**: very similar
- **0.5**: somewhat related
- **0.0**: completely unrelated
- **-1.0**: opposite meaning

You do not implement this yourself — vector databases do it for you. But you need to know what the score means when you see it.

---

## Embedding models

### OpenAI embedding models

```typescript
import OpenAI from "openai"

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY })

async function embed(text: string): Promise<number[]> {
  const response = await openai.embeddings.create({
    model: "text-embedding-3-small",   // 1536 dimensions, cheap
    // model: "text-embedding-3-large", // 3072 dimensions, better quality
    input: text
  })
  return response.data[0].embedding
}
```

| Model | Dimensions | Cost | Use case |
|-------|-----------|------|---------|
| text-embedding-3-small | 1536 | $0.02/1M tokens | Most use cases |
| text-embedding-3-large | 3072 | $0.13/1M tokens | Higher accuracy needed |
| text-embedding-ada-002 | 1536 | $0.10/1M tokens | Legacy, avoid for new projects |

### Open-source embedding models

If you want to avoid API costs or need to keep data private, you can use open-source models:

- **Sentence Transformers** (`all-MiniLM-L6-v2`): Fast, small, good quality
- **e5-large-v2**: High quality, slightly slower
- **nomic-embed-text**: Good quality, can run locally with Ollama

```bash
# Run an embedding model locally with Ollama
ollama pull nomic-embed-text
```

---

## Batch embedding

When you have many documents to embed (for example, when building your RAG knowledge base), embed them in batches instead of one at a time:

```typescript
async function embedBatch(texts: string[]): Promise<number[][]> {
  // OpenAI supports up to 2048 inputs per request
  const BATCH_SIZE = 100
  const results: number[][] = []

  for (let i = 0; i < texts.length; i += BATCH_SIZE) {
    const batch = texts.slice(i, i + BATCH_SIZE)
    const response = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: batch
    })
    results.push(...response.data.map(item => item.embedding))

    // Small delay to respect rate limits
    if (i + BATCH_SIZE < texts.length) {
      await new Promise(resolve => setTimeout(resolve, 100))
    }
  }

  return results
}
```

Batch embedding is **much faster and cheaper** than embedding one document at a time.

---

## Single embedding for queries

When a user asks a question, you embed just that one question and use it to search the vector database:

```typescript
async function searchKnowledgeBase(userQuestion: string): Promise<SearchResult[]> {
  // 1. Embed the user's question
  const queryEmbedding = await embed(userQuestion)

  // 2. Search the vector database for similar chunks
  const results = await vectorDb.search({
    vector: queryEmbedding,
    topK: 5,
    minScore: 0.7
  })

  return results
}
```

---

## What embeddings are NOT good for

Embeddings work well for semantic similarity. They do NOT work well for:

- **Exact keyword matching**: "order #12345" — use regular search for this
- **Numbers and dates**: "Find orders from 2024" — use SQL
- **Filtering by category or user**: "Show only my documents" — use metadata filters

For these cases, use hybrid search (vector + keyword) which is covered in [06 — Vector Databases](./06-vector-databases.md).

---

## Caching embeddings

Embedding the same text multiple times wastes money. Cache the results:

```typescript
@Injectable()
export class EmbeddingService {
  constructor(
    private readonly redis: Redis,
    private readonly openai: OpenAI
  ) {}

  async embed(text: string): Promise<number[]> {
    const cacheKey = `embedding:${createHash("md5").update(text).digest("hex")}`

    // Check cache first
    const cached = await this.redis.get(cacheKey)
    if (cached) return JSON.parse(cached)

    // Generate embedding
    const response = await this.openai.embeddings.create({
      model: "text-embedding-3-small",
      input: text
    })
    const embedding = response.data[0].embedding

    // Cache for 7 days (embeddings don't change)
    await this.redis.setex(cacheKey, 604800, JSON.stringify(embedding))

    return embedding
  }
}
```

---

## Key terms

| Term | Meaning |
|------|---------|
| Embedding | A list of numbers representing the meaning of text |
| Vector | Another word for an embedding (a list of numbers) |
| Dimensions | How many numbers are in the embedding (e.g., 1536) |
| Cosine similarity | A score (0–1) measuring how similar two embeddings are |
| Semantic search | Finding content by meaning, not by exact keyword match |
| Batch embedding | Embedding many texts in one API call for efficiency |

---

## What to do next

- Generate embeddings for 50 text samples and compare their cosine similarity scores
- Build a simple semantic search function: embed a query, find the 3 most similar texts
- Read [06 — Vector Databases](./06-vector-databases.md) to store and search embeddings at scale