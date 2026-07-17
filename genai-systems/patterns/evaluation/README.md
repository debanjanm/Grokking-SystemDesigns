# Pattern: LLM Evaluation

Systematically measure the quality, safety, and reliability of LLM-powered systems — both offline (before deploy) and online (in production).

---

## What It Is

LLM evaluation answers: "is this system good enough to ship, and is it getting better or worse over time?" It covers:
- **Offline eval:** run test suite before deploying a new model or prompt change
- **Online eval:** monitor production responses for quality drift
- **LLM-as-judge:** use a capable LLM to score outputs where ground truth is hard to define
- **Task-specific metrics:** RAG retrieval quality, agent task success, classification accuracy

Unlike classical ML eval (AUC, F1 on labeled dataset), LLM eval is harder because outputs are open-ended and ground truth is subjective.

---

## When to Use

| Use | Avoid |
|---|---|
| Before shipping prompt or model changes | Evaluating only with vibes / manual spot-check |
| Tracking quality over time in production | As a replacement for domain expert review (use both) |
| Comparing two models / prompts (A/B offline) | As sole signal for high-stakes decisions |
| RAG pipeline quality measurement | |
| Agent task completion measurement | |

---

## Core Concepts

### Eval Hierarchy
```
Level 1 — Unit Tests (deterministic)
  Input → expected exact output
  Fast, cheap, zero LLM calls
  Catches: regressions on known failures, format errors

Level 2 — Reference-based Metrics
  Model output vs reference answer
  Metrics: ROUGE, BLEU, BERTScore, exact match
  Catches: factual drift, key entity omission

Level 3 — LLM-as-Judge
  Judge LLM scores: quality, faithfulness, helpfulness, safety
  Slow, costs tokens, but handles open-ended outputs
  Catches: hallucination, tone, completeness

Level 4 — Human Eval
  Human rates output: gold standard
  Expensive, slow, not scalable
  Use to: calibrate LLM-as-judge; validate on critical outputs
```

Use multiple levels. Level 1+2 run on every CI commit. Level 3 on PR merges. Level 4 periodically or for major model changes.

### Golden Set
```
Curated set of (input, expected_output / criteria) pairs.

Properties:
  - Representative: covers common cases + known hard cases + edge cases
  - Stable: add to it; never remove (regression tracking)
  - Labeled: ground truth from domain experts, not model output
  - Diverse: not just easy queries; include adversarial inputs

Size: 100–500 examples minimum; 1000+ for robust signal
```

---

## Architecture

### Offline Eval Pipeline
```
Code / Prompt Change (PR)
    ↓
Eval Runner
    ├── Load golden set
    ├── Run model (new version) on all inputs
    ├── Score outputs (metrics + LLM-as-judge)
    ├── Compare vs baseline (previous model/prompt)
    └── Gate: pass if no regression > threshold
         ├── PASS → allow merge / deploy
         └── FAIL → block; report which examples regressed
```

### Online Eval Pipeline
```
Production Traffic
    ↓ (sampled: 1–5%)
Eval Queue (Kafka)
    ↓
Async Eval Worker
    ├── LLM-as-judge scores sampled responses
    ├── Compute: running quality metrics per model_version
    └── Write to metrics store (ClickHouse / BigQuery)
         ↓
Dashboard (Grafana / custom)
    └── Alert if quality metric drops > threshold
```

---

## Implementation Details

### LLM-as-Judge (Single Axis)
```python
FAITHFULNESS_PROMPT = """
You are evaluating whether an AI assistant's answer is faithful to the provided context.

Context:
{context}

Question:
{question}

Answer:
{answer}

Score the answer on FAITHFULNESS (does it only use information from the context?):
- 1: Completely faithful — no information beyond context
- 2: Mostly faithful — minor inference
- 3: Somewhat unfaithful — adds information not in context
- 4: Mostly unfaithful — significant hallucination
- 5: Completely unfaithful — contradicts or ignores context

Respond in JSON: {"score": <1-5>, "reasoning": "<one sentence>"}
"""

def eval_faithfulness(context: str, question: str, answer: str) -> dict:
    response = llm.complete(
        FAITHFULNESS_PROMPT.format(
            context=context, question=question, answer=answer
        ),
        temperature=0,      # deterministic judge
        response_format={"type": "json_object"},
    )
    return json.loads(response.content)
    # → {"score": 2, "reasoning": "Answer correctly uses context but adds minor inference"}
```

### LLM-as-Judge (Multi-Axis)
```python
AXES = {
    "faithfulness":  "Does the answer only use information from the provided context?",
    "completeness":  "Does the answer address all parts of the question?",
    "conciseness":   "Is the answer appropriately concise without omitting key info?",
    "helpfulness":   "Would a real user find this answer helpful?",
    "safety":        "Is the answer free from harmful, offensive, or dangerous content?",
}

def eval_multi_axis(context, question, answer) -> dict[str, float]:
    scores = {}
    for axis, criteria in AXES.items():
        result = eval_axis(context, question, answer, criteria)
        scores[axis] = result["score"]
    return scores
    # → {"faithfulness": 1.2, "completeness": 2.1, "conciseness": 1.5, ...}
```

### RAG-Specific Metrics (RAGAs)

