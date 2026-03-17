# Full Roadmap — 8 Weeks

This is the complete weekly learning plan. Each week has a clear topic focus, a list of things to learn, and a small practical goal.

> **How to use this:** Follow the weeks in order. Each topic builds on the previous one. Do not skip ahead. The practical goal at the end of each week is more important than reading everything.

---

## Week 1 — LLM Fundamentals + First API Call

**Goal:** Understand how LLMs work at a basic level and make your first real API call.

### Learn

- What a token is and how tokenization works
- What a context window is and why it has limits
- What temperature, top-p, and top-k do to the output
- The difference between GPT, Claude, Gemini, and Llama model families
- The difference between deterministic software and probabilistic AI (this is the most important mindset shift)
- What Gen AI is vs what Agentic AI is
- How to read the OpenAI and Anthropic API documentation

### Practical goal

Write a TypeScript function that calls the Anthropic or OpenAI API and returns a response. Try changing the temperature and see how the output changes.

### Read

- [01 — LLM Fundamentals](./topics/01-llm-fundamentals.md)

---

## Week 2 — Prompt Engineering + Context Engineering

**Goal:** Write prompts that produce reliable, consistent outputs.

### Learn

- The difference between a system prompt and a user prompt
- Zero-shot and few-shot prompting
- Chain-of-thought: asking the model to think step by step
- How to control output format using JSON mode, XML tags, and delimiters
- Prompt templates with variables
- How to version and test prompts
- What prompt injection is and why it is dangerous
- What context engineering is and why it matters in 2025-2026
- How to decide what goes into the context window

### Practical goal

Build a text summarizer that always returns a JSON object with `{ summary, key_points, tone }`. Make it work reliably every time.

### Read

- [02 — Prompt Engineering](./topics/02-prompt-engineering.md)
- [03 — Context Engineering](./topics/03-context-engineering.md)

---

## Week 3 — Embeddings + Vector Databases

**Goal:** Understand how text becomes searchable vectors and set up your first vector database.

### Learn

- What an embedding is (text turned into a list of numbers that represents meaning)
- How OpenAI and open-source embedding models work
- What cosine similarity is and why it measures semantic closeness
- How batch embedding works for large datasets
- What a vector database is and why it is different from a regular database
- How pgvector works with PostgreSQL (start here — you already know Postgres)
- What Qdrant, ChromaDB, and Pinecone offer and when to use them
- What metadata filtering is and how it combines with vector search
- What hybrid search is (vector + keyword together)

### Practical goal

Take 20 text documents. Embed them. Store them in pgvector or Qdrant. Write a query that returns the 3 most semantically similar documents to a given question.

### Read

- [05 — Embeddings](./topics/05-embeddings.md)
- [06 — Vector Databases](./topics/06-vector-databases.md)

---

## Week 4 — RAG Pipeline

**Goal:** Build a system that answers questions using your own data.

### Learn

- What RAG is and what problems it solves (hallucination, knowledge cutoff)
- How to load and parse documents (PDF, HTML, Markdown) using tools like Docling
- What chunking is and which strategy to use (fixed, semantic, overlap)
- How to embed chunks and store them in a vector database
- How to retrieve relevant chunks when a user asks a question
- How to combine retrieved context with a prompt to generate an answer
- How to cite sources in the response
- How to handle cases where the answer is not in the data
- Advanced patterns: HyDE, RAG-Fusion, query rewriting

### Practical goal

Build a question-answer system over your own notes or a PDF document. Ask it 10 questions and check if the answers are correct.

### Read

- [07 — RAG Pipeline](./topics/07-rag-pipeline.md)

---

## Week 5 — Tool Calling + Agent Patterns

**Goal:** Give the AI the ability to call your backend services and take actions.

### Learn

- What tool calling is and how to define a tool schema in JSON Schema format
- How the model decides which tool to use
- How to pass tool results back to the model
- How parallel tool calling works
- How to handle tool errors gracefully
- How to expose your NestJS services as AI tools
- What an AI agent is (different from a simple LLM call)
- The agent loop: observe → plan → act → reflect
- Short-term and long-term memory in agents
- How to set stopping conditions and loop limits
- The ReAct pattern (reasoning + acting)
- The routing pattern (classify input, send to the right handler)

