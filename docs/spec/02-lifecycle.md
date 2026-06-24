# AgentPave Dimension 2 — Lifecycle

**Spec version:** 1.1  
**Stability:** Beta  
**Depends on:** D1 (Identity)  
**Required by:** D3, D4, D5, D6  
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

Lifecycle is the gatekeeper of all agent operations. An agent without a defined lifecycle model is a process — unpredictable, ungovernable, and unauditable. The Lifecycle dimension defines the six states an agent can occupy, the valid transitions between them, and the hooks invoked at each transition.

Every other MVP dimension depends on lifecycle state:
- Communication (D3) only proceeds when agent is RUNNING
- Memory (D4) reads and writes are scoped to lifecycle context
- Observability (D5) emits state transitions as first-class events
- Reliability (D6) checkpointing is tied to tool call boundaries within RUNNING state

### 1.2 Production Consequence of Getting This Wrong

Production agents fail at lifecycle boundaries more than anywhere else. A workflow pauses at a decision point, saves the state it needs to resume, and waits for human approval — what makes this work is durable state persistence tied to lifecycle transitions.

If Lifecycle is implemented incorrectly:
- Agents that never reach RETIRED accumulate as ghost agents with lingering system access and tool permissions
- Agents that can restart from RETIRED bypass access revocation — a critical security vulnerability
- Missing lifecycle hooks mean state is not saved at pause — the agent cannot resume correctly
- Invalid state transitions (e.g. DECLARED → RUNNING, bypassing REGISTERED) skip the registry check — an unregistered agent runs with no identity validation

---

## 2. Scope Boundary

### 2.1 In Scope

- Defining and enforcing the six lifecycle states
- Defining and enforcing all valid and invalid state transitions
- Invoking lifecycle hooks on every state transition
- Recording every state transition in the immutable audit log
- Defining the lifecycle transition table
- Governing the RETIRED terminal state
- Coordinated version rollout — enforcing that dependent agents version-sync before transition

### 2.2 Out of Scope

- What the agent does while RUNNING — that is D3, D4, D5, D6
- Checkpoint creation within RUNNING state — that is D6 (Reliability)
- Human approval for state transitions — that is AgentPave-HITL extension
- Token issuance per spawn — that is D1 (Identity) `IdentityService`
- Agent retirement criteria and policy — that is D12 (Governance)

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **Six states, not fewer** | DECLARED and REGISTERED are distinct: an agent can be declared (ID assigned) but fail registry validation checks before becoming REGISTERED. Collapsing them hides validation failures. |
| **RESUMED is a transient state** | RESUMED is not a stable resting state — it exists to signal that a PAUSED agent has been explicitly asked to resume, and immediately transitions to RUNNING. This enables hooks (on_resume) to run before execution restarts. |
| **RETIRED is terminal — no transitions out** | Retiring agents properly prevents security vulnerabilities and reduces operational complexity. Allowing un-retirement creates ghost agents and re-opens revoked permissions. |
| **Hooks are synchronous** | Asynchronous hooks make state transition atomicity impossible. If a hook fails asynchronously after state has been committed, the system is in an inconsistent state. Synchronous hooks fail fast before state is committed. |
| **Audit log is immutable and append-only** | Every state transition must be traceable for compliance and debugging. Mutable audit logs are not audit logs. |
| **State is stored in AgentRegistry** | Lifecycle state is a property of the agent's identity record — not a separate service. This ensures state queries are consistent with identity queries. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **Lifecycle State** | One of six mutually exclusive states: DECLARED, REGISTERED, RUNNING, PAUSED, RESUMED, RETIRED. An agent occupies exactly one state at any point in time. |
| **State Transition** | A governed change from one LifecycleState to another. Only transitions listed in the Transition Table are valid. |
| **Lifecycle Hook** | A named callback function invoked synchronously by the runtime on every state transition. Hooks run before the new state is committed. If a hook raises an exception, the state transition is aborted. |
| **Transition Trigger** | The external event that causes a state transition: `register`, `invoke`, `pause`, `resume`, `retire`. |
| **Transition Record** | An immutable entry in the audit log recording: agent_id, from_state, to_state, trigger, reason, actor, timestamp. |
| **Ghost Agent** | An agent that has been abandoned in RUNNING or PAUSED state without being properly RETIRED. Ghost agents hold lingering permissions and consume resources silently. |
| **Actor** | The entity initiating a state transition: a user (identified by owner email or user ID) or the system itself (for automatic transitions like RESUMED → RUNNING). |
| **Retirement Reason** | A mandatory string explaining why an agent is being retired. Required for governance. Examples: "business-unit-shutdown", "replaced-by-v2", "security-policy-violation". |