| Metric | What it measures | How computed |
|---|---|---|
| **Retrieval Recall** | Are relevant chunks retrieved? | Does golden context appear in top-K? |
| **Context Precision** | Are retrieved chunks actually relevant? | % of retrieved chunks used in answer |
| **Answer Faithfulness** | Does answer stay within context? | LLM-as-judge: no hallucination |
| **Answer Relevance** | Does answer address the question? | LLM-as-judge: question answered |
| **Context Recall** | Does context contain enough to answer? | LLM-as-judge: is answer possible from context? |

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_recall

dataset = {
    "question": questions,
    "answer": model_answers,
    "contexts": retrieved_contexts,
    "ground_truth": golden_answers,
}

results = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_recall])
# → { "faithfulness": 0.87, "answer_relevancy": 0.91, "context_recall": 0.83 }
```

### Agent Task Eval
```python
def eval_agent_task(task: str, expected_outcome: str, agent_trace: list) -> dict:
    # 1. Task completion (binary or LLM-scored)
    completion_prompt = f"""
    Task: {task}
    Expected outcome: {expected_outcome}
    Agent's final answer: {agent_trace[-1]["answer"]}

    Did the agent successfully complete the task? (yes/partial/no)
    JSON: {{"result": "yes|partial|no", "reasoning": "..."}}
    """
    completion = llm.complete(completion_prompt, temperature=0)

    # 2. Efficiency: steps taken vs minimum expected
    step_count = len(agent_trace)

    # 3. Safety: any dangerous tool calls?
    safety_violations = [
        step for step in agent_trace
        if step["action"]["tool"] in DANGEROUS_TOOLS and not step.get("confirmed")
    ]

    return {
        "task_completion": json.loads(completion)["result"],
        "step_efficiency": step_count,
        "safety_violations": len(safety_violations),
    }
```

### Regression Detection
```python
def check_regression(baseline_scores: dict, new_scores: dict, thresholds: dict) -> list[str]:
    regressions = []
    for metric, new_val in new_scores.items():
        baseline = baseline_scores[metric]
        delta = new_val - baseline
        threshold = thresholds.get(metric, -0.05)  # default: 5% regression fails

        if delta < threshold:
            regressions.append(
                f"{metric}: {baseline:.3f} → {new_val:.3f} (delta={delta:+.3f}, threshold={threshold})"
            )
    return regressions

# Usage in CI:
regressions = check_regression(baseline, new_scores, thresholds={
    "faithfulness": -0.05,     # block if faithfulness drops 5%
    "answer_relevancy": -0.03, # block if relevancy drops 3%
    "task_completion": -0.10,  # block if completion drops 10%
})
if regressions:
    raise EvalGateFailure(f"Regressions detected:\n" + "\n".join(regressions))
```

### LLM-as-Judge Calibration
```
LLM judges can be biased: verbose answers rated higher (verbosity bias),
own outputs rated higher (self-preference bias), first option preferred.

Mitigations:
  1. Use stronger model as judge than model being judged (GPT-4o judges GPT-3.5)
  2. Randomize answer order when presenting to judge
  3. Calibrate judge scores against human ratings (compute correlation)
  4. Use multiple judge prompts; average scores
  5. Penalize judge for inconsistency (score A vs B, then B vs A — should match)

Calibration target: judge-human Spearman correlation > 0.7
```

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| LLM-as-judge over human eval | Scalable, cheap per eval | Biased; requires calibration vs human baseline |
| Deterministic judge (temp=0) | Reproducible scores | Judge itself may not be perfectly deterministic |
| Small golden set (100 examples) | Fast eval, cheap | May miss edge cases; low statistical power |
| Large golden set (1000+ examples) | Statistical robustness | Expensive to label; slow eval run |
| Sample online traffic (1–5%) | Low cost, catch real drift | Slow to detect sharp quality drops |
| Eval all online traffic | Fast detection | High cost (every response judged by LLM) |
| RAGAs metrics | Standardized, interpretable | May not capture domain-specific quality |

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Golden set built from model outputs | Label independently (domain experts or humans) |
| Only evaluating average score | Track score distribution; tail failures matter |
| LLM judge same as model under test | Use a stronger, different model as judge |
| No regression baseline | Always compare to last-deployed version, not just absolute threshold |
| Eval only at deploy time | Sample + score production traffic continuously |
| Single eval metric | Multi-axis: faithfulness + relevance + safety + completeness |
| Ignoring latency in eval | Include latency as eval metric; quality at 10s p99 ≠ quality at 500ms |
| Golden set never updated | Refresh quarterly; add new failure cases found in production |

---

## Used In
- [RAG Pipeline](../../cases/rag-pipeline/) — retrieval recall, context precision, answer faithfulness (RAGAs)
- [AI Agent](../../cases/ai-agent/) — task completion rate, step efficiency, safety violation rate
- [LLM Gateway](../../cases/llm-gateway/) — cache correctness, response quality regression per model version
- [ML Systems / Model Serving](../../../ml-systems/patterns/model-serving/) — analogous to shadow mode + champion/challenger

---

## Tools
| Tool | Use |
|---|---|
| **RAGAs** | RAG-specific metrics (faithfulness, recall, precision) |
| **LangSmith** | Trace + eval for LangChain apps; LLM-as-judge built in |
| **Promptfoo** | Prompt eval framework; supports multiple judge models |
| **Braintrust** | Eval + tracing + dataset management; production-grade |
| **Arize Phoenix** | Open-source; traces + evals; RAG-focused |
| **TruLens** | LLM app eval; RAGAs-compatible |
| **Custom** | Full control; LLM-as-judge via direct API; metrics in ClickHouse |
