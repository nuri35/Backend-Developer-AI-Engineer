# 08 — Streaming

**Week:** 8
**Time to learn:** 3–4 hours

---

## What is this topic?

LLMs generate text token by token. Without streaming, your user waits 3–10 seconds for a response with no feedback. With streaming, the text appears word by word as it is generated — much better experience.

As a backend developer, you need to understand how to proxy this stream through your server to the client.

---

## How LLM streaming works

The flow looks like this:

```
LLM API → your NestJS server → your frontend client
```

The LLM sends small chunks of text as they are generated. Your server receives these chunks and forwards them to the client. The client displays each chunk as it arrives.

The standard transport protocol for this is **Server-Sent Events (SSE)**.

---

## Server-Sent Events (SSE)

SSE is a one-way connection from server to client over HTTP. The server sends a stream of text events. The client listens and updates the UI.

SSE is simpler than WebSockets for this use case because:
- It is one-way (server → client)
- It works over regular HTTP
- Browsers support it natively
- It handles reconnection automatically

### SSE message format

```
data: {"type": "text_delta", "text": "Hello"}

data: {"type": "text_delta", "text": " world"}

data: {"type": "done"}

```

Each message starts with `data:`, followed by a newline, followed by another newline to end the event.

---

## Streaming in NestJS

### The streaming endpoint

```typescript
// src/ai/ai.controller.ts
import { Controller, Post, Body, Res } from "@nestjs/common"
import { Response } from "express"
import { AiService } from "./ai.service"

@Controller("ai")
export class AiController {
  constructor(private readonly aiService: AiService) {}

  @Post("chat/stream")
  async streamChat(
    @Body() body: { message: string },
    @Res() res: Response
  ) {
    // Set SSE headers
    res.setHeader("Content-Type", "text/event-stream")
    res.setHeader("Cache-Control", "no-cache")
    res.setHeader("Connection", "keep-alive")
    res.setHeader("Access-Control-Allow-Origin", "*")
    res.flushHeaders()

    try {
      await this.aiService.streamMessage(body.message, (chunk) => {
        // Send each chunk to the client
        res.write(`data: ${JSON.stringify({ text: chunk })}\n\n`)
      })

      // Signal completion
      res.write(`data: ${JSON.stringify({ done: true })}\n\n`)
    } catch (error) {
      res.write(`data: ${JSON.stringify({ error: error.message })}\n\n`)
    } finally {
      res.end()
    }
  }
}
```

### The streaming service

```typescript
// src/ai/ai.service.ts
import Anthropic from "@anthropic-ai/sdk"

@Injectable()
export class AiService {
  private readonly anthropic: Anthropic

  constructor() {
    this.anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY })
  }

  async streamMessage(
    userMessage: string,
    onChunk: (text: string) => void
  ): Promise<void> {
    const stream = this.anthropic.messages.stream({
      model: "claude-3-5-sonnet-20241022",
      max_tokens: 1024,
      messages: [{ role: "user", content: userMessage }]
    })

    // Process each chunk as it arrives
    for await (const event of stream) {
      if (
        event.type === "content_block_delta" &&
        event.delta.type === "text_delta"
      ) {
        onChunk(event.delta.text)
      }
    }
  }
}
```

---

## OpenAI streaming

```typescript
async streamWithOpenAI(
  messages: any[],
  onChunk: (text: string) => void
): Promise<void> {
  const stream = await openai.chat.completions.create({
    model: "gpt-4o",
    messages,
    stream: true
  })

  for await (const chunk of stream) {
    const text = chunk.choices[0]?.delta?.content
    if (text) onChunk(text)
  }
}
```

---

## Stream cancellation

Users sometimes want to stop a long response. You need to handle this:

