# Cost Comparison — Before & After Architectural Refactor

## Executive Summary

A 66% reduction in cost per execution was achieved through three architectural changes:
removing the central orchestrator, introducing context filtering, and aligning model selection with task complexity.

---

## Architecture Comparison

| Dimension | Original Architecture | Optimized Architecture |
|---|---|---|
| Agent structure | Orchestrator + 4 sub-agents | Sequential pipeline (4 agents) |
| Context scope | Global — shared across all agents | Local — isolated per agent |
| MCP data handling | Raw output passed between agents | Filtered and structured before passing |
| Model strategy | Same model for all tasks | Task-matched model selection |
| Orchestration overhead | High — dynamic decision-making at every step | None — deterministic sequential flow |

---

## Token Cost Breakdown

### Original architecture — cost drivers

| Source | Why it was expensive |
|---|---|
| Orchestrator reasoning | Every step required the orchestrator to decide what to do next, consuming tokens purely for coordination |
| Global context accumulation | Each agent received the full execution history, not just what it needed |
| Raw MCP outputs | Unstructured data from MCP servers passed between agents inflated the context window |
| Uniform model usage | Simple tasks (classification, formatting) ran on high-capability, high-cost models |

### Optimized architecture — cost reductions

| Change | Impact |
|---|---|
| Removed orchestrator | Eliminated all coordination reasoning tokens |
| Local context per agent | Each agent receives only its required inputs — context window reduced significantly |
| API-based data extraction + temp table | Structured, filtered data replaces raw MCP output — no noise in the context |
| Lightweight models for deterministic tasks | Lower cost per token on tasks that don't require complex reasoning |

---

## Results

| Metric | Value |
|---|---|
| Cost reduction per execution | **~66%** |
| Savings per run | **~R$ 33.00** |
| Output quality | Maintained — no degradation in results |

---

## Key Insight

> Cost in LLM systems scales with context size and reasoning complexity — not with the number of agents.
> A simpler, more specialized pipeline can outperform a centralized orchestrator both in cost and maintainability.