# AgentKit Dimension 4 — Memory

**Spec version:** 1.1  
**Stability:** Beta  
**Depends on:** D1 (Identity), D2 (Lifecycle), D3 (Communication)  
**Required by:** D6 (Reliability)  
**Owner:** AgentKit Core  

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

Memory is what transforms an agent from a stateless responder into a persistent, context-aware system. Without memory, every agent task starts blind — no knowledge of prior interactions, no learned patterns, no accumulated domain knowledge.

In 2026, memory is treated as a dedicated architectural component separate from the model's context window — not just a longer prompt. Production agents need at least three distinct memory layers, each with different storage and retrieval requirements.

AgentKit defines four memory types, each with a typed interface, a defined lifecycle, and mandatory governance declarations. This dimension also draws the critical distinction between RAG and memory — a distinction most frameworks conflate, causing silent retrieval errors.

### 1.2 Production Consequence of Getting This Wrong

A vector database stores embeddings without timestamps that affect retrieval. A six-month-old fact and a yesterday fact are equally "similar" to a query if their text matches. For domains where facts change (pricing, policies, customer status), this produces silent retrieval errors. Memory without staleness enforcement produces agents that confidently act on expired information.

Additional failure modes:
- Working memory overflow: injecting all memory into every LLM call without selection causes context explosion and degraded reasoning
- Memory privacy violations: one agent reading another agent's episodic memory exposes sensitive interaction history
- Missing expiry policies: unbounded episodic memory grows without limit, increasing retrieval latency and cost over time

---

## 2. Scope Boundary

### 2.1 In Scope

- Defining the four memory type interfaces: Working, Episodic, Semantic, Procedural
- Defining memory entry schema with timestamps and expiry
- Defining memory lifecycle: create, read, update, expire, delete
- Defining memory ownership and privacy declarations
- Enforcing staleness and expiry policies
- Distinguishing RAG (retrieval pattern) from memory (persistent state)
- Defining the memory configuration schema per agent

### 2.2 Out of Scope

- Memory backend implementations (Redis, Pinecone, pgvector etc.) — these are pluggable
- Memory consolidation (async between-session processing) — that is AgentKit-Memory extension
- Semantic caching of tool call results — that is AgentKit-Cache extension
- Cross-agent shared memory governance — that is D12 (Governance)
- Memory encryption at rest — that is D6 (Security)

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **Four memory types, not one** | The most common production combination is episodic memory for session history plus semantic memory as the organisation's knowledge authority. Working and procedural memory address different problems — all four are needed for a complete agent. |
| **RAG is a retrieval pattern, not a memory type** | RAG retrieves from a static knowledge base; memory grows and changes with every interaction. Conflating them causes agents to treat stale retrieved facts as live memory — producing silent errors. |
| **Memory entries have mandatory timestamps** | A vector database without timestamps cannot distinguish a six-month-old fact from yesterday's — causing silent retrieval errors in time-sensitive domains. |
| **Expiry policies are mandatory** | Unbounded memory grows without limit. TTL-based or event-based expiry is required for production memory hygiene. |
| **Ownership and privacy are per-store declarations** | Agents must declare who owns each memory store and at what privacy level it operates. This enables governance, data residency compliance, and multi-agent access control. |
| **Memory interfaces are technology-agnostic** | The spec defines interfaces, not implementations. Redis, DynamoDB, Pinecone, pgvector, Neo4j — all are valid backends. The interface is stable; the backend is pluggable. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **Working Memory** | The agent's in-context memory for the current task. Held in the active LLM context window. Ephemeral — lost when the task ends. Never persisted externally. |
| **Episodic Memory** | Persistent, agent-owned records of past interactions, sessions, and task outcomes. Stored externally. Survives task and session boundaries. Subject to expiry policy. |
| **Semantic Memory** | Knowledge stored as vector embeddings, retrieved by semantic similarity. Represents facts, documents, and domain knowledge. Grows through addition — entries are not typically deleted unless expired. |
| **Procedural Memory** | Learned patterns, instructions, and few-shot examples that shape agent behaviour. Stored as prompt templates, example libraries, or instruction sets. Typically read-only at runtime. |
| **RAG** | Retrieval Augmented Generation. A retrieval pattern that fetches relevant context from Semantic Memory or an external knowledge base before an LLM call. RAG is a read operation against Semantic Memory — it is not itself a memory type. |
| **Memory Entry** | A single stored unit of memory across any type. Has: `entry_id`, `agent_id`, `content`, `created_at`, `expires_at` (optional), `metadata`. |
| **Expiry Policy** | The rule governing when a memory entry becomes invalid. Three types: TTL-based (expires N seconds after creation), event-based (expires on a specific event), or never (no expiry — requires explicit justification). |
| **Privacy Level** | The access restriction on a memory store. One of: PRIVATE (agent-only), SHARED_OWNER (agents with same owner), ORG (all agents in organisation), PUBLIC. |
| **Memory Owner** | The Agent ID that owns a memory store. Only the owning agent may write to the store. Other agents may read only if privacy_level permits. |
| **Staleness** | The condition where a memory entry's content no longer accurately reflects the current state of the world. Staleness is different from expiry: an entry may be expired but not stale, or stale but not yet expired. |
| **Context Window Budget** | The maximum number of tokens the agent may use for memory context in a single LLM call. Working memory must be managed to stay within this budget. |

