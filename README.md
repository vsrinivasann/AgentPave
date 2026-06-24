# AgentPave

> The SpringBoot for AI Agents — a simple, standardised, and extensible framework for building agents that run on any runtime, any cloud, any model.

---

## What is AgentPave?

AgentKit is a **concept-first, runtime-agnostic framework** that gives developers a simple, standardised, and extensible way to create, run, and manage AI agents.

You define your agent once. AgentKit runs it on **LangGraph** or **MAF** — your choice.

```
[ Your Agent Definition ]
         ↓
    [ AgentPave ]
     ↓         ↓
LangGraph     MAF        ← you choose the runtime
     ↓         ↓
 Any LLM    Any LLM      ← model agnostic
     ↓         ↓
Any Cloud  Any Cloud     ← cloud agnostic
```

---

## Why AgentPave?

Every agent framework today makes you choose a runtime and live with its opinions. LangGraph is powerful but complex. MAF is enterprise-grade but Azure-first. CrewAI is fast but hits walls in production.

AgentKit sits **above** all of them. It gives you:

- A **clean agent definition model** — declare what your agent is, not how it runs
- **Runtime portability** — swap LangGraph for MAF without touching your agent code
- **Production-ready from day one** — memory, reliability, security, cost management, and observability built into the spec
- **Extensible by design** — start with core, add capabilities as you grow
- **Developer-first** — running your first agent in under 5 minutes

---

## Core Principles

| Principle | What it means |
|---|---|
| Progressive Autonomy | Start with human involvement. Reduce it as the agent proves itself. |
| Durability over Convenience | Agents survive failures and resume — never restart from scratch. |
| Determinism where Possible | LLM for reasoning. Typed code for execution. |
| Version Everything | Agents, LLM interfaces, schemas, tool contracts — all versioned. |
| Least Privilege Always | Agents never exceed the privilege of the task that invoked them. |
| Cost Awareness by Default | Every agent declares a budget. No agent runs unconstrained. |
| Language & Runtime Agnostic | The spec has no runtime opinions. Python is first — not only. |

---

## What AgentKit Covers

AgentKit is built around **13 dimensions** — the complete set of concerns every production agent framework must address.

| # | Dimension | What it solves |
|---|---|---|
| 1 | **Identity** | What the agent is and how it is declared |
| 2 | **Lifecycle** | Declared → Registered → Running → Paused → Resumed → Retired |
| 3 | **Communication** | MCP for tools, A2A for agents, defined human communication model |
| 4 | **Memory** | Working, Episodic, Semantic (RAG), Procedural — with lifecycle |
| 5 | **Reliability** | Durable execution, drift detection, reliability contracts |
| 6 | **Security** | Zero Trust identity, privilege model, policy-as-code |
| 7 | **Economics** | Token budgets, model routing, prompt caching |
| 8 | **Observability** | OpenTelemetry hooks, step-level tracing, pluggable backends |
| 9 | **Testability** | CI/CD integration, evaluation framework, production feedback loop |
| 10 | **Developer Experience** | CLI, local runner, mock tools, templates, 5-minute hello world |
| 11 | **Extensibility** | Core never depends on extensions. Extensions always depend on core. |
| 12 | **Governance** | Policy declaration, human oversight hooks, compliance adapters |
| 13 | **Portability** | No lock-in at any layer — runtime, model, cloud, or language |

---

## Reference Implementations

| Implementation | Runtime | Language |
|---|---|---|
| **AgentKit-LangGraph** | LangGraph | Python |
| **AgentKit-MAF** | Microsoft Agent Framework | Python + .NET |

The same agent definition runs on both. If it doesn't — the spec has a gap.

---

## Extensions

AgentKit is extensible by design. Official extensions plug into defined extension points.

| Extension | Purpose | Priority |
|---|---|---|
| AgentKit-Observe | OpenTelemetry backend integration | Release 1 |
| AgentKit-Eval | Evaluation + CI/CD + production feedback loop | Release 1 |
| AgentKit-HITL | Human-in-the-loop approval workflows | Release 1 |
| AgentKit-MultiAgent | Agent-to-agent coordination | Release 2 |
| AgentKit-Cache | Semantic + prompt caching | Release 2 |
| AgentKit-Memory | Memory consolidation | Release 2 |
| AgentKit-Govern | EU AI Act, SOC2, HIPAA, GDPR adapters | Release 3 |
| AgentKit-Marketplace | Agent registry and discovery | Release 3 |

---

## What AgentKit is NOT

- Not a cloud platform — no managed infrastructure
- Not a UI or drag-and-drop builder
- Not tied to any LLM provider
- Not tied to any cloud provider
- Not a replacement for LangGraph or MAF — it sits above them

---

## Documentation

| Document | Description |
|---|---|
| [Baseline Structure](docs/BASELINE.md) | The complete, versioned AgentKit structure — the north star document |
| AgentKit Spec v1.0 | *(coming soon)* Full specification for implementers |
| AgentKit-LangGraph Guide | *(coming soon)* Building agents with the LangGraph implementation |
| AgentKit-MAF Guide | *(coming soon)* Building agents with the MAF implementation |

---

## Status

| Item | Status |
|---|---|
| Baseline Structure | ✅ v1.0 — Baselined June 2026 |
| AgentKit Spec | 🔄 In Progress |
| AgentKit-LangGraph | 📅 Planned |
| AgentKit-MAF | 📅 Planned |

---

## Contributing

AgentKit is concept-first. The most valuable contributions right now are:

- Feedback on the [Baseline Structure](docs/BASELINE.md)
- Gaps you've experienced building agents in production that the spec doesn't address
- Reference implementation contributions (LangGraph, MAF, and beyond)

---

*AgentKit — Build agents, not plumbing.*
