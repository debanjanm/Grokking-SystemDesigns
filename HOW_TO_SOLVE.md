# How to Solve Any System Design Problem

A universal thinking framework. Domain-specific guides live in each subfolder.

---

## The Golden Rule

> **Never jump to solutions. Drive with questions.**

Interviewers don't want the "right answer" — they want to see how you think under ambiguity. A candidate who clarifies before designing beats one who designs fast but wrong.

---

## Step-by-Step Framework

### Step 1 — Clarify Requirements (5 min)

Ask before drawing anything.

**Functional (what must it do?):**
- What is the core user action? (post, search, pay, match, recommend)
- What surfaces exist? (web, mobile, API)
- What is explicitly NOT in scope? (state it yourself — shows awareness)

**Non-functional (how well must it do it?):**
- Latency SLA? (< 100ms? < 2s? best-effort?)
- Availability? (99.9%? 99.99%? what's the cost of downtime?)
- Consistency? (strong or eventual? does a stale read cause harm?)
- Scale? (DAU, QPS, data volume — get numbers)
- Geo? (single region or global?)

**Common follow-up questions:**
- Read-heavy or write-heavy?
- Real-time or batch acceptable?
- Is this a new system or redesign?

> **Signal:** saying "I'll assume X unless you tell me otherwise" shows experience.

---

### Step 2 — Back-of-Envelope Estimation (3–5 min)

Don't guess — calculate. Numbers drive every design decision that follows.

**Traffic:**
```
DAU × actions_per_user_per_day = requests/day
requests/day ÷ 86,400 = avg RPS
avg RPS × peak_multiplier (2–10×) = peak RPS
```

**Storage:**
```
record_size × writes/day = storage/day
storage/day × 365 × years = total storage
```

**Key derived questions:**
- Does this fit in memory? (< 16–64 GB = yes for single node)
- Does read traffic require cache? (> 5K RPS usually yes)
- Does write traffic require sharding? (> 10K writes/sec usually yes)
- Is media involved? (multiply storage by 10–100×)

> **Signal:** state your assumptions clearly. "I'll assume 100 bytes per record" beats leaving it implicit.

---

### Step 3 — High-Level Design (10 min)

Draw boxes and arrows. Start with the simplest design that works.

**Recipe:**
1. Identify the actors (clients, services, datastores)
2. Trace the critical path (most important user action end to end)
3. Name the boxes (don't say "backend server" — say "Feed Service", "Match Service")
4. Add storage (what data lives where, what DB type and why)

**Core question at every box:**
> "What does this component own and what does it NOT do?"

**Common first-draft components:**
```
Client → Load Balancer → API Gateway → Service(s) → Cache → DB
                                              ↓
                                        Message Queue → Workers
                                              ↓
                                        Object Store (media)
```

> **Signal:** don't over-engineer the first draft. Say "this is the simple version — I'll point out where it breaks at scale."

---

### Step 4 — Deep Dive (15–20 min)

Pick 2–3 hard problems and go deep. Don't try to cover everything equally.

**How to pick:**
- What is the hardest constraint? (latency? scale? consistency?) → go there
- What will the interviewer ask about? (if you said "Cassandra", be ready to defend it)
- What are the core algorithmic choices? (ID generation, partitioning, ranking)

**LLD checklist:**
- [ ] Data model / schema (what fields, what types, what indexes)
- [ ] API design (endpoints, request/response shape)
- [ ] Key algorithm or strategy (how does the core logic work?)
- [ ] Component internals (how does the hardest component actually function?)
- [ ] Security (auth, input validation, encryption, rate limiting)

---

### Step 5 — Scalability (5 min)

Stress-test your own design. Ask "what breaks first?"

**Scaling levers (in order of reach for):**
| Lever | When |
|---|---|
| Add cache | Read-heavy; repeated queries |
| Read replicas | DB reads saturating |
| Horizontal scaling (stateless services) | CPU/memory on app servers |
| Sharding / partitioning | DB writes saturating |
| CDN | Static assets; geographic reads |
| Message queue (async) | Write spikes; decouple producer from consumer |
| Separate read/write paths (CQRS) | Wildly different read vs write patterns |

**Bottleneck identification:**
```
Where does latency accumulate?   → optimize that component
Where does data concentrate?     → shard/partition there
Where does state make scaling hard? → externalize to Redis/DB
```

---

### Step 6 — Tradeoffs (3 min)

Explicitly name what you gave up. Interviewers penalize overconfident designs.

**Template:**
> "I chose X over Y because [reason]. The cost of this decision is [tradeoff]. It would only matter if [condition], at which point we'd [alternative]."

**Common tradeoff axes:**
- Latency vs consistency (cache = fast but stale)
- Simplicity vs scalability (monolith = simple but doesn't scale)
- Cost vs quality (more replicas = higher availability but expensive)
- Generality vs performance (general solution vs purpose-built)

---

### Step 7 — Failure Modes (2 min)

What happens when components fail? Shows production mindset.

**Cover at minimum:**
- What if the cache goes down?
- What if the DB goes down?
- What if a service crashes mid-operation?

**Answers usually involve:**
- Fallback to slower path (cache miss → DB)
- Circuit breaker (stop calling failing service)
- Retry with backoff (transient failures)
- Idempotency (safe to retry writes)
- Graceful degradation (return partial result, not error)

---

## Timing Guide (45-min Interview)

| Phase | Time |
|---|---|
| Clarify requirements | 5 min |
| Estimation | 3–5 min |
| HLD (draw + explain) | 10 min |
| Deep dive (2–3 areas) | 15–20 min |
| Tradeoffs + failure modes | 5 min |

> If interviewer steers you somewhere — follow them. They're showing you what they care about.

---

## Mental Models

### The Read/Write Split
Almost every system has asymmetric read/write load. Always ask ratio. Shapes everything:
- Write-heavy → optimize write path (async, batching, LSM-tree DB)
- Read-heavy → optimize read path (cache, read replicas, CDN)

### The Stateless/Stateful Split
Stateless services scale horizontally. Stateful services don't.
- Push state to external stores (Redis, DB)
- Keep services stateless → scale by adding instances

### The Sync/Async Split
Synchronous calls add latency on the critical path. Ask: "does the caller need this result immediately?"
- Yes → sync (REST/gRPC)
- No → async (Kafka, SQS, background job)

### The Strong/Eventual Consistency Split
Strong consistency = coordination overhead. Ask: "what is the cost of a stale read?"
- Stale read = financial harm / data corruption → strong consistency (PostgreSQL, 2PC)
- Stale read = minor UX issue → eventual consistency (Cassandra, Redis, DynamoDB)

---

## Anti-Patterns to Avoid

| Anti-pattern | Why it fails |
|---|---|
| Jumping to code/schema before HLD | Lost in details before structure is clear |
| "Just use microservices" without reason | Complexity without benefit |
| Single DB for everything at scale | First thing to crack under load |
| Ignoring the read/write ratio | Shapes every storage decision |
| No cache on read-heavy system | DB will saturate |
| Synchronous everything | Latency chains; cascading failures |
| Not naming failure modes | Shows no production experience |
| Skipping estimation | Can't justify design decisions |

---

## Red Flags Interviewers Watch For

- Jumping to solutions without clarifying
- Can't explain why they chose a specific DB
- Design that can't scale (single node everything)
- No mention of failure modes
- Confusing "availability" with "reliability"
- Using eventual consistency where strong is required (payments, inventory)
- Over-engineering a simple problem

---

## When You Don't Know Something

Say: "I'm not deeply familiar with X, but my instinct would be Y because [reasoning]. Is there a specific concern you want me to address?"

Never fake knowledge. Reasoning under uncertainty is valued. Guessing and being wrong is not.

---

## Domain Guides

- [Classical Systems](./classical/HOW_TO_SOLVE.md) — distributed systems, databases, caching, consistency
- [ML Systems](./ml-systems/HOW_TO_SOLVE.md) — ML pipelines, feature stores, model serving
- [GenAI Systems](./genai-systems/HOW_TO_SOLVE.md) — LLMs, RAG, agents, evaluation