---

## 5. Data Models

### 5.1 MemoryEntry

```python
from pydantic import BaseModel, Field
from typing import Any, Optional
from enum import Enum


class MemoryType(str, Enum):
    WORKING    = "working"
    EPISODIC   = "episodic"
    SEMANTIC   = "semantic"
    PROCEDURAL = "procedural"


class PrivacyLevel(str, Enum):
    PRIVATE      = "PRIVATE"       # Only the owning agent
    SHARED_OWNER = "SHARED_OWNER"  # Agents with the same owner
    ORG          = "ORG"           # All agents in the organisation
    PUBLIC       = "PUBLIC"        # No restriction


class ExpiryPolicyType(str, Enum):
    TTL         = "TTL"         # Expires N seconds after creation
    EVENT_BASED = "EVENT_BASED" # Expires when a named event occurs
    NEVER       = "NEVER"       # No expiry (requires justification)


class ExpiryPolicy(BaseModel):
    policy_type: ExpiryPolicyType = Field(...)
    ttl_seconds: Optional[int] = Field(
        None,
        gt=0,
        description="Required when policy_type=TTL. Seconds until expiry from created_at."
    )
    event_name: Optional[str] = Field(
        None,
        description="Required when policy_type=EVENT_BASED. Name of event that triggers expiry."
    )
    justification: Optional[str] = Field(
        None,
        min_length=10,
        description="Required when policy_type=NEVER. Why this entry never expires."
    )

    def model_post_init(self, __context) -> None:
        if self.policy_type == ExpiryPolicyType.TTL and self.ttl_seconds is None:
            raise ValueError("ttl_seconds required when policy_type is TTL.")
        if self.policy_type == ExpiryPolicyType.EVENT_BASED and self.event_name is None:
            raise ValueError("event_name required when policy_type is EVENT_BASED.")
        if self.policy_type == ExpiryPolicyType.NEVER and not self.justification:
            raise ValueError("justification required when policy_type is NEVER.")

    model_config = {"extra": "forbid"}


class MemoryEntry(BaseModel):
    """
    A single stored unit of memory.
    Used across Episodic, Semantic, and Procedural memory types.
    Working memory entries are ephemeral and are not stored as MemoryEntry objects.
    """
    entry_id: str = Field(
        ...,
        description="UUID v4. Unique identifier for this entry."
    )
    agent_id: str = Field(
        ...,
        description="UUID v4. The owning agent's ID."
    )
    memory_type: MemoryType = Field(...)
    content: Any = Field(
        ...,
        description=(
            "The stored content. Type depends on memory_type: "
            "episodic: str or dict (interaction record); "
            "semantic: str or dict (fact or document chunk); "
            "procedural: str (instruction or few-shot example)."
        )
    )
    embedding: Optional[list[float]] = Field(
        None,
        description=(
            "Vector embedding of content. Required for Semantic memory entries. "
            "Optional for Episodic. Not used for Procedural."
        )
    )
    metadata: dict = Field(
        default_factory=dict,
        description=(
            "Arbitrary structured metadata for filtering and retrieval. "
            "Examples: {'task_id': '...', 'source': 'tool_call', 'tags': ['finance']}"
        )
    )
    created_at: str = Field(..., description="ISO 8601 UTC. When this entry was created.")
    expires_at: Optional[str] = Field(
        None,
        description=(
            "ISO 8601 UTC. When this entry expires. "
            "None only when expiry_policy.policy_type == NEVER."
        )
    )
    expiry_policy: ExpiryPolicy = Field(...)
    is_expired: bool = Field(
        default=False,
        description=(
            "True when this entry has passed its expires_at timestamp. "
            "Set by the memory store on read — not stored as a persistent field."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.2 MemoryStoreConfig

```python
class MemoryStoreConfig(BaseModel):
    """
    Configuration for a memory store. Declared per-agent in the AgentDefinition.
    Mandatory governance declarations.
    """
    store_id: str = Field(..., description="UUID v4. Unique ID for this memory store.")
    memory_type: MemoryType = Field(...)
    owner_agent_id: str = Field(
        ...,
        description="UUID v4. The agent that owns this store. Only this agent may write."
    )
    privacy_level: PrivacyLevel = Field(
        ...,
        description="Access restriction level."
    )
    default_expiry_policy: ExpiryPolicy = Field(
        ...,
        description=(
            "Default expiry policy for entries that don't specify their own. "
            "All entries inherit this policy unless overridden at write time."
        )
    )
    max_entries: Optional[int] = Field(
        None,
        gt=0,
        description=(
            "Maximum number of entries in this store. "
            "None means no limit (not recommended for production). "
            "When max_entries is reached: oldest entries are expired first (LRU eviction)."
        )
    )
    backend_type: str = Field(
        ...,
        description=(
            "Identifier for the backing store technology. "
            "Examples: 'redis', 'dynamodb', 'pinecone', 'pgvector', 'chroma', 'in_memory'. "
            "AgentKit validates this field only for known backends."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.3 MemoryQuery

```python
class MemoryQuery(BaseModel):
    """
    A query against a memory store.
    Supports exact lookup (by entry_id or metadata filter) and semantic search.
    """
    query_id: str = Field(..., description="UUID v4. For tracing.")
    agent_id: str = Field(..., description="UUID v4. The querying agent.")
    store_id: str = Field(..., description="UUID v4. The store to query.")
    query_text: Optional[str] = Field(
        None,
        description=(
            "Natural language query for semantic similarity search. "
            "Used for Semantic and Episodic memory. "
            "Mutually exclusive with entry_ids."
        )
    )
    entry_ids: Optional[list[str]] = Field(
        None,
        description="List of UUID v4 entry IDs for exact lookup. Mutually exclusive with query_text."
    )
    metadata_filter: dict = Field(
        default_factory=dict,
        description="Key-value filters applied to entry metadata."
    )
    top_k: int = Field(
        default=5,
        ge=1,
        le=100,
        description="Maximum number of results to return for semantic queries."
    )
    include_expired: bool = Field(
        default=False,
        description=(
            "Whether to include expired entries in results. "
            "Default: False. Expired entries are excluded unless explicitly requested."
        )
    )

    def model_post_init(self, __context) -> None:
        if self.query_text is not None and self.entry_ids is not None:
            raise ValueError("query_text and entry_ids are mutually exclusive.")
        if self.query_text is None and self.entry_ids is None:
            raise ValueError("Either query_text or entry_ids must be provided.")

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Memory Configuration in AgentDefinition

```python
# Added to AgentDefinition (illustrative — D4 adds memory_config field)
memory_config = {
    "stores": [
        {
            "store_id": "<uuid4>",
            "memory_type": "episodic",
            "owner_agent_id": "<agent_uuid>",
            "privacy_level": "PRIVATE",
            "default_expiry_policy": {"policy_type": "TTL", "ttl_seconds": 2592000},  # 30 days
            "max_entries": 10000,
            "backend_type": "redis"
        },
        {
            "store_id": "<uuid4>",
            "memory_type": "semantic",
            "owner_agent_id": "<agent_uuid>",
            "privacy_level": "SHARED_OWNER",
            "default_expiry_policy": {
                "policy_type": "NEVER",
                "justification": "Product documentation does not expire; updated via re-embedding pipeline."
            },
            "max_entries": None,
            "backend_type": "pgvector"
        }
    ]
}
```

### 6.2 Memory Write and Read

```python
# Write an episodic memory entry
entry = episodic_memory.write(
    agent_id="<agent_uuid>",
    content={"task": "customer_inquiry", "outcome": "resolved", "sentiment": "positive"},
    metadata={"task_id": "<task_uuid>", "customer_id": "C-12345"}
)
assert entry.memory_type == MemoryType.EPISODIC
assert entry.is_expired == False

# Semantic search
results = semantic_memory.search(
    query=MemoryQuery(
        query_id=str(uuid4()),
        agent_id="<agent_uuid>",
        store_id="<store_uuid>",
        query_text="product return policy",
        top_k=3,
        include_expired=False
    )
)
assert len(results) <= 3
assert all(not r.is_expired for r in results)
```

---

## 7. Interfaces & Contracts

### 7.1 WorkingMemory

```python
from abc import ABC, abstractmethod
from typing import Any


class WorkingMemory(ABC):
    """
    Manages the agent's in-context state for the current task.
    Ephemeral — all contents are lost when the task ends.
    Not backed by an external store.

    Working memory is bounded by context_window_budget (tokens).
    The implementation is responsible for truncating or summarising
    contents when the budget is approached.
    """

    @abstractmethod
    def set(self, key: str, value: Any) -> None:
        """
        Store a value in working memory.
        Raises: MemoryBudgetError if adding this entry would exceed context_window_budget.
        """
        ...

    @abstractmethod
    def get(self, key: str) -> Any:
        """
        Retrieve a value from working memory.
        Returns: value, or None if key does not exist.
        Never raises for missing keys.
        """
        ...

    @abstractmethod
    def delete(self, key: str) -> None:
        """Remove a key from working memory. No-op if key does not exist."""
        ...

    @abstractmethod
    def clear(self) -> None:
        """Clear all working memory. Called at task end."""
        ...

    @abstractmethod
    def token_count(self) -> int:
        """Return estimated token count of current working memory contents."""
        ...

    @abstractmethod
    def to_context(self) -> str:
        """
        Serialise working memory to a string suitable for injection into LLM context.
        Respects context_window_budget — truncates oldest entries if over budget.
        """
        ...
```

### 7.2 EpisodicMemory

```python
class EpisodicMemory(ABC):
    """
    Persistent memory of past agent interactions, task outcomes, and session history.
    Backed by an external durable store.
    Entries have mandatory expiry policies.
    """

    @abstractmethod
    def write(
        self,
        agent_id: str,
        content: Any,
        metadata: dict = None,
        expiry_policy: ExpiryPolicy = None
    ) -> MemoryEntry:
        """
        Write a new episodic memory entry.
        Uses store's default_expiry_policy if expiry_policy is None.

        Raises:
            AgentKit.MemoryPrivacyViolationError: agent_id does not own this store.
            AgentKit.MemoryStoreUnavailableError: backing store unreachable.
        """
        ...

    @abstractmethod
    def read(self, entry_id: str, agent_id: str) -> MemoryEntry:
        """
        Read a specific entry by ID.

        Raises:
            AgentKit.MemoryExpiredError: entry exists but is expired.
                (Caller must explicitly pass include_expired=True to get expired entries.)
            AgentKit.MemoryPrivacyViolationError: agent_id not permitted by privacy_level.
            AgentKit.AgentNotFoundError: entry_id not in store.
        """
        ...

    @abstractmethod
    def search(self, query: MemoryQuery) -> list[MemoryEntry]:
        """
        Search episodic memory by metadata filter or semantic similarity.
        Never returns expired entries unless query.include_expired=True.
        """
        ...

    @abstractmethod
    def expire(self, entry_id: str, agent_id: str) -> None:
        """
        Manually expire an entry before its scheduled expiry time.
        Only the owning agent may expire entries.

        Raises:
            AgentKit.MemoryPrivacyViolationError: agent_id does not own the entry.
        """
        ...
```

### 7.3 SemanticMemory

```python
class SemanticMemory(ABC):
    """
    Knowledge stored as vector embeddings, retrieved by semantic similarity.
    Represents facts, documents, and domain knowledge.
    Backed by a vector store.
    """

    @abstractmethod
    def add(
        self,
        agent_id: str,
        content: str,
        embedding: list[float],
        metadata: dict = None,
        expiry_policy: ExpiryPolicy = None
    ) -> MemoryEntry:
        """
        Add a new knowledge entry with its pre-computed embedding.
        The caller is responsible for generating the embedding before calling add().

        Raises:
            AgentKit.MemoryPrivacyViolationError: agent_id not permitted to write.
            AgentKit.MemoryStoreUnavailableError: vector store unreachable.
        """
        ...

    @abstractmethod
    def search(self, query: MemoryQuery) -> list[MemoryEntry]:
        """
        Semantic similarity search using query.query_text.
        The implementation embeds query_text and performs ANN (approximate nearest neighbour) search.
        Never returns expired entries unless query.include_expired=True.

        Note: This is the RAG retrieval operation. Returning results from this method
              does NOT automatically inject them into Working Memory.
              The agent must explicitly call working_memory.set() with selected results.
        """
        ...

    @abstractmethod
    def get(self, entry_id: str, agent_id: str) -> MemoryEntry:
        """
        Exact lookup by entry_id.
        Raises: AgentKit.MemoryExpiredError if expired and include_expired not set.
        """
        ...
```

### 7.4 ProceduralMemory

```python
class ProceduralMemory(ABC):
    """
    Learned patterns, instructions, and few-shot examples.
    Typically read-only at runtime.
    Updated through offline training or manual curation — not through agent execution.
    """

    @abstractmethod
    def get_instruction(self, instruction_id: str) -> MemoryEntry:
        """Retrieve a specific instruction or few-shot example by ID."""
        ...

    @abstractmethod
    def list_instructions(
        self,
        agent_id: str,
        metadata_filter: dict = None
    ) -> list[MemoryEntry]:
        """
        List all instructions for an agent, optionally filtered by metadata.
        Returns empty list if none found.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 Memory Entry Lifecycle

**THE SYSTEM SHALL** set `created_at` at the moment an entry is persisted to the backing store.

**WHEN** an entry's `expires_at` timestamp has passed **THE SYSTEM SHALL** treat the entry as expired — it MUST NOT be returned in any query result unless `include_expired=True`.

**THE SYSTEM SHALL** raise `AgentKit.MemoryExpiredError` when a caller attempts to read an expired entry by `entry_id` without setting `include_expired=True`.

**THE SYSTEM SHALL** set `is_expired=True` on any expired entry that is returned due to `include_expired=True`.

### 8.2 Privacy and Access Control

**THE SYSTEM SHALL** enforce `privacy_level` on all memory read and write operations.

**WHEN** an agent attempts to write to a store it does not own **THE SYSTEM SHALL** raise `AgentKit.MemoryPrivacyViolationError`.

**WHEN** an agent attempts to read from a store where its access is not permitted by `privacy_level` **THE SYSTEM SHALL** raise `AgentKit.MemoryPrivacyViolationError`.

### 8.3 RAG vs Memory Distinction

**THE SYSTEM SHALL NOT** automatically inject `SemanticMemory.search()` results into Working Memory. The agent must make an explicit `working_memory.set()` call for each result it chooses to use.

**THE SYSTEM SHALL** provide a distinct code path for RAG retrieval (`semantic_memory.search()`) versus direct memory read (`episodic_memory.read()`). These MUST NOT share an interface.

### 8.4 Working Memory Budget

**WHEN** a `working_memory.set()` call would cause the estimated token count to exceed `context_window_budget` **THE SYSTEM SHALL** raise `AgentKit.MemoryBudgetError` with the current count and budget.

**THE SYSTEM SHALL** provide `working_memory.to_context()` which truncates or summarises oldest entries to fit within the budget when called.

### 8.5 Memory Store Availability

**WHEN** an external memory store (episodic, semantic, procedural) is unreachable **THE SYSTEM SHALL** raise `AgentKit.MemoryStoreUnavailableError` — not silently return empty results.

---

## 9. Three-Tier Boundary System

### ALWAYS
- Require expiry_policy on every memory store declaration
- Require owner_agent_id and privacy_level on every memory store declaration
- Enforce privacy_level on all read and write operations
- Exclude expired entries from query results by default (include_expired=False)
- Raise MemoryExpiredError on direct read of expired entries
- Require embedding to be present for semantic memory entries
- Keep RAG retrieval (search) and memory injection (set) as distinct explicit operations

### ASK FIRST
- Setting ExpiryPolicy.NEVER without justification — require justification field
- Writing to another agent's memory store (even if privacy_level permits) — confirm intentional sharing
- Clearing all working memory mid-task — confirm this is not a state loss

### NEVER
- Return expired entries in query results without explicit include_expired=True
- Allow an agent to write to a memory store it does not own
- Auto-inject semantic search results into working memory — agent must do this explicitly
- Store credentials, secrets, or PII in memory entries without encryption reference
- Silently return empty results when a backing store is unreachable — always raise MemoryStoreUnavailableError

---

## 10. Error Handling

| Scenario | Error Type | Recoverable | Required context |
|---|---|---|---|
| Direct read of expired entry | `AgentKit.MemoryExpiredError` | Yes (fetch fresh) | `entry_id, expired_at` |
| Write/read to/from unauthorised store | `AgentKit.MemoryPrivacyViolationError` | No | `agent_id, store_id, privacy_level` |
| Backing store unreachable | `AgentKit.MemoryStoreUnavailableError` | Yes (retry) | `store_id, backend_type` |
| Working memory budget exceeded | `AgentKit.MemoryBudgetError` | Yes (prune working memory) | `current_tokens, budget_tokens` |
| ExpiryPolicy.NEVER without justification | `ValueError` | No | — |
| query_text and entry_ids both set | `ValueError` | No | — |
| Neither query_text nor entry_ids set | `ValueError` | No | — |

---

## 11. Acceptance Criteria

```
Criteria ID:  D4-001
Stability:    Beta
Given:        A running agent with an episodic memory store (TTL=60 seconds)
When:         episodic_memory.write(agent_id, content={"key": "value"}, metadata={}) is called
Then:         MemoryEntry returned with:
              - entry_id: valid UUID v4
              - memory_type == MemoryType.EPISODIC
              - created_at: valid ISO 8601 UTC
              - expires_at: created_at + 60 seconds (± 1 second)
              - is_expired == False
Pass:         All conditions true
Fail:         Any condition false or expires_at not set for TTL policy
Error raised: None

---

Criteria ID:  D4-002
Stability:    Beta
Given:        A memory entry with TTL=1 second, written at time T
When:         2 seconds elapse, then episodic_memory.read(entry_id, agent_id) is called
Then:         AgentKit.MemoryExpiredError raised
              The entry is NOT returned as a valid entry
Pass:         MemoryExpiredError raised, context["entry_id"] matches
Fail:         Entry returned despite expiry, or different error
Error raised: AgentKit.MemoryExpiredError

---

Criteria ID:  D4-003
Stability:    Beta
Given:        An agent (agent_a) and a memory store owned by agent_b
              store.privacy_level == PRIVATE
When:         episodic_memory.write(agent_id=agent_a.agent_id, content=...) is called
Then:         AgentKit.MemoryPrivacyViolationError raised
              No entry is written
Pass:         MemoryPrivacyViolationError raised,
              context["agent_id"]==agent_a.agent_id, context["store_id"]==store.store_id
Fail:         Write succeeds, or different error
Error raised: AgentKit.MemoryPrivacyViolationError

---

Criteria ID:  D4-004
Stability:    Beta
Given:        A semantic memory store with 3 entries about "product returns"
When:         semantic_memory.search(MemoryQuery(query_text="how do I return a product", top_k=2))
Then:         At most 2 results returned, all non-expired
              None of the returned entries have is_expired==True
Pass:         len(results) <= 2, all results have is_expired==False
Fail:         Expired entries returned, or more than top_k results
Error raised: None

---

Criteria ID:  D4-005
Stability:    Beta
Given:        A semantic memory store with 1 expired entry and 1 valid entry
When:         semantic_memory.search(query with include_expired=False)
Then:         Only the valid entry is returned (1 result)
When:         semantic_memory.search(query with include_expired=True)
Then:         Both entries returned (2 results), expired entry has is_expired==True
Pass:         First query: 1 result, no expired entries.
              Second query: 2 results, expired entry.is_expired==True
Fail:         Expired entry appears in first query, or is_expired not set in second query
Error raised: None

---

Criteria ID:  D4-006
Stability:    Beta
Given:        A semantic memory search returns result R
When:         The result R is NOT explicitly added to working_memory.set()
Then:         R is NOT present in working_memory.to_context()
Pass:         working_memory.to_context() does not contain R's content
Fail:         R is auto-injected into working memory without explicit set() call
Error raised: None (verifies RAG ≠ auto-injection)

---

Criteria ID:  D4-007
Stability:    Beta
Given:        A working memory with context_window_budget=1000 tokens
              Current token count = 990
When:         working_memory.set("key", value_requiring_20_tokens) is called
              (total would be 1010, exceeding 1000)
Then:         AgentKit.MemoryBudgetError raised
              "key" is NOT stored in working memory
Pass:         MemoryBudgetError raised,
              context["current_tokens"]==990, context["budget_tokens"]==1000,
              working_memory.get("key") returns None
Fail:         Entry stored despite budget exceeded, or different error
Error raised: AgentKit.MemoryBudgetError

---

Criteria ID:  D4-008
Stability:    Beta
Given:        An episodic memory store backed by an unavailable store (connection refused)
When:         episodic_memory.write(agent_id, content=...) is called
Then:         AgentKit.MemoryStoreUnavailableError raised
              No empty result is returned silently
Pass:         MemoryStoreUnavailableError raised, context["backend_type"] set
Fail:         Empty result returned silently, or different error
Error raised: AgentKit.MemoryStoreUnavailableError

---

Criteria ID:  D4-009
Stability:    Beta
Given:        An ExpiryPolicy with policy_type=NEVER and no justification field
When:         ExpiryPolicy(policy_type="NEVER") is instantiated
Then:         ValueError raised mentioning justification is required
Pass:         ValueError raised
Fail:         ExpiryPolicy created without justification for NEVER type
Error raised: ValueError
```

---

## 12. Definition of Done

- [ ] `MemoryType`, `PrivacyLevel`, `ExpiryPolicyType` enums implemented
- [ ] `ExpiryPolicy` Pydantic model with cross-field validators
- [ ] `MemoryEntry` Pydantic model with all fields
- [ ] `MemoryStoreConfig` Pydantic model with all fields
- [ ] `MemoryQuery` Pydantic model with mutual exclusivity validation
- [ ] `WorkingMemory` abstract interface with all 6 methods
- [ ] `EpisodicMemory` abstract interface with all 4 methods
- [ ] `SemanticMemory` abstract interface with all 3 methods
- [ ] `ProceduralMemory` abstract interface with all 2 methods
- [ ] Concrete `WorkingMemory` implementation with token budget enforcement
- [ ] Concrete `EpisodicMemory` implementation with expiry enforcement
- [ ] Concrete `SemanticMemory` implementation (in-memory + vector search for tests)
- [ ] Privacy enforcement on all memory read/write operations
- [ ] Expired entries excluded from query results by default
- [ ] RAG and memory injection are separate explicit operations
- [ ] All 9 acceptance criteria pass: D4-001 through D4-009
- [ ] Spec invariant INV-006 verified (checkpoint includes memory state — cross-dimension, verified in D6)

---

## 13. Task Breakdown

```
Task D4-T1: Implement enums (MemoryType, PrivacyLevel, ExpiryPolicyType)
Task D4-T2: Implement ExpiryPolicy with cross-field validators
Task D4-T3: Implement MemoryEntry Pydantic model
Task D4-T4: Implement MemoryStoreConfig Pydantic model
Task D4-T5: Implement MemoryQuery with mutual exclusivity validation
Task D4-T6: Implement WorkingMemory abstract + concrete (dict-backed, token-counted)
Task D4-T7: Implement EpisodicMemory abstract + concrete (in-memory for tests)
Task D4-T8: Implement SemanticMemory abstract + concrete (numpy cosine similarity for tests)
Task D4-T9: Implement ProceduralMemory abstract + concrete (list-backed for tests)
Task D4-T10: Implement privacy enforcement on all read/write paths
Task D4-T11: Implement expiry enforcement — exclude expired by default
Task D4-T12: Verify RAG ≠ auto-injection (D4-006 criterion)
Task D4-T13: Run all 9 acceptance criteria
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Using a vector DB as the only memory store**
A vector database stores embeddings without timestamps affecting retrieval. A six-month-old fact and yesterday's fact are equally "similar." For time-sensitive domains, this produces silent errors. Use episodic memory with timestamps for time-sensitive facts.

**Anti-Pattern 2 — Auto-injecting RAG results into working memory**
Automatically stuffing all retrieved context into every LLM call causes context window overflow and degrades reasoning quality. The agent must select which retrieved facts are relevant and inject them explicitly.

**Anti-Pattern 3 — Missing expiry policies**
Memory without expiry grows unbounded. After months of production use, an agent with no expiry policy has a memory store that is slow to query, expensive to store, and full of stale information that degrades retrieval quality.

**Anti-Pattern 4 — Storing secrets in memory entries**
Memory entries may be serialised, logged, and searched. API keys, passwords, and PII stored directly in memory entries become visible in logs and traces. Use secret references instead.

**Anti-Pattern 5 — Silently returning empty results on store unavailability**
Returning an empty list when the backing store is unreachable causes the agent to believe there are no memories — it proceeds without context and produces hallucinated or incorrect outputs. Always raise MemoryStoreUnavailableError so the agent can handle it explicitly.

**Anti-Pattern 6 — NEVER expiry without justification**
ExpiryPolicy.NEVER is a governance liability. Every entry that never expires must have a human-readable justification. Without it, "NEVER" becomes the path of least resistance and every entry eventually uses it.

---

## 15. Reference Implementation Notes

### 15.1 AgentKit-LangGraph
- Working memory: use LangGraph's state graph (`State` TypedDict) for in-task state
- Episodic memory: `Mem0` (pip: `mem0ai`) for production, in-memory dict for tests
- Semantic memory: `pgvector` (PostgreSQL) for production, numpy cosine similarity for tests
- Procedural memory: YAML/JSON file-backed for development

### 15.2 AgentKit-MAF
- Working memory: MAF session context
- Episodic memory: Azure Cosmos DB with TTL support (built-in TTL fields)
- Semantic memory: Azure AI Search with vector index
- Procedural memory: Azure Blob Storage with YAML instruction files

### 15.3 Technology Decision Matrix

| Memory Type | Dev/Test Backend | Production Backend | Why |
|---|---|---|---|
| Working | Python dict | LangGraph State / MAF Session | Ephemeral — no persistence needed |
| Episodic | In-memory dict | Redis (TTL support) / Cosmos DB | Fast, TTL-native |
| Semantic | Numpy (cosine) | pgvector / Azure AI Search | Vector search + SQL hybrid |
| Procedural | YAML file | S3/Blob versioned object | Rarely changes, versioned |

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should working memory have a structured schema or remain untyped dict? | No |
| OQ-2 | Should semantic memory support hybrid search (BM25 + vector)? | No — add in v2 |
| OQ-3 | Should memory entries support versioning (read historical states)? | No |
| OQ-4 | Should cross-agent shared semantic memory be part of core or governance extension? | Yes — clarify before Stable |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec |

---

*AgentKit Dimension 4 — Memory — v1.1*
