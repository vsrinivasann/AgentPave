# AgentPave Dimension 6 — Reliability

**Spec version:** 1.1  
**Stability:** Beta  
**Depends on:** D1 (Identity), D2 (Lifecycle), D3 (Communication), D4 (Memory), D5 (Observability)  
**Required by:** Nothing in MVP — this is the final MVP dimension  
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

Reliability is what transforms an agent from a demo into a production system. It is the final MVP dimension because it depends on all others — it requires Identity (to associate checkpoints with agents), Lifecycle (to know when to checkpoint), Communication (tool call boundaries trigger checkpoints), Memory (checkpoints include memory state), and Observability (drift events are observed).

Reliability in AgentPave has four pillars:

1. **Durable Execution** — agents survive any failure and resume from their exact stopping point
2. **Deterministic Task Routing** — deterministic tasks never touch the LLM
3. **Reliability Contracts** — every agent declares its quality thresholds
4. **Drift Detection** — silent degradation is detected and surfaced

### 1.2 Production Consequence of Getting This Wrong

Complex agent workflows can take hours. If an agent fails late in a multi-step process due to an API timeout, you don't want to restart from zero. Completed model calls and tool invocations replayed from checkpoint rather than re-executed save both time and LLM cost.

Separate from failure recovery, agents that route deterministic tasks through the LLM produce non-deterministic results at unpredictable intervals — currency conversions and database queries that silently produce wrong answers. And agents that degrade silently over time as their memory becomes stale (embedding drift) or their output quality drops (behavioural drift) cause progressive, hard-to-diagnose production issues.

---

## 2. Scope Boundary

### 2.1 In Scope

- Checkpoint creation, storage, and retrieval
- Resume-from-checkpoint on any failure type
- Deterministic task router (classify + route away from LLM)
- Reliability Contract declaration and enforcement
- Embedding drift detection
- Behavioural drift detection
- Drift event emission via D5 (Observability)

### 2.2 Out of Scope

- Checkpoint encryption at rest — that is D6 (Security, post-MVP)
- Checkpoint replication across regions — that is infrastructure concern
- Agent load testing and capacity planning — that is operations concern
- Evaluation framework for quality scoring — that is AgentPave-Eval extension
- Memory consolidation between sessions — that is AgentPave-Memory extension
- Token budget enforcement — that is D8 (Economics, post-MVP)

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **Checkpoint at every tool call boundary** | Tool calls are the highest-risk boundaries — they change external state. Checkpointing before and after every tool call means the agent can always replay or skip completed tool calls. Checkpointing only at explicit developer-defined points misses failures between those points. |
| **Checkpoints include full agent state** | Partial checkpoints (context only, or memory only) leave gaps. A full checkpoint includes: working memory, task context, last LLM output, tool call history, and episodic memory reference. |
| **Deterministic tasks are classified by declaration** | Runtime LLM-based classification of "is this deterministic?" is circular — you'd use the LLM to decide whether to use the LLM. Classification is declared explicitly in the AgentDefinition. |
| **Reliability Contracts are mandatory** | The reliability contract allows an agent's quality standards to be enforced at runtime, not just tested in CI. Without it, quality regressions are invisible until users complain. |
| **Drift detection uses baseline comparison** | Both embedding drift (vector centroid shift) and behavioural drift (quality metric decline) are detected by comparing current measurements against a stored baseline. The baseline is established at deployment time. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **Checkpoint** | A durable, serialised snapshot of an agent's complete execution state at a specific point in time. Stored externally. Used to resume after failure. |
| **Checkpoint Boundary** | A point in execution where a checkpoint is created. Defined as: before every tool call and after every tool result is received. |
| **Resume** | The act of restoring an agent's state from a checkpoint and continuing execution from the point the checkpoint was taken — not from the beginning of the task. |
| **Replay** | Re-executing a completed tool call from its recorded result (stored in the checkpoint) without calling the tool again. Used for idempotency after checkpoint restore. |
| **Deterministic Task** | A task whose correct output is exactly computable from its inputs. Examples: arithmetic, currency conversion, date arithmetic, SQL query execution, data formatting. Must never be routed to the LLM. |
| **Task Router** | The component that classifies an incoming task as deterministic or non-deterministic and routes it to the appropriate handler (typed code or LLM). |
| **Reliability Contract** | A per-agent declaration of: hallucination tolerance level, grounding requirements, and quality threshold. Enforced at runtime. Violation raises an error. |
| **Quality Score** | A numeric score [0.0, 1.0] evaluating the quality of an LLM output against the agent's reliability contract. Computed by the evaluation function registered for the agent's domain. |
| **Embedding Drift** | The condition where a semantic memory store's vector representations no longer accurately reflect current reality — typically caused by the world changing while embeddings remain static. |
| **Behavioural Drift** | The condition where an agent's output quality measurably declines over time — detected by comparing current quality scores against the baseline established at deployment. |
| **Drift Baseline** | The recorded quality metrics and embedding statistics captured at deployment time. Used as the reference for drift detection comparisons. |
| **Drift Threshold** | The declared percentage deviation from baseline that triggers a drift detection event. Configured per-agent in the DriftConfig. |

