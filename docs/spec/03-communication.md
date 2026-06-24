# AgentPave Dimension 3 — Communication

**Spec version:** 1.1  
**Stability:** Beta  
**Depends on:** D1 (Identity), D2 (Lifecycle)  
**Required by:** D4, D6  
**Owner:** AgentPave Core  

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

Communication is how an agent acts on the world. Without a standardised tool invocation model, every tool integration is a custom, untested, one-off adapter. The MCP standard crossed 97 million monthly SDK downloads in 2026 and became the universal tool integration protocol for AI agents — AgentPave adopts it as the mandatory protocol for all tool communication.

This dimension governs three communication channels:
1. **Agent ↔ Tool** — via MCP (mandatory, standardised)
2. **Agent ↔ Agent** — via A2A protocol (standardised, Beta)
3. **Agent ↔ Human** — via defined interaction model (structured, auditable)

Every tool an agent uses must have a declared Integration Contract — a formal declaration of what the tool can access, how it behaves under failure, and what its rate and retry policies are.

### 1.2 Production Consequence of Getting This Wrong

Before MCP, connecting an agent to three external systems meant three custom integrations, each with its own auth flow, error handling, retry logic, and data parser. When one API changed its response format, the agent broke silently. Most production MCP failures trace to tool descriptions that are too vague for agents to use correctly, missing retry semantics for transient failures, or absent rate limit handling that causes cascading failures.

---

## 2. Scope Boundary

### 2.1 In Scope

- All tool registration and invocation via MCP
- Integration Contract definition and enforcement
- Circuit breaker pattern for tool resilience
- Agent-to-agent communication schema and versioning
- Agent-to-human communication model
- Tool discovery via MCP `tools/list`
- Error handling for all communication failure modes

### 2.2 Out of Scope

- MCP server implementation — AgentPave is an MCP client
- A2A server implementation — out of MVP scope
- Authentication of MCP server connections — that is D6 (Security)
- Tool caching — that is AgentPave-Cache extension
- Human approval workflows — that is AgentPave-HITL extension
- Cost tracking of tool calls — that is D8 (Economics)

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **MCP is mandatory for all tool calls** | MCP standardises at the protocol level: every tool declares input/output schema, the agent discovers capabilities at runtime. No hardcoded endpoint lists, no stale documentation. |
| **Integration Contracts are mandatory per tool** | The most common failure mode in MCP deployments is not the server code — it is vague tool descriptions and absent failure handling. Integration Contracts formalise this at the agent level. |
| **Circuit breaker is a first-class pattern** | Agents making 10 sequential calls with cascading timeouts cause compounding failures. Circuit breakers prevent this — they provide structured error context so the agent can reason about the failure intelligently. |
| **Inter-agent schemas are SemVer versioned** | Unversioned schemas cause silent protocol mismatches when one agent updates its output format. SemVer makes the contract explicit and breaking changes detectable. |
| **Direct function calls to tools are prohibited** | Bypassing MCP bypasses tool schema validation, rate limiting, retry logic, and observability. Direct calls create invisible, unauditable tool invocations. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **MCP** | Model Context Protocol. JSON-RPC-based protocol for agent-to-tool communication. AgentPave uses MCP as client. All tool calls use `tools/call`. Tool discovery uses `tools/list`. |
| **MCP Tool** | A named, schema-described function exposed by an MCP server that an agent can invoke. |
| **Integration Contract** | A per-tool declaration specifying: accessed systems, unavailability behaviour, retry policy, rate limit policy, and timeout policy. Mandatory for every tool. |
| **Circuit Breaker** | A resilience pattern that tracks tool failure rates and temporarily stops calling a failing tool (OPEN state) to prevent cascading failures. |
| **Circuit Breaker State** | One of three states: CLOSED (normal operation), OPEN (failing — calls blocked), HALF_OPEN (recovery probe). |
| **Retry Policy** | The declared strategy for retrying failed tool calls: max_attempts, backoff strategy (exponential), and jitter. |
| **Rate Limit Policy** | The declared strategy for handling rate limit (429) responses: per-second and per-minute limits, backoff on limit hit. |
| **Tool Result** | The structured return value from a tool call. AgentPave wraps every tool result as an `AgentResult`. |
| **A2A** | Agent-to-Agent protocol. Used for cross-runtime agent communication. Schemas are SemVer versioned. |
| **Human Interaction Request** | A structured message from an agent to a human, requiring a response before the agent continues. Used for approval, clarification, or information gathering. |
| **Timeout Policy** | The declared maximum time to wait for a tool response before raising `AgentPave.ToolTimeoutError`. |

