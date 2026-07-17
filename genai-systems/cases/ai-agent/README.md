# AI Agent

Design an LLM-powered agent that autonomously completes multi-step tasks using tools (web search, code execution, APIs).

---

## Requirements

### Functional Requirements
- Accept high-level goal from user in natural language
- Break goal into steps; select and invoke tools autonomously
- Handle tool errors and retry with corrected inputs
- Maintain memory across steps within a single task
- Maintain long-term memory across sessions (user preferences, past tasks)
- Return final answer with reasoning trace
- Support human-in-the-loop confirmation for irreversible actions

### Non-Functional Requirements
- Latency: best-effort for long tasks (async acceptable); interactive steps < 3s
- Availability: 99.9% on task submission; individual tool failures handled gracefully
- Consistency: task traces are durable and reproducible for debugging
- Security & Privacy: no irreversible action without confirmation gate; PII not stored in plaintext memory
- Scalability: 10,000 concurrent tasks; each task can span 2–50 LLM calls

### Out of Scope
- Training or fine-tuning LLM
- Building individual tools (search API, code sandbox — integrated, not built)
- UI / frontend

---

## Back-of-Envelope Estimation

### Traffic Estimation
| Metric | Value |
|---|---|
| Tasks/day | 100,000 |
| Avg steps/task | 10 |
| LLM calls/day | 1M |
| Concurrent tasks (peak) | 10,000 |

### Storage Estimation
| Metric | Value |
|---|---|
| Trace per task | ~20 KB |
| Traces/day | ~2 GB |
| Long-term memory per user | ~50 KB |
| Total memory store (1M users) | ~50 GB |

### Memory Estimation
| Metric | Value |
|---|---|
| Active task context (Redis) | ~10 GB |
| Vector memory index (long-term) | ~20 GB |

### Bandwidth Estimation
| Metric | Value |
|---|---|
| LLM input tokens/day | ~500M |
| LLM output tokens/day | ~200M |

### Cost Estimation
| Metric | Value |
|---|---|
| LLM calls/day | 1M |
| Avg tokens/call | 3,000 input + 500 output |
| Estimated cost/day | ~$3,000–$8,000 |
| Prompt caching savings (system prompt) | ~30% of input token cost |

---

## High-Level Design

### Architecture Style
**Event-driven Orchestration + Stateless Worker Nodes** — task orchestrator manages state and step sequencing; agent worker nodes are stateless and horizontally scalable; all task state lives in external stores.

### Architecture Diagram
```
User
  ↓
Task API (submit / status / cancel)
  ↓
Task Queue (Kafka)
  ↓
Agent Orchestrator
  ├── State Manager (Redis — current step, context)
  ├── LLM Planner (reason + select tool)
  │         ↓
  ├── Tool Router
  │    ├── Web Search
  │    ├── Code Executor (sandboxed)
  │    ├── API Caller
  │    └── Memory Tool (read/write)
  │         ↓
  ├── Safety Filter (pre + post tool call)
  ├── Memory Manager
  │    ├── Short-term (Redis, task-scoped)
  │    └── Long-term (Vector DB, cross-session)
  └── Trace Logger (S3 / DB — immutable audit)
        ↓
Task Result → User (webhook / polling)
```

### Core Components
| Component | Responsibility |
|---|---|
| Task API | Accept tasks, expose status + cancellation endpoints |
| Agent Orchestrator | Drive ReAct loop; manage step state; decide continue/stop |
| LLM Planner | Generate Thought → Action → Observation at each step |
| Tool Router | Parse LLM tool call → dispatch to correct tool executor |
| Safety Filter | Validate tool inputs pre-execution; filter outputs post-execution |
| Memory Manager | Read/write short-term (task) and long-term (user) memory |
| Trace Logger | Persist full task trace (steps, tool calls, results) for replay/debug |

### Data Flow

**Task Execution (ReAct loop):**
1. User submits task → Task API creates task record → publishes to Kafka
2. Agent worker picks up task → loads task context from Redis
3. LLM Planner receives (goal + history + available tools) → outputs Thought + Action
4. Tool Router parses Action → Safety Filter validates → dispatches tool
5. Tool executes → result returned as Observation
6. Orchestrator appends Observation to context → loop back to step 3
7. LLM outputs `Final Answer` → task marked complete → user notified
8. Trace written to S3; long-term memory updated asynchronously

**Memory Read (step 3 enrichment):**
- Before LLM call: query long-term memory with current goal → inject relevant past context
- Short-term: last N steps from Redis context

### Key Design Decisions
- **ReAct (Reason + Act) pattern:** LLM alternates between Thought (reasoning) and Action (tool call). Observable, debuggable, adaptive to intermediate errors. Alternative: Plan-then-Execute (upfront plan, parallel steps) for deterministic tasks.
- **Async task execution:** Agent tasks are long-running (seconds to minutes). Submit-and-poll pattern avoids holding HTTP connections.
- **External memory:** In-context window alone insufficient for long tasks (50+ steps) or cross-session memory. Redis for short-term (fast, TTL-based); Vector DB for long-term (semantic retrieval).
- **Safety gate:** Read-only tools auto-approved; write/delete/send require confirmation gate or policy check. Non-negotiable for production agents.
- **Model Selection:** Claude Sonnet / GPT-4o for planning (strong reasoning, tool use). Haiku / GPT-4o-mini for short intermediate steps to reduce cost. Routing based on step complexity.

