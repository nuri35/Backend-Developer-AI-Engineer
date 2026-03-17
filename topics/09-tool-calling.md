# 09 — Tool Calling & MCP

**Week:** 5–6
**Time to learn:** 5–6 hours

---

## What is this topic?

Tool calling (also called function calling) is one of the most powerful features of modern LLMs. It lets you give the model access to real actions — like querying your database, sending emails, looking up prices, or calling any backend service.

Without tool calling, the model can only answer questions with text. With tool calling, it can actually *do things*.

---

## How tool calling works

1. You define tools in your API call (name, description, parameters)
2. The model reads the user's request and decides which tool to use
3. The model returns a structured response saying "call this tool with these arguments"
4. **Your code** actually calls the tool
5. You send the tool result back to the model
6. The model uses the result to generate the final response

The model never directly calls anything. Your code is always in control.

---

## Defining tools

Tools are defined with a JSON Schema that describes the name, purpose, and parameters:

```typescript
const tools: Anthropic.Tool[] = [
  {
    name: "get_order_status",
    description: "Get the current status of a customer order. Use this when a customer asks about their order.",
    input_schema: {
      type: "object",
      properties: {
        order_id: {
          type: "string",
          description: "The order ID (format: ORD-XXXXX)"
        },
        include_tracking: {
          type: "boolean",
          description: "Whether to include tracking information",
          default: false
        }
      },
      required: ["order_id"]
    }
  },
  {
    name: "search_products",
    description: "Search the product catalog by keyword or category",
    input_schema: {
      type: "object",
      properties: {
        query: { type: "string", description: "Search terms" },
        category: { type: "string", description: "Product category filter (optional)" },
        max_price: { type: "number", description: "Maximum price filter (optional)" }
      },
      required: ["query"]
    }
  }
]
```

**Key rule:** Write good descriptions. The model reads the description to decide if it should use the tool. A vague description leads to wrong tool selection.

---

## Calling tools with Anthropic

```typescript
async function runWithTools(userMessage: string): Promise<string> {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: userMessage }
  ]

  while (true) {
    const response = await anthropic.messages.create({
      model: "claude-3-5-sonnet-20241022",
      max_tokens: 1024,
      tools,
      messages
    })

    // Check if the model wants to use a tool
    if (response.stop_reason === "tool_use") {
      const toolUseBlocks = response.content.filter(
        block => block.type === "tool_use"
      ) as Anthropic.ToolUseBlock[]

      // Add model's response to history
      messages.push({ role: "assistant", content: response.content })

      // Execute each tool call
      const toolResults: Anthropic.ToolResultBlockParam[] = []

      for (const toolUse of toolUseBlocks) {
        const result = await executeTool(toolUse.name, toolUse.input)
        toolResults.push({
          type: "tool_result",
          tool_use_id: toolUse.id,
          content: JSON.stringify(result)
        })
      }

      // Add tool results to history and continue
      messages.push({ role: "user", content: toolResults })
      continue
    }

    // Model is done — return the final text response
    const textBlock = response.content.find(b => b.type === "text") as Anthropic.TextBlock
    return textBlock?.text ?? ""
  }
}
```

---

## Executing tools — your NestJS services

This is where your backend skills shine. The model asks for a tool call. You execute it using your existing NestJS services:

```typescript
async function executeTool(toolName: string, input: any): Promise<any> {
  switch (toolName) {
    case "get_order_status":
      return await orderService.getOrderStatus(input.order_id, input.include_tracking)

    case "search_products":
      return await productService.search({
        query: input.query,
        category: input.category,
        maxPrice: input.max_price
      })

    case "send_notification":
      return await notificationService.send(input.user_id, input.message)

    default:
      throw new Error(`Unknown tool: ${toolName}`)
  }
}
```

Your NestJS services are already tested and reliable. You are just exposing them to the AI.

---

## Parallel tool calling

The model can call multiple tools at the same time. This is faster than sequential calls:

```typescript
// The model might return multiple tool_use blocks at once
// Execute them all in parallel
const toolResults = await Promise.all(
  toolUseBlocks.map(async toolUse => {
    const result = await executeTool(toolUse.name, toolUse.input)
    return {
      type: "tool_result" as const,
      tool_use_id: toolUse.id,
      content: JSON.stringify(result)
    }
  })
)
```

---

## Tool error handling

Tools can fail. Tell the model about the failure so it can handle it gracefully:

