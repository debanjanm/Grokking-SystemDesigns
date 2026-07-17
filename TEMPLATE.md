# [System Name]

One-line description of what the system does.

---

## Requirements

### Functional Requirements
- 
- 
- 

### Non-Functional Requirements
- Latency: 
- Availability: 
- Consistency: 
- Security & Privacy: 
- Scalability: 

### Out of Scope
- 
- 

---

## Back-of-Envelope Estimation

### Traffic Estimation
| Metric | Value |
|---|---|
| DAU / MAU | |
| Reads/sec | |
| Writes/sec | |

### Storage Estimation
| Metric | Value |
|---|---|
| Record size | |
| Records/day | |
| Storage (1yr) | |
| Storage (5yr) | |

### Memory Estimation
| Metric | Value |
|---|---|
| Cache size | |
| Vector index size | |

### Bandwidth Estimation
| Metric | Value |
|---|---|
| Ingress/sec | |
| Egress/sec | |

### Cost Estimation *(GenAI-specific)*
| Metric | Value |
|---|---|
| Tokens/day | |
| Cost/day | |

---

## High-Level Design

### Architecture Style
> e.g. Microservices / Event-driven / Serverless / Monolith / Hybrid
> Justify the choice in 1-2 lines.

### Architecture Diagram
```
Component A → Component B → Component C
                  ↓
            Component D
```

### Core Components
| Component | Responsibility |
|---|---|
| | |

### Data Flow
1. 
2. 
3. 

### Key Design Decisions
- **Decision 1:** Rationale
- **Decision 2:** Rationale
- **Model Selection *(GenAI)*:** Which model, why

---

## Low-Level Design

### Data Models
```
Table / Document / Schema name:
  field_name   TYPE   notes
```

### API Design
```
POST /endpoint
Request:  { }
Response: { }
```

### Component Deep-Dives
#### Component Name
Internal logic, data structures, concurrency considerations.

#### Guardrails / Safety Layer *(GenAI-specific)*
Input validation, output filtering, PII scrubbing, content moderation.

### Algorithms & Strategies
- **Chunking strategy *(RAG)*:** 
- **Routing logic:** 
- **Caching strategy:** 
- **Prompt design *(GenAI)*:** 

### Security Design
- AuthN/AuthZ: 
- Encryption at rest: 
- Encryption in transit: 
- Secrets management: 
- PII handling: 

---

## Observability

### Metrics
| Metric | Type | Alert threshold |
|---|---|---|
| Request latency (p99) | Histogram | |
| Error rate | Counter | |
| Cache hit rate | Gauge | |

### Logging
- Log level strategy (INFO / WARN / ERROR)
- Structured logging fields: request_id, tenant_id, latency, status
- What NOT to log: PII, raw prompts (configurable)

### Tracing
- Distributed trace across: [list components]
- Trace sampling rate: 

### Alerting
| Alert | Threshold | Action |
|---|---|---|
| High error rate | >1% for 5min | Page on-call |
| Latency spike | p99 > Xms | Page on-call |

---

## Tradeoffs

| Decision | Chose | Sacrificed |
|---|---|---|
| | | |
| | | |

---

## Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| | | |
| | | |

---

## Feedback Loop & Improvements *(GenAI-specific)*
- How user feedback is captured (thumbs up/down, corrections)
- How feedback feeds back into system (prompt tuning, retrieval improvement, fine-tuning)
- Eval cadence: automated regression on golden set per deploy

---

## Interview Tips

### What Interviewers Expect
- 

### Common Follow-ups
- 
