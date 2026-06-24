# AgentPave — Baseline Structure v1.0

**Baselined:** June 2026  
**Status:** Approved for Spec Writing  
**Purpose:** Framework for creating agents, runtime-agnostic, developer-first  
**Analogy:** SpringBoot for Agents — define once, run on LangGraph or MAF  

---

## The One-Line Definition

> AgentPave is a concept-first, runtime-agnostic framework that gives developers a simple, standardised, and extensible way to create, run, and manage AI agents — regardless of the underlying runtime or cloud provider.

---

## Design Philosophy

These principles are non-negotiable. Every decision in the spec and every reference implementation must honour them.

| Principle | What it means |
|---|---|
| **Progressive Autonomy** | Start with more human involvement. Reduce it as the agent proves itself. Never the other way around. |
| **Durability over Convenience** | Agents must survive failures and resume — not restart from scratch. |
| **Determinism where Possible** | LLM for reasoning and intent. Typed, testable code for execution. Never use LLM for things that should be exact. |
| **Version Everything** | LLM interfaces, agent definitions, communication schemas, tool contracts — all versioned. |
| **Least Privilege Always** | Agents never have more privilege than the user or task that invoked them. |
| **Cost Awareness by Default** | Every agent declares its cost constraints. No agent runs without a budget. |
| **Language & Runtime Agnostic** | The spec makes no assumption about programming language or runtime. Python is the first reference implementation — not the only one. |

---

## Core Structure — 13 Dimensions

The spec is organised around 13 universal framework dimensions. Every reference implementation must implement all 13.

---

### 1. IDENTITY
*What the agent is and how it is declared.*

- Agent declaration model (config-driven, code-first, or both)
- Unique agent identifier (immutable across versions)
- Cryptographic identity — Zero Trust, short-lived tokens
- Agent metadata: name, version, owner, purpose, domain

---

### 2. LIFECYCLE
*How the agent lives — from declaration to retirement.*

**States:**
```
Declared → Registered → Running → Paused → Resumed → Retired
```

- Lifecycle hooks: on_start, on_pause, on_error, on_resume, on_retire
- Versioned Agent Registry — tracks all agents and their versions
- Coordinated version rollout — agents in a system must version-sync
- Graceful degradation and shutdown

---

### 3. COMMUNICATION
*How the agent talks — to tools, other agents, and humans.*

- **Tool communication:** MCP (Model Context Protocol) — industry standard
- **Agent-to-agent:** A2A protocol — cross-runtime interoperability
- **Agent-to-human:** Defined communication model for HITL and approvals
- **Integration Contract** — mandatory per tool:
  - What systems the agent can access
  - Behaviour when a system is unavailable
  - Retry and fallback rules
  - Rate limit behaviour
- Versioned Communication Schema — prevents protocol mismatch failures

---

### 4. MEMORY
*What the agent remembers, at what scope, and for how long.*

| Memory Type | What it stores | Technology examples |
|---|---|---|
| **Working Memory** | Current task, active context | Context window |
| **Episodic Memory** | Past session history | Redis, DynamoDB |
| **Semantic Memory** | Knowledge, documents (RAG) | Vector DB — Pinecone, pgvector, Chroma |
| **Procedural Memory** | Learned patterns, few-shot examples | Few-shot store, fine-tune registry |

**Memory Lifecycle (mandatory):**
- Ownership and privacy declaration per memory store
- Staleness and expiry policy
- Cross-session identity resolution
- Memory consolidation hooks (async between-session processing)

> Note: RAG ≠ Memory. RAG retrieves from static knowledge. Memory grows and changes with every interaction.

---

### 5. RELIABILITY
*What happens when things go wrong — and how the agent stays trustworthy over time.*

- **Durable Execution** — agents persist through failures and resume from exact stopping points
- **Durable State Management** — state is checkpointed, not held in memory
- **Deterministic Task Router** — routes deterministic tasks (calculations, DB queries) away from the LLM
- **Reliability Contract** — declared per agent:
  - Hallucination tolerance level (e.g. zero tolerance for medical, higher for creative)
  - Grounding mechanisms required
  - Domain-specific quality thresholds
- **Drift Detection hooks:**
  - Embedding drift — vector representations diverging from reality
  - Behavioural drift — agent output quality degrading over time
- **Load Contract** — declared expected behaviour under load (concurrent requests, latency SLA)

---

### 6. SECURITY
*Who can do what, and how that is enforced.*

- **Zero Trust Agent Identity:**
  - Short-lived ephemeral tokens scoped to a single task graph
  - Tokens expire within minutes — no long-lived API keys
  - Agent-to-agent mTLS with certificate pinning
- **Privilege Model:**
  - Least privilege enforcement — minimum permissions needed
  - Confused Deputy prevention — agent never has more privilege than invoking user
- **Policy-as-code enforcement** — every tool call evaluated against declarative policy (OPA / Cedar)
- **Immutable audit trail** — append-only log of all agent decisions and data accesses

---

### 7. ECONOMICS
*What the agent costs, and how that is controlled.*

