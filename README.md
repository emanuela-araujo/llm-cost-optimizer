# LLM Cost Optimizer — 66% Cost Reduction Through Architectural Decisions

> **"Optimizing AI didn't mean adding intelligence to the system — it meant removing unnecessary complexity."**

A case study on how architectural decisions can directly impact the operational cost of LLM-based multi-agent systems.

---

## The Problem

I was presented with a common scenario in AI agent automation: a pipeline with **satisfactory output but high token cost**.

The original system worked as follows:

- User input was passed to an **orchestrator agent**
- The orchestrator coordinated the work of **4 specialized sub-agents**
- Each sub-agent extracted data from **MCP (Model Context Protocol) servers** and returned raw outputs
- The orchestrator then consolidated all intermediate results

**The root cause:** the orchestrator was accumulating too many responsibilities. Beyond interpreting user input, it had to decide which sub-agent to activate, track execution state, and consolidate intermediate results. In practice, every interaction grew the context window exponentially — and so did the cost.

---

## The Solution

### 1. Replacing the orchestrator with a sequential agent pipeline

Analysis of the workflow revealed that the steps always followed a **fixed structure**: extraction → processing → consolidation. No dynamic decision-making was needed between steps.

The orchestrator agent was removed and replaced with a **sequential agent pipeline** — each agent executes a specific step, receiving only the context required for its function. This immediately:

- Reduced the context window per agent (local context instead of global)
- Eliminated intermediate reasoning tokens used purely for coordination
- Made the pipeline deterministic and easier to debug

### 2. Context engineering

Previously, raw MCP server outputs were passed almost entirely between agents, inflating the context window with unstructured data.

The fix: an **intermediate filtering and structuring step**. Instead of connecting agents directly to MCP servers, data is now extracted dynamically from APIs and stored in a temporary structured table consulted downstream — only relevant fields, no raw noise.

### 3. Model selection by task complexity

All agents originally used the same model, regardless of task complexity. Simple deterministic tasks (classification, formatting, data insertion) were running on models designed for complex reasoning.

After agent specialization, **model selection was aligned with task requirements**:

| Task type | Model choice |
|---|---|
| Deterministic (classify, format, insert) | Lightweight, low-cost model |
| Interpretive (summarization, reasoning) | Full-capability model |

---

## Results

| Metric | Before | After |
|---|---|---|
| Architecture | Orchestrator + 4 sub-agents | Sequential pipeline (4 agents) |
| Context scope | Global (shared across all agents) | Local (per-agent only) |
| Model strategy | Same model for all tasks | Task-matched model selection |
| Cost per execution | Baseline | **−66%** |
| Savings per run | — | **~R$ 17.00** |

---

## Key Takeaways

- **Orchestration complexity has a token cost.** Every decision-making step the orchestrator takes consumes tokens. If the workflow is deterministic, remove the orchestrator.
- **Context is not free.** Raw data flowing through agents inflates cost invisibly. Filter early, pass only what's needed.
- **Not all tasks need the same model.** Matching model capability to task complexity is one of the highest-leverage cost levers available.
- **Simpler architectures are often more scalable.** A sequential pipeline with clear boundaries is easier to maintain, monitor, and extend than a centralized orchestrator.

---

## Concepts Covered

- Multi-agent system design
- Context engineering
- Sequential agent pipelines vs. orchestrator patterns
- Model Context Protocol (MCP)
- Cost-performance tradeoffs in LLM systems
- Task-based model selection

---

## Related

- [LinkedIn article (Portuguese)](https://www.linkedin.com/in/-emanuela-araujo/) — original writeup with architecture diagrams
- [My GitHub profile](https://github.com/emanuela-araujo)

---

## Author

**Emanuela Araújo** — AI Engineer & Data Scientist  
[LinkedIn](https://www.linkedin.com/in/-emanuela-araujo/) · [GitHub](https://github.com/emanuela-araujo)