---

## 5. Data Models

### 5.1 RetryPolicy

```python
from pydantic import BaseModel, Field
from typing import Literal
from enum import Enum


class BackoffStrategy(str, Enum):
    EXPONENTIAL = "exponential"   # Wait doubles each retry: 1s, 2s, 4s, 8s...
    LINEAR      = "linear"        # Wait increases linearly: 1s, 2s, 3s, 4s...
    FIXED       = "fixed"         # Same wait each retry


class RetryPolicy(BaseModel):
    """
    Declares how the agent retries failed tool calls.
    Applies to transient failures (network errors, 5xx responses, timeouts).
    Does NOT apply to client errors (4xx except 429) — those are not retried.
    """
    max_attempts: int = Field(
        ...,
        ge=1,
        le=10,
        description="Maximum number of attempts including the first. Range: [1, 10]."
    )
    backoff_strategy: BackoffStrategy = Field(
        default=BackoffStrategy.EXPONENTIAL,
        description="How wait time grows between retries."
    )
    base_delay_ms: int = Field(
        default=1000,
        ge=100,
        le=30000,
        description="Initial delay in milliseconds before first retry. Range: [100ms, 30s]."
    )
    max_delay_ms: int = Field(
        default=30000,
        ge=100,
        le=300000,
        description="Maximum delay cap in milliseconds. Range: [100ms, 5 minutes]."
    )
    jitter: bool = Field(
        default=True,
        description=(
            "Whether to add random jitter to delay to prevent thundering herd. "
            "Recommended: True for all production use."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.2 RateLimitPolicy

```python
class RateLimitPolicy(BaseModel):
    """
    Declares how the agent handles rate limit responses (HTTP 429 / MCP rate_limit error).
    """
    requests_per_second: int = Field(
        ...,
        ge=1,
        description="Maximum requests per second to this tool."
    )
    requests_per_minute: int = Field(
        ...,
        ge=1,
        description="Maximum requests per minute to this tool."
    )
    retry_after_ms: int = Field(
        default=5000,
        ge=0,
        description=(
            "How long to wait (ms) after a rate limit response before retrying. "
            "If the rate limit response includes a Retry-After value, "
            "that value takes precedence over this field."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.3 CircuitBreakerConfig

```python
class CircuitBreakerConfig(BaseModel):
    """
    Circuit breaker configuration for a tool.
    When enabled, tracks failure rates and opens the circuit to prevent cascading failures.
    """
    enabled: bool = Field(
        default=True,
        description="Whether the circuit breaker is active for this tool."
    )
    failure_threshold: int = Field(
        default=5,
        ge=1,
        le=100,
        description=(
            "Number of consecutive failures before the circuit opens (CLOSED → OPEN). "
            "Range: [1, 100]."
        )
    )
    recovery_timeout_ms: int = Field(
        default=60000,
        ge=1000,
        description=(
            "Time in ms the circuit stays OPEN before entering HALF_OPEN for recovery probe. "
            "Default: 60 seconds."
        )
    )
    half_open_max_calls: int = Field(
        default=1,
        ge=1,
        le=10,
        description=(
            "Number of probe calls allowed in HALF_OPEN state. "
            "If all succeed: circuit closes. If any fail: circuit re-opens."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.4 IntegrationContract

```python
from typing import Optional


class FallbackBehaviour(str, Enum):
    RAISE_ERROR   = "raise_error"    # Raise ToolUnavailableError immediately
    RETURN_CACHED = "return_cached"  # Return last cached result (if available)
    RETURN_NULL   = "return_null"    # Return AgentResult with status="error", output=None
    SKIP          = "skip"           # Skip this tool call and continue agent execution


class IntegrationContract(BaseModel):
    """
    Mandatory declaration for every tool an agent uses.
    Formalises what the tool accesses, how it fails, and how the agent responds.
    """

    # --- Tool identity ---
    tool_name: str = Field(
        ...,
        min_length=1,
        max_length=128,
        description=(
            "The exact tool name as registered in the MCP server. "
            "Must match the name returned by tools/list exactly."
        )
    )
    tool_version: str = Field(
        ...,
        description="SemVer version of the tool schema this contract is written for."
    )
    description: str = Field(
        ...,
        min_length=10,
        max_length=500,
        description="What this tool does and why this agent uses it."
    )

    # --- System access declaration ---
    accessed_systems: list[str] = Field(
        ...,
        min_length=1,
        description=(
            "List of external systems or APIs this tool accesses. "
            "At least one must be declared. "
            "Example: ['salesforce-api', 'internal-postgres-db']"
        )
    )

    # --- Failure handling ---
    fallback_behaviour: FallbackBehaviour = Field(
        ...,
        description=(
            "What the agent does when this tool is unavailable after exhausting retries. "
            "Choose based on the criticality of the tool to the agent's task."
        )
    )
    timeout_ms: int = Field(
        default=30000,
        ge=100,
        le=300000,
        description="Maximum time to wait for a tool response. Default: 30s. Max: 5 minutes."
    )

    # --- Retry and rate limit policies ---
    retry_policy: RetryPolicy = Field(
        ...,
        description="Mandatory retry configuration for transient failures."
    )
    rate_limit_policy: RateLimitPolicy = Field(
        ...,
        description="Mandatory rate limit configuration."
    )
    circuit_breaker: CircuitBreakerConfig = Field(
        default_factory=CircuitBreakerConfig,
        description="Circuit breaker configuration. Enabled by default."
    )

    model_config = {"extra": "forbid"}
```

### 5.5 ToolCallRequest

```python
class ToolCallRequest(BaseModel):
    """
    A structured request to invoke a tool via MCP.
    Created by the agent runtime, not by the developer directly.
    """
    call_id: str = Field(..., description="UUID v4. Unique ID for this call. For tracing.")
    agent_id: str = Field(..., description="UUID v4. The invoking agent.")
    task_id: str = Field(..., description="UUID v4. The active task.")
    tool_name: str = Field(..., description="Exact tool name from IntegrationContract.")
    arguments: dict = Field(..., description="Tool input arguments. Must match tool's input schema.")
    trace_id: str = Field(..., description="UUID v4. OpenTelemetry trace ID. Propagated to MCP server.")
    issued_at: str = Field(..., description="ISO 8601 UTC. When this call was issued.")
    runtime_token: str = Field(..., description="JWT. The agent's current Runtime Token.")

    model_config = {"extra": "forbid"}
```

### 5.6 HumanInteractionRequest

```python
class HumanInteractionType(str, Enum):
    APPROVAL   = "approval"     # Agent needs yes/no approval to proceed
    CLARIFY    = "clarify"      # Agent needs clarification to continue
    INFORM     = "inform"       # Agent is informing human — no response required
    ESCALATE   = "escalate"     # Agent is escalating a situation requiring human decision


class HumanInteractionRequest(BaseModel):
    """
    A structured message from an agent to a human.
    Used when the agent needs input, approval, or must inform the human.
    """
    request_id: str = Field(..., description="UUID v4.")
    agent_id: str = Field(..., description="UUID v4.")
    task_id: str = Field(..., description="UUID v4.")
    interaction_type: HumanInteractionType = Field(...)
    message: str = Field(
        ...,
        min_length=10,
        max_length=2000,
        description="The message shown to the human. Must be clear, specific, and actionable."
    )
    context: dict = Field(
        default_factory=dict,
        description="Structured context to help the human make a decision."
    )
    response_required: bool = Field(
        ...,
        description="True for APPROVAL and CLARIFY. False for INFORM."
    )
    timeout_seconds: Optional[int] = Field(
        None,
        ge=60,
        description=(
            "How long (seconds) to wait for a human response before timing out. "
            "None = wait indefinitely. "
            "When timeout expires: agent transitions to PAUSED state."
        )
    )
    created_at: str = Field(..., description="ISO 8601 UTC.")

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Valid IntegrationContract

```python
web_search_contract = IntegrationContract(
    tool_name="web_search",
    tool_version="2.1.0",
    description="Searches the web for current information using a search API.",
    accessed_systems=["brave-search-api"],
    fallback_behaviour=FallbackBehaviour.RAISE_ERROR,
    timeout_ms=10000,
    retry_policy=RetryPolicy(
        max_attempts=3,
        backoff_strategy=BackoffStrategy.EXPONENTIAL,
        base_delay_ms=1000,
        max_delay_ms=8000,
        jitter=True
    ),
    rate_limit_policy=RateLimitPolicy(
        requests_per_second=2,
        requests_per_minute=60,
        retry_after_ms=5000
    ),
    circuit_breaker=CircuitBreakerConfig(
        enabled=True,
        failure_threshold=5,
        recovery_timeout_ms=60000
    )
)
```

### 6.2 Tool Call Sequence

```python
# Agent invokes web_search via MCP
request = ToolCallRequest(
    call_id=str(uuid4()),
    agent_id="f47ac10b-58cc-4372-a567-0e02b2c3d479",
    task_id="b3d4e5f6-...",
    tool_name="web_search",
    arguments={"query": "AgentPave framework 2026", "num_results": 5},
    trace_id=str(uuid4()),
    issued_at="2026-06-09T12:00:00.000Z",
    runtime_token="<jwt>"
)

# Result is always an AgentResult
result = tool_caller.call(request)
assert result.status in ("success", "error", "timeout", "cancelled")
assert result.trace_id == request.trace_id  # trace_id propagated through
```

---

## 7. Interfaces & Contracts

### 7.1 ToolRegistry

```python
from abc import ABC, abstractmethod


class ToolRegistry(ABC):
    """
    Manages tool registration and Integration Contract validation.
    Backed by MCP tools/list for tool discovery.
    """

    @abstractmethod
    def register_tool(
        self,
        contract: IntegrationContract,
        mcp_server_url: str
    ) -> None:
        """
        Register a tool by validating its Integration Contract against
        the tool schema returned by the MCP server's tools/list.

        Steps:
        1. Call tools/list on the MCP server.
        2. Find the tool matching contract.tool_name.
        3. Raise ToolNotFoundError if not found.
        4. Validate contract.tool_version matches the server's reported version.
        5. Store contract internally.

        Raises:
            AgentPave.ToolNotFoundError: tool_name not found in MCP server's tools/list.
            AgentPave.IntegrationContractViolationError: tool version mismatch.
        """
        ...

    @abstractmethod
    def get_contract(self, tool_name: str) -> IntegrationContract:
        """
        Retrieve the Integration Contract for a registered tool.

        Raises:
            AgentPave.ToolNotFoundError: tool not registered.
        """
        ...

    @abstractmethod
    def list_tools(self) -> list[str]:
        """Return names of all registered tools."""
        ...
```

### 7.2 ToolCaller

```python
from abc import ABC, abstractmethod


class ToolCaller(ABC):
    """
    Executes tool calls via MCP, enforcing Integration Contract policies.
    Implements retry, rate limiting, circuit breaker, and timeout.
    """

    @abstractmethod
    def call(self, request: ToolCallRequest) -> "AgentResult":
        """
        Invoke a tool via MCP, enforcing the tool's Integration Contract.

        Execution order:
        1. Check circuit breaker state. If OPEN: raise AgentPave.CircuitBreakerOpenError.
        2. Validate agent is in RUNNING state (via LifecycleManager).
        3. Validate request.tool_name is registered (via ToolRegistry).
        4. Check rate limit. If exceeded: wait per RateLimitPolicy or raise RateLimitError.
        5. Send MCP tools/call with timeout from IntegrationContract.
        6. On success: record circuit breaker success, return AgentResult.
        7. On transient failure: apply RetryPolicy (exponential backoff + jitter).
        8. On retry exhaustion: apply fallback_behaviour from IntegrationContract.
        9. On circuit break trip: transition circuit to OPEN, raise ToolUnavailableError.

        Note: All calls are logged as OpenTelemetry spans (D5 Observability).
              trace_id from request MUST be propagated to the MCP server as a header.

        Raises:
            AgentPave.ToolNotFoundError: tool not registered.
            AgentPave.ToolUnavailableError: tool unreachable after retry exhaustion.
            AgentPave.RateLimitError: rate limit exceeded.
            AgentPave.ToolTimeoutError: tool did not respond within timeout_ms.
            AgentPave.CircuitBreakerOpenError: circuit is OPEN for this tool.
            AgentPave.InvalidStateTransitionError: agent not in RUNNING state.
        """
        ...
```

### 7.3 AgentCommunicator

```python
class AgentCommunicator(ABC):
    """
    Manages agent-to-agent and agent-to-human communication.
    """

    @abstractmethod
    def send_to_agent(
        self,
        from_agent_id: str,
        to_agent_id: str,
        schema_version: str,
        payload: dict
    ) -> "AgentResult":
        """
        Send a versioned message to another agent via A2A protocol.

        Args:
            schema_version: SemVer. Must match the receiving agent's declared schema version.
            payload: Must conform to the schema for schema_version.

        Raises:
            AgentPave.SchemaMismatchError: schema_version incompatible with receiver.
            AgentPave.AgentNotFoundError: to_agent_id not in registry.
        """
        ...

    @abstractmethod
    def request_human_input(
        self,
        request: HumanInteractionRequest
    ) -> Optional[dict]:
        """
        Send a structured message to a human and optionally await response.

        Returns:
            dict: Human's response payload (for APPROVAL/CLARIFY).
            None: For INFORM type (no response expected).

        Raises:
            AgentPave.HumanInputTimeoutError: if timeout_seconds exceeded.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 Tool Registration

**THE SYSTEM SHALL** require every tool to have a registered Integration Contract before it may be invoked.

**THE SYSTEM SHALL** validate Integration Contracts against the MCP server's `tools/list` at registration time — not at call time.

**WHEN** an agent attempts to invoke a tool without a registered Integration Contract **THE SYSTEM SHALL** raise `AgentPave.ToolNotFoundError`.

### 8.2 Tool Invocation

**THE SYSTEM SHALL** route all tool calls through MCP `tools/call` — direct function calls to tools are prohibited.

**THE SYSTEM SHALL** propagate the `trace_id` from `ToolCallRequest` to the MCP server as a request header (`X-AgentPave-Trace-Id`).

**THE SYSTEM SHALL** enforce `timeout_ms` from the Integration Contract — if the tool does not respond within `timeout_ms`, raise `AgentPave.ToolTimeoutError`.

**WHILE** an agent is not in RUNNING state **THE SYSTEM SHALL** reject all tool invocations with `AgentPave.InvalidStateTransitionError`.

### 8.3 Retry Logic

**WHEN** a tool call fails with a transient error (network timeout, 5xx HTTP, MCP transport error) **THE SYSTEM SHALL** retry according to the `RetryPolicy` declared in the Integration Contract.

**THE SYSTEM SHALL** apply exponential backoff with jitter when `backoff_strategy=EXPONENTIAL` and `jitter=True`.

**WHEN** `max_attempts` is exhausted **THE SYSTEM SHALL** apply `fallback_behaviour` from the Integration Contract.

**THE SYSTEM SHALL NOT** retry client errors (4xx except 429) — these indicate a bug in the agent's tool arguments.

### 8.4 Circuit Breaker

**THE SYSTEM SHALL** maintain a circuit breaker per registered tool.

**WHEN** consecutive failures reach `failure_threshold` **THE SYSTEM SHALL** transition the circuit to OPEN and raise `AgentPave.CircuitBreakerOpenError` with structured metadata: `breaker_name`, `retry_after`, `failure_count`.

**WHEN** `recovery_timeout_ms` elapses in OPEN state **THE SYSTEM SHALL** transition to HALF_OPEN and allow `half_open_max_calls` probe calls.

**WHEN** all probe calls in HALF_OPEN succeed **THE SYSTEM SHALL** close the circuit (return to CLOSED).

**WHEN** any probe call in HALF_OPEN fails **THE SYSTEM SHALL** re-open the circuit and reset `recovery_timeout_ms`.

### 8.5 Rate Limiting

**THE SYSTEM SHALL** track request rates per tool and enforce `requests_per_second` and `requests_per_minute` limits.

**WHEN** a rate limit is exceeded **THE SYSTEM SHALL** wait `retry_after_ms` (or the Retry-After header value if present) before retrying — not immediately raise `RateLimitError` unless the RetryPolicy is also exhausted.

### 8.6 Inter-Agent Communication

**THE SYSTEM SHALL** require a `schema_version` on all A2A messages.

**WHEN** the receiving agent's declared schema version is incompatible with `schema_version` **THE SYSTEM SHALL** raise `AgentPave.SchemaMismatchError`.

---

## 9. Three-Tier Boundary System

### ALWAYS
- Route all tool calls through MCP — never bypass
- Require an Integration Contract for every tool before any invocation
- Propagate trace_id to every MCP server call
- Enforce timeout_ms from Integration Contract
- Apply retry policy on transient failures
- Maintain circuit breaker state per tool
- Version all inter-agent message schemas

### ASK FIRST
- Invoking a tool when the circuit breaker is in HALF_OPEN — confirm this is a valid probe
- Sending a HUMAN_INTERACTION_REQUEST of type ESCALATE — confirm the escalation is warranted
- Registering a tool whose version differs from the contract's declared tool_version

### NEVER
- Invoke a tool without a registered Integration Contract
- Bypass MCP for any tool call
- Retry 4xx client errors (they are bugs, not transient failures)
- Issue a tool call when the agent is not in RUNNING state
- Open a tool connection without propagating the trace_id

---

## 10. Error Handling

| Scenario | Error Type | Recoverable | Required context |
|---|---|---|---|
| Tool not registered | `AgentPave.ToolNotFoundError` | No | `tool_name` |
| Tool unreachable after retries | `AgentPave.ToolUnavailableError` | Yes (per fallback) | `tool_name, attempts, last_error` |
| Rate limit exceeded and retries exhausted | `AgentPave.RateLimitError` | Yes (wait) | `tool_name, retry_after_ms` |
| Tool did not respond in time | `AgentPave.ToolTimeoutError` | Yes (retry) | `tool_name, timeout_ms` |
| Circuit breaker OPEN | `AgentPave.CircuitBreakerOpenError` | Yes (wait recovery_timeout) | `tool_name, retry_after, failure_count` |
| Tool response violates schema | `AgentPave.IntegrationContractViolationError` | No | `tool_name, schema_field, received_value` |
| A2A schema version mismatch | `AgentPave.SchemaMismatchError` | No | `expected_version, received_version` |
| Human input timeout | `AgentPave.HumanInputTimeoutError` | Yes (agent pauses) | `request_id, timeout_seconds` |
| Tool call when agent not RUNNING | `AgentPave.InvalidStateTransitionError` | No | `agent_id, current_state` |

---

## 11. Acceptance Criteria

```
Criteria ID:  D3-001
Stability:    Beta
Given:        A running agent with a registered web_search tool (valid Integration Contract)
When:         tool_caller.call(ToolCallRequest(..., tool_name="web_search", ...))
Then:         Tool is invoked via MCP tools/call.
              AgentResult returned with status="success", trace_id matches request.
Pass:         MCP call log shows tools/call was used, AgentResult.trace_id == request.trace_id
Fail:         Tool called directly (not via MCP), or trace_id not propagated
Error raised: None

---

Criteria ID:  D3-002
Stability:    Beta
Given:        A running agent attempting to call an unregistered tool "unknown_tool"
When:         tool_caller.call(ToolCallRequest(..., tool_name="unknown_tool", ...))
Then:         AgentPave.ToolNotFoundError is raised
Pass:         ToolNotFoundError raised, context["tool_name"] == "unknown_tool"
Fail:         Call succeeds or different error raised
Error raised: AgentPave.ToolNotFoundError

---

Criteria ID:  D3-003
Stability:    Beta
Given:        A tool whose MCP server returns errors on every call
              RetryPolicy: max_attempts=3, backoff=exponential
              IntegrationContract: fallback_behaviour=RAISE_ERROR
When:         tool_caller.call(request) is called
Then:         3 total attempts are made (1 original + 2 retries)
              AgentPave.ToolUnavailableError raised after all attempts exhausted
Pass:         MCP call log shows exactly 3 attempts,
              ToolUnavailableError raised with context["attempts"]==3
Fail:         Fewer or more than 3 attempts, or different error, or no error
Error raised: AgentPave.ToolUnavailableError

---

Criteria ID:  D3-004
Stability:    Beta
Given:        A tool with circuit breaker: failure_threshold=3
              The tool has already failed 3 consecutive times (circuit is OPEN)
When:         tool_caller.call(request) is called
Then:         No MCP call is made (circuit is OPEN — call is blocked)
              AgentPave.CircuitBreakerOpenError raised immediately
Pass:         Zero MCP calls made, CircuitBreakerOpenError raised,
              context["tool_name"] set, context["retry_after"] is a future timestamp
Fail:         MCP call attempt made despite OPEN circuit, or different error
Error raised: AgentPave.CircuitBreakerOpenError

---

Criteria ID:  D3-005
Stability:    Beta
Given:        A tool with timeout_ms=5000
              The MCP server takes 6000ms to respond
When:         tool_caller.call(request) is called
Then:         AgentPave.ToolTimeoutError raised after 5000ms
              No partial result is returned
Pass:         ToolTimeoutError raised, elapsed time ~5000ms (±100ms),
              context["timeout_ms"] == 5000
Fail:         Call waits 6000ms, or different error, or partial result returned
Error raised: AgentPave.ToolTimeoutError

---

Criteria ID:  D3-006
Stability:    Beta
Given:        A paused agent (lifecycle_state == PAUSED)
When:         tool_caller.call(request) is called
Then:         AgentPave.InvalidStateTransitionError raised
              No MCP call is made
Pass:         Error raised, zero MCP calls logged
Fail:         Tool invoked despite agent being PAUSED, or different error
Error raised: AgentPave.InvalidStateTransitionError

---

Criteria ID:  D3-007
Stability:    Beta
Given:        Two agents: sender declares schema_version="1.0.0", receiver expects "2.0.0"
When:         agent_communicator.send_to_agent(schema_version="1.0.0", payload={...})
Then:         AgentPave.SchemaMismatchError raised
Pass:         SchemaMismatchError raised,
              context["expected_version"]=="2.0.0", context["received_version"]=="1.0.0"
Fail:         Message delivered despite version mismatch, or different error
Error raised: AgentPave.SchemaMismatchError

---

Criteria ID:  D3-008
Stability:    Beta
Given:        A tool that returns a response violating its declared output schema
              (e.g. tool declares output.results as list[str] but returns list[int])
When:         tool_caller.call(request) returns the invalid response
Then:         AgentPave.IntegrationContractViolationError raised
              The invalid response is NOT passed to the agent
Pass:         IntegrationContractViolationError raised,
              context["tool_name"] set, invalid output not accessible to agent
Fail:         Invalid output passed to agent, or different error
Error raised: AgentPave.IntegrationContractViolationError
```

---

## 12. Definition of Done

- [ ] `RetryPolicy` Pydantic model implemented with all fields
- [ ] `RateLimitPolicy` Pydantic model implemented with all fields
- [ ] `CircuitBreakerConfig` Pydantic model implemented with all fields
- [ ] `IntegrationContract` Pydantic model implemented with all fields
- [ ] `ToolCallRequest` Pydantic model implemented with all fields
- [ ] `HumanInteractionRequest` Pydantic model implemented with all fields
- [ ] `ToolRegistry` abstract interface implemented
- [ ] `ToolCaller` abstract interface implemented
- [ ] `AgentCommunicator` abstract interface implemented
- [ ] Concrete `ToolCaller` implements: MCP routing, retry, rate limit, circuit breaker, timeout
- [ ] Circuit breaker state machine: CLOSED → OPEN → HALF_OPEN → CLOSED
- [ ] `trace_id` propagated to every MCP call header
- [ ] Direct tool function calls are blocked (no bypass path exists)
- [ ] All 8 acceptance criteria pass: D3-001 through D3-008
- [ ] Spec invariants verified: INV-004, INV-005

---

## 13. Task Breakdown

```
Task D3-T1: Implement data models (RetryPolicy, RateLimitPolicy, CircuitBreakerConfig,
            IntegrationContract, ToolCallRequest, HumanInteractionRequest)
Task D3-T2: Implement ToolRegistry abstract + concrete (MCP tools/list backed)
Task D3-T3: Implement circuit breaker state machine (CLOSED/OPEN/HALF_OPEN)
Task D3-T4: Implement retry logic with exponential backoff + jitter
Task D3-T5: Implement rate limit tracking and enforcement
Task D3-T6: Implement timeout enforcement
Task D3-T7: Implement ToolCaller with full integration contract enforcement pipeline
Task D3-T8: Implement trace_id propagation to MCP server headers
Task D3-T9: Implement AgentCommunicator (A2A schema versioning + human interaction)
Task D3-T10: Run all 8 acceptance criteria
Task D3-T11: Verify INV-004 and INV-005 invariants
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Direct function calls to tools**
Bypassing MCP creates invisible, unauditable, unretried, unobserved tool calls. Agents that don't use MCP have no standard tool schema validation, no rate limiting, no circuit breaking, and no tracing.

**Anti-Pattern 2 — Retrying 4xx client errors**
A 400 Bad Request means the agent's arguments are wrong — retrying the same wrong arguments wastes quota and time. Only transient errors (5xx, network, timeout) should be retried.

**Anti-Pattern 3 — Absent circuit breakers**
Without circuit breakers, a failing tool causes cascading timeouts across all agent tasks. An agent making 10 sequential calls to a timing-out tool will hang for `10 × timeout_ms` before failing. Circuit breakers limit this to a single recovery timeout period.

**Anti-Pattern 4 — Not propagating trace_id**
Without trace_id propagation, tool call spans are disconnected from agent spans. Debugging a production failure requires manually correlating disconnected traces — near-impossible at scale.

**Anti-Pattern 5 — Unversioned inter-agent schemas**
An agent that updates its output format without bumping schema_version silently breaks all receivers. This is the agent equivalent of a breaking API change with no version bump.

---

## 15. Reference Implementation Notes

### 15.1 AgentPave-LangGraph
- Use the `mcp` Python package (PyPI: `mcp>=1.0`) for all MCP client operations
- LangGraph's `ToolNode` should be wrapped to enforce IntegrationContract checks
- Circuit breaker can use `tenacity` library with custom circuit breaker state machine

### 15.2 AgentPave-MAF
- MAF has native MCP client support — use it directly
- Rate limiting can leverage Azure API Management policies if tools are Azure-hosted
- A2A protocol support is native in MAF — use it for agent-to-agent calls

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should AgentPave define a standard tool output schema that all tools must conform to? | No (MCP tools define their own schemas) |
| OQ-2 | Should circuit breaker state be shared across agent instances of the same type? | No — per-agent by default is safer |
| OQ-3 | Should retry budgets count against the agent's token budget? | Yes — add to D8 (Economics) spec |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec |

---

*AgentPave Dimension 3 — Communication — v1.1*