---

## 5. Data Models

### 5.1 Lifecycle Transition Table

This is the authoritative, exhaustive list of valid transitions. Any transition not in this table is INVALID and MUST raise `AgentPave.InvalidStateTransitionError`.

```python
from typing import FrozenSet, Tuple

# Type: (from_state, to_state) — both as LifecycleState string values
VALID_TRANSITIONS: FrozenSet[Tuple[str, str]] = frozenset({
    # Normal flow
    ("DECLARED",    "REGISTERED"),  # Trigger: register
    ("REGISTERED",  "RUNNING"),     # Trigger: invoke
    ("RUNNING",     "PAUSED"),      # Trigger: pause (explicit or HITL)
    ("RUNNING",     "RETIRED"),     # Trigger: retire (completion or forced)
    ("PAUSED",      "RESUMED"),     # Trigger: resume
    ("PAUSED",      "RETIRED"),     # Trigger: retire (forced from paused state)
    ("RESUMED",     "RUNNING"),     # Trigger: automatic (RESUMED is transient)
})

# These are examples of INVALID transitions for documentation clarity.
# Any transition not in VALID_TRANSITIONS is invalid.
INVALID_TRANSITION_EXAMPLES: FrozenSet[Tuple[str, str]] = frozenset({
    ("DECLARED",    "RUNNING"),     # Must pass through REGISTERED
    ("DECLARED",    "RETIRED"),     # Must register before retiring
    ("REGISTERED",  "PAUSED"),      # Cannot pause before running
    ("REGISTERED",  "RESUMED"),     # Cannot resume before running
    ("RETIRED",     "DECLARED"),    # RETIRED is terminal
    ("RETIRED",     "REGISTERED"),  # RETIRED is terminal
    ("RETIRED",     "RUNNING"),     # RETIRED is terminal
    ("RETIRED",     "PAUSED"),      # RETIRED is terminal
    ("RETIRED",     "RESUMED"),     # RETIRED is terminal
    ("RUNNING",     "DECLARED"),    # Cannot un-register a running agent
    ("RUNNING",     "REGISTERED"),  # Cannot go backwards
})
```

### 5.2 LifecycleHookName

```python
from enum import Enum

class LifecycleHookName(str, Enum):
    ON_REGISTER  = "on_register"   # DECLARED → REGISTERED
    ON_START     = "on_start"      # REGISTERED → RUNNING
    ON_PAUSE     = "on_pause"      # RUNNING → PAUSED
    ON_RESUME    = "on_resume"     # PAUSED → RESUMED
    ON_RUNNING   = "on_running"    # RESUMED → RUNNING (automatic)
    ON_RETIRE    = "on_retire"     # RUNNING|PAUSED → RETIRED
    ON_ERROR     = "on_error"      # Any state: unhandled exception during transition
```

### 5.3 TransitionRecord

```python
from pydantic import BaseModel, Field
from typing import Optional


class TransitionRecord(BaseModel):
    """
    An immutable record of a single lifecycle state transition.
    Written to the immutable audit log at the moment the transition completes.
    Never updated or deleted.
    """
    record_id: str = Field(
        ...,
        description="UUID v4. Unique identifier for this transition record."
    )
    agent_id: str = Field(
        ...,
        description="UUID v4. The agent that transitioned."
    )
    from_state: str = Field(
        ...,
        description="The LifecycleState the agent was in before this transition."
    )
    to_state: str = Field(
        ...,
        description="The LifecycleState the agent is in after this transition."
    )
    trigger: str = Field(
        ...,
        description=(
            "The event that caused this transition. "
            "One of: register, invoke, pause, resume, retire, automatic."
        )
    )
    reason: str = Field(
        ...,
        min_length=1,
        max_length=500,
        description=(
            "Human-readable reason for this transition. "
            "Required for RETIRE transitions. Optional but recommended for others."
        )
    )
    actor: str = Field(
        ...,
        description=(
            "Entity initiating this transition. "
            "Format: user email, team path, or 'system' for automatic transitions."
        )
    )
    timestamp: str = Field(
        ...,
        description="ISO 8601 UTC timestamp of the moment the transition completed."
    )
    hook_results: dict = Field(
        default_factory=dict,
        description=(
            "Results from lifecycle hooks invoked during this transition. "
            "Key: hook name. Value: 'success' or error message string."
        )
    )
    task_id: Optional[str] = Field(
        None,
        description=(
            "UUID v4. The task_id associated with this transition, if applicable. "
            "Present for invoke (REGISTERED→RUNNING) and resume (PAUSED→RESUMED) transitions."
        )
    )

    model_config = {"extra": "forbid"}
```