- Token budget declaration — per agent, per task, per day
- **Model routing rules:**
  - Simple/deterministic tasks → cost-optimised models
  - Complex reasoning tasks → frontier models
- Prompt caching hooks — 50–90% input token savings on repeated patterns
- Semantic caching — reuse responses for semantically equivalent queries
- Cost alerting thresholds — alert at 50% and 80% of budget before hard limits

---

### 8. OBSERVABILITY
*How you know what the agent is doing.*

- Structured logging — built into core, not optional
- OpenTelemetry trace hooks — standard, pluggable
- Step-level trace capture:
  - Every tool call and result
  - Every reasoning step
  - Every decision point and branch taken
- Pluggable observability backends:
  - LangSmith, Langfuse, Arize, Braintrust, custom

---

### 9. TESTABILITY
*How you verify the agent works — before and after deployment.*

- Unit testable agent components — each component independently testable
- Mock tool interface — offline development without real tool calls
- Evaluation framework hooks:
  - Ground truth validation
  - Semantic similarity scoring
  - Full trajectory evaluation (tool choice + outcomes)
- CI/CD integration contract — evaluations run on every pull request
- **Production → Test feedback loop:**
  - Production failures automatically converted to regression test cases
  - Regression dataset grows weekly with new failure patterns

---

### 10. DEVELOPER EXPERIENCE
*How easy it is to build, debug, and ship an agent.*

- **CLI** — scaffold, run, debug, deploy from terminal
- **Local runner** — mirrors production behaviour exactly, runs offline
- **Local trace visualizer** — inspect every step without a cloud dependency
- **Mock tool server** — simulate any tool response for offline testing
- **Agent templates** — scaffold common agent patterns instantly
- **"Hello World" agent** — a developer must be able to run their first agent in under 5 minutes

> Developer Experience is a first-class concern — not an afterthought. A framework that is hard to use will not be adopted, regardless of how technically complete it is.

---

### 11. EXTENSIBILITY
*How new capabilities are added without breaking existing ones.*

- Core never depends on extensions
- Extensions always depend on core
- Extension points are explicitly defined in the spec — not open-ended
- Every extension has its own versioning contract
- Adding an extension must never require changing core agent definitions

---

### 12. GOVERNANCE
*How agents are audited, approved, and controlled at scale.*

- Policy declaration per agent — what it is and isn't allowed to do
- Human oversight interface hooks — mandatory for high-risk agent actions
- Compliance adapter extension points:
  - EU AI Act (Article 14 — human oversight for high-risk systems)
  - SOC2, HIPAA, GDPR adapters
- Agent approval workflow hooks — for regulated environments

---

### 13. PORTABILITY
*How lock-in is avoided at every layer.*

- Spec is language-agnostic — Python first, Java / Go / TypeScript later
- Spec is runtime-agnostic — no LangGraph or MAF concepts leak into core
- No cloud provider dependencies in core
- MCP and A2A as universal interface contracts — not proprietary protocols
- Swapping runtime (LangGraph ↔ MAF) requires zero changes to agent definition

---

## Extension Registry

Extensions are official AgentPave modules that plug into defined extension points. They are versioned, optional, and independently deployable.

### Priority 1 — Ship with first release
| Extension | Purpose |
|---|---|
| **AgentPave-Observe** | OpenTelemetry backend integration |
| **AgentPave-Eval** | Evaluation framework + CI/CD integration + feedback loop |
| **AgentPave-HITL** | Human-in-the-loop approval workflows |

### Priority 2 — Ship with second release
| Extension | Purpose |
|---|---|
| **AgentPave-MultiAgent** | Agent-to-agent coordination layer |
| **AgentPave-Cache** | Semantic caching + prompt caching |
| **AgentPave-Memory** | Memory consolidation (async between-session processing) |

### Priority 3 — Ship with third release
| Extension | Purpose |
|---|---|
| **AgentPave-Govern** | Compliance adapters — EU AI Act, SOC2, HIPAA, GDPR |
| **AgentPave-Marketplace** | Agent registry and discovery |

---

## Reference Implementations

Two reference implementations prove the spec works across runtimes.

| Implementation | Runtime | Language | Status |
|---|---|---|---|
| **AgentPave-LangGraph** | LangGraph | Python | Planned |
| **AgentPave-MAF** | Microsoft Agent Framework | Python + .NET | Planned |

> The same agent definition must run on both implementations without modification. If it doesn't, the spec has a gap.

---

## What AgentPave is NOT

- Not a cloud platform — no managed infrastructure
- Not a UI builder — no drag and drop
- Not tied to any LLM provider
- Not tied to any cloud provider
- Not a replacement for LangGraph or MAF — it sits above them

---

## Versioning & Change Log

| Version | Date | Change |
|---|---|---|
| v1.0 | June 2026 | Initial baseline — 13 dimensions, 7 principles, 2 reference implementations |

> Any changes to this baseline must be versioned. The spec is derived from this document. If the baseline changes, the spec version increments.

---

*This document is the baseline. It is the north star for the AgentPave Spec, all reference implementations, and all extensions.*
