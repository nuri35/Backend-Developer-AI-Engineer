# 07 — RAG Pipeline

**Week:** 4
**Time to learn:** 6–8 hours

---

## What is this topic?

RAG (Retrieval-Augmented Generation) is the most important AI pattern for backend developers. It is how you connect an LLM to your own data.

Without RAG, an LLM can only answer questions using information from its training data (which has a cutoff date and does not include your company's internal data). With RAG, you give the model access to any information you want — in real time.

---

## The problem RAG solves

LLMs have two limitations:

1. **Knowledge cutoff**: The model does not know about events or data after its training date
2. **No access to your data**: The model has never seen your product documentation, user data, internal policies, etc.

RAG solves both problems by retrieving relevant information at query time and giving it to the model as context.

---

## How RAG works

RAG has two phases:

### Phase 1: Ingestion (happens once, or periodically)

```
Documents → Parse → Chunk → Embed → Store in Vector DB
```

### Phase 2: Retrieval and Generation (happens every user request)

```
User Question → Embed Question → Search Vector DB → Get Relevant Chunks → Build Prompt → LLM → Answer
```

---

## Phase 1: Ingestion pipeline

### Step 1 — Load documents

```typescript
import * as fs from "fs"
import * as path from "path"

async function loadDocuments(folderPath: string): Promise<RawDocument[]> {
  const files = fs.readdirSync(folderPath)
  const documents: RawDocument[] = []

  for (const file of files) {
    const filePath = path.join(folderPath, file)
    const content = fs.readFileSync(filePath, "utf-8")
    documents.push({
      id: file,
      content,
      source: filePath,
      type: path.extname(file).slice(1)
    })
  }

  return documents
}
```

For PDFs, use [Docling](https://github.com/DS4SD/docling) — a library that converts PDF and other formats into clean text:

```bash
pip install docling  # Python tool, but worth using for parsing
# Or use pdf-parse in Node.js for simpler PDFs
npm install pdf-parse
```

### Step 2 — Chunk documents

You cannot embed an entire document as one vector. Long texts lose meaning. You need to split them into smaller, overlapping chunks.

```typescript
function chunkText(
  text: string,
  chunkSize = 500,    // characters per chunk
  overlap = 100       // characters of overlap between chunks
): string[] {
  const chunks: string[] = []
  let start = 0

  while (start < text.length) {
    const end = Math.min(start + chunkSize, text.length)
    chunks.push(text.slice(start, end))
    start += chunkSize - overlap  // move forward but keep some overlap
  }

  return chunks
}
```

**Why overlap?** If an answer spans two chunks, the overlap ensures the relevant content appears in at least one chunk.

**Chunk size guidelines:**
- Too small (< 100 chars): Not enough context in each chunk
- Too large (> 1000 chars): The chunk contains too many unrelated ideas
- Sweet spot: 300–600 characters for most use cases

### Step 3 — Embed and store

```typescript
async function ingestDocuments(documents: RawDocument[]): Promise<void> {
  for (const doc of documents) {
    const chunks = chunkText(doc.content)

    for (let i = 0; i < chunks.length; i++) {
      const embedding = await embedText(chunks[i])

      await qdrant.upsert("knowledge", {
        points: [{
          id: `${doc.id}-chunk-${i}`,
          vector: embedding,
          payload: {
            content: chunks[i],
            source: doc.source,
            chunkIndex: i,
            totalChunks: chunks.length,
            documentId: doc.id
          }
        }]
      })
    }
  }
  console.log("Ingestion complete")
}
```

---

## Phase 2: Retrieval and generation

### Step 4 — Retrieve relevant chunks

```typescript
async function retrieveRelevantChunks(
  question: string,
  topK = 5
): Promise<Chunk[]> {
  const queryEmbedding = await embedText(question)

  const results = await qdrant.search("knowledge", {
    vector: queryEmbedding,
    limit: topK,
    score_threshold: 0.65,
    with_payload: true
  })

  return results.map(r => ({
    content: r.payload.content as string,
    source: r.payload.source as string,
    score: r.score
  }))
}
```

### Step 5 — Build the prompt with context

```typescript
function buildRagPrompt(question: string, chunks: Chunk[]): string {
  const context = chunks
    .map((c, i) => `[Source ${i + 1}: ${c.source}]\n${c.content}`)
    .join("\n\n---\n\n")

  return `
You are a helpful assistant. Answer the user's question using ONLY the information provided below.

If the answer is not in the provided information, say: "I don't have enough information to answer this question."

Do not make up information. Do not use knowledge from outside the provided context.

CONTEXT:
${context}

QUESTION: ${question}
  `.trim()
}
```

### Step 6 — Generate the answer

```typescript
async function askQuestion(question: string): Promise<RagResponse> {
  // 1. Retrieve relevant chunks
  const chunks = await retrieveRelevantChunks(question)

  if (chunks.length === 0) {
    return {
      answer: "I don't have any information about this topic.",
      sources: []
    }
  }

  // 2. Build prompt
  const prompt = buildRagPrompt(question, chunks)

  // 3. Generate answer
  const response = await anthropic.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 1000,
    messages: [{ role: "user", content: prompt }]
  })

  return {
    answer: response.content[0].text,
    sources: chunks.map(c => c.source)
  }
}
```

---

## Advanced RAG patterns

### Query rewriting

The user's question is often not written in a way that searches well. Rewrite it before searching:

```typescript
async function rewriteQuery(originalQuery: string): Promise<string> {
  const response = await anthropic.messages.create({
    model: "claude-3-5-haiku-20241022",
    max_tokens: 100,
    messages: [{
      role: "user",
      content: `Rewrite this question to improve search results. 
      Make it more specific and include key terms.
      Return only the rewritten question, nothing else.
      
      Original: ${originalQuery}`
    }]
  })
  return response.content[0].text
}
```

### HyDE (Hypothetical Document Embedding)

Instead of embedding the question, generate a hypothetical answer first, then embed that. This often finds better matches:

```typescript
async function hydeSearch(question: string): Promise<Chunk[]> {
  // 1. Generate a hypothetical answer
  const hypotheticalAnswer = await anthropic.messages.create({
    model: "claude-3-5-haiku-20241022",
    max_tokens: 200,
    messages: [{
      role: "user",
      content: `Write a short answer to this question. It can be approximate.
      Question: ${question}`
    }]
  })

  // 2. Embed the hypothetical answer (not the question)
  const queryEmbedding = await embedText(hypotheticalAnswer.content[0].text)

  // 3. Search with the hypothetical answer's embedding
  return retrieveWithEmbedding(queryEmbedding)
}
```

### Reranking

After getting the top 10 results from vector search, use a reranker to pick the best 3:

```typescript
async function rerankResults(
  question: string,
  candidates: Chunk[]
): Promise<Chunk[]> {
  const response = await anthropic.messages.create({
    model: "claude-3-5-haiku-20241022",
    max_tokens: 200,
    messages: [{
      role: "user",
      content: `Given this question: "${question}"
      
Rank these ${candidates.length} text chunks from most to least relevant.
Return only the indices in order, like: [2, 0, 4, 1, 3]

${candidates.map((c, i) => `[${i}]: ${c.content.slice(0, 200)}`).join('\n\n')}`
    }]
  })

  const indices = JSON.parse(response.content[0].text)
  return indices.slice(0, 3).map((i: number) => candidates[i])
}
```

---

## RAG evaluation

How do you know if your RAG system is working well?

```typescript
interface RagEvalMetrics {
  faithfulness: number    // Is the answer based on the retrieved context?
  relevance: number       // Is the retrieved context relevant to the question?
  correctness: number     // Is the answer actually correct?
}

async function evaluateRagResponse(
  question: string,
  retrievedChunks: Chunk[],
  generatedAnswer: string
): Promise<RagEvalMetrics> {
  const evalPrompt = `
You are an evaluator. Score these metrics from 0.0 to 1.0.

Question: ${question}
Retrieved context: ${retrievedChunks.map(c => c.content).join('\n')}
Generated answer: ${generatedAnswer}

Return JSON: {"faithfulness": X, "relevance": X, "correctness": X}
  `

  const response = await anthropic.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 100,
    messages: [{ role: "user", content: evalPrompt }]
  })

  return JSON.parse(response.content[0].text)
}
```

See [11 — Evaluation](./11-evaluation.md) for a complete evaluation system.

---

## Key terms

| Term | Meaning |
|------|---------|
| RAG | Retrieval-Augmented Generation — connecting LLMs to external data |
| Ingestion | The process of loading, chunking, embedding, and storing documents |
| Chunking | Splitting documents into smaller pieces for embedding |
| Chunk overlap | Repeating some text between consecutive chunks |
| Retrieval | Finding the most relevant chunks for a given query |
| Grounding | Answering questions based only on retrieved context |
| HyDE | Hypothetical Document Embedding — embed a generated answer to find better matches |
| Reranking | Re-scoring retrieved results to pick the best ones |
| Faithfulness | Whether the answer is supported by the retrieved context |

---

## What to do next

- Build a complete RAG system over your own documentation
- Add query rewriting and see if it improves results
- Set up evaluation with 10 test questions and expected answers
- Read [11 — Evaluation](./11-evaluation.md) for measuring RAG quality