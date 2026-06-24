# AgentPave Dimension 9 — Testability

**Spec version:** 1.1  
**Stability:** Alpha  
**Depends on:** D1–D6 (all MVP dimensions)  
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

AI agents fail differently from traditional software: they produce well-formed but wrong outputs, make semantically invalid decisions, and degrade silently over time. Traditional unit testing with deterministic assertions cannot catch these failures. D9 defines a testability model purpose-built for AI agents.

D9 delivers four capabilities:
1. **Unit testable components** — every AgentPave component is independently testable
2. **Mock tool interface** — full offline development without real tool calls
3. **Evaluation framework** — semantic quality evaluation beyond pass/fail
4. **CI/CD integration** — evaluation gates that block merges on quality regressions

### 1.2 Production Consequence of Getting This Wrong

Without D9:
- Quality regressions go undetected until users report them
- Developers cannot test agents locally without live API keys and external services
- CI/CD has no quality gate — agents are deployed blind
- The feedback loop from production failures to test datasets never closes

---

## 2. Scope Boundary

### 2.1 In Scope

- Mock tool server for offline development
- Agent test harness (unit and integration)
- Evaluation dataset management
- Evaluation metrics: exact match, semantic similarity, trajectory evaluation
- CI/CD integration contract (GitHub Actions, etc.)
- Production → test feedback loop (failures become regression cases)
- Evaluation result reporting schema

### 2.2 Out of Scope

- Load/performance testing — that is infrastructure concern
- Security penetration testing — that is D7
- Full end-to-end UI testing — that is application concern
- Managed evaluation platforms (Confident AI, LangSmith etc.) — these are pluggable backends

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **Evaluation fixtures are binary pass/fail** | Consistent with the rest of the spec. An evaluation criterion that returns a score is useful; a criterion that cannot tell you pass/fail is ambiguous. Both are supported — but CI gates require binary pass/fail. |
| **Mock tool server mimics MCP protocol** | The mock server must use the same MCP protocol as real tools, otherwise tests don't reflect production behaviour. |
| **Production failures auto-convert to regression cases** | The feedback loop from production trace to regression dataset is the most valuable testing investment. Teams that close this loop ship features with an order-of-magnitude fewer regressions. |
| **Trajectory evaluation is first-class** | Evaluating only the final output misses reasoning errors. Trajectory evaluation checks that the agent took the right steps in the right order — not just that the final answer was correct. |
| **CI gates block on pass-rate threshold** | A configurable pass-rate threshold (e.g., 90%) means minor regressions don't block; significant quality drops do. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **Evaluation Dataset** | A collection of (input, expected_output, metadata) fixtures used to evaluate agent quality. |
| **Evaluation Fixture** | A single test case: input to send to the agent, expected output or behaviour, and metadata tags. |
| **Evaluation Metric** | A function that computes a score [0.0, 1.0] for a single (actual_output, expected_output) pair. |
| **Evaluation Run** | The execution of the agent against all fixtures in a dataset, producing a set of EvaluationResults. |
| **Trajectory** | The sequence of steps an agent takes: tool calls, LLM calls, memory operations, and decisions. |
| **Trajectory Evaluation** | Evaluation that checks the correctness of the agent's reasoning path — not just the final output. |
| **Mock Tool Server** | An MCP-compliant server that returns pre-configured responses to tool calls without real external calls. |
| **CI Gate** | A threshold condition in CI/CD that blocks merge if evaluation pass-rate falls below the configured threshold. |
| **Regression Case** | A production failure that has been converted into an evaluation fixture to prevent recurrence. |
| **Pass-Rate Threshold** | The minimum percentage of evaluation fixtures that must pass for the CI gate to allow merge. |

---

## 5. Data Models

### 5.1 EvaluationFixture