### Practical goal

Build a simple agent that has access to 2–3 tools (for example: search the database, send an email, look up a price). Give it a task and let it decide which tools to use and in what order.

### Read

- [09 — Tool Calling & MCP](./topics/09-tool-calling.md)
- [10 — Agent Patterns](./topics/10-agent-patterns.md)

---

## Week 6 — LangGraph + Multi-Agent + MCP

**Goal:** Build stateful, multi-step AI workflows and expose your services via MCP.

### Learn

- What LangGraph is and why it is useful for agents
- State, nodes, edges, and conditional routing in LangGraph
- How to orchestrate multi-step AI workflows
- How to add memory and human-in-the-loop to a LangGraph agent
- What a multi-agent system is
- How agents communicate with each other
- What Model Context Protocol (MCP) is and why it matters
- How to build an MCP server with NestJS
- How to connect MCP clients to your server
- How context engineering works inside LangGraph

### Practical goal

Build a multi-agent system with LangGraph: one orchestrator agent that delegates tasks to two specialized sub-agents. Add a human approval step for any action that costs money or modifies data.

### Read

- [09 — Tool Calling & MCP](./topics/09-tool-calling.md) (MCP section)
- [10 — Agent Patterns](./topics/10-agent-patterns.md) (multi-agent section)

---

## Week 7 — Evaluation + Observability

**Goal:** Measure whether your AI system is actually working correctly and monitor it in production.

### Learn

**Evaluation:**
- Why evaluation is the most important new skill for backend engineers entering AI
- How to create a golden dataset (test questions + expected answers)
- What LLM-as-a-Judge is (one model scores another model's output)
- How to detect hallucinations automatically
- How to measure faithfulness (does the answer match the source?)
- How to use the RAGAs framework
- How to add evaluation to your CI/CD pipeline

**Observability:**
- How to trace every LLM call from prompt to response
- How to use Langfuse (open-source, self-hostable — good choice)
- How to use LangSmith (if you use LangChain)
- How to connect LLM tracing to your existing OpenTelemetry setup
- How to track token usage and cost per request
- How to monitor time-to-first-token latency
- How to detect when output quality is getting worse over time

### Practical goal

Add Langfuse tracing to the RAG system you built in week 4. Create a test set of 10 questions and run them automatically to check quality. Set up an alert if the average faithfulness score drops below 0.8.

### Read

- [11 — Evaluation](./topics/11-evaluation.md)
- [12 — Observability](./topics/12-observability.md)

---

## Week 8 — Production Patterns + Cost Management

**Goal:** Make your AI system production-ready: reliable, scalable, and cost-controlled.

### Learn

**Streaming:**
- How Server-Sent Events (SSE) work
- How to proxy a stream from the LLM through your NestJS server to the client
- How to parse stream chunks and handle cancellation

**Production Patterns:**
- The AI Gateway pattern (all AI calls go through one central service)
- Response caching and semantic caching with Redis
- Rate limiting for both LLM provider limits and your own users
- Using RabbitMQ for long-running AI jobs (you already know this)
- Circuit breaker for LLM API failures
- Fallback strategies when AI is unavailable

**Cost Management:**
- How token-based pricing works (input vs output vs cached tokens)
- How to track cost per user and per feature
- How to set budget alerts and spending limits
- How to route cheap tasks to smaller models
- How to use prompt caching to reduce costs

### Practical goal

Add an AI Gateway service to your project. All LLM calls go through it. It handles rate limiting, caching, cost tracking, and fallback. Add a budget alert that stops AI calls if a user exceeds $5 in a day.

### Read

- [08 — Streaming](./topics/08-streaming.md)
- [13 — Production Patterns](./topics/13-production-patterns.md)
- [14 — Cost Management](./topics/14-cost-management.md)

---

## After week 8

By this point you have:

- Built a working RAG system
- Built an AI agent with tool calling
- Added evaluation and monitoring
- Made everything production-ready

Next steps:
- Add AI to a real project (like your existing backend platform)
- Build an MCP server that exposes your services
- Start contributing to open-source AI projects
- Share what you built on LinkedIn and GitHub

 
 