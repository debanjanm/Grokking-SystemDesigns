# GenAI Systems

LLM-native architecture design.

**Core concerns:** Context window limits, Cost per token, Hallucination, Latency, Evaluation

## Cases

| System | Key Challenge |
|---|---|
| [RAG Pipeline](./cases/rag-pipeline/) | Retrieval quality, chunking strategy, context stuffing |
| [AI Agent](./cases/ai-agent/) | Tool use, memory, multi-step reasoning, failure recovery |
| [LLM Gateway](./cases/llm-gateway/) | Multi-provider routing, rate limits, cost tracking, caching |

## Patterns

| Pattern | Used In |
|---|---|
| [Prompt Caching](./patterns/prompt-caching/) | Cost reduction for repeated system prompts |
| [Vector Search](./patterns/vector-search/) | Semantic retrieval in RAG, memory systems |
| [Evaluation](./patterns/evaluation/) | LLM-as-judge, regression testing, golden sets |