```python
from pydantic import BaseModel, Field
from typing import Any, Optional
from enum import Enum


class EvaluationMetricType(str, Enum):
    EXACT_MATCH        = "exact_match"
    SEMANTIC_SIMILARITY = "semantic_similarity"
    TRAJECTORY         = "trajectory"
    CUSTOM             = "custom"


class EvaluationFixture(BaseModel):
    """A single test case for evaluating an agent."""
    fixture_id: str = Field(..., description="UUID v4.")
    dataset_id: str = Field(..., description="UUID v4. The dataset this fixture belongs to.")
    agent_id: str = Field(..., description="UUID v4. The agent this fixture tests.")
    input: str = Field(..., min_length=1, description="The task input to send to the agent.")
    expected_output: Optional[Any] = Field(
        None,
        description=(
            "Expected output for EXACT_MATCH and SEMANTIC_SIMILARITY metrics. "
            "None for TRAJECTORY metrics (evaluated against expected_trajectory)."
        )
    )
    expected_trajectory: Optional[list[dict]] = Field(
        None,
        description=(
            "Expected sequence of steps for TRAJECTORY evaluation. "
            "Each step: {'action_type': str, 'tool_name': Optional[str], 'order': int}"
        )
    )
    metric_type: EvaluationMetricType = Field(...)
    pass_threshold: float = Field(
        default=0.8,
        ge=0.0,
        le=1.0,
        description="Minimum score [0.0, 1.0] to consider this fixture passed."
    )
    tags: list[str] = Field(default_factory=list)
    source: str = Field(
        default="manual",
        description="Origin: 'manual', 'production_failure', 'generated'."
    )
    created_at: str = Field(...)
    metadata: dict = Field(default_factory=dict)

    model_config = {"extra": "forbid"}
```

### 5.2 EvaluationResult

```python
class EvaluationResult(BaseModel):
    """Result of evaluating a single fixture."""
    result_id: str = Field(..., description="UUID v4.")
    fixture_id: str = Field(...)
    run_id: str = Field(..., description="UUID v4. The evaluation run this result belongs to.")
    agent_id: str = Field(...)
    actual_output: Any = Field(..., description="What the agent actually produced.")
    actual_trajectory: Optional[list[dict]] = Field(None)
    score: float = Field(..., ge=0.0, le=1.0)
    passed: bool = Field(..., description="score >= fixture.pass_threshold.")
    metric_type: EvaluationMetricType = Field(...)
    failure_reason: Optional[str] = Field(None, description="Why it failed, if passed==False.")
    duration_ms: int = Field(..., ge=0)
    evaluated_at: str = Field(...)
    mock_tool_calls_used: int = Field(default=0, ge=0)

    model_config = {"extra": "forbid"}
```

### 5.3 EvaluationRun

```python
class EvaluationRun(BaseModel):
    """Summary of a complete evaluation run."""
    run_id: str = Field(..., description="UUID v4.")
    dataset_id: str = Field(...)
    agent_id: str = Field(...)
    agent_version: str = Field(...)
    total_fixtures: int = Field(..., ge=0)
    passed: int = Field(..., ge=0)
    failed: int = Field(..., ge=0)
    pass_rate: float = Field(..., ge=0.0, le=1.0, description="passed / total_fixtures.")
    ci_gate_passed: bool = Field(
        ...,
        description="True if pass_rate >= configured pass_rate_threshold."
    )
    pass_rate_threshold: float = Field(..., ge=0.0, le=1.0)
    results: list[EvaluationResult] = Field(default_factory=list)
    started_at: str = Field(...)
    completed_at: str = Field(...)

    model_config = {"extra": "forbid"}
```

### 5.4 MockToolResponse

