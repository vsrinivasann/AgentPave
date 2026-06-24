# AgentPave Specification v1.1

**Status:** Draft — MVP Scope  
**Baselined from:** [AgentPave Baseline Structure v1.0](BASELINE.md)  
**Spec Type:** Executable — suitable for human engineers and autonomous coding agents  
**License:** Apache 2.0  
**Stability:** See Section 6 for per-dimension stability levels  

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [How to Read This Spec](#2-how-to-read-this-spec)
3. [Glossary](#3-glossary)
4. [Design Philosophy](#4-design-philosophy)
5. [Architecture Overview](#5-architecture-overview)
6. [Specification Overview — 13 Dimensions](#6-specification-overview--13-dimensions)
7. [Standard Error Taxonomy](#7-standard-error-taxonomy)
8. [Non-Functional Requirements](#8-non-functional-requirements)
9. [Spec Invariants](#9-spec-invariants)
10. [MVP Scope](#10-mvp-scope)
11. [Extension Model](#11-extension-model)
12. [Conformance](#12-conformance)
13. [Document Index](#13-document-index)
14. [Versioning & Changelog](#14-versioning--changelog)

---

## 1. Introduction

### 1.1 What is AgentPave?

AgentPave is a concept-first, runtime-agnostic framework that gives developers a simple, standardised, and extensible way to create, run, and manage AI agents — regardless of the underlying runtime or cloud provider.

AgentPave is to AI agents what SpringBoot is to microservices. It provides the framework. The developer provides the agent definition. The runtime (LangGraph or MAF) provides the execution engine.

```
[ Agent Definition ]        ← developer writes this
        ↓
  [ AgentPave Core ]         ← this spec defines this
     ↓         ↓
LangGraph     MAF           ← runtime implementations
     ↓         ↓
 Any LLM    Any LLM         ← model agnostic
     ↓         ↓
Any Cloud  Any Cloud        ← cloud agnostic
```

### 1.2 Who This Spec Is For

This specification is written for two audiences:

**Human Engineers**
- Implementers building AgentPave-LangGraph or AgentPave-MAF
- Developers building agents using AgentPave
- Architects evaluating AgentPave for their organisation

**Autonomous Coding Agents**
- Agents assigned to implement AgentPave reference implementations
- Agents assigned to build conformant AgentPave extensions
- Agents assigned to generate AgentPave-compliant agent definitions

Both audiences must be able to read this spec and produce correct, conformant output without asking clarifying questions. Any ambiguity in this spec is a bug in the spec — open an issue.

### 1.3 Spec-Driven Development Contract

This spec operates under the following contract:

> **The spec is the source of truth. Code is the output.**

When a conflict exists between this spec and any implementation:
- The spec wins
- The implementation must be updated to match the spec
- The spec is never changed to match a broken implementation

When this spec is updated:
- The version number increments
- The changelog records what changed and why
- All implementations must update to match within one release cycle

### 1.4 Machine-Readable Schema

In addition to this human-readable document, AgentPave publishes a machine-readable JSON Schema at:

```
/schema/agentpave-schema.json
```

The JSON Schema is the authoritative source for all data model definitions in this spec. Where any conflict exists between prose descriptions and the JSON Schema, the JSON Schema wins. Autonomous agents and tooling SHOULD validate agent definitions against the JSON Schema programmatically.

---

## 2. How to Read This Spec

### 2.1 Notation Guide

This spec uses RFC 2119 keyword notation throughout. These keywords have precise meanings and must not be interpreted loosely.

| Keyword | Meaning |
|---|---|
| **MUST** | Absolute requirement. No exceptions. Non-conformant if violated. |
| **MUST NOT** | Absolute prohibition. No exceptions. Non-conformant if violated. |
| **SHOULD** | Strongly recommended. Valid reasons may justify omission — but implications must be fully understood and documented. |
| **SHOULD NOT** | Strongly discouraged. Valid reasons may justify inclusion — but implications must be fully understood and documented. |
| **MAY** | Truly optional. Implementations may include or omit freely without affecting conformance. |

### 2.2 EARS Notation

Behavioural requirements in this spec use EARS — Easy Approach to Requirements Syntax. EARS produces unambiguous, machine-readable requirements.

EARS patterns used in this spec:

| Pattern | Syntax | Example |
|---|---|---|
| Ubiquitous | THE SYSTEM SHALL [action] | THE SYSTEM SHALL assign a unique identifier to every agent. |
| Event-driven | WHEN [trigger] THE SYSTEM SHALL [action] | WHEN an agent raises an exception THE SYSTEM SHALL invoke the on_error lifecycle hook. |
| State-driven | WHILE [state] THE SYSTEM SHALL [action] | WHILE an agent is in PAUSED state THE SYSTEM SHALL reject all task invocations. |
| Conditional | IF [condition] THEN THE SYSTEM SHALL [action] | IF token budget is exceeded THEN THE SYSTEM SHALL terminate the task and raise AgentPave.BudgetExceededError. |
| Optional feature | WHERE [feature] THE SYSTEM SHALL [action] | WHERE observability extension is enabled THE SYSTEM SHALL emit an OpenTelemetry span per tool call. |

### 2.3 Three-Tier Boundary System

Every dimension spec defines a three-tier boundary system. These tiers govern what an implementation or agent must always do, must confirm before doing, and must never do.

| Tier | Label | Meaning |
|---|---|---|
| Tier 1 | **ALWAYS** | The implementation must always do this. No conditions, no exceptions. Violation is non-conformant. |
| Tier 2 | **ASK FIRST** | The implementation must obtain explicit human or policy confirmation before doing this. Proceeding without confirmation is non-conformant. |
| Tier 3 | **NEVER** | The implementation must never do this under any circumstance. Violation is non-conformant. |

> The most commonly omitted tier is ASK FIRST. Without it, implementations treat ambiguous situations as either always permitted or always prohibited — both of which are wrong and dangerous.

### 2.4 Acceptance Criteria Format

All acceptance criteria in this spec follow Given/When/Then format and resolve to a binary pass/fail. There is no partial pass.

```
Criteria ID:  [DIMENSION]-[NUMBER]
Stability:    [Alpha | Beta | Stable]
Given:        [precondition — the exact state of the system before the action]
When:         [action — the event or input that triggers behaviour]
Then:         [outcome — the exact, verifiable result, referencing defined types]
Pass:         [the precise programmatic check that verifies Then is true]
Fail:         [the exact condition that constitutes failure]
Error raised: [the AgentPave standard error type raised on failure, if applicable]
```

An autonomous agent implementing AgentPave MUST run all acceptance criteria for each dimension against its own implementation before declaring that dimension complete. No exceptions.

### 2.5 Definition of Done

Each dimension in Section 6 includes an explicit Definition of Done (DoD) checklist. A dimension is complete if and only if every item in its DoD is checked. An autonomous agent MUST NOT proceed to the next dimension until all DoD items for the current dimension are satisfied.

### 2.6 Stability Levels

Every dimension and every acceptance criterion carries a stability label:

| Level | Meaning | Commitment |
|---|---|---|
| **Alpha** | Early design. May change without notice. Not for production use. | None |
| **Beta** | Stable enough to build on. Breaking changes announced one release ahead. | One release notice |
| **Stable** | Frozen contract. Breaking changes require MAJOR version increment. | Full SemVer protection |

All MVP dimensions (1–6) are **Beta** in v1.1 of this spec. No dimension reaches Stable until both reference implementations pass their conformance suites.

### 2.7 Reading Order for Autonomous Agents

If you are an autonomous coding agent consuming this spec, follow this order precisely:

```
Step 1:  Read Section 3 (Glossary) — understand every term before reading anything else.
         Do not proceed until you can define every term without referring back.

Step 2:  Read Section 4 (Design Philosophy) — every implementation decision must honour
         these principles. Violation is not acceptable regardless of technical justification.

Step 3:  Read Section 2.3 (Three-Tier Boundary System) — internalize what you must always
         do, must confirm before doing, and must never do.

Step 4:  Read Section 7 (Standard Error Taxonomy) — you will raise these errors throughout
         implementation. Know them before you write a single line.

Step 5:  Read Section 8 (Non-Functional Requirements) — know the performance and
         reliability targets your implementation must meet.

Step 6:  Read Section 9 (Spec Invariants) — these are the absolute constraints that must
         hold true in every state of the system.

Step 7:  Read Section 10 (MVP Scope) — understand exactly what you are building.

Step 8:  Read and implement each MVP dimension spec in this exact order:
         01-identity.md → 02-lifecycle.md → 03-communication.md →
         04-memory.md → 05-observability.md → 06-reliability.md

Step 9:  For each dimension:
         a. Read the full dimension spec including its DoD checklist
         b. Implement the dimension
         c. Run every acceptance criterion — all must pass before proceeding
         d. Verify every DoD item is satisfied
         e. Only then proceed to the next dimension

Step 10: Run the full conformance test suite (Section 12).
         Zero failures required. Do not declare implementation complete until this passes.

Step 11: Produce the conformance declaration file (agentpave-conformance.yaml).
         Validate it against /schema/agentpave-conformance-schema.json.
```

**Rollback guidance:** If you have partially implemented a dimension and encounter a blocking ambiguity or failure, do not proceed. Revert the partial implementation to the last known-good state (previous passing dimension). Open an issue on the AgentPave spec repo describing the ambiguity precisely. Do not invent behaviour to resolve ambiguity — ambiguity in this spec is a bug.

---

## 3. Glossary

All terms used across this specification and all dimension specs are defined here. Each term has exactly one definition. No synonyms are used anywhere in this spec. If a term appears that is not in this glossary, it is a bug in the spec.

| Term | Definition |
|---|---|
| **Agent** | An autonomous software entity that perceives input, reasons using an LLM, executes actions via tools, observes results, and loops until a goal is achieved or a termination condition is met. |
| **Agent Definition** | The complete, declarative description of an agent — its identity, purpose, tools, memory configuration, reliability contract, security policy, and cost constraints. Written once. Runtime-agnostic. Validated against agentpave-schema.json. |
| **Agent ID** | A globally unique, immutable string identifier assigned to an agent at declaration time. Format: UUID v4. Never reused. Never reassigned. |
| **AgentPave** | This framework. The concept-first, runtime-agnostic specification and set of reference implementations for building AI agents. |
| **AgentPave Core** | The mandatory set of 13 dimensions every conformant AgentPave implementation must implement. |
| **AgentPave Extension** | An optional, versioned module that adds capabilities to AgentPave Core via defined extension points. Extensions depend on core. Core never depends on extensions. |
| **Runtime** | The execution engine that runs an AgentPave agent. Currently: LangGraph and MAF. The same agent definition must run on any conformant runtime without modification. |
| **LLM** | Large Language Model. The reasoning engine inside an agent. AgentPave treats the LLM as a pluggable, versioned dependency. It does not mandate any specific model or provider. |
| **LLM Interface Version** | The declared version of the LLM API contract used by an agent. Incrementing this version signals a potentially breaking change in model behaviour. |
| **Tool** | An external capability an agent can invoke — an API, a function, a database query, a file operation. All tools in AgentPave are registered and invoked via MCP. |
| **MCP** | Model Context Protocol. The industry-standard protocol for agent-to-tool communication. All tool registration and invocation in AgentPave MUST use MCP. |
| **A2A** | Agent-to-Agent protocol. The industry-standard protocol for cross-runtime agent communication. |
| **Lifecycle State** | One of six mutually exclusive states an agent can occupy: DECLARED, REGISTERED, RUNNING, PAUSED, RESUMED, RETIRED. An agent occupies exactly one state at any point in time. |
| **Lifecycle Hook** | A named callback invoked by the runtime on a lifecycle state transition: on_start, on_pause, on_error, on_resume, on_retire. |
| **Working Memory** | The agent's in-context memory — the current task state held in the active LLM context window. Ephemeral. Lost when the task ends. Not persisted. |
| **Episodic Memory** | Persistent memory of past sessions and interactions. Stored in an external store (e.g. Redis, DynamoDB). Survives task and session boundaries. Owned by the agent. Subject to expiry policy. |
| **Semantic Memory** | Knowledge stored as vector embeddings, retrieved via similarity search (RAG). Represents facts, documents, and domain knowledge. Read-heavy. Updated less frequently than Episodic Memory. |
| **Procedural Memory** | Learned patterns and few-shot examples that shape agent behaviour. Stored as prompt templates, fine-tune datasets, or example libraries. Typically read-only at runtime. |
| **RAG** | Retrieval Augmented Generation. The pattern of retrieving relevant context from Semantic Memory before an LLM call. RAG is a retrieval pattern — not a memory type. RAG retrieves from a static or slowly-changing knowledge base. Agent memory grows dynamically with every interaction. |
| **Checkpoint** | A durable, serialised snapshot of an agent's complete execution state at a specific point in time. Stored externally. Used to resume after failure. |
| **Durable Execution** | The property of an agent that allows it to survive any failure and resume from its most recent Checkpoint — never restarting from scratch. |
| **Reliability Contract** | A per-agent declaration of its hallucination tolerance level, required grounding mechanisms, and domain-specific quality thresholds. Declared at definition time. Enforced at runtime. |
| **Drift** | The silent degradation of agent behaviour or memory accuracy over time. Two subtypes: Embedding Drift (vector representations no longer reflect current reality) and Behavioural Drift (agent output quality measurably declines). |
| **Zero Trust** | A security model where no agent, user, or system is trusted by default. Every request is independently authenticated, authorised, and audited, regardless of source. |
| **Confused Deputy** | A security vulnerability where an agent is manipulated — via prompt injection or malicious tool output — into exercising its privileges on behalf of an unauthorised party. |
| **Conformance** | The property of an implementation that correctly implements all MUST requirements in this spec for every dimension it claims to support, as verified by the conformance test suite. |
| **MVP** | Minimum Viable Product. The minimum set of dimensions that produce a working, deployable, observable agent: Dimensions 1–6. |
| **EARS** | Easy Approach to Requirements Syntax. The notation used in this spec for writing unambiguous, machine-readable behavioural requirements. |
| **Deterministic Task** | A task whose correct output is exactly computable from its inputs — a mathematical calculation, a database query, a sort operation. Deterministic Tasks MUST NOT be routed through the LLM. |
| **Extension Point** | A named, versioned hook in AgentPave Core where an extension may attach behaviour. Extension points are explicitly enumerated in this spec. No implicit or undeclared extension points exist. |
| **Integration Contract** | A per-tool declaration specifying: the system the tool accesses, the agent's behaviour when that system is unavailable, retry rules, fallback rules, and rate limit behaviour. Mandatory for every tool an agent uses. |
| **Token Budget** | The maximum number of LLM tokens an agent is permitted to consume. Scoped to: per-task, per-day, or per-agent-lifetime. Declared in the Agent Definition. |
| **Policy-as-Code** | Security and governance rules expressed as declarative, machine-readable, version-controlled code (e.g. OPA Rego or Cedar policies) rather than documentation or convention. |
| **Definition of Done** | An explicit checklist attached to each dimension. A dimension is complete if and only if every item on its DoD checklist is satisfied. |
| **Spec Invariant** | A statement that must be true in every valid state of the system. Invariants are machine-checkable. Violations indicate a bug in an implementation. |
| **Interoperability** | The property of two conformant AgentPave implementations being able to run the same Agent Definition and produce equivalent observable behaviour. Verified by the interoperability test suite. |
| **AgentResult** | The standard structured return type from any agent action. Contains: status, output, error (if any), trace_id, and timestamp. Defined in agentpave-schema.json. |

---

## 4. Design Philosophy

These seven principles are non-negotiable. Every implementation decision, every extension, and every agent definition must honour them. A decision that violates a principle is wrong — regardless of its technical merit or business justification.

Each principle carries an **invariant** — a machine-checkable statement that must hold true in every conformant implementation.

### 4.1 Progressive Autonomy
Start with more human involvement. Reduce it as the agent proves itself. Never the other way around.

**Consequence of violation:** Agents deployed with full autonomy before they are proven will cause irreversible production incidents. Trust must be earned through demonstrated, measured reliability.

**Invariant:** `every_agent.initial_autonomy_level <= every_agent.current_autonomy_level`  
An agent's autonomy level must only increase — never be set higher at deployment than it has earned.

### 4.2 Durability over Convenience
Agents must survive failures and resume from their exact stopping point. Restarting from scratch is never acceptable in a production agent.

**Consequence of violation:** Long-running agent tasks that restart on failure cause compounding side effects — duplicate writes, inconsistent state, wasted compute, and broken user trust.

**Invariant:** `every_tool_call_boundary → checkpoint_exists`  
A Checkpoint must exist for every tool call boundary before the next tool call begins.

### 4.3 Determinism where Possible
Use the LLM for reasoning and intent classification. Use typed, testable code for execution. Never use the LLM for tasks that must be exact.

**Consequence of violation:** Agents that route Deterministic Tasks through the LLM produce non-deterministic results — currency conversions, database queries, and calculations that silently produce wrong answers at unpredictable intervals.

**Invariant:** `is_deterministic_task(task) → NOT routed_to_llm(task)`  
No task classified as deterministic may be routed to the LLM.

### 4.4 Version Everything
LLM interfaces, agent definitions, communication schemas, tool contracts — all must be versioned. Nothing in AgentPave is unversioned.

**Consequence of violation:** Unversioned agents in a multi-agent system cause silent protocol mismatches. A new agent version misinterprets shared context from an old version. The failure is silent, hard to reproduce, and nearly impossible to debug.

**Invariant:** `every_artifact.version IS NOT NULL AND every_artifact.version matches SemVer`  
Every versioned artifact must have a non-null, SemVer-compliant version field.

### 4.5 Least Privilege Always
Agents never have more privilege than the user or task that invoked them. Privilege is scoped to the minimum required for the specific task.

**Consequence of violation:** Over-privileged agents are exploitable via prompt injection. A malicious tool output can instruct an over-privileged agent to take actions the invoking user was never authorised to perform — the Confused Deputy problem.

**Invariant:** `agent.effective_privilege <= invoking_user.privilege`  
An agent's effective privilege must never exceed the privilege of the entity that invoked it.

### 4.6 Cost Awareness by Default
Every agent declares a Token Budget. No agent runs without declared cost constraints. Cost is a first-class production concern — not a post-launch optimisation.

**Consequence of violation:** Unbounded agents in production consume unbounded resources. A single runaway agent task can exhaust an organisation's entire LLM spend in minutes. This is not hypothetical — it is a documented production incident pattern.

**Invariant:** `every_agent.token_budget IS NOT NULL AND every_agent.token_budget > 0`  
Every agent must declare a positive, non-null token budget before it may run.

### 4.7 Language and Runtime Agnostic
The AgentPave spec makes no assumption about programming language or runtime. Python is the first reference implementation — not the only one. The same Agent Definition must run on LangGraph and MAF without modification.

**Consequence of violation:** A spec that leaks runtime-specific concepts creates lock-in. Developers who adopt AgentPave expecting portability will find they have traded one form of vendor lock-in for another — a trust-destroying outcome.

**Invariant:** `agent_definition.contains_no_runtime_specific_fields()`  
No field in a valid Agent Definition may reference a runtime-specific type, class, or concept.

---

## 5. Architecture Overview

### 5.1 Layered Architecture

AgentPave is organised in four layers. Each layer depends only on the layer below it. No layer depends on the layer above it. This is a hard architectural rule — not a guideline.

```
┌─────────────────────────────────────────────────────┐
│                   AGENT DEFINITIONS                  │
│         (written by developers using AgentPave)       │
│         validated against agentpave-schema.json       │
└─────────────────────────┬───────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────┐
│                  AGENTKIT EXTENSIONS                 │
│   Observe │ Eval │ HITL │ MultiAgent │ Cache │ ...   │
│   (optional, versioned, attach at extension points)  │
└─────────────────────────┬───────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────┐
│                   AGENTKIT CORE                      │
│  Identity │ Lifecycle │ Communication │ Memory       │
│  Reliability │ Security │ Economics │ Observability  │
│  Testability │ DX │ Extensibility │ Governance       │
│  Portability                                         │
└──────────────┬──────────────────────┬───────────────┘
               │                      │
┌──────────────▼──────────┐  ┌────────▼───────────────┐
│   AGENTKIT-LANGGRAPH    │  │     AGENTKIT-MAF        │
│   Runtime Implementation│  │  Runtime Implementation │
│   (Python)              │  │  (Python + .NET)        │
└──────────────┬──────────┘  └────────┬───────────────┘
               │                      │
┌──────────────▼──────────────────────▼───────────────┐
│         LLM PROVIDERS (via versioned LLM interface)  │
│    OpenAI │ Anthropic │ Google │ Mistral │ Local     │
└─────────────────────────────────────────────────────┘
```

### 5.2 Dimension Dependency Graph

Dimensions have explicit dependencies. An implementation MUST NOT implement a dimension before all its dependencies are complete and passing their acceptance criteria.

```
Identity (D1) ─────────────────────────────► All dimensions
    │                     depends on D1: every dimension uses Agent ID
    ▼
Lifecycle (D2) ─────────────────────────────► D3, D4, D5, D6
    │                     depends on D2: lifecycle states gate all operations
    ▼
Communication (D3) ─────────────────────────► D4, D6
    │                     depends on D3: tool calls consume and update memory
    │                                        tool call boundaries trigger checkpoints
    ▼
Memory (D4) ────────────────────────────────► D6
    │                     depends on D4: checkpoint includes memory state
    ▼
Observability (D5) ─────────────────────────► (no MVP dependencies)
    │                     depends on D5: drift detection emits observability events
    ▼
Reliability (D6)
```

**Formal dependency declarations for autonomous agents:**

```
D1 (Identity):      depends_on = []
D2 (Lifecycle):     depends_on = [D1]
D3 (Communication): depends_on = [D1, D2]
D4 (Memory):        depends_on = [D1, D2, D3]
D5 (Observability): depends_on = [D1, D2]
D6 (Reliability):   depends_on = [D1, D2, D3, D4, D5]
```

Before implementing any dimension, verify: all dimensions it depends on have complete implementations with passing acceptance criteria.

### 5.3 Core vs Extensions

```
AgentPave Core
│   Defines: what an agent IS and what it MUST do
│   Rule: Core MUST NOT import or depend on any extension module
│
AgentPave Extensions
│   Defines: additional capabilities agents MAY have
│   Rule: Extensions MUST import and depend on AgentPave Core
│   Rule: Extensions MUST attach only at declared extension points
│   Rule: Each extension is independently versioned
│
Declared Extension Points (exhaustive — no others exist):
    ├── observability.backend      ← AgentPave-Observe attaches here
    ├── evaluation.runner          ← AgentPave-Eval attaches here
    ├── hitl.approval              ← AgentPave-HITL attaches here
    ├── multiagent.coordinator     ← AgentPave-MultiAgent attaches here
    ├── cache.semantic             ← AgentPave-Cache attaches here
    ├── memory.consolidation       ← AgentPave-Memory attaches here
    ├── governance.policy          ← AgentPave-Govern attaches here
    └── agent.registry             ← AgentPave-Marketplace attaches here
```

### 5.4 Protocol Standards

AgentPave uses two open, vendor-neutral protocols. AgentPave does not define proprietary communication protocols.

| Protocol | Purpose | Scope | Stability |
|---|---|---|---|
| **MCP** (Model Context Protocol) | Agent ↔ Tool communication | All tool registration and invocation | Stable |
| **A2A** (Agent-to-Agent) | Agent ↔ Agent communication | Cross-runtime agent coordination | Beta |

---

## 6. Specification Overview — 13 Dimensions

Each dimension summary includes: definition, stability level, core MUST requirements, formal dependencies, Definition of Done checklist, and link to the full dimension spec.

---

### Dimension 1 — Identity
**MVP: YES | Stability: Beta**

Every agent in AgentPave has a unique, immutable identity. Identity is declared once. It is the anchor that every other dimension references. No operation in AgentPave may proceed without a valid Agent ID.

**Depends on:** nothing — this is the root dimension.

**Core MUST requirements:**
- THE SYSTEM SHALL assign a UUID v4 Agent ID to every agent at declaration time
- THE SYSTEM SHALL make Agent ID immutable — it MUST NOT change after declaration
- THE SYSTEM SHALL associate a cryptographic identity (short-lived token) with every agent for Zero Trust authentication
- THE SYSTEM SHALL include `name`, `version`, `owner`, `purpose`, `domain`, and `created_at` in every agent's metadata
- THE SYSTEM SHALL validate every agent definition against agentpave-schema.json before accepting it

**Definition of Done:**
- [ ] Agent ID is a valid UUID v4
- [ ] Agent ID is immutable after declaration
- [ ] Agent metadata contains all required fields
- [ ] Agent definition is validated against agentpave-schema.json
- [ ] Cryptographic identity token is generated and associated
- [ ] All D1 acceptance criteria pass

→ [Full spec: spec/01-identity.md](spec/01-identity.md)

---

### Dimension 2 — Lifecycle
**MVP: YES | Stability: Beta**

Every agent moves through exactly six defined states. State transitions are governed and explicit. No agent may perform operations outside its permitted states. Every state transition invokes the appropriate lifecycle hook.

**Depends on:** D1 (Identity)

**States:** `DECLARED → REGISTERED → RUNNING → PAUSED → RESUMED → RETIRED`

**Valid transitions:**
```
DECLARED  → REGISTERED  (registration)
REGISTERED → RUNNING    (invocation)
RUNNING   → PAUSED      (pause)
RUNNING   → RETIRED     (completion or forced retirement)
RUNNING   → DECLARED    ✗ INVALID
PAUSED    → RESUMED     (resume request)
PAUSED    → RETIRED     (forced retirement from paused state)
RESUMED   → RUNNING     (automatic — RESUMED is transient)
RETIRED   → any         ✗ INVALID — RETIRED is terminal
```

**Core MUST requirements:**
- THE SYSTEM SHALL enforce valid state transitions and raise `AgentPave.InvalidStateTransitionError` on invalid ones
- THE SYSTEM SHALL invoke the appropriate lifecycle hook synchronously on every state transition
- THE SYSTEM SHALL register every agent in the versioned agent registry before it may transition to RUNNING
- WHEN an agent enters RETIRED state THE SYSTEM SHALL prevent all further state transitions
- THE SYSTEM SHALL record the timestamp and reason for every state transition in the immutable audit log

**Definition of Done:**
- [ ] All valid state transitions succeed
- [ ] All invalid transitions raise `AgentPave.InvalidStateTransitionError`
- [ ] All lifecycle hooks are invoked correctly on transition
- [ ] Agent is in versioned registry before RUNNING is reachable
- [ ] RETIRED is terminal — no transition out is possible
- [ ] All D2 acceptance criteria pass

→ [Full spec: spec/02-lifecycle.md](spec/02-lifecycle.md)

---

### Dimension 3 — Communication
**MVP: YES | Stability: Beta**

Agents communicate with tools via MCP, with other agents via A2A, and with humans via a defined interaction model. Every tool must have a declared Integration Contract. All inter-agent schemas are versioned.

**Depends on:** D1 (Identity), D2 (Lifecycle)

**Core MUST requirements:**
- THE SYSTEM SHALL use MCP for all tool registration and invocation — no direct function calls to tools are permitted
- THE SYSTEM SHALL require a valid Integration Contract for every tool an agent declares
- THE SYSTEM SHALL version all inter-agent communication schemas using SemVer
- WHEN a tool is unavailable THE SYSTEM SHALL follow the Integration Contract's declared fallback rule — not invent a fallback
- WHEN a tool call exceeds the declared rate limit THE SYSTEM SHALL raise `AgentPave.RateLimitError` and follow the Integration Contract's rate limit rule
- THE SYSTEM SHALL raise `AgentPave.ToolUnavailableError` when a tool cannot be reached after exhausting its retry policy

**Definition of Done:**
- [ ] All tool calls go through MCP
- [ ] Direct tool function calls are blocked
- [ ] Every tool has a valid Integration Contract
- [ ] Unavailable tool triggers correct fallback from Integration Contract
- [ ] Rate limit breach raises `AgentPave.RateLimitError`
- [ ] All inter-agent schemas are SemVer versioned
- [ ] All D3 acceptance criteria pass

→ [Full spec: spec/03-communication.md](spec/03-communication.md)

---

### Dimension 4 — Memory
**MVP: YES | Stability: Beta**

Agents have four memory types. Each has a defined interface, lifecycle, privacy policy, and expiry rule. RAG is a retrieval pattern — not a memory type. These are distinct and must not be conflated.

**Depends on:** D1 (Identity), D2 (Lifecycle), D3 (Communication)

**Memory type interfaces — summary:**

| Memory Type | Interface | Persistence | Scope |
|---|---|---|---|
| Working | `WorkingMemory` | Ephemeral (task duration) | Current task |
| Episodic | `EpisodicMemory` | Durable (external store) | Per-agent, cross-session |
| Semantic | `SemanticMemory` | Durable (vector store) | Knowledge base |
| Procedural | `ProceduralMemory` | Durable (template store) | Per-agent |

**Core MUST requirements:**
- THE SYSTEM SHALL provide concrete implementations of all four memory interfaces
- THE SYSTEM SHALL require an `owner`, `privacy_level`, and `expiry_policy` declaration for every Episodic and Semantic memory store
- THE SYSTEM SHALL enforce expiry policies — expired memory entries MUST NOT be returned in any query
- THE SYSTEM SHALL NOT route RAG retrieval results directly to Working Memory without an explicit agent step
- THE SYSTEM SHALL raise `AgentPave.MemoryExpiredError` when an expired entry is accessed directly

**Definition of Done:**
- [ ] All four memory interfaces are implemented
- [ ] Every memory store has declared owner, privacy_level, and expiry_policy
- [ ] Expired entries are never returned from any memory query
- [ ] RAG and memory are distinct — no conflation
- [ ] `AgentPave.MemoryExpiredError` is raised on direct expired entry access
- [ ] All D4 acceptance criteria pass

→ [Full spec: spec/04-memory.md](spec/04-memory.md)

---

### Dimension 5 — Observability
**MVP: YES | Stability: Beta**

Every agent action is observable. Structured logs and OpenTelemetry traces are emitted for every tool call, reasoning step, and decision point. Observability is built into core — not added later by extension. The observability backend is pluggable.

**Depends on:** D1 (Identity), D2 (Lifecycle)

**Core MUST requirements:**
- THE SYSTEM SHALL emit a structured log entry for every agent action — including tool calls, LLM calls, state transitions, and errors
- THE SYSTEM SHALL emit an OpenTelemetry span for every tool call — containing: `agent_id`, `tool_name`, `input_schema`, `output` (as `AgentResult`), `duration_ms`, `trace_id`, `span_id`
- THE SYSTEM SHALL include `agent_id`, `action_type`, `timestamp` (ISO 8601 UTC), `trace_id`, and `AgentResult` in every structured log entry
- THE SYSTEM SHALL support a pluggable observability backend via the `observability.backend` extension point
- THE SYSTEM SHALL NOT require a specific observability backend to be running — a null backend that logs to stdout MUST be the default

**AgentResult schema (used in all log entries and spans):**
```json
{
  "status": "success | error | timeout | cancelled",
  "output": "<any serialisable value or null>",
  "error": "<AgentPave error type and message or null>",
  "trace_id": "<UUID v4>",
  "timestamp": "<ISO 8601 UTC>"
}
```

**Definition of Done:**
- [ ] Structured log entry emitted for every agent action
- [ ] OpenTelemetry span emitted for every tool call with all required fields
- [ ] Every log entry contains all required fields including AgentResult
- [ ] Null/stdout backend is default — no backend required to run
- [ ] Pluggable backend extension point works correctly
- [ ] All D5 acceptance criteria pass

→ [Full spec: spec/05-observability.md](spec/05-observability.md)

---

### Dimension 6 — Reliability
**MVP: YES | Stability: Beta**

Agents survive failures. State is checkpointed at every tool call boundary. Deterministic tasks never touch the LLM. Every agent declares a Reliability Contract. Drift is detected and surfaced via observability events.

**Depends on:** D1, D2, D3, D4, D5 (all prior MVP dimensions)

**Core MUST requirements:**
- THE SYSTEM SHALL create a durable Checkpoint before every tool call and after every tool result is received
- WHEN an agent raises any unhandled exception THE SYSTEM SHALL resume from the most recent Checkpoint — never from the beginning of the task
- THE SYSTEM SHALL route every Deterministic Task to typed, testable code — never to the LLM
- THE SYSTEM SHALL require a Reliability Contract declaration in every Agent Definition
- THE SYSTEM SHALL raise `AgentPave.ReliabilityContractViolationError` when agent output violates its declared quality thresholds
- THE SYSTEM SHALL emit a `drift.detected` observability event when Embedding Drift or Behavioural Drift is detected

**Definition of Done:**
- [ ] Checkpoint is created before every tool call
- [ ] Checkpoint is created after every tool result is received
- [ ] Agent resumes from latest Checkpoint on failure — never restarts
- [ ] Deterministic tasks are never routed to LLM
- [ ] Every Agent Definition has a Reliability Contract
- [ ] `AgentPave.ReliabilityContractViolationError` is raised on threshold breach
- [ ] `drift.detected` event is emitted when drift is detected
- [ ] All D6 acceptance criteria pass

→ [Full spec: spec/06-reliability.md](spec/06-reliability.md)

---

### Dimensions 7–13 — Post-MVP

| # | Dimension | Stability | Target Release |
|---|---|---|---|
| 7 | Security | Alpha | Release 2 |
| 8 | Economics | Alpha | Release 2 |
| 9 | Testability | Alpha | Release 2 |
| 10 | Developer Experience | Alpha | Release 2 |
| 11 | Extensibility | Alpha | Release 2 |
| 12 | Governance | Alpha | Release 3 |
| 13 | Portability | Alpha | Release 3 |

Post-MVP dimension specs are linked in Section 13.

---

## 7. Standard Error Taxonomy

All AgentPave errors are defined here. Implementations MUST raise exactly these error types — not custom equivalents. All errors are in the `AgentPave` namespace.

Errors are structured as follows:

```python
class AgentPaveError(Exception):
    error_code: str        # e.g. "AGENTKIT_INVALID_STATE_TRANSITION"
    message: str           # human-readable description
    agent_id: str          # UUID v4 of the agent that raised the error
    dimension: str         # dimension where the error originated
    trace_id: str          # UUID v4 — links to the observability trace
    timestamp: str         # ISO 8601 UTC
    recoverable: bool      # whether the agent can resume after this error
    context: dict          # additional structured context (error-type specific)
```

### 7.1 Identity Errors

| Error Type | Error Code | Recoverable | When Raised |
|---|---|---|---|
| `AgentPave.DuplicateAgentIDError` | `AGENTKIT_DUPLICATE_AGENT_ID` | No | An Agent ID is already registered in the agent registry |
| `AgentPave.InvalidAgentDefinitionError` | `AGENTKIT_INVALID_AGENT_DEFINITION` | No | Agent Definition fails agentpave-schema.json validation |
| `AgentPave.AgentNotFoundError` | `AGENTKIT_AGENT_NOT_FOUND` | No | A referenced Agent ID does not exist in the registry |

### 7.2 Lifecycle Errors

| Error Type | Error Code | Recoverable | When Raised |
|---|---|---|---|
| `AgentPave.InvalidStateTransitionError` | `AGENTKIT_INVALID_STATE_TRANSITION` | No | An invalid state transition is attempted |
| `AgentPave.RetiredAgentError` | `AGENTKIT_RETIRED_AGENT` | No | Any operation is attempted on a RETIRED agent |
| `AgentPave.LifecycleHookError` | `AGENTKIT_LIFECYCLE_HOOK_ERROR` | Yes (with retry) | A lifecycle hook raises an unhandled exception |

### 7.3 Communication Errors

| Error Type | Error Code | Recoverable | When Raised |
|---|---|---|---|
| `AgentPave.ToolNotFoundError` | `AGENTKIT_TOOL_NOT_FOUND` | No | Tool invoked without a registered Integration Contract |
| `AgentPave.ToolUnavailableError` | `AGENTKIT_TOOL_UNAVAILABLE` | Yes (per Integration Contract) | A tool cannot be reached after exhausting retry policy |
| `AgentPave.ToolTimeoutError` | `AGENTKIT_TOOL_TIMEOUT` | Yes (retry) | Tool did not respond within declared timeout_ms |
| `AgentPave.RateLimitError` | `AGENTKIT_RATE_LIMIT` | Yes (with backoff) | A tool call exceeds the declared rate limit |
| `AgentPave.CircuitBreakerOpenError` | `AGENTKIT_CIRCUIT_BREAKER_OPEN` | Yes (wait recovery_timeout) | Tool circuit breaker is in OPEN state |
| `AgentPave.IntegrationContractViolationError` | `AGENTKIT_INTEGRATION_CONTRACT_VIOLATION` | No | A tool response violates its declared output schema |
| `AgentPave.SchemaMismatchError` | `AGENTKIT_SCHEMA_MISMATCH` | No | An inter-agent message does not match the declared schema version |
| `AgentPave.HumanInputTimeoutError` | `AGENTKIT_HUMAN_INPUT_TIMEOUT` | Yes (agent pauses) | Human did not respond within timeout_seconds |

### 7.4 Memory Errors

| Error Type | Error Code | Recoverable | When Raised |
|---|---|---|---|
| `AgentPave.MemoryExpiredError` | `AGENTKIT_MEMORY_EXPIRED` | Yes (fetch fresh) | An expired memory entry is directly accessed |
| `AgentPave.MemoryPrivacyViolationError` | `AGENTKIT_MEMORY_PRIVACY_VIOLATION` | No | An agent attempts to access memory it does not own |
| `AgentPave.MemoryStoreUnavailableError` | `AGENTKIT_MEMORY_STORE_UNAVAILABLE` | Yes (per retry policy) | The external memory store cannot be reached |
| `AgentPave.MemoryBudgetError` | `AGENTKIT_MEMORY_BUDGET` | Yes (prune working memory) | Working memory set() would exceed context_window_budget |

### 7.5 Observability Errors

| Error Type | Error Code | Recoverable | When Raised |
|---|---|---|---|
| `AgentPave.ObservabilityBackendError` | `AGENTKIT_OBSERVABILITY_BACKEND_ERROR` | Yes (log to stdout fallback) | The observability backend cannot be reached |

### 7.6 Reliability Errors

| Error Type | Error Code | Recoverable | When Raised |
|---|---|---|---|
| `AgentPave.CheckpointError` | `AGENTKIT_CHECKPOINT_ERROR` | No (halt and alert) | A checkpoint cannot be created or retrieved |
| `AgentPave.ReliabilityContractViolationError` | `AGENTKIT_RELIABILITY_CONTRACT_VIOLATION` | No | Agent output violates its declared quality thresholds |
| `AgentPave.DeterministicTaskLLMRouteError` | `AGENTKIT_DETERMINISTIC_TASK_LLM_ROUTE` | No | A Deterministic Task is routed to the LLM |
| `AgentPave.BudgetExceededError` | `AGENTKIT_BUDGET_EXCEEDED` | No | Token budget is exhausted |

### 7.7 Security Errors (D7 — Release 2)

| Error Type | Error Code | Recoverable | When Raised |
|---|---|---|---|
| `AgentPave.PolicyDeniedError` | `AGENTKIT_POLICY_DENIED` | No | PDP returns DENY for a tool call |
| `AgentPave.PrivilegeEscalationError` | `AGENTKIT_PRIVILEGE_ESCALATION` | No | Agent attempts action exceeding invoking user's privilege |
| `AgentPave.PromptInjectionDetectedError` | `AGENTKIT_PROMPT_INJECTION` | No | HIGH/CRITICAL injection pattern detected in tool output or user input |
| `AgentPave.AuditTrailError` | `AGENTKIT_AUDIT_TRAIL_ERROR` | No — block the action | Audit trail write fails |
| `AgentPave.PolicyLoadError` | `AGENTKIT_POLICY_LOAD_ERROR` | No — fix the policy | Policy document is invalid or cannot be loaded |

### 7.8 Extensibility Errors (D11 — Release 2)

| Error Type | Error Code | Recoverable | When Raised |
|---|---|---|---|
| `AgentPave.ExtensionLoadError` | `AGENTKIT_EXTENSION_LOAD_ERROR` | No — fix manifest or entry point | Extension manifest invalid, entry point not found, or wrong interface |
| `AgentPave.ExtensionCompatibilityError` | `AGENTKIT_EXTENSION_COMPATIBILITY` | No — update extension or core | Extension core version requirement not satisfied |
| `AgentPave.ExtensionConflictError` | `AGENTKIT_EXTENSION_CONFLICT` | No — unload existing first | Two extensions attempt to register at the same extension point |

### 7.9 Governance Errors (D12 — Release 3)

| Error Type | Error Code | Recoverable | When Raised |
|---|---|---|---|
| `AgentPave.GovernanceError` | `AGENTKIT_GOVERNANCE_ERROR` | No — fix governance issue | UNACCEPTABLE risk agent attempts operation, or HIGH_RISK agent deploys without approval |

---

## 8. Non-Functional Requirements

These requirements apply to every conformant AgentPave implementation. They are testable and must be included in the conformance test suite.

### 8.1 Performance

| Requirement | Target | Test Method |
|---|---|---|
| Agent declaration time | < 100ms (p99) | Time from declaration call to Agent ID assigned |
| State transition latency | < 50ms (p99) | Time from transition request to new state confirmed |
| Checkpoint write latency | < 200ms (p99) | Time from checkpoint initiation to durable write confirmed |
| Checkpoint resume latency | < 500ms (p99) | Time from failure detection to agent resumed at checkpoint |
| Structured log emission | < 10ms (p99) | Time from action completion to log entry written |
| OpenTelemetry span emission | < 10ms (p99) | Time from tool call completion to span emitted |

### 8.2 Reliability

| Requirement | Target | Test Method |
|---|---|---|
| Checkpoint durability | 100% | Checkpoint survives process restart, memory loss, and infrastructure fault |
| Resume success rate | > 99.9% | Agent resumes from checkpoint — not beginning — after any failure type |
| Memory expiry enforcement | 100% | Expired entries are never returned in any query |
| Error taxonomy coverage | 100% | Every error raised is a defined AgentPave error type |

### 8.3 Observability

| Requirement | Target | Test Method |
|---|---|---|
| Log coverage | 100% | Every agent action produces a structured log entry |
| Span coverage | 100% | Every tool call produces an OpenTelemetry span |
| Log field completeness | 100% | Every log entry contains all required fields |
| Trace ID consistency | 100% | All log entries and spans for one task share a single trace_id |

### 8.4 Interoperability

| Requirement | Target | Test Method |
|---|---|---|
| Cross-runtime agent definition | 100% | Same Agent Definition runs on LangGraph and MAF without modification |
| Error type consistency | 100% | Same operation raises same AgentPave error type on both runtimes |
| AgentResult consistency | 100% | Same action produces structurally identical AgentResult on both runtimes |

---

## 9. Spec Invariants

These invariants must hold true in every valid state of every conformant AgentPave implementation. They are machine-checkable. A conformant implementation MUST provide a method to verify each invariant programmatically.

```
INV-001: every_agent.agent_id IS UUID v4 AND IS NOT NULL AND IS UNIQUE
INV-002: every_agent.lifecycle_state IN {DECLARED, REGISTERED, RUNNING, PAUSED, RESUMED, RETIRED}
INV-003: every_agent.lifecycle_state == RETIRED → no_further_state_transitions_possible
INV-004: every_tool_call → mcp_used == True
INV-005: every_tool → integration_contract_exists == True
INV-006: every_tool_call_boundary → checkpoint_exists == True
INV-007: is_deterministic_task(task) → llm_not_invoked == True
INV-008: every_agent.token_budget IS NOT NULL AND > 0
INV-009: every_agent_action → structured_log_entry_emitted == True
INV-010: every_tool_call → otel_span_emitted == True
INV-011: every_agent.reliability_contract IS NOT NULL
INV-012: agent.effective_privilege <= invoking_user.privilege
INV-013: every_versioned_artifact.version matches SemVer regex
INV-014: every_agent_definition validates against agentpave-schema.json
INV-015: every_error raised IS subclass of AgentPaveError
```

---

## 10. MVP Scope

### 10.1 MVP Definition

An AgentPave MVP implementation is a conformant implementation that can:

1. Declare an agent with a UUID v4 Agent ID and all required metadata
2. Move an agent through its full lifecycle (DECLARED → REGISTERED → RUNNING → RETIRED)
3. Register and invoke at least one tool via MCP with a declared Integration Contract
4. Maintain Working Memory across a multi-step task
5. Persist Episodic Memory across sessions
6. Retrieve context from Semantic Memory via RAG
7. Emit structured logs and OpenTelemetry traces for every action
8. Checkpoint state at every tool call boundary and resume from that checkpoint on failure
9. Route Deterministic Tasks to typed code — never to the LLM
10. Enforce the agent's Reliability Contract and raise the correct error on violation

### 10.2 MVP Dimensions and Implementation Order

| Order | Dimension | Formal Dependencies | Rationale |
|---|---|---|---|
| 1 | Identity | None | Root — everything depends on it |
| 2 | Lifecycle | D1 | Gates all operations — nothing runs without it |
| 3 | Communication | D1, D2 | Enables action — agents do nothing without tools |
| 4 | Memory | D1, D2, D3 | Enables context — agents are stateless without it |
| 5 | Observability | D1, D2 | Enables trust — unobservable agents cannot be trusted |
| 6 | Reliability | D1–D5 | Enables production — agents that can't survive failure can't ship |

### 10.3 MVP Acceptance Criteria

All criteria must pass. No partial credit. All errors raised must be the correct AgentPave error type.

```
Criteria ID:  MVP-001
Stability:    Beta
Given:        A valid agent definition conforming to agentpave-schema.json
When:         The agent is declared via AgentPave.declare_agent(definition)
Then:         The agent is assigned a UUID v4 Agent ID
Pass:         agent.agent_id is a valid UUID v4, is non-null, and does not exist in the registry prior to this call
Fail:         agent_id is null, malformed, or already exists in registry
Error raised: AgentPave.DuplicateAgentIDError if agent_id already exists

Criteria ID:  MVP-002
Stability:    Beta
Given:        A REGISTERED agent
When:         The agent is invoked with a goal string
Then:         Agent transitions to RUNNING state and on_start lifecycle hook is invoked
Pass:         agent.lifecycle_state == RUNNING, on_start was called with the agent_id and goal
Fail:         State does not change, or on_start is not called
Error raised: AgentPave.InvalidStateTransitionError if agent is not in REGISTERED state

Criteria ID:  MVP-003
Stability:    Beta
Given:        A RUNNING agent with a registered MCP tool that has a valid Integration Contract
When:         The agent decides to invoke the tool
Then:         The tool is invoked via MCP, result is returned as AgentResult
Pass:         Tool call went through MCP (verifiable via MCP call log), AgentResult.status == "success", AgentResult.output is non-null
Fail:         Tool called directly (not via MCP), AgentResult is null, or agent halts
Error raised: AgentPave.ToolUnavailableError if tool unreachable after retry policy exhausted

Criteria ID:  MVP-004
Stability:    Beta
Given:        A RUNNING agent mid-task with at least one Checkpoint written
When:         An unhandled RuntimeError is raised in the agent
Then:         The agent resumes from the most recent Checkpoint — not from task start
Pass:         Agent resumes, task continues from checkpoint step N (not step 0), no steps 0..N-1 are repeated
Fail:         Agent restarts from step 0, or fails permanently without resume
Error raised: AgentPave.CheckpointError if checkpoint cannot be retrieved

Criteria ID:  MVP-005
Stability:    Beta
Given:        A RUNNING agent with a task classified as deterministic (e.g. "convert 100 USD to EUR at rate 0.92")
When:         The agent processes the task
Then:         The task is computed by typed code — no LLM call is made
Pass:         Zero LLM API calls are made during the deterministic task execution (verifiable via LLM call log)
Fail:         Any LLM API call is made during deterministic task execution
Error raised: AgentPave.DeterministicTaskLLMRouteError

Criteria ID:  MVP-006
Stability:    Beta
Given:        A RUNNING agent that invokes a tool
When:         The tool call completes (success or failure)
Then:         A structured log entry and an OpenTelemetry span are both emitted
Pass:
  log_entry.agent_id == agent.agent_id (UUID v4)
  log_entry.action_type == "tool_call"
  log_entry.timestamp is valid ISO 8601 UTC
  log_entry.trace_id is a valid UUID v4
  log_entry.result is a valid AgentResult (status, output, error, trace_id, timestamp all present)
  otel_span.agent_id == agent.agent_id
  otel_span.tool_name is non-null and matches the registered tool name
  otel_span.duration_ms > 0
  otel_span.trace_id == log_entry.trace_id  ← same trace_id links log and span
Fail:         Any required field is missing, null, or malformed; or trace_ids do not match
Error raised: AgentPave.ObservabilityBackendError if backend unavailable (fallback to stdout)
```

### 10.4 Post-MVP Dimensions

| Dimension | Target Release |
|---|---|
| Security | Release 2 |
| Economics | Release 2 |
| Testability | Release 2 |
| Developer Experience | Release 2 |
| Extensibility | Release 2 |
| Governance | Release 3 |
| Portability | Release 3 |

---

## 11. Extension Model

### 11.1 What Extensions Are

Extensions are optional, versioned AgentPave modules that add capabilities via defined extension points. They are plug-ins — not forks and not modifications of core.

Rules — no exceptions:
- Extensions MUST depend on AgentPave Core
- AgentPave Core MUST NOT import or depend on any extension module
- Extensions MUST attach only at the declared extension points listed in Section 5.3
- Extensions MUST be independently versioned using SemVer
- Adding or removing an extension MUST NOT require changes to any Agent Definition

### 11.2 Extension Conformance

A conformant extension must:
1. Declare: `extension_point` — which extension point it attaches to
2. Declare: `agentpave_core_min_version` — minimum core version required
3. Declare: its own `version` in SemVer
4. Implement: the abstract interface defined for its extension point in the core spec
5. Include: its own acceptance criteria
6. Pass: all its acceptance criteria in CI before release

### 11.3 Official Extensions

See Section 8.3 in the previous version — extension schedule unchanged. Release 1 ships with MVP.

---

## 12. Conformance

### 12.1 Conformance Levels

| Level | Description | Badge |
|---|---|---|
| **MVP Conformant** | All 15 spec invariants hold. All 6 MVP acceptance criteria pass. Dimensions 1–6 implemented. | `AgentPave MVP Conformant v1.1` |
| **Full Conformant** | All 13 dimensions implemented. All invariants hold. All acceptance criteria pass. | `AgentPave Conformant v1.1` |

### 12.2 Conformance Declaration File

Every conformant implementation MUST include this file at the repository root:

```yaml
# agentpave-conformance.yaml
agentpave_spec_version: "1.1"
implementation_name: "AgentPave-LangGraph"
implementation_version: "0.1.0"
conformance_level: "MVP"             # "MVP" or "Full"
dimensions_implemented:
  identity:       { version: "1.1", stability: "Beta", criteria_passed: true }
  lifecycle:      { version: "1.1", stability: "Beta", criteria_passed: true }
  communication:  { version: "1.1", stability: "Beta", criteria_passed: true }
  memory:         { version: "1.1", stability: "Beta", criteria_passed: true }
  observability:  { version: "1.1", stability: "Beta", criteria_passed: true }
  reliability:    { version: "1.1", stability: "Beta", criteria_passed: true }
invariants_verified: true
conformance_test_results: "conformance-results.json"
interoperability_verified: true      # true only if tested against the other reference implementation
last_verified: "2026-06-09"
```

### 12.3 Conformance Results Schema

`conformance-results.json` MUST conform to this schema:

```json
{
  "spec_version": "1.1",
  "implementation": "AgentPave-LangGraph",
  "run_timestamp": "2026-06-09T12:00:00Z",
  "summary": {
    "total": 6,
    "passed": 6,
    "failed": 0
  },
  "criteria": [
    {
      "id": "MVP-001",
      "status": "pass",
      "duration_ms": 42,
      "error": null
    }
  ],
  "invariants": [
    {
      "id": "INV-001",
      "status": "pass",
      "error": null
    }
  ],
  "nfr_results": [
    {
      "requirement": "checkpoint_write_latency_p99",
      "target_ms": 200,
      "actual_ms": 87,
      "status": "pass"
    }
  ]
}
```

### 12.4 Interoperability Test

To claim `interoperability_verified: true`, an implementation MUST:
1. Take the AgentPave interoperability test suite Agent Definition
2. Run it on both AgentPave-LangGraph and AgentPave-MAF
3. Verify that: Agent IDs are structurally identical, lifecycle transitions produce identical state sequences, tool call results produce structurally identical AgentResult objects, and error types are identical for identical failure scenarios

### 12.5 Conformance Verification Loop for Autonomous Agents

```
For each dimension in [D1, D2, D3, D4, D5, D6]:
    1. Read full dimension spec
    2. Verify all declared dependencies are passing
    3. Implement the dimension
    4. Run dimension acceptance criteria
    5. IF any fail → fix, re-run, do not proceed until all pass
    6. Verify all DoD checklist items
    7. Record results in conformance-results.json

After D1–D6:
    8.  Run all 15 invariant checks
    9.  Run NFR tests (performance and reliability targets)
    10. Run interoperability test if both runtimes available
    11. Produce agentpave-conformance.yaml
    12. Validate agentpave-conformance.yaml against conformance schema
    13. Only then declare MVP Conformance achieved
```

---

## 13. Document Index

### MVP Dimension Specs — Implement in This Order

| # | Document | Stability | Status |
|---|---|---|---|
| 1 | [spec/01-identity.md](spec/01-identity.md) | Beta | Draft |
| 2 | [spec/02-lifecycle.md](spec/02-lifecycle.md) | Beta | Draft |
| 3 | [spec/03-communication.md](spec/03-communication.md) | Beta | Draft |
| 4 | [spec/04-memory.md](spec/04-memory.md) | Beta | Draft |
| 5 | [spec/05-observability.md](spec/05-observability.md) | Beta | Draft |
| 6 | [spec/06-reliability.md](spec/06-reliability.md) | Beta | Draft |

### Post-MVP Dimension Specs

| # | Document | Stability | Status |
|---|---|---|---|
| 7 | [spec/07-security.md](spec/07-security.md) | Alpha | Planned |
| 8 | [spec/08-economics.md](spec/08-economics.md) | Alpha | Planned |
| 9 | [spec/09-testability.md](spec/09-testability.md) | Alpha | Planned |
| 10 | [spec/10-developer-experience.md](spec/10-developer-experience.md) | Alpha | Planned |
| 11 | [spec/11-extensibility.md](spec/11-extensibility.md) | Alpha | Planned |
| 12 | [spec/12-governance.md](spec/12-governance.md) | Alpha | Planned |
| 13 | [spec/13-portability.md](spec/13-portability.md) | Alpha | Planned |

### Supporting Documents

| Document | Purpose |
|---|---|
| [BASELINE.md](BASELINE.md) | AgentPave Baseline Structure v1.0 — the north star |
| [README.md](../README.md) | Repository homepage |
| [schema/agentpave-schema.json](../schema/agentpave-schema.json) | Machine-readable JSON Schema — authoritative for all data models |
| [agentpave-conformance.yaml](../agentpave-conformance.yaml) | Conformance declaration template |
| [conformance-results.json](../conformance-results.json) | Conformance test results (generated by CI) |

---

## 14. Versioning & Changelog

### 14.1 Spec Versioning Model

AgentPave Spec uses semantic versioning: `MAJOR.MINOR.PATCH`

| Change type | Version increment |
|---|---|
| Breaking change to any MUST requirement in any Stable dimension | MAJOR |
| New dimension, new invariant, new error type, or new extension point | MINOR |
| Clarification, correction, or non-breaking addition to any section | PATCH |

### 14.2 Breaking Change Policy

Breaking changes to this spec:
- MUST be announced at least one release cycle before taking effect
- MUST include a migration guide detailing exactly what must change in implementations
- MUST NOT be applied to any dimension at Stable stability without a MAJOR version increment
- MUST NOT be made to MVP dimensions (D1–D6) without explicit maintainer approval and a MAJOR increment

### 14.3 Changelog

| Version | Date | Change |
|---|---|---|
| 1.0.0 | June 2026 | Initial spec — MVP scope, 6 dimensions, master spec structure |
| 1.1.0 | June 2026 | Added: Standard Error Taxonomy (Section 7), Non-Functional Requirements (Section 8), Spec Invariants (Section 9), stability levels per dimension, formal dependency declarations, Definition of Done per dimension, AgentResult type definition, conformance results schema, interoperability test definition, machine-readable schema reference, rollback guidance for autonomous agents, spec linting rules |
| 1.1.1 | June 2026 | Fixed: Added missing error types to taxonomy: ToolNotFoundError, ToolTimeoutError, CircuitBreakerOpenError, HumanInputTimeoutError (D3), MemoryBudgetError (D4) — discovered via cross-dimension automated validation |
| 1.1.2 | June 2026 | Added: Post-MVP error taxonomy sections 7.7–7.9: Security errors (D7), Extensibility errors (D11), Governance errors (D12). Added post-MVP dimension specs D7–D13 to Document Index |

---

*AgentPave Specification v1.1 — Build agents, not plumbing.*
