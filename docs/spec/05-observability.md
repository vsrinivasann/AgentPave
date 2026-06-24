# AgentPave Dimension 5 — Observability

**Spec version:** 1.1  
**Stability:** Beta  
**Depends on:** D1 (Identity), D2 (Lifecycle)  
**Required by:** D6 (Reliability — drift detection emits observability events)  
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

Agents fail in ways that look like success — well-formed but wrong outputs, redundant tool calls, semantically invalid actions. Agent observability is the practice of tracing, monitoring, and evaluating autonomous agents in production — capturing every model call, tool execution, and reasoning step as structured spans so you can answer the one question that matters when something goes wrong: **why did the agent do that?**

In 2026, observability is built on OpenTelemetry's GenAI semantic conventions — a vendor-neutral standard that every major agent observability platform (Langfuse, LangSmith, Braintrust, Arize, Laminar) supports as an exporter target.

AgentPave makes observability a core primitive — not an optional extension. The observability backend is pluggable; the instrumentation is mandatory.

### 1.2 Production Consequence of Getting This Wrong

Three principles separate teams that ship reliable AI agents at scale from those that struggle: instrument everything before you optimize anything; close the loop from production trace to regression dataset; let automated evaluations replace instinct-based release decisions.

Without observability:
- A production failure in a 2,000-span trace that fails 4 tool calls deep cannot be debugged
- Quality regressions go undetected until users complain
- The feedback loop from production to evaluation dataset never closes
- Compliance requirements for audit trails cannot be met

---

## 2. Scope Boundary

### 2.1 In Scope

- Structured log emission for every agent action
- OpenTelemetry span emission for every tool call
- Lifecycle state transition events
- AgentResult schema (used in all spans and logs)
- Pluggable observability backend via extension point
- Null/stdout backend as default (no external dependency required)
- Trace ID propagation across all events for a single task

### 2.2 Out of Scope

- Observability backend implementations (LangSmith, Langfuse, Braintrust etc.) — those are the AgentPave-Observe extension
- Online evaluation and scoring — that is AgentPave-Eval extension
- Alerting and dashboards — that is the observability backend
- LLM call tracing beyond what OpenTelemetry GenAI conventions define — that is a future spec version
- Session replay — that is an advanced observability backend feature

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **OpenTelemetry is the standard** | OTel is the CNCF standard for distributed tracing. Every major agent observability platform supports it as an exporter. Building on OTel means AgentPave traces integrate with any existing APM infrastructure. |
| **Structured logs use the same schema as OTel spans** | Consistency between logs and spans means the same trace_id links both — enabling correlation without custom tooling. |
| **Null/stdout backend is the default** | An observability backend being unavailable MUST NOT prevent agent execution. The default backend writes to stdout and never fails. Production teams replace this with their chosen platform. |
| **SDK-based instrumentation, not proxy-based** | SDK instrumentation means traces are captured within the agent's own infrastructure. No proxy = no credential exposure, no single point of failure between the agent and its observability backend. |
| **trace_id is the primary correlation key** | All logs and spans for a single task share one trace_id. This enables cross-service debugging without custom correlation fields. |
| **AgentResult is the standard output schema** | All tool calls, LLM calls, and agent actions return an AgentResult. Standardising this schema means observability backends can parse any AgentPave event without agent-specific configuration. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **Structured Log Entry** | A JSON-serialisable record emitted for every agent action. Contains all required fields defined in this spec. Written to the configured observability backend or stdout. |
| **OTel Span** | An OpenTelemetry span representing a unit of work (tool call, LLM call, lifecycle transition). Follows OTel GenAI semantic conventions. |
| **Trace ID** | A UUID v4 that identifies all spans and log entries belonging to a single agent task execution. Propagated through all downstream calls. |
| **Span ID** | A UUID v4 that uniquely identifies a single span within a trace. |
| **Parent Span ID** | The span_id of the parent span. Used to construct the span hierarchy (trace tree). |
| **AgentResult** | The standard structured return type for any agent action. Schema: `{status, output, error, trace_id, timestamp}`. |
| **Action Type** | The category of agent action being logged. One of: `tool_call`, `llm_call`, `lifecycle_transition`, `memory_read`, `memory_write`, `drift_event`, `error`. |
| **Observability Backend** | The external system that receives structured logs and OTel spans. Examples: LangSmith, Langfuse, Braintrust, stdout (default). |
| **Null Backend** | The default observability backend. Writes to stdout in JSON format. Never fails. Requires no external dependencies. |
| **OTel GenAI Conventions** | OpenTelemetry Semantic Conventions for Generative AI — the vendor-neutral standard for AI observability spans and attributes. |

