# AgentPave Dimension 8 — Economics

**Spec version:** 1.1  
**Stability:** Alpha  
**Depends on:** D1 (Identity), D2 (Lifecycle), D3 (Communication), D5 (Observability)  
**Required by:** Nothing in Release 2  
**Owner:** AgentPave Core  
**Target Release:** Release 2  

---

## Table of Contents

1. [Purpose & Outcomes](#1-purpose--outcomes)
2. [Scope Boundary](#2-scope-boundary)
3. [Prior Decisions](#3-prior-decisions)
4. [Definitions](#4-definitions)
5. [Data Models](#5-data-models)
6. [Canonical Example](#6-canonical-example)
7. [Interfaces & Contracts](#7-interfaces--contracts)
8. [Behaviour Specification](#8-behaviour-specification)
9. [Three-Tier Boundary System](#9-three-tier-boundary-system)
10. [Error Handling](#10-error-handling)
11. [Acceptance Criteria](#11-acceptance-criteria)
12. [Definition of Done](#12-definition-of-done)
13. [Task Breakdown](#13-task-breakdown)
14. [Anti-Patterns](#14-anti-patterns)
15. [Reference Implementation Notes](#15-reference-implementation-notes)
16. [Open Questions](#16-open-questions)
17. [Changelog](#17-changelog)

---

## 1. Purpose & Outcomes

### 1.1 Purpose

Agents make 3–10x more LLM calls than simple chatbots — a single user request can trigger planning, tool selection, execution, verification, and response generation, easily consuming 5x the token budget of a direct chat completion. An unconstrained agent solving a software engineering task can cost $5–8 per task in API fees alone.

D8 makes cost a first-class engineering concern. It delivers four capabilities:
1. **Token budget enforcement** — hard limits, soft alerts, hierarchical budgets
2. **Model routing** — route simple tasks to cheaper models, complex tasks to frontier models
3. **Prompt caching** — 50–90% savings on cached input tokens
4. **Semantic caching** — reuse responses for semantically equivalent queries

### 1.2 Production Consequence of Getting This Wrong

Unoptimised production agents can cost $10–100+ per session due to long context windows and multi-step loops. Using strategies like model routing and prompt caching can reduce this spend by over 80%. A hierarchical architecture using budget models for worker agents and frontier models only for the lead orchestrator can achieve 97.7% of full-frontier accuracy at ~61% of the cost. A single runaway agent loop that ignores budget limits can exhaust an organisation's entire monthly LLM spend in hours.

---

## 2. Scope Boundary

### 2.1 In Scope

- Token budget declaration, tracking, and enforcement per agent, task, and day
- Soft alerts (50% and 80% of budget) and hard limits (100%)
- Model routing rules: complexity-based routing to appropriate model tiers
- Prompt caching hooks: stable prefix preservation for provider-side caching
- Semantic caching: reuse of responses for semantically equivalent queries
- Cost attribution: per-task and per-agent cost tracking
- Budget hierarchy: org → team → agent → task

### 2.2 Out of Scope

- LLM gateway implementation (LiteLLM, Portkey etc.) — these are pluggable backends
- Payment and billing integration — that is enterprise deployment concern
- Model fine-tuning cost tracking — out of scope for MVP
- Multi-cloud cost aggregation — that is FinOps tooling concern

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **Budget enforcement is in core, not extension** | An unconstrained agent is an existential business risk. Budget enforcement cannot be optional — it is declared in AgentDefinition (D1) and enforced here. |
| **Hierarchical budgets** | Org-level budgets prevent any single agent or team from monopolising resources. Team budgets enable cost attribution. Agent budgets provide per-agent guardrails. |
| **Semantic caching uses vector similarity** | Exact-match caching misses semantically equivalent queries. Semantic caching (embedding + cosine similarity) achieves up to 73% cost reduction in high-repetition workloads. |
| **Model routing is complexity-based** | Simple tasks (classification, formatting) routed to cheap models; complex reasoning to frontier models. This achieves 97.7% accuracy at 61% cost for multi-agent hierarchies. |
| **Prompt caching is prefix-based** | Provider-side prompt caching works by matching the prefix of the prompt. AgentPave structures prompts with stable prefixes (system prompt first) to maximise cache hits. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **Token Budget** | The maximum number of LLM tokens an agent may consume. Declared in AgentDefinition. Three scopes: per_task, per_day, per_lifetime. |
| **Budget Hierarchy** | Four-level budget structure: Organisation → Team → Agent → Task. Lower levels cannot exceed higher-level caps. |
| **Soft Alert Threshold** | The percentage of budget consumed that triggers a warning (not a block). Defaults: 50% and 80%. |
| **Hard Limit** | The 100% budget threshold that triggers immediate task suspension and raises BudgetExceededError. |
| **Model Tier** | Classification of LLM models by capability and cost: ECONOMY (simple tasks, cheapest), STANDARD (general reasoning), FRONTIER (complex reasoning, most expensive). |
| **Model Router** | The component that classifies task complexity and selects the appropriate model tier. |
| **Prompt Caching** | Provider-side caching of stable prompt prefixes. AgentPave structures prompts to maximise cache hits by placing stable content (system prompt, instructions) before variable content. |
| **Semantic Cache** | A local cache of LLM responses keyed by embedding vector similarity. Returns cached responses for semantically equivalent queries without an LLM API call. |
| **Cache Hit Rate** | The percentage of LLM calls served from the semantic cache. Target: > 30% in production. |
| **Cost Attribution** | The process of tracking and reporting token spend per agent, per task, and per team. |
| **Prompt Compression** | The process of removing low-information tokens from long prompts using a small, fast model (LLMLingua pattern). Used when prompts approach the context window limit. |

---

## 5. Data Models

### 5.1 BudgetHierarchy

```python
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum


class BudgetScope(str, Enum):
    ORG   = "org"
    TEAM  = "team"
    AGENT = "agent"
    TASK  = "task"


class BudgetLevel(BaseModel):
    """A single level in the budget hierarchy."""
    scope: BudgetScope
    entity_id: str = Field(..., description="Org ID, team ID, agent_id, or task_id.")
    per_day_tokens: int = Field(..., gt=0)
    per_task_tokens: Optional[int] = Field(None, gt=0, description="Only for AGENT and TASK scope.")
    soft_alert_pct_50: bool = Field(default=True)
    soft_alert_pct_80: bool = Field(default=True)
    current_day_usage: int = Field(default=0, ge=0)
    current_task_usage: int = Field(default=0, ge=0)

    model_config = {"extra": "forbid"}
```

### 5.2 ModelTier and ModelRoutingRule

```python
class ModelTier(str, Enum):
    ECONOMY  = "economy"   # Simple tasks: classification, formatting, extraction
    STANDARD = "standard"  # General reasoning, moderate complexity
    FRONTIER = "frontier"  # Complex multi-step reasoning, mission-critical


class TaskComplexity(str, Enum):
    SIMPLE   = "simple"    # → ECONOMY model
    MODERATE = "moderate"  # → STANDARD model
    COMPLEX  = "complex"   # → FRONTIER model


class ModelRoutingRule(BaseModel):
    """Maps task complexity to a specific model."""
    complexity: TaskComplexity
    model_id: str = Field(
        ...,
        description=(
            "The model identifier to use for this complexity tier. "
            "Examples: 'claude-haiku-4-5', 'claude-sonnet-4-6', 'claude-opus-4-6'"
        )
    )
    max_tokens: int = Field(..., gt=0, description="Max tokens for this tier.")
    cost_per_1k_input_tokens: float = Field(..., ge=0.0)
    cost_per_1k_output_tokens: float = Field(..., ge=0.0)

    model_config = {"extra": "forbid"}


class ModelRoutingConfig(BaseModel):
    """Complete model routing configuration for an agent."""
    rules: list[ModelRoutingRule] = Field(
        ...,
        min_length=1,
        description="At least one routing rule required."
    )
    complexity_classifier: str = Field(
        default="heuristic",
        description=(
            "Method for classifying task complexity. "
            "'heuristic': based on token count and keywords. "
            "'llm': use a cheap model to classify (adds one call). "
            "'declared': agent declares complexity explicitly per task."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.3 PromptCacheConfig

```python
class PromptCacheConfig(BaseModel):
    """
    Configuration for provider-side prompt caching.
    AgentPave structures prompts to maximise stable prefix length.
    """
    enabled: bool = Field(default=True)
    stable_prefix_fields: list[str] = Field(
        default_factory=lambda: ["system_prompt", "tool_definitions", "instructions"],
        description=(
            "Fields placed at the start of the prompt (before variable content) "
            "to maximise the stable prefix for provider caching."
        )
    )
    min_cacheable_tokens: int = Field(
        default=1024,
        gt=0,
        description=(
            "Minimum token count for the stable prefix to be worth caching. "
            "Prompts shorter than this are not submitted for caching."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.4 SemanticCacheConfig

```python
class SemanticCacheConfig(BaseModel):
    """Configuration for semantic response caching."""
    enabled: bool = Field(default=True)
    similarity_threshold: float = Field(
        default=0.95,
        ge=0.8,
        le=1.0,
        description=(
            "Cosine similarity threshold for cache hit. "
            "0.95 = very similar queries get cached response. "
            "Lower values increase hit rate but risk returning wrong responses."
        )
    )
    ttl_seconds: int = Field(
        default=3600,
        gt=0,
        description="Cache entry TTL. Entries older than this are evicted."
    )
    max_entries: int = Field(
        default=10000,
        gt=0,
        description="Maximum cache size. LRU eviction when full."
    )
    excluded_action_types: list[str] = Field(
        default_factory=lambda: ["tool:database_write:invoke", "tool:email:send"],
        description=(
            "Action types excluded from semantic caching. "
            "Write operations and side-effectful calls must never be cached."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.5 CostRecord

```python
class CostRecord(BaseModel):
    """A single cost attribution record for one LLM call."""
    record_id: str = Field(..., description="UUID v4.")
    agent_id: str = Field(...)
    task_id: str = Field(...)
    trace_id: str = Field(...)
    model_id: str = Field(...)
    model_tier: ModelTier = Field(...)
    input_tokens: int = Field(..., ge=0)
    output_tokens: int = Field(..., ge=0)
    cached_input_tokens: int = Field(default=0, ge=0, description="Tokens served from prompt cache.")
    semantic_cache_hit: bool = Field(default=False)
    estimated_cost_usd: float = Field(..., ge=0.0)
    recorded_at: str = Field(..., description="ISO 8601 UTC.")

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Budget Declaration in AgentDefinition

```python
# TokenBudget is already in AgentDefinition (D1)
# D8 extends enforcement with hierarchy and alerting

token_budget = TokenBudget(
    per_task=2000,
    per_day=500000,
    per_lifetime=None
)

model_routing_config = ModelRoutingConfig(
    rules=[
        ModelRoutingRule(
            complexity=TaskComplexity.SIMPLE,
            model_id="claude-haiku-4-5",
            max_tokens=1000,
            cost_per_1k_input_tokens=0.00025,
            cost_per_1k_output_tokens=0.00125
        ),
        ModelRoutingRule(
            complexity=TaskComplexity.MODERATE,
            model_id="claude-sonnet-4-6",
            max_tokens=4000,
            cost_per_1k_input_tokens=0.003,
            cost_per_1k_output_tokens=0.015
        ),
        ModelRoutingRule(
            complexity=TaskComplexity.COMPLEX,
            model_id="claude-opus-4-6",
            max_tokens=8000,
            cost_per_1k_input_tokens=0.015,
            cost_per_1k_output_tokens=0.075
        ),
    ],
    complexity_classifier="heuristic"
)
```

---

## 7. Interfaces & Contracts

### 7.1 BudgetEnforcer

```python
from abc import ABC, abstractmethod


class BudgetEnforcer(ABC):
    """
    Tracks and enforces token budgets across all scopes.
    Called before every LLM call.
    """

    @abstractmethod
    def check_budget(
        self,
        agent_id: str,
        task_id: str,
        estimated_tokens: int
    ) -> None:
        """
        Check if estimated_tokens can be consumed without exceeding any budget level.

        Raises:
            AgentPave.BudgetExceededError: if any budget level would be exceeded.
                context = {"scope": str, "entity_id": str, "available": int, "requested": int}
        Emits:
            Soft alert event via AgentObserver when 50% or 80% threshold crossed.
        """
        ...

    @abstractmethod
    def record_usage(
        self,
        agent_id: str,
        task_id: str,
        trace_id: str,
        cost_record: CostRecord
    ) -> None:
        """
        Record actual token consumption after an LLM call completes.
        Updates running totals at all budget levels.
        """
        ...

    @abstractmethod
    def get_remaining(self, agent_id: str, task_id: str) -> dict:
        """
        Return remaining budget at all scopes for this agent+task.
        Returns: {"per_task": int, "per_day": int, "per_lifetime": Optional[int]}
        """
        ...
```

### 7.2 ModelRouter

```python
class ModelRouter(ABC):
    """Routes LLM calls to the appropriate model tier based on task complexity."""

    @abstractmethod
    def classify_complexity(self, task_input: str, context: dict) -> TaskComplexity:
        """
        Classify task complexity as SIMPLE, MODERATE, or COMPLEX.
        Uses the method declared in ModelRoutingConfig.complexity_classifier.
        Never raises — returns FRONTIER on classification failure (safe default).
        """
        ...

    @abstractmethod
    def select_model(
        self,
        agent_id: str,
        task_input: str,
        context: dict
    ) -> ModelRoutingRule:
        """
        Select the appropriate model for this task.
        Returns the ModelRoutingRule for the classified complexity tier.
        """
        ...
```

### 7.3 SemanticCache

```python
class SemanticCache(ABC):
    """
    Caches LLM responses keyed by embedding vector similarity.
    Write operations and side-effectful calls must never be cached.
    """

    @abstractmethod
    def get(self, query_embedding: list[float], action_type: str) -> Optional[str]:
        """
        Look up a cached response for a semantically similar query.

        Returns:
            Cached response string if similarity >= threshold AND action_type not excluded.
            None if no cache hit.
        """
        ...

    @abstractmethod
    def set(
        self,
        query_embedding: list[float],
        response: str,
        action_type: str,
        ttl_seconds: int
    ) -> None:
        """
        Store a response in the cache.
        No-op if action_type is in excluded_action_types.
        """
        ...

    @abstractmethod
    def get_stats(self) -> dict:
        """
        Return cache statistics: hit_count, miss_count, hit_rate_pct, entry_count.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 Budget Enforcement

**THE SYSTEM SHALL** call `BudgetEnforcer.check_budget()` before every LLM call.

**WHEN** any budget level (task, day, lifetime) would be exceeded **THE SYSTEM SHALL** raise `AgentPave.BudgetExceededError` and block the LLM call.

**THE SYSTEM SHALL** emit a soft alert observability event when consumption crosses 50% of any budget level.

**THE SYSTEM SHALL** emit a soft alert observability event when consumption crosses 80% of any budget level.

**THE SYSTEM SHALL** record actual consumption via `BudgetEnforcer.record_usage()` after every LLM call completes.

### 8.2 Model Routing

**THE SYSTEM SHALL** classify task complexity before every LLM call when `ModelRoutingConfig` is declared.

**THE SYSTEM SHALL** select the model declared for the classified complexity tier.

**WHEN** complexity classification fails **THE SYSTEM SHALL** default to the FRONTIER tier (safe, not cheap).

### 8.3 Semantic Caching

**THE SYSTEM SHALL** check the semantic cache before every LLM call.

**WHEN** a cache hit is found (similarity >= threshold, action_type not excluded) **THE SYSTEM SHALL** return the cached response without making an LLM API call.

**THE SYSTEM SHALL NOT** cache responses for action_types in `excluded_action_types` — write operations must never return cached responses.

**THE SYSTEM SHALL** record `semantic_cache_hit=True` in the `CostRecord` when a cached response is used.

### 8.4 Prompt Caching

**THE SYSTEM SHALL** structure prompts with stable content (system prompt, tool definitions) first to maximise provider-side cache prefix length.

**THE SYSTEM SHALL** record `cached_input_tokens` in `CostRecord` when provider reports cached token usage.

---

## 9. Three-Tier Boundary System

### ALWAYS
- Call BudgetEnforcer.check_budget() before every LLM call
- Record actual usage via BudgetEnforcer.record_usage() after every LLM call
- Emit soft alerts at 50% and 80% budget consumption
- Raise BudgetExceededError at 100% — never silently allow overage
- Never cache responses for write operations or side-effectful calls
- Default to FRONTIER model tier on classification failure

### ASK FIRST
- Granting a temporary budget increase mid-task — confirm with operator
- Lowering similarity_threshold below 0.90 — high risk of returning wrong cached responses
- Disabling semantic caching for an agent — confirm and document reason

### NEVER
- Allow LLM calls without budget check
- Cache responses for write operations (database_write, email_send, etc.)
- Silently absorb budget overages — always raise BudgetExceededError
- Route a COMPLEX task to ECONOMY model — accuracy risk is too high

---

## 10. Error Handling

| Scenario | Error Type | Recoverable | Required context |
|---|---|---|---|
| Any budget level exceeded | `AgentPave.BudgetExceededError` | No — task must halt | `scope, entity_id, available, requested` |
| Budget at 50% | Observability event (not error) | N/A | `scope, entity_id, pct_used` |
| Budget at 80% | Observability event (not error) | N/A | `scope, entity_id, pct_used` |
| Complexity classification fails | No error — default to FRONTIER | N/A | Logged as warning |

---

## 11. Acceptance Criteria

```
Criteria ID:  D8-001
Stability:    Alpha
Given:        Agent with per_task token budget of 1000
              Current task has consumed 950 tokens
When:         LLM call estimated to consume 100 more tokens is attempted
Then:         AgentPave.BudgetExceededError raised
              LLM call is blocked
Pass:         BudgetExceededError raised, context["available"]==50, context["requested"]==100
Fail:         LLM call proceeds despite budget exceeded
Error raised: AgentPave.BudgetExceededError

---

Criteria ID:  D8-002
Stability:    Alpha
Given:        Agent with per_day budget 1000, currently consumed 510 (51%)
When:         BudgetEnforcer.check_budget() is called
Then:         Soft alert event emitted (50% crossed)
              Call proceeds (not blocked — 51% < 100%)
Pass:         Soft alert observability event emitted, call not blocked
Fail:         No alert emitted, or call blocked
Error raised: None

---

Criteria ID:  D8-003
Stability:    Alpha
Given:        ModelRoutingConfig with SIMPLE→haiku, COMPLEX→opus
              Task input: "What is 2+2?" (SIMPLE complexity)
When:         ModelRouter.select_model(agent_id, task_input, {}) is called
Then:         ModelRoutingRule for SIMPLE tier returned (haiku)
Pass:         rule.model_id == "claude-haiku-4-5"
Fail:         Frontier model selected for simple task
Error raised: None

---

Criteria ID:  D8-004
Stability:    Alpha
Given:        SemanticCache with similarity_threshold=0.95
              Cached response for "What is the return policy?" (embedding E1)
When:         SemanticCache.get(embedding_for("How do I return an item?"), "tool:search:invoke")
              (cosine similarity between E1 and new embedding >= 0.95)
Then:         Cached response returned without LLM API call
Pass:         Cached response returned, LLM call log shows zero new calls
Fail:         LLM called despite cache hit
Error raised: None

---

Criteria ID:  D8-005
Stability:    Alpha
Given:        SemanticCache with excluded_action_types=["tool:database_write:invoke"]
              Cached response exists for a write operation
When:         SemanticCache.get(embedding, "tool:database_write:invoke")
Then:         None returned (cache miss enforced for write operations)
Pass:         None returned, write operation proceeds to LLM/handler
Fail:         Cached response returned for write operation
Error raised: None

---

Criteria ID:  D8-006
Stability:    Alpha
Given:        Agent completes an LLM call consuming 500 input tokens (200 cached) and 100 output
When:         BudgetEnforcer.record_usage(agent_id, task_id, trace_id, cost_record) is called
Then:         Cost record stored with correct token counts
              Running total updated: task_usage += 600, day_usage += 600
Pass:         cost_record.input_tokens==500, cost_record.cached_input_tokens==200,
              cost_record.output_tokens==100, budget totals updated correctly
Fail:         Incorrect token counts stored, or budgets not updated
Error raised: None
```

---

## 12. Definition of Done

- [ ] `BudgetScope`, `ModelTier`, `TaskComplexity` enums implemented
- [ ] `BudgetLevel`, `ModelRoutingRule`, `ModelRoutingConfig` models implemented
- [ ] `PromptCacheConfig`, `SemanticCacheConfig`, `CostRecord` models implemented
- [ ] `BudgetEnforcer` abstract + concrete implemented
- [ ] `ModelRouter` abstract + concrete (heuristic classifier) implemented
- [ ] `SemanticCache` abstract + concrete (Redis/in-memory) implemented
- [ ] BudgetEnforcer.check_budget() called before every LLM call
- [ ] BudgetEnforcer.record_usage() called after every LLM call
- [ ] Soft alerts emitted at 50% and 80%
- [ ] Write operations excluded from semantic cache
- [ ] Prompt structure optimised for stable prefix caching
- [ ] All 6 acceptance criteria pass: D8-001 through D8-006

---

## 13. Task Breakdown

```
Task D8-T1: Implement data models (BudgetLevel, ModelRoutingRule, PromptCacheConfig,
            SemanticCacheConfig, CostRecord)
Task D8-T2: Implement BudgetEnforcer abstract + concrete
            (hierarchical budget tracking with alerts)
Task D8-T3: Wire BudgetEnforcer into LLM call path (before and after every call)
Task D8-T4: Implement ModelRouter abstract + concrete (heuristic complexity classifier)
Task D8-T5: Implement SemanticCache abstract + concrete (in-memory with cosine similarity)
Task D8-T6: Wire SemanticCache into LLM call path (check before, set after)
Task D8-T7: Implement prompt structure optimisation for stable prefix caching
Task D8-T8: Run all 6 acceptance criteria
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — No budget limits in production**
An agent without budget limits is a runaway process waiting to happen. A coding bug causing an infinite loop can exhaust a month's LLM budget in hours.

**Anti-Pattern 2 — Same model for everything**
Using frontier models for simple classification tasks wastes 15–50x the budget compared to economy models. For multi-agent systems, hierarchical routing achieves near-identical accuracy at 61% of cost.

**Anti-Pattern 3 — Caching write operations**
Returning a cached "insert successful" response for a database write that was never actually executed causes silent data corruption. Write operations and side-effectful calls must never be cached.

**Anti-Pattern 4 — Low similarity threshold**
A semantic cache threshold of 0.80 will return cached responses for queries that are merely in the same topic area — not semantically equivalent. This produces incorrect responses that look correct. Use 0.95 as the minimum for production.

**Anti-Pattern 5 — No cost attribution**
Without CostRecord tracking per agent and task, cost optimisation is guesswork. You cannot reduce what you cannot measure.

---

## 15. Reference Implementation Notes

### 15.1 AgentPave-LangGraph
- Budget enforcement: custom middleware wrapping LangGraph's LLM node
- Model routing: LangChain's model switching via configurable LLM wrapper
- Semantic cache: Redis with `redis-py` + `numpy` cosine similarity
- Cost tracking: LangSmith's built-in cost tracking as observability backend

### 15.2 AgentPave-MAF
- Budget enforcement: MAF middleware hook before every model call
- Semantic cache: Azure Cache for Redis
- Model routing: Azure AI model catalog with tier-based routing rules

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should budget alerts be emitted via AgentObserver (D5) or a dedicated cost event? | No — use AgentObserver |
| OQ-2 | Should cost estimates be exact or approximate before the LLM call? | No — approximate is acceptable |
| OQ-3 | Should the semantic cache be shared across agent instances of the same type? | Yes — privacy and data isolation implications |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec |

---

*AgentPave Dimension 8 — Economics — v1.1*
