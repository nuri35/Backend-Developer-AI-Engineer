# 06 — Vector Databases

**Week:** 3
**Time to learn:** 4–5 hours

---

## What is this topic?

A vector database stores embeddings and lets you search them by similarity. Without a vector database, you cannot build a scalable RAG system.

You already know PostgreSQL. The easiest starting point is **pgvector** — a PostgreSQL extension that adds vector search capabilities to a database you already know.

---

## Why a regular database is not enough

Regular databases search by exact values or text patterns:
```sql
SELECT * FROM documents WHERE content LIKE '%machine learning%'
```

This only finds exact keyword matches. It misses "artificial intelligence", "deep learning", "neural networks" — all related concepts.

A vector database searches by meaning:
```
Find documents that mean something similar to "machine learning"
```
It returns relevant results even when the exact words do not appear.

---

## How vector search works

1. During indexing: Every document is embedded into a vector and stored
2. During search: The query is embedded into a vector
3. The database finds stored vectors that are closest to the query vector
4. Returns the top K most similar documents

This is called **Approximate Nearest Neighbor (ANN)** search.

---

## pgvector — start here

pgvector is a PostgreSQL extension. If you already use PostgreSQL, this is the easiest path.

### Setup

```sql
-- Enable the extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create a table with a vector column
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content TEXT NOT NULL,
  embedding VECTOR(1536),  -- 1536 for text-embedding-3-small
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Create an index for fast search
CREATE INDEX ON documents 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

### Insert a document

```typescript
const embedding = await embedText(content)

await db.query(
  `INSERT INTO documents (content, embedding, metadata) 
   VALUES ($1, $2, $3)`,
  [content, JSON.stringify(embedding), JSON.stringify({ source: "user-manual", page: 3 })]
)
```

### Search for similar documents

```typescript
async function searchSimilar(
  queryText: string,
  limit = 5,
  minScore = 0.7
): Promise<Document[]> {
  const queryEmbedding = await embedText(queryText)

  const result = await db.query(
    `SELECT 
      id,
      content,
      metadata,
      1 - (embedding <=> $1::vector) AS similarity
     FROM documents
     WHERE 1 - (embedding <=> $1::vector) > $2
     ORDER BY embedding <=> $1::vector
     LIMIT $3`,
    [JSON.stringify(queryEmbedding), minScore, limit]
  )

  return result.rows
}
```

The `<=>` operator calculates cosine distance. `1 - distance = similarity`.

---

## Qdrant — production choice

Qdrant is an open-source vector database built specifically for vector search. It is faster than pgvector for large datasets and has more features.

### Setup with Docker

```yaml
# docker-compose.yml
services:
  qdrant:
    image: qdrant/qdrant
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage
```

### TypeScript client

```bash
npm install @qdrant/js-client-rest
```

```typescript
import { QdrantClient } from "@qdrant/js-client-rest"

const qdrant = new QdrantClient({ host: "localhost", port: 6333 })

// Create a collection
await qdrant.createCollection("documents", {
  vectors: { size: 1536, distance: "Cosine" }
})

// Insert documents
await qdrant.upsert("documents", {
  points: [
    {
      id: "uuid-here",
      vector: embedding,
      payload: {
        content: "The text content here",
        source: "user-manual",
        page: 3
      }
    }
  ]
})

// Search
const results = await qdrant.search("documents", {
  vector: queryEmbedding,
  limit: 5,
  score_threshold: 0.7
})
```

---

## ChromaDB — for local development

ChromaDB is the easiest to set up locally. Good for prototyping and development.

```bash
npm install chromadb
```

```typescript
import { ChromaClient } from "chromadb"

const chroma = new ChromaClient()

const collection = await chroma.createCollection({ name: "documents" })

await collection.add({
  ids: ["doc-1", "doc-2"],
  embeddings: [embedding1, embedding2],
  documents: ["content 1", "content 2"],
  metadatas: [{ source: "manual" }, { source: "faq" }]
})

const results = await collection.query({
  queryEmbeddings: [queryEmbedding],
  nResults: 5
})
```

---

## Pinecone — fully managed

Pinecone is a cloud-hosted vector database. No infrastructure to manage.

```bash
npm install @pinecone-database/pinecone
```

```typescript
import { Pinecone } from "@pinecone-database/pinecone"

const pinecone = new Pinecone({ apiKey: process.env.PINECONE_API_KEY })
const index = pinecone.index("documents")

// Upsert
await index.upsert([{
  id: "doc-1",
  values: embedding,
  metadata: { content: "...", source: "manual" }
}])

// Query
const results = await index.query({
  vector: queryEmbedding,
  topK: 5,
  includeMetadata: true
})
```

---

## Metadata filtering

Vector search finds semantically similar documents. But sometimes you also need to filter by properties — like user ID, document category, or date.

```typescript
// Qdrant — filter by metadata while searching
const results = await qdrant.search("documents", {
  vector: queryEmbedding,
  limit: 5,
  filter: {
    must: [
      { key: "metadata.user_id", match: { value: currentUserId } },
      { key: "metadata.source", match: { value: "product-docs" } }
    ]
  }
})
```

This is important for **multi-tenant systems** where users should only see their own data.

---

## Hybrid search

Pure vector search is great for semantic meaning but misses exact keywords. Pure keyword search (BM25) is great for exact matches but misses related concepts.

**Hybrid search combines both.** It is the most reliable approach for production RAG systems.

```typescript
async function hybridSearch(query: string): Promise<Document[]> {
  // 1. Vector search — finds semantically similar documents
  const vectorResults = await qdrant.search("documents", {
    vector: await embedText(query),
    limit: 10,
    with_payload: true
  })

  // 2. Keyword search — finds exact keyword matches
  const keywordResults = await db.query(
    `SELECT id, content, ts_rank(to_tsvector(content), query) as rank
     FROM documents, to_tsquery($1) query
     WHERE to_tsvector(content) @@ query
     ORDER BY rank DESC LIMIT 10`,
    [query.split(" ").join(" & ")]
  )

  // 3. Combine and rerank results
  return mergeAndRerank(vectorResults, keywordResults)
}
```

---

## Choosing the right vector database

| Database | Best for | Cost | Setup |
|----------|---------|------|-------|
| pgvector | Teams already using PostgreSQL | Free | Easy (just an extension) |
| Qdrant | Production systems needing high performance | Free / Cloud | Medium |
| ChromaDB | Local development and prototyping | Free | Very easy |
| Pinecone | Teams wanting no infrastructure | Paid | Very easy |
| Weaviate | Hybrid search built-in | Free / Cloud | Medium |

**Recommendation:** Start with pgvector. Move to Qdrant when you need better performance. Use Pinecone if you want zero infrastructure management.

---

## Key terms

| Term | Meaning |
|------|---------|
| Vector database | A database optimized for storing and searching embeddings |
| ANN search | Approximate Nearest Neighbor — fast similarity search |
| Cosine distance | A way to measure the angle between two vectors |
| Metadata filtering | Filtering search results by document properties |
| Hybrid search | Combining vector search with keyword search |
| Collection / Index | A group of related vectors in the database |
| Top-k | The number of most similar results to return |

---

## What to do next

- Set up pgvector with Docker and store 100 document embeddings
- Build a similarity search function that returns the top 5 results
- Add metadata filtering so users only see their own documents
- Read [07 — RAG Pipeline](./07-rag-pipeline.md) to combine embeddings and vector search into a full system