---

## 5. Data Models

### 5.1 AgentResult (canonical definition)

```python
from pydantic import BaseModel, Field
from typing import Any, Optional
from enum import Enum


class AgentResultStatus(str, Enum):
    SUCCESS   = "success"
    ERROR     = "error"
    TIMEOUT   = "timeout"
    CANCELLED = "cancelled"


class AgentResult(BaseModel):
    """
    The standard structured return type for any agent action.
    Used in all structured log entries and OTel spans.
    This is the CANONICAL definition — all other dimensions reference this schema.
    """
    status: AgentResultStatus = Field(
        ...,
        description="Outcome of the action."
    )
    output: Optional[Any] = Field(
        None,
        description=(
            "The action's output payload. "
            "None when status is error, timeout, or cancelled. "
            "Must be JSON-serialisable."
        )
    )
    error: Optional[str] = Field(
        None,
        description=(
            "AgentPave error type and message when status is error. "
            "Format: '<ErrorClassName>: <message>'. "
            "None when status is success."
        )
    )
    trace_id: str = Field(
        ...,
        description="UUID v4. The task-level trace ID. Shared by all spans in this task."
    )
    timestamp: str = Field(
        ...,
        description="ISO 8601 UTC. When this result was produced."
    )

    model_config = {"extra": "forbid"}
```

### 5.2 ActionType

```python
class ActionType(str, Enum):
    TOOL_CALL            = "tool_call"
    LLM_CALL             = "llm_call"
    LIFECYCLE_TRANSITION = "lifecycle_transition"
    MEMORY_READ          = "memory_read"
    MEMORY_WRITE         = "memory_write"
    DRIFT_EVENT          = "drift_event"
    ERROR                = "error"
```

### 5.3 StructuredLogEntry

```python
class StructuredLogEntry(BaseModel):
    """
    A structured log entry emitted for every agent action.
    Written to the configured observability backend.
    """

    # --- Required correlation fields ---
    log_id: str = Field(..., description="UUID v4. Unique ID for this log entry.")
    trace_id: str = Field(..., description="UUID v4. Task-level trace ID.")
    span_id: str = Field(..., description="UUID v4. ID of the OTel span for this action.")
    parent_span_id: Optional[str] = Field(
        None,
        description="UUID v4. Parent span ID. None for root spans (task start)."
    )

    # --- Required identity fields ---
    agent_id: str = Field(..., description="UUID v4.")
    agent_name: str = Field(..., description="Human-readable agent name.")
    agent_version: str = Field(..., description="SemVer. Agent definition version.")
    task_id: str = Field(..., description="UUID v4. Active task ID.")

    # --- Required action fields ---
    action_type: ActionType = Field(...)
    action_name: str = Field(
        ...,
        description=(
            "Specific name of the action. "
            "For tool_call: the tool name. "
            "For llm_call: the model identifier. "
            "For lifecycle_transition: '<from_state>→<to_state>'. "
            "For others: a descriptive identifier."
        )
    )
    timestamp: str = Field(..., description="ISO 8601 UTC. When this action occurred.")
    duration_ms: int = Field(
        ...,
        ge=0,
        description="Duration of the action in milliseconds."
    )

    # --- Result ---
    result: AgentResult = Field(
        ...,
        description="Outcome of the action. AgentResult schema."
    )

    # --- Optional enrichment ---
    metadata: dict = Field(
        default_factory=dict,
        description=(
            "Additional structured context. Not part of the required schema. "
            "Examples: {'tool_input_tokens': 150, 'model': 'claude-sonnet-4-6'}"
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.4 OTelSpanAttributes

```python
class OTelSpanAttributes(BaseModel):
    """
    OpenTelemetry span attributes following OTel GenAI semantic conventions.
    These attributes are set on every OTel span emitted by AgentPave.
    """

    # --- AgentPave-specific attributes ---
    agentpave_agent_id: str         # gen_ai.agent.id
    agentpave_agent_name: str       # gen_ai.agent.name
    agentpave_agent_version: str    # gen_ai.agent.version
    agentpave_task_id: str          # gen_ai.request.id (repurposed)
    agentpave_action_type: str      # gen_ai.operation.name

    # --- OTel GenAI conventions (tool calls) ---
    gen_ai_tool_name: Optional[str] = None      # gen_ai.tool.name
    gen_ai_tool_call_id: Optional[str] = None   # gen_ai.tool.call.id

    # --- OTel GenAI conventions (LLM calls) ---
    gen_ai_system: Optional[str] = None              # gen_ai.system (e.g. "anthropic")
    gen_ai_request_model: Optional[str] = None       # gen_ai.request.model
    gen_ai_usage_input_tokens: Optional[int] = None  # gen_ai.usage.input_tokens
    gen_ai_usage_output_tokens: Optional[int] = None # gen_ai.usage.output_tokens

    # --- Standard OTel attributes ---
    span_kind: str = "client"   # Always "client" for agent-initiated actions
    status_code: str            # "OK" or "ERROR"
    status_message: Optional[str] = None  # Error message if status_code is "ERROR"

    model_config = {"extra": "allow"}  # Allow additional OTel attributes
