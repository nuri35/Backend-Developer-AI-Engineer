# 10 — Agent Patterns

**Week:** 5–6
**Time to learn:** 6–8 hours

---

## What is this topic?

An AI agent is a system where an LLM takes multiple steps to complete a task. It reasons, decides what to do, calls tools, observes results, and continues until the task is done.

This is more complex than a simple LLM call, but it is also much more powerful.

---

## Agent vs simple LLM call

**Simple LLM call:**
```
User question → Prompt → LLM → Answer
```
One step. No actions. No feedback loop.

**AI Agent:**
```
User task → LLM thinks → Calls tool → Gets result → LLM thinks again → Calls another tool → ... → Final answer
```
Multiple steps. Real actions. Feedback loop.

---

## The agent loop

Every agent runs a loop:

```
1. OBSERVE: What is the current state? What tools are available?
2. PLAN: What should I do next?
3. ACT: Call the appropriate tool or generate a response
4. REFLECT: What was the result? Is the task done?
5. Repeat from step 1 until done
```

In code:

```typescript
async function runAgent(userTask: string): Promise<string> {
  const messages: Message[] = [
    { role: "user", content: userTask }
  ]

  let iterations = 0
  const MAX_ITERATIONS = 10  // IMPORTANT: always set a limit

  while (iterations < MAX_ITERATIONS) {
    iterations++

    const response = await anthropic.messages.create({
      model: "claude-3-5-sonnet-20241022",
      max_tokens: 2048,
      tools: availableTools,
      messages
    })

    // Add model response to history
    messages.push({ role: "assistant", content: response.content })

    // Check if done
    if (response.stop_reason === "end_turn") {
      const textBlock = response.content.find(b => b.type === "text")
      return textBlock?.text ?? "Task completed."
    }

    // Handle tool calls
    if (response.stop_reason === "tool_use") {
      const toolResults = await executeAllTools(response.content)
      messages.push({ role: "user", content: toolResults })
    }
  }

  return "Max iterations reached. Task may not be complete."
}
```

---

## Memory in agents

Agents need memory to track what has happened so far.

### Short-term memory (within one task)

This is just the messages array. Every tool call and result is added to the history. The model can see everything that happened.

```typescript
// The messages array IS the short-term memory
const messages = [
  { role: "user", content: "Research the top 3 competitors and summarize their pricing" },
  { role: "assistant", content: [toolUseBlock] },  // Model called web_search
  { role: "user", content: [toolResultBlock] },    // Search results returned
  { role: "assistant", content: [anotherToolUse] }, // Model called it again
  // ... and so on
]
```

### Long-term memory (across sessions)

For persistence between conversations, store important facts in a database:

```typescript
@Injectable()
export class AgentMemoryService {
  constructor(
    private readonly db: DatabaseService,
    private readonly embeddingService: EmbeddingService,
    private readonly vectorDb: VectorDbService
  ) {}

  async saveMemory(userId: string, fact: string): Promise<void> {
    const embedding = await this.embeddingService.embed(fact)
    await this.vectorDb.upsert("memories", {
      id: `${userId}-${Date.now()}`,
      vector: embedding,
      payload: { userId, fact, createdAt: new Date() }
    })
  }

  async recallRelevant(userId: string, context: string): Promise<string[]> {
    const queryEmbedding = await this.embeddingService.embed(context)
    const results = await this.vectorDb.search("memories", {
      vector: queryEmbedding,
      filter: { must: [{ key: "payload.userId", match: { value: userId } }] },
      limit: 5
    })
    return results.map(r => r.payload.fact)
  }
}
```

---

## Agent patterns

### ReAct (Reasoning + Acting)

The model explicitly reasons before acting. You ask it to think through the problem step by step before choosing a tool:

```typescript
const systemPrompt = `
You are a research assistant. For each task:
1. Think: What do I need to find out?
2. Act: Choose the right tool to get that information
3. Observe: What did the tool return?
4. Repeat until you have enough information
5. Answer: Give a clear, complete response

Always think out loud before acting.
`
```

### Routing pattern

Classify the user's request first, then route to the right specialized handler:

```typescript
async function routeRequest(userMessage: string): Promise<string> {
  // 1. Classify the request
  const classification = await classifyRequest(userMessage)

  // 2. Route to the right handler
  switch (classification) {
    case "billing":
      return billingAgent.handle(userMessage)
    case "technical_support":
      return technicalAgent.handle(userMessage)
    case "product_info":
      return productAgent.handle(userMessage)
    default:
      return generalAgent.handle(userMessage)
  }
}
```

### Orchestrator-Worker pattern

A main agent breaks the task into subtasks and delegates them to specialized agents:

```typescript
async function orchestratorAgent(task: string): Promise<string> {
  // 1. Break task into subtasks
  const subtasks = await planSubtasks(task)

  // 2. Execute subtasks (in parallel when possible)
  const results = await Promise.all(
    subtasks.map(subtask => workerAgent.execute(subtask))
  )

  // 3. Combine results
  return synthesizeResults(task, results)
}
```

---

## Guardrails — keeping agents safe

Agents can cause real damage if they run out of control. Always add guardrails:

### Iteration limit

```typescript
const MAX_ITERATIONS = 10
if (iterations >= MAX_ITERATIONS) {
  return "Agent stopped: max iterations reached"
}
```

### Cost ceiling

```typescript
let totalCost = 0
const COST_LIMIT = 0.50  // $0.50 max per task

function checkCost(usage: TokenUsage): void {
  totalCost += calculateCost(usage)
  if (totalCost > COST_LIMIT) {
    throw new Error(`Cost limit exceeded: $${totalCost.toFixed(4)}`)
  }
}
```

### Timeout enforcement

```typescript
const AGENT_TIMEOUT = 30000  // 30 seconds

const result = await Promise.race([
  runAgent(task),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Agent timeout")), AGENT_TIMEOUT)
  )
])
```

### Allowed actions list

```typescript
const ALLOWED_TOOLS_FOR_READONLY = ["search_products", "get_order_status", "lookup_faq"]
const ALLOWED_TOOLS_FOR_EDITOR = [...ALLOWED_TOOLS_FOR_READONLY, "update_order", "send_notification"]

function filterTools(userRole: string): Tool[] {
  const allowed = userRole === "editor" ? ALLOWED_TOOLS_FOR_EDITOR : ALLOWED_TOOLS_FOR_READONLY
  return allTools.filter(tool => allowed.includes(tool.name))
}
```

---

## Human-in-the-loop

Some actions are too important to automate completely. Add a human approval step:

```typescript
async function agentWithApproval(task: string): Promise<string> {
  const plannedActions = await planActions(task)

  // Check if any action requires approval
  const requiresApproval = plannedActions.some(action =>
    HIGH_RISK_ACTIONS.includes(action.tool)
  )

  if (requiresApproval) {
    // Store the plan and wait for human review
    const reviewId = await reviewQueue.submit({
      task,
      plannedActions,
      requestedBy: currentUser.id
    })

    return `This task requires approval. Review ID: ${reviewId}. You will be notified when approved.`
  }

  // No approval needed — execute directly
  return executeActions(plannedActions)
}
```

---

## Multi-agent systems with LangGraph

LangGraph is a framework for building stateful, multi-step agents as graphs. Each node is a step. Edges connect steps. The state flows through the graph.

```typescript
import { StateGraph, END } from "@langchain/langgraph"

// Define the state shape
interface AgentState {
  messages: Message[]
  toolResults: any[]
  finalAnswer: string | null
}

// Define nodes
async function callModel(state: AgentState): Promise<Partial<AgentState>> {
  const response = await callLlm(state.messages)
  return { messages: [...state.messages, response] }
}

async function executeTools(state: AgentState): Promise<Partial<AgentState>> {
  const toolCalls = extractToolCalls(state.messages)
  const results = await Promise.all(toolCalls.map(tc => executeTool(tc)))
  return { toolResults: results }
}

// Define the graph
const workflow = new StateGraph<AgentState>({
  channels: { messages: ..., toolResults: ..., finalAnswer: ... }
})

workflow.addNode("call_model", callModel)
workflow.addNode("execute_tools", executeTools)

workflow.addEdge("call_model", shouldCallTools ? "execute_tools" : END)
workflow.addEdge("execute_tools", "call_model")

const app = workflow.compile()
```

---

## Key terms

| Term | Meaning |
|------|---------|
| AI agent | A system where an LLM takes multiple steps using tools to complete a task |
| Agent loop | The observe-plan-act-reflect cycle the agent runs repeatedly |
| Short-term memory | The conversation history within a single task |
| Long-term memory | Stored facts that persist across sessions (in a database) |
| ReAct | A pattern where the model reasons before acting |
| Orchestrator | An agent that breaks tasks into subtasks and delegates them |
| Worker | A specialized agent that handles one type of subtask |
| Guardrails | Limits that prevent agents from running indefinitely or taking dangerous actions |
| Human-in-the-loop | A pattern where a human must approve high-risk actions |
| LangGraph | A framework for building stateful multi-step agent workflows |

---

## What to do next

- Build a basic agent that can use 3 tools to complete a research task
- Add iteration limit and cost ceiling guardrails
- Add human-in-the-loop for any action that modifies data
- Read [11 — Evaluation](./11-evaluation.md) to test your agent's reliability