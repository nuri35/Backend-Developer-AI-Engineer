# 11 — Evaluation

**Week:** 7
**Time to learn:** 4–5 hours

---

## What is this topic?

Evaluation is how you measure whether your AI system is actually working correctly. It is one of the most important new skills for backend developers entering AI.

> "Evaluation is the most important new skill. Most engineers struggle here initially." — Alexey Grigorev

In traditional software, you test with exact values:
```typescript
expect(add(2, 3)).toBe(5)  // exact match
```

In AI systems, the output is probabilistic and variable. You cannot test exact strings. You need to test *quality*:
```typescript
expect(faithfulness).toBeGreaterThan(0.8)   // quality score
expect(hallucinations).toBe(0)
```

---

## Why evaluation is different in AI systems

| Traditional software | AI systems |
|---------------------|-----------|
| Same input → same output | Same input → different output each time |
| Test exact values | Test quality scores |
| Bugs are reproducible | Some failures are random |
| Pass/fail tests | Grade-based evaluation (0.0–1.0) |
| Unit tests catch regressions | Eval suites catch quality regressions |

---

## The golden dataset

A golden dataset is a curated set of test questions with expected answers. It is the foundation of evaluation.

```typescript
interface GoldenExample {
  id: string
  question: string
  expectedAnswer: string        // What the correct answer should be
  expectedSources?: string[]    // Which documents should be retrieved (for RAG)
  difficulty: "easy" | "medium" | "hard"
  category: string
}

const goldenDataset: GoldenExample[] = [
  {
    id: "test-001",
    question: "What is the return policy for electronics?",
    expectedAnswer: "Electronics can be returned within 30 days with original packaging",
    expectedSources: ["return-policy.pdf"],
    difficulty: "easy",
    category: "policies"
  },
  // ... more examples
]
```

**How to create a good golden dataset:**
- Start with 20–50 examples covering different topics and difficulties
- Include edge cases (questions with no answer in your data, ambiguous questions)
- Have domain experts verify the expected answers
- Add new examples whenever you find a real failure in production

---

## LLM-as-a-Judge

Instead of writing rules to check answers, use an LLM to score the quality:

```typescript
async function evaluateAnswer(
  question: string,
  context: string,
  generatedAnswer: string,
  expectedAnswer: string
): Promise<EvaluationResult> {
  const evalPrompt = `
You are an objective evaluator. Score the following response on these criteria.

QUESTION: ${question}

CONTEXT (retrieved documents):
${context}

GENERATED ANSWER: ${generatedAnswer}

EXPECTED ANSWER: ${expectedAnswer}

Score each criterion from 0.0 to 1.0:

1. FAITHFULNESS: Is the answer fully supported by the context? (0 = made up, 1 = fully grounded)
2. RELEVANCE: Does the answer actually answer the question? (0 = off-topic, 1 = directly answers)
3. CORRECTNESS: Is the answer factually correct compared to expected? (0 = wrong, 1 = correct)
4. COMPLETENESS: Does the answer cover all important points? (0 = missing key info, 1 = complete)

Return only valid JSON:
{
  "faithfulness": 0.0,
  "relevance": 0.0,
  "correctness": 0.0,
  "completeness": 0.0,
  "reasoning": "Brief explanation of the scores"
}
  `

  const response = await anthropic.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 500,
    messages: [{ role: "user", content: evalPrompt }]
  })

  return JSON.parse(response.content[0].text)
}
```

---

## Running evaluation on your RAG system

```typescript
async function runEvaluation(system: RagSystem): Promise<EvalReport> {
  const results: EvalResult[] = []

  for (const example of goldenDataset) {
    // Run the system
    const { answer, retrievedChunks } = await system.query(example.question)

    // Evaluate the result
    const context = retrievedChunks.map(c => c.content).join("\n\n")
    const scores = await evaluateAnswer(
      example.question,
      context,
      answer,
      example.expectedAnswer
    )

    results.push({
      id: example.id,
      question: example.question,
      answer,
      scores
    })

    // Small delay to avoid rate limits
    await sleep(500)
  }

  return {
    totalExamples: results.length,
    averageFaithfulness: average(results.map(r => r.scores.faithfulness)),
    averageRelevance: average(results.map(r => r.scores.relevance)),
    averageCorrectness: average(results.map(r => r.scores.correctness)),
    failedExamples: results.filter(r => r.scores.faithfulness < 0.7),
    results
  }
}
```