```

### 5.5 ObservabilityEvent

```python
class ObservabilityEventType(str, Enum):
    AGENT_STARTED      = "agent.started"
    AGENT_COMPLETED    = "agent.completed"
    TOOL_CALL_STARTED  = "tool_call.started"
    TOOL_CALL_COMPLETED = "tool_call.completed"
    LLM_CALL_STARTED   = "llm_call.started"
    LLM_CALL_COMPLETED = "llm_call.completed"
    DRIFT_DETECTED     = "drift.detected"
    ERROR_OCCURRED     = "error.occurred"
    LIFECYCLE_CHANGED  = "lifecycle.changed"


class ObservabilityEvent(BaseModel):
    """
    High-level event emitted at key points in agent execution.
    Wraps a StructuredLogEntry with event type metadata.
    """
    event_id: str = Field(..., description="UUID v4.")
    event_type: ObservabilityEventType = Field(...)
    log_entry: StructuredLogEntry = Field(...)

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Tool Call Log Entry and Span

```python
# Emitted when a tool call completes
log_entry = StructuredLogEntry(
    log_id="<uuid4>",
    trace_id="<task_trace_id>",
    span_id="<uuid4>",
    parent_span_id="<parent_span_id>",
    agent_id="f47ac10b-58cc-4372-a567-0e02b2c3d479",
    agent_name="customer-support-router",
    agent_version="1.0.0",
    task_id="<task_uuid>",
    action_type=ActionType.TOOL_CALL,
    action_name="web_search",
    timestamp="2026-06-09T12:00:01.234Z",
    duration_ms=342,
    result=AgentResult(
        status=AgentResultStatus.SUCCESS,
        output={"results": ["result 1", "result 2"]},
        error=None,
        trace_id="<task_trace_id>",  # Same trace_id as log entry
        timestamp="2026-06-09T12:00:01.234Z"
    ),
    metadata={"tool_call_id": "<call_id>"}
)
# Key invariant: log_entry.trace_id == log_entry.result.trace_id
assert log_entry.trace_id == log_entry.result.trace_id
```

---

## 7. Interfaces & Contracts

### 7.1 ObservabilityBackend

```python
from abc import ABC, abstractmethod


class ObservabilityBackend(ABC):
    """
    Pluggable backend for receiving structured logs and OTel spans.
    Attaches at extension point: observability.backend.

    Contract:
    - emit() MUST NOT raise. If the backend is unavailable, it MUST
      fall back to stdout and log the backend failure. Agent execution
      MUST NOT be blocked by observability backend unavailability.
    - emit() MUST complete within 10ms p99 (NFR from SPEC.md Section 8.3).
    """

    @abstractmethod
    def emit(self, event: ObservabilityEvent) -> None:
        """
        Emit an observability event to the backend.
        MUST NOT raise. MUST NOT block agent execution.
        If backend unavailable: write to stdout and continue.
        """
        ...

    @abstractmethod
    def flush(self) -> None:
        """
        Flush any buffered events to the backend.
        Called at task completion and agent retirement.
        MAY raise if flush fails after retries.
        """
        ...
```

### 7.2 AgentObserver