```python
class MockToolResponse(BaseModel):
    """Pre-configured response for a mock tool call."""
    tool_name: str = Field(...)
    arguments_match: Optional[dict] = Field(
        None,
        description=(
            "If set: only return this response when tool call arguments match this dict. "
            "If None: return this response for any arguments to this tool."
        )
    )
    response: Any = Field(..., description="The response to return. Must be JSON-serialisable.")
    should_fail: bool = Field(
        default=False,
        description="If True: simulate tool failure (raise ToolUnavailableError)."
    )
    latency_ms: int = Field(
        default=0,
        ge=0,
        description="Simulated latency in milliseconds."
    )

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Mock Tool Configuration

```python
# Configure mock tool server for offline testing
mock_responses = [
    MockToolResponse(
        tool_name="web_search",
        arguments_match={"query": "AgentPave framework"},
        response={"results": ["AgentPave is a framework for building AI agents"]},
        latency_ms=50
    ),
    MockToolResponse(
        tool_name="web_search",
        arguments_match=None,   # Any other query
        response={"results": []},
    ),
    MockToolResponse(
        tool_name="database_lookup",
        should_fail=True,   # Simulate tool failure
    )
]
```

### 6.2 Evaluation Fixture

```python
fixture = EvaluationFixture(
    fixture_id=str(uuid4()),
    dataset_id="<dataset_uuid>",
    agent_id="<agent_uuid>",
    input="Find information about AgentPave framework",
    expected_output="AgentPave is a framework for building AI agents",
    metric_type=EvaluationMetricType.SEMANTIC_SIMILARITY,
    pass_threshold=0.85,
    tags=["search", "happy-path"],
    source="manual",
    created_at="2026-06-09T12:00:00Z"
)
```

---

## 7. Interfaces & Contracts

### 7.1 MockToolServer

```python
from abc import ABC, abstractmethod


class MockToolServer(ABC):
    """
    MCP-compliant mock server for offline agent testing.
    Implements the same tools/list and tools/call interface as real MCP servers.
    """

    @abstractmethod
    def configure(self, responses: list[MockToolResponse]) -> None:
        """Configure mock responses. Called before running tests."""
        ...

    @abstractmethod
    def start(self) -> str:
        """
        Start the mock server. Returns the server URL for MCP client connection.
        The URL can be used as the mcp_server_url in ToolRegistry.register_tool().
        """
        ...

    @abstractmethod
    def stop(self) -> None:
        """Stop the mock server and clean up resources."""
        ...

    @abstractmethod
    def get_call_log(self) -> list[dict]:
        """
        Return the log of all tool calls made to this server during the test.
        Each entry: {'tool_name': str, 'arguments': dict, 'timestamp': str}
        """
        ...

    @abstractmethod
    def reset(self) -> None:
        """Reset call log and response queue. Called between test cases."""
        ...
```

### 7.2 EvaluationRunner

```python
class EvaluationRunner(ABC):
    """
    Runs evaluation fixtures against an agent and produces EvaluationResults.
    Attaches at extension point: evaluation.runner
    """

    @abstractmethod
    def run(
        self,
        agent_id: str,
        dataset_id: str,
        pass_rate_threshold: float = 0.9,
        use_mock_tools: bool = True
    ) -> EvaluationRun:
        """
        Run all fixtures in a dataset against the agent.

        Args:
            pass_rate_threshold: Minimum pass rate for ci_gate_passed=True.
            use_mock_tools: If True, use MockToolServer instead of real tools.

        Returns:
            EvaluationRun with all results and pass/fail summary.
        """
        ...

    @abstractmethod
    def run_single(
        self,
        fixture: EvaluationFixture,
        use_mock_tools: bool = True
    ) -> EvaluationResult:
        """Run a single fixture. Used for interactive debugging."""
        ...


class EvaluationMetric(ABC):
    """Base class for evaluation metrics."""

    @abstractmethod
    def score(self, actual: Any, expected: Any, context: dict = None) -> float:
        """
        Compute score [0.0, 1.0] for (actual, expected) pair.
        0.0 = completely wrong. 1.0 = perfect match.
        Never raises — return 0.0 on any error.
        """
        ...