---

## 5. Data Models

### 5.1 CheckpointData

```python
from pydantic import BaseModel, Field
from typing import Any, Optional


class ToolCallRecord(BaseModel):
    """Record of a completed tool call — stored in checkpoint for replay."""
    call_id: str
    tool_name: str
    arguments: dict
    result: "AgentResult"   # From D5
    completed_at: str       # ISO 8601 UTC
    replayed: bool = False  # True if this result was replayed from checkpoint

    model_config = {"extra": "forbid"}


class CheckpointData(BaseModel):
    """
    Complete serialised state of an agent at a checkpoint boundary.
    Stored durably. Used to resume after any failure type.
    """
    checkpoint_id: str = Field(..., description="UUID v4.")
    agent_id: str = Field(..., description="UUID v4.")
    task_id: str = Field(..., description="UUID v4.")
    trace_id: str = Field(..., description="UUID v4. From D5.")
    sequence_number: int = Field(
        ...,
        ge=0,
        description=(
            "Monotonically increasing checkpoint number within a task. "
            "Starts at 0 for the first checkpoint. "
            "Used to identify the 'latest' checkpoint and detect ordering issues."
        )
    )

    # --- Execution state ---
    working_memory_snapshot: dict = Field(
        ...,
        description=(
            "Serialised contents of working memory at checkpoint time. "
            "Used to restore working memory on resume."
        )
    )
    task_context: dict = Field(
        ...,
        description="The original task goal, parameters, and any task-level state."
    )
    last_llm_output: Optional[str] = Field(
        None,
        description="The most recent LLM output before this checkpoint."
    )
    completed_tool_calls: list[ToolCallRecord] = Field(
        default_factory=list,
        description=(
            "All tool calls completed up to this checkpoint. "
            "On resume, these calls are replayed from their stored results "
            "rather than re-executed — prevents duplicate side effects."
        )
    )
    pending_tool_call: Optional[dict] = Field(
        None,
        description=(
            "The tool call that was in-flight when this checkpoint was taken (if any). "
            "On resume, this call is re-executed (not replayed) "
            "since we don't know if it completed before the failure."
        )
    )

    # --- Memory references ---
    episodic_store_ids: list[str] = Field(
        default_factory=list,
        description="UUIDs of episodic memory stores active at checkpoint time."
    )

    # --- Metadata ---
    created_at: str = Field(..., description="ISO 8601 UTC.")
    runtime: str = Field(..., description="Runtime that created this checkpoint: 'langgraph' or 'maf'.")

    model_config = {"extra": "forbid"}
```

### 5.2 DeterministicTaskDefinition

