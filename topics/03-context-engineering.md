# 03 — Context Engineering

**Week:** 2
**Time to learn:** 2–3 hours

---

## What is this topic?

Context engineering is a newer and more advanced skill than prompt engineering. While prompt engineering focuses on writing good instructions, context engineering focuses on **managing everything that goes into the context window**.

The context window is limited. What you put in it directly affects the quality of the output, the speed of the response, and how much you pay.

> "Context engineering is the practice of deciding what information to include in the context window, how to structure it, and how to keep it within limits — so the model always has exactly what it needs."

---

## Why it matters

A model can only work with what it can see. If you put too little context in, the model lacks information and gives wrong answers. If you put too much, you:

- Hit the context window limit
- Make the model confused by too much irrelevant information
- Pay more for input tokens
- Get slower responses

The goal is: **the right information, at the right time, in the right amount.**

---

## What goes into the context window?

Every token you send to the model uses up context space. This includes:

| Part | Description |
|------|-------------|
| System prompt | Your instructions and persona definition |
| Conversation history | Previous messages in the current session |
| Retrieved documents | Text chunks from RAG (week 4) |
| Tool results | Outputs from tool calls the model made |
| User message | The current input from the user |

All of these compete for space. You need to manage all of them.

---

## The three main context engineering problems

### Problem 1: Conversation history grows forever

In a chatbot, every message you add to the history uses more tokens. After 20–30 exchanges, you may hit the limit.

**Solutions:**

**Summarization:** After a certain number of messages, ask the model to summarize the conversation. Replace the old messages with the summary.

```typescript
async function summarizeHistory(messages: Message[]): Promise<string> {
  const response = await anthropic.messages.create({
    model: "claude-3-5-haiku-20241022",
    max_tokens: 500,
    messages: [{
      role: "user",
      content: `Summarize this conversation in 3-5 sentences, keeping the key facts:
      
${messages.map(m => `${m.role}: ${m.content}`).join('\n')}`
    }]
  })
  return response.content[0].text
}
```

**Sliding window:** Keep only the last N messages. Older messages are dropped.

**Selective history:** Keep only the messages that are relevant to the current question (using semantic similarity — requires embeddings from week 3).

### Problem 2: Retrieved documents are too long

When you use RAG (week 4), you retrieve chunks of text from your database. These chunks need to fit in the context window along with everything else.

**Solutions:**

- Chunk documents into smaller pieces during ingestion (covered in RAG)
- Retrieve only the top 3–5 most relevant chunks, not 20
- Use reranking to pick the best chunks after retrieval
- Compress retrieved text: summarize each chunk before adding it to the context

### Problem 3: Too much irrelevant information

Every irrelevant sentence in the context window is a distraction. The model may focus on the wrong things.

**Solutions:**

- Be selective: only include what is necessary for this specific request
- Use metadata filters when retrieving from a vector database (only get documents from the right category, time range, or author)
- Pre-process documents to remove boilerplate (headers, footers, legal disclaimers)

---

## Context compression techniques

### Prompt caching

Both OpenAI and Anthropic support prompt caching. If you send the same system prompt in many requests, the API can cache it and charge you less for the cached part.

```typescript
// Anthropic prompt caching
const response = await anthropic.messages.create({
  model: "claude-3-5-sonnet-20241022",
  max_tokens: 1024,
  system: [
    {
      type: "text",
      text: "You are a helpful assistant...",
      cache_control: { type: "ephemeral" }  // cache this
    }
  ],
  messages: [...]
})
```

This is especially useful when your system prompt is long (500+ tokens) and used in many requests.

### Dynamic context selection

Instead of always including the same context, select what to include based on the current request:

```typescript
function buildContext(userMessage: string, availableContext: ContextItem[]): string {
  // Only include context items relevant to this specific message
  const relevant = availableContext.filter(item => 
    isRelevant(item, userMessage)
  )
  return relevant.slice(0, 5).map(item => item.content).join('\n\n')
}
```

---

## Model Context Protocol (MCP)

MCP is a standard protocol that defines how AI models connect to external tools and data sources. Instead of each developer building custom integrations, MCP provides a standard way to:

- Define what tools are available
- Describe what each tool does
- Send and receive tool calls

Think of MCP as a standard API design pattern for AI tools — similar to how REST is a standard for HTTP APIs.

**Why this matters for backend developers:**

You can build an MCP server that exposes your NestJS services to any AI model that supports the protocol. This is covered in detail in [09 — Tool Calling & MCP](./09-tool-calling.md).

---

## Context window management in practice

Here is a simple priority system for managing context:

```
HIGH PRIORITY (always include):
- System prompt (instructions)
- Current user message
- Most recent 2-3 conversation turns

MEDIUM PRIORITY (include if space allows):
- Retrieved relevant documents (top 3)
- Tool call results from this conversation

LOW PRIORITY (include only if specifically needed):
- Older conversation history (summarized)
- Background information
```

When you are close to the token limit, remove items from the lowest priority first.

---

## Token counting before sending

Always count tokens before sending to avoid hitting limits and to control costs:

```typescript
import Anthropic from "@anthropic-ai/sdk"

const anthropic = new Anthropic()

async function countTokens(messages: any[]): Promise<number> {
  const response = await anthropic.messages.countTokens({
    model: "claude-3-5-sonnet-20241022",
    messages
  })
  return response.input_tokens
}
```

---

## Key terms

| Term | Meaning |
|------|---------|
| Context window | The maximum amount of text the model can process at once |
| Context engineering | Managing what goes into the context window |
| Prompt caching | Reusing cached tokens to reduce cost and latency |
| Summarization | Compressing conversation history to save tokens |
| MCP | Model Context Protocol — standard for AI tool connections |
| Sliding window | Keeping only the most recent N messages in history |

---

## What to do next

- Add conversation summarization to your chatbot when it reaches 20 messages
- Implement token counting in your LLM service wrapper
- Read [07 — RAG Pipeline](./07-rag-pipeline.md) to see how context engineering applies to document retrieval
- Read [09 — Tool Calling & MCP](./09-tool-calling.md) for MCP server implementation