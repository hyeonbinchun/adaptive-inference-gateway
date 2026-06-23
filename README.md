# Adaptive Inference Gateway

**LLM Cost & Latency Optimization Gateway**


An intelligent gateway that sits in front of LLM APIs and reduces cost and latency through:

* **Semantic Caching** using vector similarity search
* **Dynamic Model Routing** based on query complexity
* **Agentic Quality Control** with automatic escalation
* **Feedback Memory RAG** for continuous improvement

---

## Motivation

Most applications send every request to the most capable (and expensive) model.

However:

* Many user queries are repeated or semantically equivalent.
* Simple questions do not require frontier models.
* Expensive models should only be used when necessary.

This project aims to optimize the **cost-quality-latency tradeoff** by combining semantic caching, model routing, and feedback-driven learning.

---

## Architecture

```text
                      ┌─────────────────┐
                      │     Request     │
                      └────────┬────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Semantic Cache    │
                    │      (Qdrant)       │
                    └──────────┬──────────┘
                               │
                               │ Cache Miss
                               │
                               ▼
                  ┌─────────────────────────┐
                  │  Feedback Memory RAG    │
                  └────────────┬────────────┘
                               │
                               ▼
                      ┌─────────────────┐
                      │     Router      │
                      └────────┬────────┘
                               │
                               ▼
                      ┌─────────────────┐
                      │      Model      │
                      └────────┬────────┘
                               │
                               ▼
                      ┌─────────────────┐
                      │      Judge      │
                      └────────┬────────┘
                               │
                               ▼
                      ┌─────────────────┐
                      │   PostgreSQL    │
                      └────────┬────────┘
                               │
                               ▼
                      ┌─────────────────┐
                      │   Batch ETL     │
                      └────────┬────────┘
                               │
                               ▼
                      ┌─────────────────┐
                      │     Qdrant      │
                      └─────────────────┘
```

---

## Core Components

### 1. Semantic Cache

Before calling an LLM, the gateway searches Qdrant for semantically similar requests.

```text
User Query
    ↓
Embedding
    ↓
Qdrant Similarity Search
    ↓
Cached Response (if similarity > threshold)
```

Benefits:

* Reduced API cost
* Lower latency
* Higher throughput

### 2. Model Router

Selects the most appropriate model based on query complexity.

**Strategy 1: Baseline**

Always use GPT-5.

```python
response = gpt5(question)
```

**Strategy 2: Rule-Based Router**
Simple heuristics.

```python
if token_count > 100:
    model = "gpt-5"
else:
    model = "haiku"
```

Possible features:

* Token count
* Presence of code
* Mathematical reasoning keywords
* Multi-step instruction patterns

**Strategy 3: LLM Router**

A lightweight model acts as a classifier.

```text
Question
    ↓
Router LLM
    ↓
Model Selection
    ↓
Answer Generation
```

Example output:

```json
{
  "difficulty": "medium",
  "recommended_model": "gpt-4o-mini"
}
```

**Strategy 4: Agentic Router**

A multi-step decision system with quality verification.

```text
Question
    ↓
Router Agent
    ↓
Model Selection
    ↓
Answer Generation
    ↓
Judge Agent
    ↓
Quality Score
    ↓
Escalation (if needed)
    ↓
Feedback Storage
```

Advantages:

* Automatic quality assurance
* Cost-aware routing
* Continuous learning

### 3. Feedback Memory RAG

Stores successful interactions and routing decisions.

The router can retrieve previous experiences before selecting a model.

```text
Question
    ↓
Retrieve Similar Cases
    ↓
Prior Routing Decisions
    ↓
Better Model Selection
```

### 4. Batch ETL Pipeline

Only high-quality interactions are promoted into long-term memory.

```text
PostgreSQL Logs
      ↓
Quality Filtering
      ↓
Embedding Generation
      ↓
Qdrant Memory Store
```

---

## Technology Stack

| Component           | Technology                |
| ------------------- | ------------------------- |
| API Gateway         | FastAPI                   |
| Agent Orchestration | LangGraph                 |
| Vector Database     | Qdrant                    |
| Relational Storage  | PostgreSQL                |
| Embeddings          | OpenAI Embeddings         |
| LLM Providers       | OpenAI, Anthropic, Etc    |
| Containerization    | Docker, Docker Compose.   |

---

## Project Phases

### Phase 1

Focus on cost and latency optimization.

```text
Semantic Cache
+
Rule Router
+
LLM Router
+
Agent Router
```

Goals:

* Measure semantic cache effectiveness
* Compare routing strategies
* Analyze cost-quality tradeoffs


### Phase 2

Add learning from historical interactions.

```text
Feedback Memory RAG
+
Agent Router
```

Goals:

* Improve routing decisions
* Reduce unnecessary escalations
* Build adaptive behavior over time
* How many API calls are avoided?

## Expected Outcomes

* 30–70% reduction in LLM API costs
* Significant latency improvement from semantic cache hits
* Better cost-quality tradeoff than always using GPT-5
* Adaptive routing through feedback-driven learning