---

## Hallucination detection

Hallucination is when the model states something confidently that is not true. In RAG systems, you can detect it by checking if the answer is supported by the retrieved context:

```typescript
async function detectHallucination(
  answer: string,
  context: string
): Promise<{ isHallucination: boolean; confidence: number }> {
  const response = await anthropic.messages.create({
    model: "claude-3-5-sonnet-20241022",
    max_tokens: 200,
    messages: [{
      role: "user",
      content: `Does this answer contain any claims that are NOT supported by the context?

Context: ${context}

Answer: ${answer}

Return JSON: {"isHallucination": true/false, "confidence": 0.0-1.0, "problematicClaims": []}`
    }]
  })

  return JSON.parse(response.content[0].text)
}
```

---

## RAGAs framework

RAGAs is an open-source framework specifically for evaluating RAG pipelines. It automates many of the metrics:

```bash
pip install ragas  # Python library — worth knowing even if you use Node.js
```

Key RAGAs metrics:
- **Faithfulness**: Is the answer grounded in the retrieved context?
- **Answer Relevance**: Does the answer address the question?
- **Context Precision**: Are the retrieved chunks actually relevant?
- **Context Recall**: Were all the relevant chunks retrieved?

---

## Regression testing

When you change your prompts, models, or chunking strategy, run your evaluation suite to check if quality went up or down:

```typescript
// Run before making changes
const baselineReport = await runEvaluation(currentSystem)
saveReport("baseline", baselineReport)

// Make your changes...

// Run after making changes
const newReport = await runEvaluation(updatedSystem)

// Compare
const regression = compareReports(baselineReport, newReport)
if (regression.faithfulnessDrop > 0.05) {
  console.error("REGRESSION: Faithfulness dropped by", regression.faithfulnessDrop)
  process.exit(1)
}
```

---

## Adding evaluation to CI/CD

```yaml
# .github/workflows/eval.yml
name: AI Evaluation

on:
  pull_request:
    paths:
      - "src/ai/**"
      - "prompts/**"

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run evaluation suite
        run: npx ts-node scripts/run-eval.ts

      - name: Check quality thresholds
        run: |
          if [ $(cat eval-results.json | jq '.averageFaithfulness') -lt 0.8 ]; then
            echo "Faithfulness below threshold"
            exit 1
          fi
```

---

## Production monitoring

Beyond offline testing, monitor quality in production:

```typescript
// After every real user interaction, log quality signals
async function logProductionQuality(
  question: string,
  answer: string,
  retrievedChunks: Chunk[],
  userId: string
): Promise<void> {
  // Run lightweight evaluation asynchronously (don't block the response)
  setImmediate(async () => {
    const scores = await quickEvaluate(question, answer, retrievedChunks)

    await metricsService.record({
      metric: "ai.rag.quality",
      tags: { userId },
      values: {
        faithfulness: scores.faithfulness,
        relevance: scores.relevance
      }
    })

    // Alert if quality drops
    if (scores.faithfulness < 0.6) {
      await alertService.send("Low faithfulness detected", { question, answer, scores })
    }
  })
}
```

---

## Key terms

| Term | Meaning |
|------|---------|
| Evaluation | Measuring the quality of AI system outputs |
| Golden dataset | A curated set of test questions with known correct answers |
| LLM-as-a-Judge | Using an LLM to score the quality of another LLM's output |
| Faithfulness | Whether the answer is supported by the retrieved context |
| Hallucination | When the model confidently states something that is not true |
| Regression testing | Running evaluation after changes to detect quality drops |
| RAGAs | An open-source framework for evaluating RAG pipelines |

---

## What to do next

- Create a golden dataset of 20 questions for your RAG system
- Run LLM-as-a-Judge evaluation and calculate average faithfulness
- Add the evaluation to your CI/CD pipeline
- Read [12 — Observability](./12-observability.md) to monitor quality in production