### 5.4 LifecycleHookContext

```python
class LifecycleHookContext(BaseModel):
    """
    Context passed to every lifecycle hook invocation.
    Hooks receive this object and MAY use it to perform setup/teardown.
    """
    agent_id: str = Field(..., description="UUID v4 of the transitioning agent.")
    agent_name: str = Field(..., description="Human-readable name from AgentDefinition.")
    from_state: str = Field(..., description="State before transition.")
    to_state: str = Field(..., description="State after transition.")
    trigger: str = Field(..., description="Event that caused this transition.")
    task_id: Optional[str] = Field(None, description="Task ID if transition is task-scoped.")
    reason: str = Field(..., description="Reason string for this transition.")
    actor: str = Field(..., description="Entity initiating this transition.")
    timestamp: str = Field(..., description="ISO 8601 UTC timestamp of transition.")
    metadata: dict = Field(
        default_factory=dict,
        description="Optional additional context. Runtime-specific. Not part of spec contract."
    )

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Valid Lifecycle Sequence (Normal Flow)

```python
# Step 1: Agent declared (Identity D1)
record = registry.declare(definition)
assert record.lifecycle_state == "DECLARED"

# Step 2: Agent registered
record = lifecycle_manager.register(agent_id=record.agent_id, actor="platform@example.com")
assert record.lifecycle_state == "REGISTERED"

# Step 3: Agent invoked (task starts)
record = lifecycle_manager.invoke(
    agent_id=record.agent_id,
    task_id=new_uuid4(),
    actor="user@example.com"
)
assert record.lifecycle_state == "RUNNING"

# Step 4: Agent paused (e.g. awaiting HITL approval)
record = lifecycle_manager.pause(
    agent_id=record.agent_id,
    reason="Awaiting human approval for high-risk action",
    actor="system"
)
assert record.lifecycle_state == "PAUSED"

# Step 5: Agent resumed
record = lifecycle_manager.resume(
    agent_id=record.agent_id,
    task_id=existing_task_id,
    actor="approver@example.com"
)
# RESUMED is transient — immediately becomes RUNNING
assert record.lifecycle_state == "RUNNING"

# Step 6: Agent retired on task completion
record = lifecycle_manager.retire(
    agent_id=record.agent_id,
    reason="Task completed successfully",
    actor="system"
)
assert record.lifecycle_state == "RETIRED"

# Step 7: Retired agent cannot be invoked
try:
    lifecycle_manager.invoke(agent_id=record.agent_id, task_id=new_uuid4(), actor="user@example.com")
    assert False, "Should have raised RetiredAgentError"
except AgentPave.RetiredAgentError:
    pass  # Expected
```

### 6.2 Invalid Transition Example

```python
# Attempting DECLARED → RUNNING (skipping REGISTERED)
try:
    lifecycle_manager.invoke(
        agent_id=declared_agent_id,  # agent is in DECLARED state
        task_id=new_uuid4(),
        actor="user@example.com"
    )
    assert False, "Should have raised InvalidStateTransitionError"
except AgentPave.InvalidStateTransitionError as e:
    assert e.context["from_state"] == "DECLARED"
    assert e.context["to_state"] == "RUNNING"
```

---

## 7. Interfaces & Contracts

### 7.1 LifecycleManager

```python
from abc import ABC, abstractmethod
from typing import Callable


# Hook signature type
LifecycleHookFn = Callable[[LifecycleHookContext], None]