```typescript
@Post("chat/stream")
async streamChat(
  @Body() body: { message: string },
  @Res() res: Response,
  @Req() req: Request
) {
  res.setHeader("Content-Type", "text/event-stream")
  res.flushHeaders()

  const abortController = new AbortController()

  // Cancel when client disconnects
  req.on("close", () => {
    abortController.abort()
  })

  try {
    const stream = this.anthropic.messages.stream({
      model: "claude-3-5-sonnet-20241022",
      max_tokens: 1024,
      messages: [{ role: "user", content: body.message }]
    }, { signal: abortController.signal })

    for await (const event of stream) {
      if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
        res.write(`data: ${JSON.stringify({ text: event.delta.text })}\n\n`)
      }
    }
  } catch (error) {
    if (error.name !== "AbortError") {
      res.write(`data: ${JSON.stringify({ error: "Stream failed" })}\n\n`)
    }
  } finally {
    res.end()
  }
}
```

---

## Streaming with tool calls

When the model uses tools (function calling), the stream includes both text and tool call deltas. You need to handle both:

```typescript
let toolCallBuffer = ""
let currentToolName = ""

for await (const event of stream) {
  if (event.type === "content_block_start") {
    if (event.content_block.type === "tool_use") {
      currentToolName = event.content_block.name
      toolCallBuffer = ""
    }
  }

  if (event.type === "content_block_delta") {
    if (event.delta.type === "text_delta") {
      // Regular text — send to client
      onChunk(event.delta.text)
    } else if (event.delta.type === "input_json_delta") {
      // Tool call argument being built
      toolCallBuffer += event.delta.partial_json
    }
  }

  if (event.type === "content_block_stop" && currentToolName) {
    // Tool call is complete, execute it
    const toolInput = JSON.parse(toolCallBuffer)
    const toolResult = await executeTool(currentToolName, toolInput)
    // ... handle tool result
    currentToolName = ""
  }
}
```

---

## Timeout configuration

Streaming connections are long-lived. You need to configure your infrastructure to allow this.

### Nginx

```nginx
location /ai/chat/stream {
  proxy_pass http://localhost:3000;
  proxy_read_timeout 300s;    # 5 minutes
  proxy_send_timeout 300s;
  proxy_buffering off;        # CRITICAL: disable buffering for SSE
  proxy_cache off;
  proxy_set_header Connection "";
  proxy_http_version 1.1;
}
```

### AWS API Gateway

AWS API Gateway has a 29-second timeout limit. For streaming, you need to either:
- Use AWS Lambda with response streaming
- Use an ALB (Application Load Balancer) instead of API Gateway
- Use WebSockets via API Gateway

This is a common gotcha — plan for it early.

---

## Frontend consumption (for reference)

```typescript
// React example
const streamMessage = async (message: string) => {
  const response = await fetch("/api/ai/chat/stream", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message })
  })

  const reader = response.body.getReader()
  const decoder = new TextDecoder()

  while (true) {
    const { done, value } = await reader.read()
    if (done) break

    const text = decoder.decode(value)
    const lines = text.split("\n\n").filter(Boolean)

    for (const line of lines) {
      if (line.startsWith("data: ")) {
        const data = JSON.parse(line.slice(6))
        if (data.text) {
          setCurrentResponse(prev => prev + data.text)
        }
      }
    }
  }
}
```

---

## Key terms

| Term | Meaning |
|------|---------|
| Streaming | Sending data piece by piece as it is generated, rather than all at once |
| SSE (Server-Sent Events) | A protocol for one-way server-to-client streaming over HTTP |
| Text delta | A small piece of the model's response as it is being generated |
| Abort controller | A standard way to cancel an ongoing async operation |
| Proxy buffering | When a proxy server collects the full response before sending it — disable this for SSE |
| Time to first token | How long it takes for the first character to appear in the stream |

---

## What to do next

- Build a streaming chat endpoint in NestJS
- Test it with a simple HTML page using EventSource
- Add cancellation when the client disconnects
- Read [13 — Production Patterns](./13-production-patterns.md) for rate limiting and reliability