```python
class AgentObserver(ABC):
    """
    The core observability interface injected into every AgentPave component.
    Produces ObservabilityEvents and routes them to the configured backend.

    Usage: every component that performs observable actions (ToolCaller,
    LifecycleManager, MemoryStore) receives an AgentObserver instance
    and calls the appropriate observe_*() method.
    """

    @abstractmethod
    def observe_tool_call(
        self,
        agent_id: str,
        task_id: str,
        trace_id: str,
        tool_name: str,
        arguments: dict,
        result: AgentResult,
        duration_ms: int
    ) -> None:
        """
        Record a tool call event. Called by ToolCaller after every tool invocation.
        Emits both a StructuredLogEntry and an OTel span.
        MUST NOT raise. MUST complete within 10ms p99.
        """
        ...

    @abstractmethod
    def observe_lifecycle_transition(
        self,
        agent_id: str,
        from_state: str,
        to_state: str,
        trigger: str,
        actor: str,
        trace_id: str,
        duration_ms: int
    ) -> None:
        """
        Record a lifecycle state transition event. Called by LifecycleManager.
        MUST NOT raise.
        """
        ...

    @abstractmethod
    def observe_memory_operation(
        self,
        agent_id: str,
        task_id: str,
        trace_id: str,
        memory_type: str,
        operation: str,      # "read" or "write"
        store_id: str,
        result: AgentResult,
        duration_ms: int
    ) -> None:
        """
        Record a memory read or write event. Called by memory implementations.
        MUST NOT raise.
        """
        ...

    @abstractmethod
    def observe_drift_event(
        self,
        agent_id: str,
        drift_type: str,     # "embedding" or "behavioural"
        severity: str,       # "low", "medium", "high"
        details: dict,
        trace_id: str
    ) -> None:
        """
        Record a drift detection event. Called by D6 drift detection.
        MUST NOT raise.
        """
        ...

    @abstractmethod
    def observe_error(
        self,
        agent_id: str,
        task_id: str,
        trace_id: str,
        error: Exception,
        action_type: ActionType,
        action_name: str
    ) -> None:
        """
        Record an error event. Called whenever an AgentPaveError is raised.
        MUST NOT raise.
        """
        ...

    @abstractmethod
    def start_trace(self, agent_id: str, task_id: str) -> str:
        """
        Start a new trace for a task. Returns the trace_id (UUID v4).
        Called at task start (REGISTERED → RUNNING transition).
        """
        ...

    @abstractmethod
    def end_trace(self, trace_id: str, final_result: AgentResult) -> None:
        """
        End a trace and flush all buffered events.
        Called at task completion or agent RETIRED.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 Mandatory Emission

**THE SYSTEM SHALL** emit a `StructuredLogEntry` for every agent action of types: `tool_call`, `llm_call`, `lifecycle_transition`, `memory_read`, `memory_write`, `drift_event`, `error`.

**THE SYSTEM SHALL** emit an OTel span for every `tool_call` action.

**THE SYSTEM SHALL** emit an OTel span for every `llm_call` action.

**THE SYSTEM SHALL** set the same `trace_id` on every log entry and span belonging to the same agent task.

**THE SYSTEM SHALL** set `log_entry.trace_id == log_entry.result.trace_id` — the trace ID in the result must match the log entry's trace ID.

### 8.2 Non-Blocking

**THE SYSTEM SHALL NOT** allow observability backend unavailability to block or fail agent execution.

**WHEN** the configured observability backend is unavailable **THE SYSTEM SHALL** fall back to the null/stdout backend, log the backend failure to stderr, and continue agent execution.

**THE SYSTEM SHALL** emit all observability events within 10ms p99 (non-blocking, buffered emission is acceptable).

### 8.3 Default Backend

**THE SYSTEM SHALL** use the null/stdout backend when no observability backend is configured.

**THE SYSTEM SHALL** make the null/stdout backend the default — no environment variables or configuration are required to run an agent with basic observability.

### 8.4 Trace ID Propagation

**THE SYSTEM SHALL** propagate `trace_id` to all downstream calls from a tool invocation — including MCP tool calls (via `X-AgentPave-Trace-Id` header, D3) and A2A messages.

**THE SYSTEM SHALL** create a new `trace_id` for every new task execution. Different task executions MUST have different trace IDs even for the same agent.

### 8.5 Span Hierarchy

**THE SYSTEM SHALL** set `parent_span_id` on every span that is a child of another span, enabling trace tree construction.

**THE SYSTEM SHALL** make the task-level span (agent started) the root span — it has no `parent_span_id`.

---

## 9. Three-Tier Boundary System

### ALWAYS
- Emit a StructuredLogEntry for every agent action (all 7 action types)
- Emit an OTel span for every tool_call and llm_call
- Set trace_id consistently across all events for one task
- Set log_entry.trace_id == log_entry.result.trace_id
- Default to null/stdout backend when no backend is configured
- Fall back to stdout when configured backend is unavailable
- Complete emission within 10ms p99

### ASK FIRST
- Disabling observability for a specific action type — confirm this is intentional and documented
- Sending observability data to an external third-party backend — confirm data residency compliance

### NEVER
- Block agent execution due to observability backend failure
- Raise from emit() — always catch and fall back to stdout
- Log sensitive data (secrets, PII) in structured log entries
- Create a new trace_id mid-task — one task = one trace_id

---

## 10. Error Handling

| Scenario | Error Type | Recoverable | Required context |
|---|---|---|---|
| Observability backend unavailable | `AgentPave.ObservabilityBackendError` | Yes — fall back to stdout | `backend_type` |
| Backend unavailability during flush | `AgentPave.ObservabilityBackendError` | Yes — retry once, then warn | `backend_type, pending_events` |

Note: `ObservabilityBackendError` is logged to stderr but NEVER propagated to the agent caller. Observability failures are always handled internally by the observer.

---

## 11. Acceptance Criteria

```
Criteria ID:  D5-001
Stability:    Beta
Given:        A running agent that invokes a tool (success)
When:         The tool call completes
Then:         A StructuredLogEntry is emitted with:
              - action_type == ActionType.TOOL_CALL
              - action_name == the tool name
              - result.status == "success"
              - result.trace_id == log_entry.trace_id
              - duration_ms >= 0
              - all required fields present and non-null