```typescript
for (const toolUse of toolUseBlocks) {
  let result: any
  let isError = false

  try {
    result = await executeTool(toolUse.name, toolUse.input)
  } catch (error) {
    result = { error: error.message, success: false }
    isError = true
  }

  toolResults.push({
    type: "tool_result",
    tool_use_id: toolUse.id,
    content: JSON.stringify(result),
    is_error: isError  // Anthropic supports this flag
  })
}
```

---

## Exposing NestJS services as tools

Here is a pattern for a clean tool registry in NestJS:

```typescript
// src/ai/tools/tool-registry.ts
@Injectable()
export class ToolRegistry {
  private tools = new Map<string, ToolHandler>()

  constructor(
    private readonly orderService: OrderService,
    private readonly productService: ProductService,
    private readonly notificationService: NotificationService
  ) {
    this.register("get_order_status", async (input) => {
      return this.orderService.getStatus(input.order_id)
    })

    this.register("search_products", async (input) => {
      return this.productService.search(input)
    })

    this.register("send_notification", async (input) => {
      return this.notificationService.send(input.user_id, input.message)
    })
  }

  register(name: string, handler: ToolHandler): void {
    this.tools.set(name, handler)
  }

  async execute(name: string, input: any): Promise<any> {
    const handler = this.tools.get(name)
    if (!handler) throw new Error(`Tool not found: ${name}`)
    return handler(input)
  }

  getDefinitions(): Anthropic.Tool[] {
    // Return the tool schemas for the API call
    return TOOL_DEFINITIONS
  }
}
```

---

## Model Context Protocol (MCP)

MCP is a standard protocol that defines how AI models connect to tools and data sources. Instead of each project having its own custom tool system, MCP provides a universal standard.

Think of MCP like REST for AI tools. It is a way to define and expose capabilities in a way that any AI model (or client) can understand and use.

### Why MCP matters

- **Reusability**: Build a tool once, use it from any MCP-compatible AI client
- **Discoverability**: Clients can discover available tools automatically
- **Standardization**: No more custom integration for each model

### Building an MCP server with NestJS

```typescript
// The MCP server exposes your NestJS services
// using the standard MCP protocol over HTTP or stdio

import { Server } from "@modelcontextprotocol/sdk/server/index.js"
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js"

const server = new Server({
  name: "my-backend-tools",
  version: "1.0.0"
}, {
  capabilities: { tools: {} }
})

// Register your tools
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_order_status",
      description: "Get order status from the e-commerce backend",
      inputSchema: {
        type: "object",
        properties: {
          order_id: { type: "string" }
        },
        required: ["order_id"]
      }
    }
  ]
}))

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params

  if (name === "get_order_status") {
    const status = await orderService.getStatus(args.order_id)
    return {
      content: [{ type: "text", text: JSON.stringify(status) }]
    }
  }

  throw new Error(`Unknown tool: ${name}`)
})

// Start the server
const transport = new StdioServerTransport()
await server.connect(transport)
```

---

## Tool call security

Tools can do real things — send emails, modify data, call external APIs. You need guards:

```typescript
async function executeToolSafely(
  toolName: string,
  input: any,
  userId: string
): Promise<any> {
  // 1. Check if this user is allowed to use this tool
  const allowed = await permissionService.canUse(userId, toolName)
  if (!allowed) throw new Error(`Not authorized: ${toolName}`)

  // 2. Validate input schema before executing
  const validated = toolSchemas[toolName].parse(input)

  // 3. Log the tool call for audit
  await auditLog.record({ userId, toolName, input: validated })

  // 4. Execute with timeout
  return await Promise.race([
    toolRegistry.execute(toolName, validated),
    timeout(10000)  // 10 second limit
  ])
}
```

---

## Key terms

| Term | Meaning |
|------|---------|
| Tool calling | Giving an LLM the ability to call external functions or services |
| Function calling | Same as tool calling (OpenAI's older name for it) |
| Tool schema | JSON Schema that describes a tool's name, purpose, and parameters |
| Tool result | The output of executing a tool, sent back to the model |
| Parallel tool calling | The model calling multiple tools at the same time |
| MCP | Model Context Protocol — a standard for AI tool connections |
| Tool registry | A service that manages available tools and executes them |

---

## What to do next

- Define 3 tools based on your existing NestJS services
- Build a simple agent loop that can call tools and use the results
- Add error handling and permission checks
- Read [10 — Agent Patterns](./10-agent-patterns.md) to build more complex tool-using systems