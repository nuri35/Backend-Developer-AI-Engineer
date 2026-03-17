# 01 — LLM Fundamentals

**Week:** 1
**Time to learn:** 3–5 hours

---

## What is this topic?

Before you call an LLM API, you need to understand what is actually happening on the other side. You do not need to know the math. You need a working mental model — enough to debug problems, choose the right settings, and design good systems.

---

## Core concepts

### What is a Large Language Model?

A Large Language Model (LLM) is a software system trained on a huge amount of text. It learned patterns from that text. When you send it a message, it predicts what words should come next, one piece at a time.

It does not "think" the way humans do. It is a very sophisticated pattern matcher. But the patterns it learned are so complex that the outputs often look like real reasoning.

### What is a token?

An LLM does not read text character by character or word by word. It reads **tokens**.

A token is a small piece of text — usually a word, part of a word, or a punctuation mark. For example:
- "hello" → 1 token
- "backend" → 1 token
- "tokenization" → 2–3 tokens
- " " (space) → part of the next token

Why does this matter?
- The API **charges you per token** (input + output)
- The model has a **maximum number of tokens** it can process at once (the context window)
- Long inputs cost more and can hit the limit

You can count tokens before sending a request using the `tiktoken` library (OpenAI) or the Anthropic token counter.

### What is a context window?

The context window is the maximum amount of text the model can "see" at one time. Everything you send (system prompt + conversation history + user message + retrieved documents) must fit inside this window.

Examples (approximate):
- GPT-4o: 128,000 tokens (~96,000 words)
- Claude 3.5 Sonnet: 200,000 tokens
- Llama 3.1 70B: 128,000 tokens

When the context is full, the model cannot see older parts of the conversation. You need to manage this — either by summarizing old messages or by removing them.

### What is temperature?

Temperature controls how random the model's output is.

- **Temperature 0**: The model always picks the most likely next token. Output is very consistent and predictable. Use this for structured data extraction, classification, and anything where you need the same answer every time.
- **Temperature 1**: The model is more creative and varied. Use this for creative writing, brainstorming, and generating diverse options.
- **Temperature > 1**: Very random. Usually not useful for backend systems.

### What is top-p?

Top-p (also called nucleus sampling) is another way to control randomness. Instead of using temperature, the model only considers the smallest set of tokens whose total probability adds up to `p`.

- `top_p = 0.1`: Only very likely tokens are considered. Very focused output.
- `top_p = 0.9`: A wider set of tokens. More variety.

In practice, you usually only change temperature OR top-p, not both.

### What is top-k?

Top-k limits the model to only consider the `k` most likely next tokens at each step.

- `top_k = 1`: Always picks the single most likely token (deterministic)
- `top_k = 40`: Picks from the 40 most likely tokens

This is used more by open-source models (like Llama) than by the OpenAI or Anthropic APIs.

---

## Model families

### OpenAI (GPT series)
- **GPT-4o**: Fast, multimodal, good balance of cost and quality
- **GPT-4o mini**: Cheaper, faster, good for simpler tasks
- **o1, o3**: Reasoning-focused models, slower but better at complex problems
- **GPT-3.5 Turbo**: Cheap, fast, older — use GPT-4o mini instead

### Anthropic (Claude series)
- **Claude 3.5 Sonnet**: Best balance of quality, speed, and cost
- **Claude 3.5 Haiku**: Fast and cheap
- **Claude 3 Opus**: Most powerful, most expensive

### Google (Gemini series)
- **Gemini 1.5 Pro**: Long context (up to 2 million tokens), multimodal
- **Gemini 1.5 Flash**: Fast and cheap

### Open-source (Llama, Mistral)
- **Llama 3.1 70B**: Strong open-source model, can run locally
- **Mistral 7B**: Small but capable, very fast
- You can run these locally with **Ollama** — useful for development and privacy

---

## Gen AI vs Agentic AI

**Gen AI** (Generative AI): The model generates text, code, or images in response to a prompt. One call → one response. Simple.

**Agentic AI**: The model takes multiple steps to complete a task. It can call tools, search for information, make decisions, and loop until the task is done. More complex, more powerful, harder to control.

You will start with Gen AI (weeks 1–4) and move to Agentic AI (weeks 5–6).

---

## The most important mindset shift

Traditional backend code is **deterministic**:
- Same input → always same output
- You can write tests that check exact values
- Bugs are reproducible

AI systems are **probabilistic**:
- Same input → different output each time (unless temperature = 0)
- You cannot test for exact values — you test for quality and correctness
- Bugs are sometimes random and hard to reproduce

This changes how you design, test, and monitor AI systems. Instead of unit tests that check exact strings, you need evaluation systems that score quality. This is a new skill, and it is covered in [11 — Evaluation](./11-evaluation.md).

---

## Running models locally with Ollama

For development, you can run open-source models on your own machine using [Ollama](https://ollama.ai):

```bash
# Install and run Llama 3
ollama run llama3

# It exposes an OpenAI-compatible API at http://localhost:11434
```

This is useful when:
- You want to experiment without paying for API calls
- You are working with sensitive data that cannot leave your machine
- You want to test different models quickly

---

## Key terms to know

| Term | Meaning |
|------|---------|
| Token | A piece of text (word or part of a word) that the model processes |
| Context window | Maximum tokens the model can process at once |
| Temperature | How random the output is (0 = consistent, 1 = creative) |
| Top-p | Controls output randomness via probability threshold |
| Top-k | Limits token selection to the k most likely options |
| Foundation model | A large model trained on general data (GPT-4, Claude, etc.) |
| Prompt | The input text you send to the model |
| Completion | The output text the model returns |
| Hallucination | When the model confidently states something that is not true |
| LLM | Large Language Model |

---

## What to do next

1. Read the [OpenAI API documentation](https://platform.openai.com/docs) — focus on the Chat Completions section
2. Read the [Anthropic documentation](https://docs.anthropic.com) — focus on the Messages API
3. Make your first API call → see [04 — LLM API Integration](./04-llm-api-integration.md)
4. Try changing temperature between 0 and 1 on the same prompt and compare the results