Pass:         All conditions true
Fail:         Any field missing, null, or trace_ids mismatched
Error raised: None

---

Criteria ID:  D5-002
Stability:    Beta
Given:        A running agent that invokes a tool
When:         The tool call completes
Then:         An OTel span is emitted with:
              - gen_ai_tool_name == tool name
              - agentpave_agent_id == agent.agent_id
              - trace_id matches the task's trace_id
              - duration set correctly
              - parent_span_id set (not None) for non-root spans
Pass:         All OTel span attributes present and correctly set
Fail:         Span not emitted, or required attributes missing
Error raised: None

---

Criteria ID:  D5-003
Stability:    Beta
Given:        A task execution that includes 3 tool calls
When:         All 3 tool calls complete
Then:         All 3 StructuredLogEntry objects have the same trace_id
              All 3 OTel spans have the same trace_id
              All log entries and spans have matching trace_ids
Pass:         trace_id identical across all 6 emitted objects
Fail:         Any trace_id differs from the others
Error raised: None

---

Criteria ID:  D5-004
Stability:    Beta
Given:        No observability backend is configured
When:         An agent action is observed
Then:         The null/stdout backend emits the event to stdout in JSON format
              Agent execution is NOT blocked
Pass:         JSON output appears on stdout, agent continues executing normally
Fail:         Agent execution blocked, exception raised, or no output
Error raised: None

---

Criteria ID:  D5-005
Stability:    Beta
Given:        An observability backend that raises an exception on emit()
When:         An agent action is observed
Then:         The AgentObserver catches the exception
              Falls back to stdout emission
              Agent execution is NOT blocked or interrupted
Pass:         Event appears on stdout as fallback, agent continues, no exception propagated
Fail:         Exception propagated to agent, agent execution blocked or fails
Error raised: None (internally caught)

---

Criteria ID:  D5-006
Stability:    Beta
Given:        A running agent that executes a lifecycle transition (RUNNING → PAUSED)
When:         The transition completes
Then:         A StructuredLogEntry is emitted with:
              - action_type == ActionType.LIFECYCLE_TRANSITION
              - action_name == "RUNNING→PAUSED"
              - result.status == "success"
Pass:         Log entry emitted with correct action_type and action_name
Fail:         No log entry for lifecycle transition, or wrong action_type
Error raised: None

---

Criteria ID:  D5-007
Stability:    Beta
Given:        Two separate task executions of the same agent
When:         Both tasks complete
Then:         Each task has a distinct trace_id
              No log entries from task A share a trace_id with log entries from task B
Pass:         trace_id(task_A) != trace_id(task_B),
              no cross-task trace_id sharing
