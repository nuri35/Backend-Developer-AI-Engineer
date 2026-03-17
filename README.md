# Backend Developer → AI Engineer

![No Python Required](https://img.shields.io/badge/No_Python_Required-success?style=flat)
![No ML Training](https://img.shields.io/badge/No_ML_Training-important?style=flat)
![TypeScript](https://img.shields.io/badge/Node.js-TypeScript-3178c6?style=flat&logo=typescript)
![NestJS](https://img.shields.io/badge/NestJS-focused-E0234E?style=flat&logo=nestjs)
![PRs Welcome](https://img.shields.io/badge/PRs-Welcome-brightgreen?style=flat)
![License MIT](https://img.shields.io/badge/license-MIT-blue?style=flat)

> **"An AI engineer is first and foremost an engineer. The AI part comes second."**
> — Alexey Grigorev (based on analysis of 1,765 real AI job postings)

A practical, opinionated roadmap for **backend developers** who want to integrate AI into their systems — using ready-made models like Claude or GPT-4, **without training ML models from scratch** and **without switching to Python**.

---

## Who is this for?

This roadmap is for you if:

- You are a backend developer with Node.js, NestJS, or TypeScript experience
- You want to add AI features to your backend systems
- You do **not** want to become a Machine Learning Engineer
- You want to use existing LLMs (Claude, GPT-4, Gemini) as services in your architecture
- You already understand APIs, databases, queues, and async patterns

You do **not** need:
- Python knowledge
- Math or statistics background
- Experience with model training or fine-tuning

---

## What you will learn

| # | Topic | Why it matters |
|---|-------|----------------|
| 01 | [LLM Fundamentals](./topics/01-llm-fundamentals.md) | Understand how the AI you are calling actually works |
| 02 | [Prompt Engineering](./topics/02-prompt-engineering.md) | Write instructions that give consistent, reliable outputs |
| 03 | [Context Engineering](./topics/03-context-engineering.md) | Manage what goes into the context window efficiently |
| 04 | [LLM API Integration](./topics/04-llm-api-integration.md) | Call OpenAI and Anthropic APIs properly in TypeScript |
| 05 | [Embeddings](./topics/05-embeddings.md) | Turn text into vectors for semantic search |
| 06 | [Vector Databases](./topics/06-vector-databases.md) | Store and search embeddings at scale |
| 07 | [RAG Pipeline](./topics/07-rag-pipeline.md) | Connect LLMs to your own data |
| 08 | [Streaming](./topics/08-streaming.md) | Stream LLM responses to users in real time |
| 09 | [Tool Calling & MCP](./topics/09-tool-calling.md) | Let the AI use your backend services as tools |
| 10 | [Agent Patterns](./topics/10-agent-patterns.md) | Build systems where AI takes multi-step actions |
| 11 | [Evaluation](./topics/11-evaluation.md) | Measure and improve AI output quality |
| 12 | [Observability](./topics/12-observability.md) | Monitor traces, costs, and latency in production |
| 13 | [Production Patterns](./topics/13-production-patterns.md) | Make AI systems reliable, scalable, and safe |
| 14 | [Cost Management](./topics/14-cost-management.md) | Control and optimize LLM API spending |

---

## Learning path (8 weeks)

```
Week 1 → LLM Fundamentals + First API call
Week 2 → Prompt Engineering + Context Engineering
Week 3 → Embeddings + Vector Databases
Week 4 → RAG Pipeline
Week 5 → Tool Calling + Agent Patterns
Week 6 → LangGraph + Multi-Agent + MCP
Week 7 → Evaluation + Observability
Week 8 → Production Patterns + Cost Management
```

Full weekly plan with resources → [roadmap.md](./roadmap.md)

---

## The 6-Layer AI Backend Framework

When you add AI to a backend system, think of it as new layers on top of what you already know:

```
Layer 6: Defense           → guardrails, prompt injection, security
Layer 5: AI Systems        → agents, multi-agent, human-in-the-loop
Layer 4: AI Infrastructure → vector DB, embeddings, RAG  ← YOU ADD THIS
─────────────────────────────────────────────────────────
Layer 3: Production        → retry, circuit breaker, rate limiting (you know this)
Layer 2: Business Logic    → services, validation (you know this)
Layer 1: Foundation        → APIs, auth, databases (you know this)
```

You already have layers 1–3. This roadmap teaches you layers 4–6.

---

## Your advantage as a backend developer

Because you already know backend engineering, you understand:

- REST API design and microservice architecture
- PostgreSQL, Redis, and async queue systems (RabbitMQ)
- Error handling, retry logic, and circuit breakers
- Docker, CI/CD, and production deployment
- Observability and monitoring

Most AI engineers who come from a research background struggle with production engineering. You already have this. You just need to add the AI layer on top.

---

## Repository structure

```
backend-to-ai-engineer/
├── README.md
├── roadmap.md
├── topics/
│   ├── 01-llm-fundamentals.md
│   ├── 02-prompt-engineering.md
│   ├── 03-context-engineering.md
│   ├── 04-llm-api-integration.md
│   ├── 05-embeddings.md
│   ├── 06-vector-databases.md
│   ├── 07-rag-pipeline.md
│   ├── 08-streaming.md
│   ├── 09-tool-calling.md
│   ├── 10-agent-patterns.md
│   ├── 11-evaluation.md
│   ├── 12-observability.md
│   ├── 13-production-patterns.md
│   └── 14-cost-management.md
├── resources.md
├── projects.md
└── CONTRIBUTING.md
```

---

## Sources

Built by combining research from:

- [alexeygrigorev/ai-engineering-field-guide](https://github.com/alexeygrigorev/ai-engineering-field-guide) — 1,765 real AI job postings
- [daveebbelaar/ai-cookbook](https://github.com/daveebbelaar/ai-cookbook) — AI Engineer Roadmap 2026
- [Codebasics — AI Engineering Fast Track for SWEs 2026](https://codebasics.io)
- [roadmap.sh/ai-engineer](https://roadmap.sh/ai-engineer)
- [zenvanriel.com — Backend Developer to AI Engineer](https://zenvanriel.com/learning-path/backend-developer-to-ai-engineer/)
- [leverage.to — AI Engineering Patterns](https://www.leverage.to/learn/dev/ai_engineering_patterns)
- [scrimba.com — The AI Engineer Path](https://scrimba.com/the-ai-engineer-path-c02v)

---

## Timeline

> You can go from backend developer to productive AI engineer in **2–3 months**.
> You already have all the engineering skills. You just need to learn how they apply to AI.

---

## License

MIT — free to use, share, and build upon.