```python
from enum import Enum
from typing import Callable


class DeterministicTaskType(str, Enum):
    ARITHMETIC          = "arithmetic"          # Math calculations
    CURRENCY_CONVERSION = "currency_conversion" # Currency exchange
    DATE_ARITHMETIC     = "date_arithmetic"     # Date/time calculations
    DATA_FORMATTING     = "data_formatting"     # String/data transformations
    SQL_QUERY           = "sql_query"           # Database queries
    UNIT_CONVERSION     = "unit_conversion"     # Unit/measurement conversion
    LOOKUP              = "lookup"              # Exact key-value lookups
    CUSTOM              = "custom"              # Developer-defined


class DeterministicTaskDefinition(BaseModel):
    """
    Declaration of a deterministic task that should be routed to typed code.
    Included in AgentDefinition.deterministic_tasks[].
    """
    task_id: str = Field(..., description="Unique identifier for this task type.")
    task_type: DeterministicTaskType = Field(...)
    description: str = Field(
        ...,
        min_length=5,
        description="Human-readable description of what this deterministic task does."
    )
    input_pattern: str = Field(
        ...,
        description=(
            "Regex or keyword pattern that identifies this task in the agent's input. "
            "Used by the TaskRouter to classify incoming tasks."
        )
    )
    handler_function: str = Field(
        ...,
        description=(
            "Fully-qualified Python function path for the handler. "
            "Example: 'myagent.handlers.convert_currency'. "
            "This function receives the task input and returns the result."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.3 DriftConfig

```python
class DriftConfig(BaseModel):
    """
    Configuration for drift detection. Included in AgentDefinition.
    """
    embedding_drift_enabled: bool = Field(
        default=True,
        description="Whether to monitor for embedding drift in semantic memory."
    )
    behavioural_drift_enabled: bool = Field(
        default=True,
        description="Whether to monitor for behavioural (quality) drift."
    )
    embedding_drift_threshold_pct: float = Field(
        default=15.0,
        ge=1.0,
        le=50.0,
        description=(
            "Percentage shift in mean embedding cosine distance from baseline "
            "that triggers an embedding drift event. Default: 15%."
        )
    )
    behavioural_drift_threshold_pct: float = Field(
        default=10.0,
        ge=1.0,
        le=50.0,
        description=(
            "Percentage decline in mean quality score from baseline "
            "that triggers a behavioural drift event. Default: 10%."
        )
    )
    measurement_window: int = Field(
        default=100,
        ge=10,
        le=10000,
        description=(
            "Number of recent task outputs used to compute the current quality measurement. "
            "Drift is assessed when this window fills."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.4 DriftEvent

```python
class DriftSeverity(str, Enum):
    LOW    = "low"     # 10–20% deviation from baseline
    MEDIUM = "medium"  # 20–35% deviation from baseline
    HIGH   = "high"    # >35% deviation from baseline


class DriftEvent(BaseModel):
    """
    Emitted via D5 (Observability) when drift is detected.
    """
    event_id: str = Field(..., description="UUID v4.")
    agent_id: str = Field(..., description="UUID v4.")
    drift_type: str = Field(..., description="'embedding' or 'behavioural'.")
    severity: DriftSeverity = Field(...)
    baseline_value: float = Field(..., description="The baseline measurement.")
    current_value: float = Field(..., description="The current measurement.")
    deviation_pct: float = Field(..., description="Percentage deviation from baseline.")
    detected_at: str = Field(..., description="ISO 8601 UTC.")
    details: dict = Field(default_factory=dict)

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Checkpoint Sequence

```python
# Before tool call: checkpoint created (sequence_number=0)
checkpoint_0 = CheckpointData(
    checkpoint_id=str(uuid4()),
    agent_id="<agent_uuid>",
    task_id="<task_uuid>",
    trace_id="<trace_uuid>",
    sequence_number=0,
    working_memory_snapshot={"goal": "find best price for product X"},
    task_context={"goal": "find best price", "product": "X"},
    last_llm_output=None,
    completed_tool_calls=[],
    pending_tool_call={"tool": "web_search", "args": {"query": "product X price"}},
    episodic_store_ids=["<store_uuid>"],
    created_at="2026-06-09T12:00:00Z",
    runtime="langgraph"
)

# Tool call completes. After tool result: checkpoint created (sequence_number=1)
checkpoint_1 = CheckpointData(
    sequence_number=1,
    completed_tool_calls=[
        ToolCallRecord(tool_name="web_search", result=AgentResult(status="success", output={...}), ...)
    ],
    pending_tool_call=None,  # No pending call
    ...
)

# If agent fails now: resume from checkpoint_1 (sequence_number=1)
# completed_tool_calls[0] is replayed from stored result — web_search is NOT called again
```

### 6.2 Deterministic Task Routing

```python
# Declaration in AgentDefinition
deterministic_tasks = [
    DeterministicTaskDefinition(
        task_id="currency_conversion",
        task_type=DeterministicTaskType.CURRENCY_CONVERSION,
        description="Convert between currency amounts using a fixed rate.",
        input_pattern=r"convert \d+ [A-Z]{3} to [A-Z]{3}",
        handler_function="myagent.handlers.convert_currency"
    )
]

# At runtime:
task_router.route("convert 100 USD to EUR at rate 0.92")
# → routes to convert_currency() handler
# → result: {"eur": 92.0}
# → NO LLM call made
```

---

## 7. Interfaces & Contracts

### 7.1 CheckpointStore

```python
from abc import ABC, abstractmethod


class CheckpointStore(ABC):
    """
    Durable storage for agent checkpoints.
    Must be durable: checkpoints survive process restart, memory failure,
    and infrastructure fault.
    """

    @abstractmethod
    def save(self, checkpoint: CheckpointData) -> None:
        """
        Durably persist a checkpoint.
        MUST complete before returning (synchronous, durable write).

        Raises:
            AgentPave.CheckpointError: checkpoint could not be saved.
                If this raises, the calling code MUST halt the agent task —
                continuing without a checkpoint violates the durability invariant.
        """
        ...

    @abstractmethod
    def load_latest(self, agent_id: str, task_id: str) -> CheckpointData:
        """
        Load the most recent checkpoint for a task (highest sequence_number).

        Raises:
            AgentPave.CheckpointError: no checkpoint found for this task,
                or checkpoint data is corrupted.
        """
        ...

    @abstractmethod
    def load(self, checkpoint_id: str) -> CheckpointData:
        """
        Load a specific checkpoint by ID.

        Raises:
            AgentPave.CheckpointError: checkpoint not found or corrupted.
        """
        ...

    @abstractmethod
    def list_checkpoints(self, agent_id: str, task_id: str) -> list[CheckpointData]:
        """
        List all checkpoints for a task, ordered by sequence_number ascending.
        Returns empty list if no checkpoints exist.
        """
        ...
```

### 7.2 TaskRouter

```python
class TaskRouter(ABC):
    """
    Classifies and routes incoming tasks.
    Deterministic tasks → typed handler function.
    Non-deterministic tasks → LLM.
    """

    @abstractmethod
    def route(self, task_input: str, agent_id: str) -> "TaskRouteResult":
        """
        Classify the task and return routing decision.

        Classification steps:
        1. For each DeterministicTaskDefinition registered for agent_id:
           check if task_input matches input_pattern (regex match).
        2. If match found: route = DETERMINISTIC, handler = definition.handler_function.
        3. If no match: route = LLM.

        Returns:
            TaskRouteResult with route_type and optional handler reference.

        Raises:
            AgentPave.DeterministicTaskLLMRouteError: if a task classified as
                DETERMINISTIC is attempted to be sent to LLM. (Called as a
                safeguard — should not happen in normal flow.)
        """
        ...

    @abstractmethod
    def execute_deterministic(
        self,
        handler_function: str,
        task_input: str,
        agent_id: str
    ) -> "AgentResult":
        """
        Execute a deterministic task using the registered handler function.

        Steps:
        1. Import and call the handler function.
        2. Wrap result in AgentResult.
        3. Verify NO LLM call was made during handler execution.

        Raises:
            AgentPave.DeterministicTaskLLMRouteError: if handler made an LLM call.
            Any exception from the handler function (propagated as-is).
        """
        ...


class TaskRouteType(str, Enum):
    DETERMINISTIC = "deterministic"
    LLM           = "llm"


class TaskRouteResult(BaseModel):
    route_type: TaskRouteType
    handler_function: Optional[str] = None   # Set when route_type == DETERMINISTIC
    deterministic_task_id: Optional[str] = None

    model_config = {"extra": "forbid"}
```

### 7.3 ReliabilityEnforcer

```python
class ReliabilityEnforcer(ABC):
    """
    Enforces the Reliability Contract declared in the AgentDefinition.
    Evaluates LLM outputs against quality thresholds.
    """

    @abstractmethod
    def evaluate(
        self,
        agent_id: str,
        llm_output: str,
        task_context: dict
    ) -> float:
        """
        Evaluate an LLM output and return a quality score [0.0, 1.0].
        Score is domain-specific. The evaluation function is registered per domain.

        Returns:
            float: Quality score. 0.0 = lowest quality, 1.0 = highest.

        Raises:
            AgentPave.ReliabilityContractViolationError: if score < quality_threshold
                from the agent's ReliabilityContract.
                context = {"score": float, "threshold": float, "agent_id": str}
        """
        ...

    @abstractmethod
    def register_evaluator(
        self,
        domain: str,
        evaluator_fn: "Callable[[str, dict], float]"
    ) -> None:
        """
        Register a domain-specific evaluator function.
        Called at agent setup. One evaluator per domain.

        Args:
            domain: The agent domain (e.g. 'finance', 'customer-support').
            evaluator_fn: Function(llm_output: str, context: dict) -> float [0.0, 1.0].
        """
        ...
```

### 7.4 DriftDetector

```python
class DriftDetector(ABC):
    """
    Detects embedding drift in semantic memory and behavioural drift in outputs.
    Emits DriftEvents via D5 (AgentObserver) when drift is detected.
    """

    @abstractmethod
    def record_baseline(self, agent_id: str, embeddings: list[list[float]], quality_scores: list[float]) -> None:
        """
        Record the baseline embedding statistics and quality scores at deployment time.
        Called once when an agent transitions to REGISTERED.
        """
        ...

    @abstractmethod
    def check_embedding_drift(self, agent_id: str, current_embeddings: list[list[float]]) -> Optional[DriftEvent]:
        """
        Compare current embedding distribution against baseline.
        Returns DriftEvent if deviation exceeds embedding_drift_threshold_pct.
        Returns None if no drift detected.
        Emits the DriftEvent via AgentObserver if drift detected.
        """
        ...

    @abstractmethod
    def check_behavioural_drift(self, agent_id: str, current_quality_scores: list[float]) -> Optional[DriftEvent]:
        """
        Compare current quality scores against baseline mean.
        Returns DriftEvent if deviation exceeds behavioural_drift_threshold_pct.
        Returns None if no drift detected.
        Emits the DriftEvent via AgentObserver if drift detected.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 Checkpoint Creation

**THE SYSTEM SHALL** create a CheckpointData before every tool call begins — with `pending_tool_call` set to the tool call about to execute.

**THE SYSTEM SHALL** create a CheckpointData after every tool call result is received — with the tool call moved to `completed_tool_calls` and `pending_tool_call` set to None.

**THE SYSTEM SHALL** persist checkpoints synchronously (durable write) before returning control to the agent. An unsaved checkpoint is not a checkpoint.

**WHEN** `CheckpointStore.save()` raises `AgentPave.CheckpointError` **THE SYSTEM SHALL** halt the agent task, transition to PAUSED state, and raise the error. Continuing without a saved checkpoint violates the durability invariant.

### 8.2 Resume from Checkpoint

**WHEN** an unhandled exception occurs during agent execution **THE SYSTEM SHALL** automatically call `CheckpointStore.load_latest()` for the current task.

**THE SYSTEM SHALL** restore working memory, task context, and completed tool call history from the checkpoint.

**THE SYSTEM SHALL** replay all tool calls in `checkpoint.completed_tool_calls` from their stored results — these tool calls MUST NOT be re-executed.

**THE SYSTEM SHALL** re-execute the tool call in `checkpoint.pending_tool_call` (if present) — this call's completion status is unknown, so it must be run again.

**THE SYSTEM SHALL** continue execution from the point after the restored state — not from the beginning of the task.

### 8.3 Deterministic Task Routing

**THE SYSTEM SHALL** classify every incoming task through `TaskRouter.route()` before routing it to the LLM or a handler.

**WHEN** a task matches a `DeterministicTaskDefinition.input_pattern` **THE SYSTEM SHALL** route it to the registered `handler_function` — NOT to the LLM.

**WHEN** a deterministic task is routed to the LLM **THE SYSTEM SHALL** raise `AgentPave.DeterministicTaskLLMRouteError` immediately.

**THE SYSTEM SHALL** verify that `execute_deterministic()` makes no LLM calls during handler execution. Any LLM call from within a deterministic handler raises `DeterministicTaskLLMRouteError`.

### 8.4 Reliability Contract Enforcement

**THE SYSTEM SHALL** evaluate every LLM output using `ReliabilityEnforcer.evaluate()`.

**WHEN** the quality score is below `reliability_contract.quality_threshold` **THE SYSTEM SHALL** raise `AgentPave.ReliabilityContractViolationError` with `context = {"score": float, "threshold": float}`.

**WHEN** `reliability_contract.grounding_required = True` **THE SYSTEM SHALL** verify that the LLM output is grounded (references retrieved context) before passing it to the quality evaluator.

### 8.5 Drift Detection

**THE SYSTEM SHALL** call `DriftDetector.check_embedding_drift()` after every batch of `drift_config.measurement_window` semantic memory queries.

**THE SYSTEM SHALL** call `DriftDetector.check_behavioural_drift()` after every batch of `drift_config.measurement_window` task completions.

**WHEN** drift is detected **THE SYSTEM SHALL** emit a `drift.detected` event via `AgentObserver.observe_drift_event()` with the DriftEvent payload.

**THE SYSTEM SHALL NOT** automatically pause or retire an agent on drift detection — this is a governance decision. Drift events are surfaced; action is taken by the operator.

---

## 9. Three-Tier Boundary System

### ALWAYS
- Create a checkpoint before every tool call and after every tool result
- Persist checkpoints synchronously before continuing execution
- Resume from the latest checkpoint on any unhandled exception
- Replay completed tool calls from stored results — never re-execute them
- Route all tasks through TaskRouter before LLM or handler invocation
- Raise DeterministicTaskLLMRouteError if a deterministic task reaches the LLM
- Evaluate every LLM output against the Reliability Contract
- Record drift baseline at agent deployment (REGISTERED transition)

### ASK FIRST
- Continuing execution after a CheckpointError — confirm with operator before proceeding without checkpoint
- Resetting the drift baseline — confirm this is an intentional recalibration, not a drift suppression
- Disabling drift detection for an agent — confirm and document justification

### NEVER
- Re-execute a tool call that is in completed_tool_calls (replay, don't re-execute)
- Continue a task after CheckpointStore.save() raises — halt and pause
- Route a deterministic task to the LLM
- Suppress or ignore a ReliabilityContractViolationError
- Auto-pause an agent on drift detection without operator instruction (emit event, don't act)
- Store checkpoints in ephemeral memory — checkpoints must survive process restart

---

## 10. Error Handling

| Scenario | Error Type | Recoverable | Required context |
|---|---|---|---|
| Checkpoint cannot be saved | `AgentPave.CheckpointError` | No — halt task, transition to PAUSED | `checkpoint_id, cause` |
| Checkpoint not found on resume | `AgentPave.CheckpointError` | No — task must restart | `agent_id, task_id` |
| Deterministic task routed to LLM | `AgentPave.DeterministicTaskLLMRouteError` | No | `task_input, task_id` |
| LLM output below quality threshold | `AgentPave.ReliabilityContractViolationError` | No | `score, threshold, agent_id` |

---

## 11. Acceptance Criteria

```
Criteria ID:  D6-001
Stability:    Beta
Given:        A running agent about to invoke a tool
When:         The tool call is initiated
Then:         A CheckpointData is persisted with:
              - sequence_number == N (monotonically increasing)
              - pending_tool_call set to the tool being called
              - completed_tool_calls contains all prior tool calls
Pass:         CheckpointStore has a record with the correct pending_tool_call,
              sequence_number is N, persisted before MCP call is made
Fail:         Checkpoint not created before tool call, or pending_tool_call not set
Error raised: None (if checkpoint save fails: AgentPave.CheckpointError)

---

Criteria ID:  D6-002
Stability:    Beta
Given:        A running agent that has completed 2 tool calls (checkpoints at seq 0, 1, 2)
              Agent raises RuntimeError during LLM reasoning after tool call 2
When:         The RuntimeError is caught by the runtime
Then:         Agent resumes from the checkpoint at sequence_number=2
              Both prior tool calls are replayed from stored results
              The RuntimeError point is NOT re-executed
              Agent continues from the step after sequence_number=2
Pass:         Agent resumes correctly, tool calls 1 and 2 NOT re-invoked via MCP,
              replayed=True on both ToolCallRecords after resume
Fail:         Agent restarts from task beginning, tool calls re-executed, or agent fails permanently
Error raised: None (RuntimeError is caught and handled by durable execution)

---

Criteria ID:  D6-003
Stability:    Beta
Given:        Agent with DeterministicTaskDefinition:
              input_pattern=r"convert \d+ [A-Z]{3} to [A-Z]{3}",
              handler_function="myagent.handlers.convert_currency"
When:         task_router.route("convert 100 USD to EUR at rate 0.92")
Then:         TaskRouteResult.route_type == DETERMINISTIC
              TaskRouteResult.handler_function == "myagent.handlers.convert_currency"
              execute_deterministic() is called — NOT the LLM
Pass:         Route result is DETERMINISTIC, LLM call log shows zero calls
Fail:         Task routed to LLM, or route_type == LLM
Error raised: None

---

Criteria ID:  D6-004
Stability:    Beta
Given:        Agent with ReliabilityContract.quality_threshold=0.8
              LLM output receives quality score 0.65 from the evaluator
When:         ReliabilityEnforcer.evaluate(agent_id, llm_output, context) is called
Then:         AgentPave.ReliabilityContractViolationError raised
Pass:         ReliabilityContractViolationError raised,
              context["score"]==0.65, context["threshold"]==0.8
Fail:         Output accepted despite low score, or different error
Error raised: AgentPave.ReliabilityContractViolationError

---

Criteria ID:  D6-005
Stability:    Beta
Given:        Agent with DriftConfig.embedding_drift_threshold_pct=15.0
              Baseline mean cosine distance = 0.20
              Current mean cosine distance = 0.25 (25% increase > 15% threshold)
When:         drift_detector.check_embedding_drift(agent_id, current_embeddings) is called
Then:         DriftEvent is returned with drift_type="embedding"
              AgentObserver.observe_drift_event() is called with the DriftEvent
              DriftEvent.severity determined by deviation_pct:
                 deviation_pct = (0.25 - 0.20) / 0.20 * 100 = 25%
                 → severity = MEDIUM (20-35% range)
Pass:         DriftEvent returned, observe_drift_event called, severity==MEDIUM
Fail:         No drift event despite exceeding threshold, or wrong severity
Error raised: None (drift events are surfaced, not raised)

---

Criteria ID:  D6-006
Stability:    Beta
Given:        A running agent with DeterministicTaskDefinition registered
When:         A deterministic task is classified and execute_deterministic() is called,
              but the handler function internally makes an LLM API call
Then:         AgentPave.DeterministicTaskLLMRouteError raised
Pass:         DeterministicTaskLLMRouteError raised when handler makes LLM call
Fail:         LLM call proceeds, or different error, or no error
Error raised: AgentPave.DeterministicTaskLLMRouteError

---

Criteria ID:  D6-007
Stability:    Beta
Given:        CheckpointStore configured with a failing backend (raises on save())
When:         A tool call boundary is reached and checkpoint save is attempted
Then:         AgentPave.CheckpointError raised
              Agent task is halted — execution does NOT continue
              Agent transitions to PAUSED state
Pass:         CheckpointError raised, execution halted,
              agent.lifecycle_state == PAUSED after the error
Fail:         Agent continues execution despite failed checkpoint, or no error
Error raised: AgentPave.CheckpointError
```

---

## 12. Definition of Done

- [ ] `ToolCallRecord` and `CheckpointData` Pydantic models implemented
- [ ] `DeterministicTaskDefinition` and `DeterministicTaskType` implemented
- [ ] `DriftConfig`, `DriftEvent`, `DriftSeverity` models implemented
- [ ] `TaskRouteResult`, `TaskRouteType` models implemented
- [ ] `CheckpointStore` abstract interface implemented with 4 methods
- [ ] `TaskRouter` abstract interface implemented
- [ ] `ReliabilityEnforcer` abstract interface implemented
- [ ] `DriftDetector` abstract interface implemented
- [ ] Concrete `CheckpointStore` is durable (survives process restart — tested)
- [ ] Checkpoint created before and after every tool call boundary
- [ ] Resume from checkpoint replays completed calls, re-executes pending call
- [ ] Deterministic task routing classification works for all registered patterns
- [ ] LLM calls in deterministic handlers are detected and raise error
- [ ] Quality evaluation runs on every LLM output
- [ ] Drift baseline recorded at REGISTERED transition
- [ ] Drift events emitted via AgentObserver.observe_drift_event()
- [ ] All 7 acceptance criteria pass: D6-001 through D6-007
- [ ] Checkpoint write latency p99 < 200ms; resume latency p99 < 500ms (NFR)
- [ ] INV-006 verified: every tool_call_boundary → checkpoint_exists
- [ ] INV-007 verified: deterministic tasks never routed to LLM

---

## 13. Task Breakdown

```
Task D6-T1: Implement ToolCallRecord and CheckpointData Pydantic models
Task D6-T2: Implement DeterministicTaskDefinition, DeterministicTaskType
Task D6-T3: Implement DriftConfig, DriftEvent, DriftSeverity
Task D6-T4: Implement TaskRouteResult, TaskRouteType
Task D6-T5: Implement CheckpointStore abstract interface
Task D6-T6: Implement concrete CheckpointStore (SQLite for dev, PostgreSQL for prod)
Task D6-T7: Implement checkpoint creation at tool call boundaries (wire into ToolCaller D3)
Task D6-T8: Implement resume-from-checkpoint logic (restore + replay + re-execute pending)
Task D6-T9: Implement TaskRouter abstract + concrete (regex-based classification)
Task D6-T10: Implement LLM call detection inside deterministic handlers
Task D6-T11: Implement ReliabilityEnforcer abstract + concrete (domain evaluator registry)
Task D6-T12: Implement DriftDetector abstract + concrete (cosine distance + quality score comparison)
Task D6-T13: Wire DriftDetector into task completion and semantic memory query paths
Task D6-T14: Run all 7 acceptance criteria
Task D6-T15: Verify INV-006 and INV-007
Task D6-T16: Verify checkpoint write p99 < 200ms and resume p99 < 500ms
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Re-executing completed tool calls after resume**
Re-executing a write tool call (database insert, API POST) after a resume causes duplicate side effects. Completed tool calls must be replayed from stored results — not re-executed.

**Anti-Pattern 2 — In-memory checkpoint store**
Storing checkpoints in process memory means they're lost on any process restart or crash — the most common failure scenario. Checkpoints must be durable: persisted to external storage before returning from `save()`.

**Anti-Pattern 3 — LLM-based determinism classification**
Using the LLM to decide "is this task deterministic?" creates a circular dependency and non-determinism in the classification itself. Deterministic tasks are declared upfront in the AgentDefinition — not classified at runtime by the LLM.

**Anti-Pattern 4 — Continuing execution after failed checkpoint**
An agent that fails to save a checkpoint but continues execution is running without a safety net. If it fails in the next step, there's no checkpoint to resume from. Halt and pause on checkpoint failure — always.

**Anti-Pattern 5 — Acting on drift without operator instruction**
Auto-pausing an agent when drift is detected removes operator control. Drift events are information — they should be surfaced and observed. The operator decides whether to pause, retrain, or do nothing.

**Anti-Pattern 6 — Missing grounding check**
When `reliability_contract.grounding_required = True`, checking quality score alone is insufficient. An LLM can produce a high-quality-sounding answer that is entirely fabricated. Grounding verification must confirm the output references retrieved context.

---

## 15. Reference Implementation Notes

### 15.1 AgentPave-LangGraph
- Checkpointing: LangGraph's `SqliteSaver` (dev) / `PostgresSaver` (prod) as CheckpointStore backend
- LangGraph's native `interrupt()` and checkpoint-and-resume maps directly to D6 durable execution
- TaskRouter: use LangGraph's conditional edges for routing decisions
- DriftDetector: use numpy for cosine similarity comparisons in development

### 15.2 AgentPave-MAF
- Checkpointing: Azure Cosmos DB with TTL disabled (checkpoints should not expire automatically)
- MAF's actor model handles resume natively — wire AgentPave CheckpointData into MAF's state persistence
- TaskRouter: MAF's function invocation framework maps to handler_function routing

### 15.3 Checkpoint Storage Requirements

| Requirement | Development | Production |
|---|---|---|
| Durability | Process restart survival | Multi-AZ redundancy |
| Latency | < 200ms write p99 | < 200ms write p99 |
| Read latency | < 500ms (resume) | < 500ms (resume) |
| Retention | Until task RETIRED | Until task RETIRED + 90 days |
| Encryption | Optional | Required (D7 Security, post-MVP) |

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should checkpoints support partial restore (restore working memory only, not tool call history)? | No |
| OQ-2 | Should the drift baseline be updated incrementally, or only at explicit recalibration? | Yes — affects drift accuracy over long agent lifetimes |
| OQ-3 | Should ReliabilityContract violations trigger automatic PAUSED transition or just raise? | Yes — governance decision, clarify before Stable |
| OQ-4 | Should checkpoint compression be part of the spec? (Large working memory snapshots) | No — implementation detail |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec |

---

*AgentPave Dimension 6 — Reliability — v1.1*