---

## Low-Level Design

### Data Models
```
tasks (PostgreSQL):
  task_id       UUID      PK
  user_id       UUID
  goal          TEXT
  status        ENUM (queued, running, paused, complete, failed)
  result        TEXT      (nullable)
  created_at    TIMESTAMP
  updated_at    TIMESTAMP
  step_count    INTEGER
  max_steps     INTEGER   (default 50)

task_steps (PostgreSQL):
  step_id       UUID      PK
  task_id       UUID      FK
  step_index    INTEGER
  thought       TEXT
  action        JSONB     { tool, args }
  observation   TEXT
  timestamp     TIMESTAMP

task_context (Redis):
  key: "ctx:{task_id}"
  value: {
    goal, step_history (last 20), current_step,
    tools_available, user_id
  }
  TTL: 2 hours

long_term_memory (Vector DB):
  id        = memory_id
  vector    = float32[1536]   (embed of memory content)
  payload   = { user_id, content, memory_type, created_at, task_id }

user_memory_index (PostgreSQL):
  memory_id     UUID      PK
  user_id       UUID
  content       TEXT
  memory_type   ENUM (preference, fact, past_task)
  created_at    TIMESTAMP
```

### API Design
```
POST /tasks
Request:  { "goal": "Research competitors and summarize pricing", "max_steps": 30 }
Response: { "task_id": "...", "status": "queued" }

GET /tasks/{task_id}
Response: { "status": "running", "step_count": 7, "latest_thought": "..." }

GET /tasks/{task_id}/trace
Response: { "steps": [{ "thought": "...", "action": {...}, "observation": "..." }] }

POST /tasks/{task_id}/confirm
Request:  { "action_id": "...", "approved": true }
Response: { "status": "resumed" }

DELETE /tasks/{task_id}
Response: { "status": "cancelled" }
```

### Component Deep-Dives

#### ReAct Loop (Orchestrator)
```
while step_count < max_steps:
    context = build_context(goal, history, memory, tools)
    llm_response = llm.complete(context)

    if llm_response.is_final_answer:
        complete_task(llm_response.answer)
        break

    action = parse_tool_call(llm_response)
    safety_result = safety_filter.check(action)

    if safety_result == REQUIRES_CONFIRMATION:
        pause_task(); notify_user(); wait_for_confirm()

    observation = tool_router.execute(action)
    append_step(thought, action, observation)
    step_count += 1
else:
    fail_task("max_steps exceeded")
```

#### Memory Manager — Memory Types
| Type | Scope | Storage | TTL | Retrieval |
|---|---|---|---|---|
| In-context | Current request | LLM context window | — | Direct injection |
| Short-term | Current task | Redis | 2hr | Key lookup |
| Long-term | Cross-session | Vector DB + PostgreSQL | Permanent | Semantic search |
| Episodic | Specific past events | Vector DB | Permanent | Semantic search |

Long-term memory write: after task complete → embed key takeaways → store in Vector DB with user_id filter.

Long-term memory read: before each LLM call → embed current goal → top-3 relevant memories → inject into context.

#### Tool Design Contract
```json
{
  "name": "web_search",
  "description": "Search the web for current information. Use for recent events, facts, prices.",
  "parameters": {
    "query": { "type": "string", "description": "search query" },
    "max_results": { "type": "integer", "default": 5 }
  },
  "returns": "list of { title, url, snippet }",
  "safety_class": "read_only"
}
```

`safety_class` values:
- `read_only` → auto-approve
- `write` → policy check (rate limit, scope validation)
- `destructive` → require explicit user confirmation

#### Guardrails / Safety Layer

**Pre-execution (input validation):**
- Schema validation: tool args match declared parameter types
- Scope check: agent cannot call tools outside task-granted permissions
- Prompt injection detection: scan tool outputs before feeding back to LLM (`ignore previous instructions`, etc.)
- Destructive action gate: DELETE / SEND / POST to external services require confirmation

**Post-execution (output filtering):**
- Strip PII from tool observations before storing in memory
- Truncate large outputs before context injection (max 2000 tokens per observation)
- Flag anomalous outputs (e.g., tool returned HTML error page instead of JSON)

#### Loop Detector
```
if last 3 (action, args) tuples are identical → break loop
if step_count > 0.8 * max_steps → inject "you are running out of steps, aim to finalize" hint
```