Fail:         Same trace_id used for both tasks
Error raised: None
```

---

## 12. Definition of Done

- [ ] `AgentResultStatus` enum and `AgentResult` Pydantic model implemented
- [ ] `ActionType` enum implemented
- [ ] `StructuredLogEntry` Pydantic model implemented with all required fields
- [ ] `OTelSpanAttributes` model implemented
- [ ] `ObservabilityEventType` enum and `ObservabilityEvent` model implemented
- [ ] `ObservabilityBackend` abstract interface implemented
- [ ] `AgentObserver` abstract interface implemented with all 7 methods
- [ ] Concrete null/stdout backend implemented (never raises, always emits JSON)
- [ ] Concrete `AgentObserver` implementation routes to configured backend
- [ ] Backend unavailability falls back to stdout without propagating exceptions
- [ ] trace_id is consistent across all events for one task
- [ ] log_entry.trace_id == log_entry.result.trace_id enforced
- [ ] Emission completes within 10ms p99 (NFR)
- [ ] All 7 acceptance criteria pass: D5-001 through D5-007
- [ ] Log coverage 100% and span coverage 100% for tool calls verified (NFR)

---

## 13. Task Breakdown

```
Task D5-T1: Implement AgentResult Pydantic model (canonical definition)
Task D5-T2: Implement ActionType, AgentResultStatus, ObservabilityEventType enums
Task D5-T3: Implement StructuredLogEntry Pydantic model
Task D5-T4: Implement OTelSpanAttributes model
Task D5-T5: Implement ObservabilityEvent Pydantic model
Task D5-T6: Implement ObservabilityBackend abstract interface
Task D5-T7: Implement null/stdout concrete backend (JSON, never raises)
Task D5-T8: Implement AgentObserver abstract interface (7 methods)
Task D5-T9: Implement concrete AgentObserver with backend routing and fallback
Task D5-T10: Implement trace_id lifecycle (start_trace, propagation, end_trace)
Task D5-T11: Wire AgentObserver into ToolCaller (D3), LifecycleManager (D2), MemoryStores (D4)
Task D5-T12: Run all 7 acceptance criteria
Task D5-T13: Verify 10ms p99 emission latency (NFR)
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Blocking agent execution on observability**
Observability is a side effect of execution — it MUST NOT be in the critical path. An observability backend going down should never stop an agent from completing its task.

**Anti-Pattern 2 — Logging sensitive data**
Memory entries, tool arguments, and LLM outputs may contain PII, secrets, or sensitive business data. Logging these verbatim creates compliance liabilities. Use field-level redaction or reference patterns for sensitive fields.

**Anti-Pattern 3 — Reusing trace_id across tasks**
Reusing trace IDs from previous tasks corrupts the trace tree — it becomes impossible to distinguish which events belong to which task execution. One task = one UUID v4 trace_id.

**Anti-Pattern 4 — Separate log and trace systems without correlation**
Using a different ID in logs and spans breaks cross-system debugging. Always ensure log_entry.trace_id == span.trace_id for the same action. This is the most common observability implementation mistake.

**Anti-Pattern 5 — Synchronous blocking backends**
An observability backend that writes synchronously and blocks on network I/O adds latency to every agent action. Use async or buffered emission. The 10ms p99 target is only achievable with non-blocking emission.

---

## 15. Reference Implementation Notes

### 15.1 AgentPave-LangGraph
- Use `opentelemetry-sdk` + `opentelemetry-exporter-otlp` for OTel span emission
- LangSmith integration: `LangSmithSpanExporter` as the observability backend
- Langfuse integration: `LangfuseSpanExporter` as the observability backend
- AgentObserver should be injected as a dependency into ToolCaller and LifecycleManager

### 15.2 AgentPave-MAF
- MAF has native OpenTelemetry integration — wire AgentObserver to MAF's built-in telemetry
- Azure Monitor / Application Insights as the default observability backend for MAF
- `X-AgentPave-Trace-Id` header propagation integrates with Azure distributed tracing

### 15.3 OTel Backend Compatibility

| Backend | OTel Support | Notes |
|---|---|---|
| LangSmith | ✅ Full (March 2026) | Best for LangGraph stacks |
| Langfuse | ✅ Full | Open source, self-hostable |
| Braintrust | ✅ Full | Best evaluation integration |
| Arize AX | ✅ Full | Best for drift detection |
| Azure Monitor | ✅ Full | Best for MAF stacks |
| stdout (null) | ✅ Always | Default, no dependencies |

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should AgentPave define a standard sampling rate for high-volume agents? | No |
| OQ-2 | Should drift_event spans include the full embedding vector? (privacy concern) | Yes — clarify before Stable |
| OQ-3 | Should LLM call spans include input/output content, or only token counts? | Yes — data residency implications |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec |

---

*AgentPave Dimension 5 — Observability — v1.1*