```

### 7.3 ProductionFeedbackLoop

```python
class ProductionFeedbackLoop(ABC):
    """
    Converts production failures into evaluation fixtures.
    Closes the loop from production trace to regression dataset.
    """

    @abstractmethod
    def capture_failure(
        self,
        trace_id: str,
        failure_description: str,
        dataset_id: str
    ) -> EvaluationFixture:
        """
        Create an EvaluationFixture from a production failure trace.

        Steps:
        1. Retrieve the trace from observability backend (D5).
        2. Extract task input from trace.
        3. Create fixture with source='production_failure' and appropriate metric.
        4. Add to dataset_id.

        Returns:
            EvaluationFixture ready for the regression dataset.
        """
        ...

    @abstractmethod
    def get_regression_dataset(self, agent_id: str) -> list[EvaluationFixture]:
        """
        Return all production-failure-sourced fixtures for an agent.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 Mock Tool Server

**THE SYSTEM SHALL** implement MockToolServer using the same MCP `tools/list` and `tools/call` protocol as real MCP servers.

**THE SYSTEM SHALL** return `MockToolResponse.response` when a tool call matches the configured `tool_name` and `arguments_match`.

**WHEN** `should_fail=True` **THE SYSTEM SHALL** raise `AgentPave.ToolUnavailableError` to simulate tool failure.

**THE SYSTEM SHALL** add every tool call to the call log, regardless of match result.

### 8.2 Evaluation Metrics

**THE SYSTEM SHALL** implement at minimum three built-in metrics:

- `ExactMatchMetric`: score=1.0 if actual==expected (string or dict), 0.0 otherwise
- `SemanticSimilarityMetric`: score = cosine similarity between embeddings of actual and expected
- `TrajectoryMetric`: score = fraction of expected trajectory steps that appear in actual trajectory in correct order

**THE SYSTEM SHALL** support custom metrics by implementing the `EvaluationMetric` abstract class.

### 8.3 CI/CD Integration

**THE SYSTEM SHALL** produce a machine-readable `evaluation-results.json` file after every `EvaluationRun`.

**THE SYSTEM SHALL** set `EvaluationRun.ci_gate_passed = True` when `pass_rate >= pass_rate_threshold`.

**THE SYSTEM SHALL** exit with non-zero exit code when `ci_gate_passed=False` — enabling CI/CD systems to block merges.

### 8.4 Production Feedback Loop

**THE SYSTEM SHALL** provide `ProductionFeedbackLoop.capture_failure()` to convert production failure traces into evaluation fixtures.

**THE SYSTEM SHALL** tag production-sourced fixtures with `source='production_failure'` for tracking.

---

## 9. Three-Tier Boundary System

### ALWAYS
- Use MockToolServer with same MCP protocol as real servers
- Exit with non-zero code when ci_gate_passed=False
- Tag production-sourced fixtures with source='production_failure'
- Log all mock tool calls to call log
- Return 0.0 from metric.score() on any internal error (never raise)

### ASK FIRST
- Lowering pass_rate_threshold below 0.80 — confirm acceptable quality regression
- Running evaluation against production tools (not mocks) in CI — high cost and latency risk

### NEVER
- Use real LLM calls in unit tests (use mocked LLM responses)
- Allow CI gate to pass when evaluation errors occurred (errors ≠ passes)
- Silently ignore evaluation fixture failures in CI

---

## 10. Error Handling

| Scenario | Error Type | Recoverable | Required context |
|---|---|---|---|
| Mock tool not configured for call | `AgentPave.ToolNotFoundError` | No — add mock config | `tool_name` |
| Evaluation metric raises | Caught — score returns 0.0 | Yes | Logged as warning |
| Production trace not found | `AgentPave.AgentNotFoundError` | No — check trace_id | `trace_id` |

---

## 11. Acceptance Criteria

```
Criteria ID:  D9-001
Stability:    Alpha
Given:        MockToolServer configured with web_search response
When:         Agent runs offline (no real MCP server) using MockToolServer
Then:         Tool call reaches MockToolServer, response returned correctly
              Call log shows one entry for web_search
Pass:         Mock response returned, call log has one entry, no real API call made
Fail:         Real API called, or mock response not returned
Error raised: None

---

Criteria ID:  D9-002
Stability:    Alpha
Given:        MockToolServer configured with should_fail=True for database_lookup
When:         Agent calls database_lookup
Then:         AgentPave.ToolUnavailableError raised by mock server
Pass:         ToolUnavailableError raised, agent handles failure per Integration Contract
Fail:         Mock server returns success despite should_fail=True
Error raised: AgentPave.ToolUnavailableError

---

Criteria ID:  D9-003
Stability:    Alpha
Given:        EvaluationFixture with expected_output="AgentPave is a framework"
              Agent produces actual_output="AgentPave is a tool for building agents"
              Metric: SemanticSimilarityMetric, pass_threshold=0.80
When:         EvaluationRunner.run_single(fixture) is called
Then:         Score computed between 0.80 and 1.0 (semantically similar)
              passed==True
Pass:         result.score >= 0.80, result.passed==True
Fail:         Score < 0.80 or passed==False for semantically similar outputs
Error raised: None

---

Criteria ID:  D9-004
Stability:    Alpha
Given:        EvaluationRun with 10 fixtures, 8 pass, 2 fail
              pass_rate_threshold=0.90
When:         EvaluationRunner.run() completes
Then:         EvaluationRun.pass_rate == 0.80
              ci_gate_passed == False (0.80 < 0.90)
              Process exits with non-zero code
Pass:         pass_rate==0.80, ci_gate_passed==False, exit code != 0
Fail:         ci_gate_passed==True despite failing threshold, or zero exit code
Error raised: None (SystemExit with non-zero code)

---

Criteria ID:  D9-005
Stability:    Alpha
Given:        A production failure trace with trace_id="<known_uuid>"
              Failure description: "Agent returned wrong department routing"
When:         ProductionFeedbackLoop.capture_failure(trace_id, description, dataset_id)
Then:         EvaluationFixture created with:
              - source == "production_failure"
              - input extracted from trace
              - fixture added to dataset_id
Pass:         Fixture created, source=="production_failure",
              get_regression_dataset(agent_id) includes this fixture
Fail:         Fixture not created or source not set correctly
Error raised: None
```

---

## 12. Definition of Done

- [ ] `EvaluationMetricType`, `EvaluationFixture`, `EvaluationResult`, `EvaluationRun` models implemented
- [ ] `MockToolResponse` model implemented
- [ ] `MockToolServer` abstract + concrete (MCP-compliant) implemented
- [ ] `EvaluationRunner` abstract + concrete implemented
- [ ] Three built-in metrics: ExactMatch, SemanticSimilarity, Trajectory
- [ ] `ProductionFeedbackLoop` abstract + concrete implemented
- [ ] CI/CD integration: evaluation-results.json output + non-zero exit on gate failure
- [ ] All 5 acceptance criteria pass: D9-001 through D9-005

---

## 13. Task Breakdown

```
Task D9-T1: Implement data models (EvaluationFixture, EvaluationResult, EvaluationRun, MockToolResponse)
Task D9-T2: Implement MockToolServer (MCP-compliant, with call log)
Task D9-T3: Implement EvaluationMetric abstract + ExactMatch, SemanticSimilarity, Trajectory
Task D9-T4: Implement EvaluationRunner abstract + concrete
Task D9-T5: Implement CI/CD integration (evaluation-results.json + exit code)
Task D9-T6: Implement ProductionFeedbackLoop abstract + concrete
Task D9-T7: Run all 5 acceptance criteria
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Mock servers that don't use MCP protocol**
A mock that bypasses MCP tests a different code path than production. Bugs in the MCP integration layer will only surface in production.

**Anti-Pattern 2 — Evaluating only final output**
An agent that produces the right answer via the wrong reasoning path is a reliability risk. Use trajectory evaluation for complex multi-step agents.

**Anti-Pattern 3 — Fixed pass-rate threshold of 100%**
A 100% threshold means any model update that changes wording (but not meaning) blocks the CI gate. Use semantic similarity with a configurable threshold.

**Anti-Pattern 4 — Never adding production failures to regression dataset**
Every production failure that is fixed without adding a regression case will recur. The feedback loop from production to evaluation dataset is mandatory.

---

## 15. Reference Implementation Notes

### 15.1 AgentPave-LangGraph
- MockToolServer: FastAPI app implementing MCP tools/list and tools/call endpoints
- SemanticSimilarityMetric: `sentence-transformers` for embedding generation
- EvaluationRunner: integrates with LangSmith's evaluation API as optional backend

### 15.2 AgentPave-MAF
- MockToolServer: same FastAPI implementation (language-agnostic MCP protocol)
- CI/CD: GitHub Actions workflow template included in AgentPave-MAF distribution

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should the mock LLM also be standardised (for unit testing without API keys)? | Yes — major DX improvement |
| OQ-2 | Should evaluation datasets be version-controlled alongside agent definitions? | No — good practice, not spec requirement |
| OQ-3 | Should trajectory evaluation support partial order (not strict sequence)? | No — add in v2 |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec |

---

*AgentPave Dimension 9 — Testability — v1.1*