### Algorithms & Strategies
- **Context window management:** Keep last 20 steps in context. Summarize older steps via LLM ("summarize these steps in 200 tokens") when approaching limit.
- **Tool selection:** LLM selects from tool schema list injected in system prompt. Descriptions must be precise — LLM matches intent to description.
- **Prompt design:** System prompt includes: agent role, tool schemas, memory snippets, output format (JSON with `thought`, `action` or `final_answer`). Prompt cached at provider level (Anthropic/OpenAI prompt caching) to reduce cost.
- **Retry strategy:** Tool failures → retry up to 3× with exponential backoff. After 3 failures → LLM observes failure and selects alternative approach.
- **Plan-and-Execute variant:** For complex tasks, first call LLM to generate full plan (list of steps) → execute steps sequentially or in parallel → reduces total LLM calls for deterministic flows.

### Security Design
- AuthN/AuthZ: task_id + user_id scoped; agent cannot access other users' tasks or memory
- Tool permissions: per-task permission manifest; agent cannot escalate
- Code execution: sandboxed (gVisor / Firecracker VM); no network access from sandbox; timeout 30s; resource limits (CPU, mem, disk)
- Secrets: no secrets in agent context; tools call APIs server-side, credentials not exposed to LLM
- Audit: all tool calls logged immutably (action, args, result, user_id, timestamp) to S3 with Object Lock
- PII: strip from memory before storage; task goal hashed in logs

---

## Observability

### Metrics
| Metric | Type | Alert threshold |
|---|---|---|
| Task completion rate | Gauge | < 80% |
| Avg steps per task | Histogram | > 40 (approaching limit) |
| Tool error rate (per tool) | Counter | > 5% |
| Safety gate trigger rate | Counter | > 1% (unexpected spike) |
| LLM call latency (p95) | Histogram | > 5s |
| Memory retrieval latency | Histogram | > 100ms |

### Logging
- Per step: task_id, step_index, tool_name, tool_args_hash, observation_length, latency, safety_result
- Per task: task_id, user_id, goal_hash, step_count, outcome (complete/failed/cancelled), total_cost
- Do NOT log: raw goal text (PII risk), raw tool observations, memory content

### Tracing
- Trace spans: task_submit → orchestrator_pickup → llm_plan → tool_dispatch → tool_execute → memory_update
- Cross-step trace: single trace_id per task, child spans per step
- Sample: 20% normal; 100% on failures or tasks > 30 steps

### Alerting
| Alert | Threshold | Action |
|---|---|---|
| Task failure rate spike | > 20% for 5min | Page on-call |
| Code sandbox timeout | > 5% requests | Check sandbox capacity |
| LLM provider errors | > 2% for 2min | Failover to backup model |
| Infinite loop detection | > 50 tasks in loop state | Alert + auto-cancel |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| ReAct over Plan-Execute | Adaptive to intermediate errors | More LLM calls = higher cost |
| External memory (Redis + Vector DB) | Long task + cross-session support | Retrieval latency + sync complexity |
| Async task execution | UX responsiveness | Harder error surfacing (polling vs push) |
| Strict tool schemas | Reduces hallucinated tool calls | Less flexible; schema updates need deploy |
| Safety gate for destructive actions | Prevents irreversible mistakes | Adds friction for power users |
| Context window summarization | Handles long tasks | Summarization loses detail; LLM cost |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| Infinite loop | LLM stuck in same reasoning pattern | Loop detector; max_steps hard cap; inject "finalize now" hint |
| Tool hallucination | LLM invents args or fabricates results | Strict JSON schema validation; observation consistency check |
| Context overflow | Long task exceeds context window | Sliding window (last 20 steps) + episodic summary |
| Irreversible action | Agent acts without confirmation | Safety gate; `destructive` tool class blocks without user ack |
| Cascading errors | One bad tool result poisons next steps | Checkpoint state; allow human-in-loop recovery |
| Memory poisoning | Bad data stored in long-term memory | TTL + confidence scoring; user can clear memory |
| Code sandbox escape | Malicious code in tool input | gVisor isolation; no network; resource limits; auto-kill on timeout |

---

## Feedback Loop & Improvements
- Task completion rated by user (1-5 or success/failure)
- Failed tasks reviewed: classify failure (tool error, reasoning error, safety block, timeout)
- Low-rated task traces used to: improve tool descriptions, add examples to system prompt, adjust safety thresholds
- Memory quality: user can flag bad memories for removal
- Eval cadence: benchmark on fixed task suite per model version change (task completion rate, avg steps)

---

## Interview Tips

### What Interviewers Expect
- ReAct loop as baseline — must explain Thought / Action / Observation clearly
- Memory design is the crux — distinguish in-context vs short-term vs long-term
- Safety: how to prevent irreversible actions — confirmation gate + safety_class per tool
- Async execution: why agent tasks can't be synchronous HTTP calls

### Common Follow-ups
- Multi-agent: supervisor routes subtasks to specialized sub-agents (researcher, coder, critic) → results synthesized
- Evaluation: task completion rate + step efficiency (fewer steps = better planning) + human eval
- How to handle streaming? SSE for step-by-step thought visibility
- Cost control: route short steps to cheaper model; cache system prompt; limit max_steps