class LifecycleManager(ABC):
    """
    Governs all lifecycle state transitions for AgentPave agents.

    Contract rules:
    - All state transitions are atomic: the state is only committed if the
      transition is valid AND all hooks complete without raising.
    - Hooks are invoked synchronously, before the new state is committed.
      If any hook raises, the transition is aborted and the original state is preserved.
    - Every completed transition is written to the immutable audit log.
    - RETIRED is terminal: any operation on a RETIRED agent raises RetiredAgentError.
    - RESUMED is transient: invoke resumed_to_running() immediately after resume().
    """

    @abstractmethod
    def register(self, agent_id: str, actor: str) -> "AgentRecord":
        """
        Transition agent from DECLARED to REGISTERED.
        Verifies the agent exists in the registry with state DECLARED.

        Steps:
        1. Validate agent exists and is in DECLARED state.
        2. Invoke on_register hook.
        3. If hook succeeds: commit REGISTERED state, write TransitionRecord.
        4. If hook fails: abort transition, raise LifecycleHookError.

        Args:
            agent_id: UUID v4.
            actor: Entity initiating registration (email or team path).

        Returns:
            AgentRecord with lifecycle_state == REGISTERED.

        Raises:
            AgentPave.AgentNotFoundError: agent_id not in registry.
            AgentPave.InvalidStateTransitionError: agent not in DECLARED state.
            AgentPave.LifecycleHookError: on_register hook raised an exception.
        """
        ...

    @abstractmethod
    def invoke(self, agent_id: str, task_id: str, actor: str) -> "AgentRecord":
        """
        Transition agent from REGISTERED to RUNNING.
        Issues a Runtime Token (via IdentityService) before transition completes.

        Steps:
        1. Validate agent exists and is in REGISTERED state.
        2. Issue Runtime Token via IdentityService.issue_token().
        3. Invoke on_start hook with task_id in context.
        4. If hook succeeds: commit RUNNING state, write TransitionRecord.
        5. If hook fails: revoke Runtime Token, abort transition, raise LifecycleHookError.

        Args:
            agent_id: UUID v4.
            task_id: UUID v4. The task being started.
            actor: Entity initiating invocation (email or team path).

        Returns:
            AgentRecord with lifecycle_state == RUNNING.

        Raises:
            AgentPave.AgentNotFoundError: agent_id not in registry.
            AgentPave.InvalidStateTransitionError: agent not in REGISTERED state.
            AgentPave.LifecycleHookError: on_start hook raised an exception.
        """
        ...

    @abstractmethod
    def pause(self, agent_id: str, reason: str, actor: str) -> "AgentRecord":
        """
        Transition agent from RUNNING to PAUSED.
        The agent's current execution state MUST be checkpointed by D6
        before pause() is called. This method assumes the checkpoint exists.

        Steps:
        1. Validate agent exists and is in RUNNING state.
        2. Invoke on_pause hook.
        3. If hook succeeds: commit PAUSED state, write TransitionRecord.
        4. If hook fails: abort transition, raise LifecycleHookError.

        Args:
            agent_id: UUID v4.
            reason: Mandatory reason for pausing (e.g. "Awaiting HITL approval").
            actor: Entity initiating pause.

        Returns:
            AgentRecord with lifecycle_state == PAUSED.

        Raises:
            AgentPave.AgentNotFoundError: agent_id not in registry.
            AgentPave.InvalidStateTransitionError: agent not in RUNNING state.
            AgentPave.LifecycleHookError: on_pause hook raised an exception.
        """
        ...

    @abstractmethod
    def resume(self, agent_id: str, task_id: str, reason: str, actor: str) -> "AgentRecord":
        """
        Transition agent from PAUSED to RESUMED, then immediately to RUNNING.
        RESUMED is transient — this method performs both transitions atomically.

        Steps:
        1. Validate agent exists and is in PAUSED state.
        2. Invoke on_resume hook.
        3. If hook succeeds: commit RESUMED state (transient).
        4. Invoke on_running hook.
        5. If hook succeeds: commit RUNNING state, write single TransitionRecord
           covering PAUSED → RUNNING with RESUMED as intermediate.
        6. If any hook fails: abort both transitions, restore PAUSED state.

        Args:
            agent_id: UUID v4.
            task_id: UUID v4. The task being resumed.
            reason: Reason for resuming (e.g. "HITL approval received").
            actor: Entity initiating resume.

        Returns:
            AgentRecord with lifecycle_state == RUNNING.

        Raises:
            AgentPave.AgentNotFoundError: agent_id not in registry.
            AgentPave.InvalidStateTransitionError: agent not in PAUSED state.
            AgentPave.LifecycleHookError: on_resume or on_running hook raised.
        """
        ...

    @abstractmethod
    def retire(self, agent_id: str, reason: str, actor: str) -> "AgentRecord":
        """
        Transition agent from RUNNING or PAUSED to RETIRED.
        RETIRED is terminal — no further transitions are possible after this.

        Steps:
        1. Validate agent exists and is in RUNNING or PAUSED state.
        2. Invoke on_retire hook with reason in context.
        3. If hook succeeds: commit RETIRED state, write TransitionRecord,
           revoke all active Runtime Tokens for this agent.
        4. If hook fails: abort transition, raise LifecycleHookError.
           Agent remains in its current state (RUNNING or PAUSED).

        Args:
            agent_id: UUID v4.
            reason: Mandatory retirement reason (min 3 words).
            actor: Entity initiating retirement.

        Returns:
            AgentRecord with lifecycle_state == RETIRED.

        Raises:
            AgentPave.AgentNotFoundError: agent_id not in registry.
            AgentPave.RetiredAgentError: agent is already RETIRED.
            AgentPave.InvalidStateTransitionError: agent in DECLARED or REGISTERED state.
            AgentPave.LifecycleHookError: on_retire hook raised an exception.
            ValueError: reason contains fewer than 3 words.
        """
        ...

    @abstractmethod
    def get_transitions(self, agent_id: str) -> list[TransitionRecord]:
        """
        Retrieve the complete, ordered transition history for an agent.
        Ordered by timestamp ascending (oldest first).

        Returns:
            list[TransitionRecord]: All transitions for this agent.
                Empty list if agent exists but has no transitions yet.

        Raises:
            AgentPave.AgentNotFoundError: agent_id not in registry.
        """
        ...

    @abstractmethod
    def register_hook(
        self,
        hook_name: LifecycleHookName,
        hook_fn: LifecycleHookFn
    ) -> None:
        """
        Register a lifecycle hook function to be invoked on a specific transition.
        Multiple hooks may be registered for the same hook_name.
        Hooks are invoked in registration order.

        Args:
            hook_name: Which lifecycle event this hook fires on.
            hook_fn: Callable accepting LifecycleHookContext. Must not return a value.
                     Raise any exception to abort the transition.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 State Transition Enforcement

**THE SYSTEM SHALL** validate every state transition against the VALID_TRANSITIONS table before executing it.

**WHEN** a transition is not in VALID_TRANSITIONS **THE SYSTEM SHALL** raise `AgentPave.InvalidStateTransitionError` with context `{"from_state": str, "to_state": str, "agent_id": str}` and abort the transition without modifying the agent's state.

**THE SYSTEM SHALL** make state transitions atomic: either the full transition (hook invocation + state commit + audit log write) completes, or nothing changes.

### 8.2 Lifecycle Hooks

**THE SYSTEM SHALL** invoke lifecycle hooks synchronously before committing the new state.

**WHEN** a lifecycle hook raises any exception **THE SYSTEM SHALL** abort the state transition, preserve the agent's current state, and raise `AgentPave.LifecycleHookError` with the original exception captured in `context["cause"]`.

**THE SYSTEM SHALL** invoke hooks in registration order when multiple hooks are registered for the same event.

**THE SYSTEM SHALL** pass a `LifecycleHookContext` to every hook invocation.

### 8.3 RETIRED Terminal State

**WHEN** an agent enters RETIRED state **THE SYSTEM SHALL** prevent all further state transitions — any attempt raises `AgentPave.RetiredAgentError`.

**WHEN** an agent is retired **THE SYSTEM SHALL** revoke all active Runtime Tokens for that agent via `IdentityService`.

**THE SYSTEM SHALL NOT** allow RETIRED agents to invoke tools, read or write memory, or receive task invocations.

### 8.4 RESUMED Transitivity

**WHEN** an agent transitions to RESUMED **THE SYSTEM SHALL** immediately and automatically transition it to RUNNING.

**THE SYSTEM SHALL** record PAUSED → RUNNING in the audit log with `trigger="resume"` and an intermediate note indicating RESUMED state was passed through.

**WHILE** an agent is in RESUMED state **THE SYSTEM SHALL** reject all external state transition requests — RESUMED → RUNNING is exclusively an internal automatic transition.

### 8.5 Audit Log

**THE SYSTEM SHALL** write a TransitionRecord to the immutable audit log for every completed state transition.

**THE SYSTEM SHALL NOT** modify or delete any TransitionRecord after it is written.

**THE SYSTEM SHALL** record `actor` in every TransitionRecord — anonymous transitions are not permitted.

### 8.6 Retirement Validation

**THE SYSTEM SHALL** reject retirement requests where `reason` contains fewer than 3 words, raising `ValueError`.

**WHEN** retiring an agent with active RUNNING tasks (ASK FIRST tier) **THE SYSTEM SHALL** require explicit `force=True` flag, otherwise raise `ValueError` with message `"Agent has active tasks. Pass force=True to retire forcefully."`.

---

## 9. Three-Tier Boundary System

### ALWAYS

- Validate every transition against VALID_TRANSITIONS before executing
- Invoke the correct lifecycle hook synchronously before committing state
- Write a TransitionRecord to the immutable audit log for every completed transition
- Make transitions atomic — all-or-nothing
- Prevent any transition out of RETIRED state
- Revoke all active Runtime Tokens when agent is RETIRED
- Record a non-anonymous actor on every transition

### ASK FIRST

- Retiring an agent that has active RUNNING tasks — require explicit `force=True` flag
- Registering an agent that will replace an existing RUNNING agent of the same name+domain — confirm intention to avoid accidental duplicate registration
- Pausing an agent in the middle of a multi-step atomic tool sequence — confirm the pause can be safely applied at the current step

### NEVER

- Allow any transition out of RETIRED state
- Commit a state change without writing a TransitionRecord
- Allow a hook failure to silently pass — always raise LifecycleHookError
- Allow DECLARED → RUNNING (bypassing REGISTERED)
- Allow anonymous transitions (actor must always be set)
- Allow retirement with an empty or one-word reason

---

## 10. Error Handling

| Scenario | Error Type | Recoverable | Required context |
|---|---|---|---|
| Transition not in VALID_TRANSITIONS | `AgentPave.InvalidStateTransitionError` | No | `agent_id, from_state, to_state` |
| Any operation on RETIRED agent | `AgentPave.RetiredAgentError` | No | `agent_id, retired_at` |
| Lifecycle hook raises exception | `AgentPave.LifecycleHookError` | Yes (retry) | `agent_id, hook_name, cause: str` |
| agent_id not found | `AgentPave.AgentNotFoundError` | No | `agent_id` |
| retire() reason < 3 words | `ValueError` | No | — |
| retire() with active tasks, no force=True | `ValueError` | No | — |

---

## 11. Acceptance Criteria

```
Criteria ID:  D2-001
Stability:    Beta
Given:        A declared agent (lifecycle_state == DECLARED)
When:         lifecycle_manager.register(agent_id, actor="platform@example.com") is called
Then:         AgentRecord is returned with lifecycle_state == REGISTERED
              TransitionRecord exists with from_state=DECLARED, to_state=REGISTERED,
              trigger="register", actor="platform@example.com"
              on_register hook was invoked exactly once
Pass:         All conditions above true
Fail:         Any condition false
Error raised: None

---

Criteria ID:  D2-002
Stability:    Beta
Given:        A registered agent (lifecycle_state == REGISTERED)
When:         lifecycle_manager.invoke(agent_id, task_id=<new_uuid4>, actor="user@example.com")
Then:         AgentRecord returned with lifecycle_state == RUNNING
              on_start hook invoked with context.task_id == task_id
              TransitionRecord written with trigger="invoke"
Pass:         All conditions true
Fail:         Any condition false
Error raised: None

---

Criteria ID:  D2-003
Stability:    Beta
Given:        A declared agent (lifecycle_state == DECLARED)
When:         lifecycle_manager.invoke(agent_id, task_id=<uuid>, actor="user@example.com")
Then:         AgentPave.InvalidStateTransitionError is raised
              Agent remains in DECLARED state
Pass:         Error raised, context["from_state"]=="DECLARED", context["to_state"]=="RUNNING",
              agent.lifecycle_state == "DECLARED" (unchanged)
Fail:         Agent transitions to RUNNING, or different error raised
Error raised: AgentPave.InvalidStateTransitionError

---

Criteria ID:  D2-004
Stability:    Beta
Given:        A running agent (lifecycle_state == RUNNING)
When:         lifecycle_manager.pause(agent_id, reason="Awaiting HITL approval", actor="system")
Then:         AgentRecord returned with lifecycle_state == PAUSED
              TransitionRecord written with reason="Awaiting HITL approval"
              on_pause hook invoked exactly once
Pass:         All conditions true
Fail:         Any condition false
Error raised: None

---

Criteria ID:  D2-005
Stability:    Beta
Given:        A paused agent (lifecycle_state == PAUSED)
When:         lifecycle_manager.resume(agent_id, task_id=<existing_uuid>, reason="Approval received", actor="approver@example.com")
Then:         AgentRecord returned with lifecycle_state == RUNNING
              on_resume hook invoked, then on_running hook invoked
              TransitionRecord records PAUSED → RUNNING (RESUMED as intermediate)
Pass:         lifecycle_state == RUNNING, both hooks invoked, TransitionRecord present
Fail:         lifecycle_state == RESUMED (RESUMED must not be stable), or hooks not invoked
Error raised: None

---

Criteria ID:  D2-006
Stability:    Beta
Given:        A running agent (lifecycle_state == RUNNING)
When:         lifecycle_manager.retire(agent_id, reason="Task completed successfully", actor="system")
Then:         AgentRecord returned with lifecycle_state == RETIRED
              on_retire hook invoked
              All active Runtime Tokens for agent are revoked
              TransitionRecord written
Pass:         lifecycle_state == RETIRED, on_retire invoked,
              IdentityService.validate_token() returns False for any previously issued token
Fail:         Any condition false
Error raised: None

---

Criteria ID:  D2-007
Stability:    Beta
Given:        A retired agent (lifecycle_state == RETIRED)
When:         lifecycle_manager.invoke(agent_id, task_id=<uuid>, actor="user@example.com")
Then:         AgentPave.RetiredAgentError is raised
              Agent remains in RETIRED state
Pass:         RetiredAgentError raised, context["agent_id"] matches, lifecycle_state unchanged
Fail:         Agent transitions out of RETIRED, or different error raised
Error raised: AgentPave.RetiredAgentError

---

Criteria ID:  D2-008
Stability:    Beta
Given:        A running agent with an on_start hook that raises ValueError("hook failure")
When:         lifecycle_manager.invoke(agent_id, task_id=<uuid>, actor="user@example.com")
Then:         AgentPave.LifecycleHookError is raised
              Agent remains in REGISTERED state (transition aborted)
              No TransitionRecord written for the failed transition
Pass:         LifecycleHookError raised, context["hook_name"]=="on_start",
              context["cause"] contains "hook failure",
              agent.lifecycle_state == "REGISTERED"
Fail:         Agent transitions to RUNNING despite hook failure, or state changes
Error raised: AgentPave.LifecycleHookError

---

Criteria ID:  D2-009
Stability:    Beta
Given:        A running agent
When:         lifecycle_manager.retire(agent_id, reason="ok", actor="system")
              (reason has only 1 word)
Then:         ValueError is raised. Agent remains RUNNING.
Pass:         ValueError raised with message mentioning "3 words", agent state unchanged
Fail:         Agent retired with short reason, or different error
Error raised: ValueError

---

Criteria ID:  D2-010
Stability:    Beta
Given:        A registered agent
When:         lifecycle_manager.get_transitions(agent_id) is called after
              register() and invoke() have been completed
Then:         A list of 2 TransitionRecords is returned, ordered oldest-first:
              [DECLARED→REGISTERED, REGISTERED→RUNNING]
Pass:         len(records)==2, records[0].to_state=="REGISTERED",
              records[1].to_state=="RUNNING", records[0].timestamp <= records[1].timestamp
Fail:         Wrong count, wrong order, wrong states
Error raised: None
```

---

## 12. Definition of Done

- [ ] `LifecycleHookName` enum implemented with all 7 hook names
- [ ] `TransitionRecord` Pydantic model implemented with all fields
- [ ] `LifecycleHookContext` Pydantic model implemented with all fields
- [ ] `VALID_TRANSITIONS` frozenset defined and used in all transition methods
- [ ] `LifecycleManager` abstract interface implemented with all 6 methods
- [ ] Concrete `LifecycleManager` implementation enforces VALID_TRANSITIONS
- [ ] Lifecycle hooks are synchronous and abort transition on exception
- [ ] RETIRED is enforced as terminal — no transitions out possible
- [ ] RESUMED is transient — immediately becomes RUNNING
- [ ] TransitionRecord written to immutable audit log on every completed transition
- [ ] Runtime Tokens revoked on RETIRE
- [ ] All 10 acceptance criteria pass: D2-001 through D2-010
- [ ] Spec invariants verified: INV-002, INV-003
- [ ] State transition latency p99 < 50ms (NFR SPEC.md Section 8.1)

---

## 13. Task Breakdown

```
Task D2-T1: Implement LifecycleHookName enum (7 values)
Task D2-T2: Implement TransitionRecord Pydantic model
Task D2-T3: Implement LifecycleHookContext Pydantic model
Task D2-T4: Define VALID_TRANSITIONS frozenset
Task D2-T5: Implement LifecycleManager abstract interface
Task D2-T6: Implement concrete LifecycleManager with transition enforcement
  - validate_transition() helper using VALID_TRANSITIONS
  - atomic transition: hook → state commit → audit log write
  - all 5 transition methods: register, invoke, pause, resume, retire
Task D2-T7: Implement hook registry (register_hook, invoke hooks in order)
Task D2-T8: Implement TransitionRecord audit log (append-only durable store)
Task D2-T9: Implement RESUMED transitivity (auto-advance to RUNNING)
Task D2-T10: Implement Runtime Token revocation on RETIRE
Task D2-T11: Run all 10 acceptance criteria, record in conformance-results.json
Task D2-T12: Verify INV-002 and INV-003 invariants
Task D2-T13: Verify state transition latency p99 < 50ms
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Asynchronous lifecycle hooks**
Async hooks make transition atomicity impossible. If a hook fires after state is committed and then fails, the system is in an inconsistent state with no rollback path.

**Anti-Pattern 2 — Allowing RETIRED → any state**
Un-retirement reopens revoked access, permissions, and tool connections — a critical security vulnerability. Ghost agents with lingering system access are the second-most common agent security failure mode.

**Anti-Pattern 3 — Skipping REGISTERED**
DECLARED → RUNNING bypasses the registry validation step. An agent that runs without being registered has no validated identity, no audit trail entry for the registration step, and no scope validation.

**Anti-Pattern 4 — Silent hook failures**
Catching hook exceptions and logging them instead of aborting the transition means hooks become decorative. A hook that fails silently is useless for enforcement.

**Anti-Pattern 5 — Anonymous transitions**
Transitions without an actor cannot be audited or attributed. In a governance or compliance investigation, "the system did it" is not an acceptable answer.

**Anti-Pattern 6 — Mutable audit log**
An audit log that can be edited is not an audit log. Using a database with UPDATE or DELETE permissions on the transitions table undermines the entire compliance and forensics use case.

---

## 15. Reference Implementation Notes

### 15.1 AgentPave-LangGraph
- LangGraph's native interrupt() mechanism maps to PAUSED state — use it for HITL pause points
- LangGraph's `SqliteSaver`/`PostgresSaver` can back the TransitionRecord audit log
- State transitions should wrap LangGraph's `update_state()` call inside the atomic transition block

### 15.2 AgentPave-MAF
- MAF's actor model maps naturally to lifecycle states — each actor has an active/inactive concept
- Use Azure Cosmos DB change feed for append-only TransitionRecord storage
- MAF's built-in OpenTelemetry emits lifecycle events automatically — wire to TransitionRecord writes

### 15.3 Known Divergences

| Behaviour | AgentPave-LangGraph | AgentPave-MAF | Impact |
|---|---|---|---|
| Pause mechanism | LangGraph interrupt() | MAF actor suspension | None — observable state identical |
| Audit log store | SqliteSaver / PostgresSaver | Cosmos DB change feed | None — record schema identical |

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should lifecycle support a SUSPENDED state (like PAUSED but system-initiated, not user-initiated)? | No |
| OQ-2 | Should TransitionRecord include a `duration_ms` field (time spent in the previous state)? | No |
| OQ-3 | Should retire() support a scheduled retirement timestamp ("retire this agent at midnight")? | No |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec |

---

*AgentPave Dimension 2 — Lifecycle — v1.1*
