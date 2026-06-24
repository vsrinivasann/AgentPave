# AgentPave Dimension 13 — Portability

**Spec version:** 1.1  
**Stability:** Alpha  
**Depends on:** D1–D12 (all dimensions)  
**Required by:** Nothing — this is the final dimension  
**Owner:** AgentPave Core  
**Target Release:** Release 3  

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

Portability is the promise that started AgentPave: write your agent definition once, run it anywhere. D13 formalises this promise into verifiable contracts and an interoperability test suite.

D13 delivers four capabilities:
1. **Runtime portability** — same AgentDefinition runs on LangGraph and MAF without modification
2. **Language portability** — the spec enables non-Python implementations
3. **Cloud portability** — no cloud provider dependencies in core
4. **Protocol portability** — MCP and A2A as universal, vendor-neutral interfaces

Portability is not a feature — it is the validation that every other dimension fulfilled its contract.

### 1.2 Production Consequence of Getting This Wrong

A spec that leaks runtime-specific concepts creates lock-in. Developers who adopt AgentPave expecting portability will find they have traded one form of vendor lock-in for another — a trust-destroying outcome. The interoperability test suite is the mechanism that proves portability is real, not claimed.

---

## 2. Scope Boundary

### 2.1 In Scope

- Interoperability test suite: same AgentDefinition produces equivalent behaviour on both runtimes
- Language portability contract: spec requirements that must hold in any language implementation
- Cloud portability requirements: no cloud-specific types, APIs, or dependencies in core
- Protocol portability: MCP and A2A as the only inter-system communication protocols
- Runtime adapter specification: what a new runtime must implement to be AgentPave-conformant
- Portability validation tooling: `agentpave portability-check` command

### 2.2 Out of Scope

- Building new language implementations (Java, Go, TypeScript) — community implementations
- Cloud migration tooling — that is infrastructure concern
- Protocol implementation — MCP and A2A are implemented by D3 (Communication)

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **Interoperability is tested, not assumed** | Claiming portability without a test suite is marketing. The interoperability test suite is the proof. |
| **Python is the reference language, not the only language** | The spec uses Python for data model examples because it is the first reference implementation. Nothing in the spec prevents Java, Go, or TypeScript implementations. |
| **MCP and A2A are the only protocol dependencies** | Any other protocol creates a portability barrier. No proprietary protocols are permitted in AgentPave Core. |
| **Cloud providers are pluggable backends only** | AWS, Azure, GCP may back storage, vector DBs, or observability — but they are never imported in core code. Core must run on a developer's laptop with no cloud account. |
| **Runtime adapter interface is fully specified** | A new runtime (LangGraph v2, a future framework) must only implement the RuntimeAdapter interface to be AgentPave-conformant. No other integration work required. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **Portability** | The property of an agent definition that allows it to run on any conformant AgentPave runtime without modification. |
| **Interoperability** | The property of two conformant AgentPave runtimes producing equivalent observable behaviour for the same agent definition. |
| **Runtime Adapter** | The implementation-specific bridge between AgentPave Core abstractions and a concrete runtime (LangGraph, MAF). |
| **Language Implementation** | An implementation of the AgentPave spec in a programming language other than Python. Must implement all core interfaces and pass the conformance test suite. |
| **Portability Violation** | A spec or implementation detail that prevents an agent definition from running on more than one runtime without modification. |
| **Equivalent Behaviour** | Two executions are behaviourally equivalent if: Agent IDs are structurally identical, lifecycle transitions produce identical state sequences, tool call results produce structurally identical AgentResults, and error types are identical for identical failure scenarios. |
| **Cloud-Agnostic** | The property of core code that contains no imports, dependencies, or types from any cloud provider SDK (AWS boto3, Azure SDK, GCP SDK). |

---

## 5. Data Models

### 5.1 PortabilityCheckResult

```python
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum


class PortabilityViolationType(str, Enum):
    RUNTIME_SPECIFIC_IMPORT  = "runtime_specific_import"   # Core imports a runtime module
    CLOUD_SPECIFIC_IMPORT    = "cloud_specific_import"     # Core imports a cloud SDK
    PROTOCOL_VIOLATION       = "protocol_violation"        # Non-MCP/A2A protocol used
    SCHEMA_DIVERGENCE        = "schema_divergence"         # Data models differ across runtimes
    BEHAVIOUR_DIVERGENCE     = "behaviour_divergence"      # Observable behaviour differs


class PortabilityViolation(BaseModel):
    violation_type: PortabilityViolationType
    severity: str = Field(..., description="'critical' or 'warning'.")
    location: str = Field(..., description="File and line number where violation occurs.")
    description: str = Field(...)
    fix: str = Field(...)

    model_config = {"extra": "forbid"}


class PortabilityCheckResult(BaseModel):
    """Result of running agentpave portability-check."""
    portable: bool = Field(..., description="True only if zero critical violations.")
    violations: list[PortabilityViolation] = Field(default_factory=list)
    runtimes_tested: list[str] = Field(default_factory=list)
    spec_version_checked: str = Field(...)
    checked_at: str = Field(...)

    model_config = {"extra": "forbid"}
```

### 5.2 InteroperabilityTestCase

```python
class InteroperabilityTestCase(BaseModel):
    """
    A single interoperability test case.
    Run against both LangGraph and MAF runtimes.
    Both must produce equivalent behaviour.
    """
    test_id: str = Field(..., description="UUID v4.")
    name: str = Field(...)
    description: str = Field(...)
    agent_definition: dict = Field(..., description="The AgentDefinition to test (as dict).")
    task_input: str = Field(...)
    expected_lifecycle_sequence: list[str] = Field(
        ...,
        description="Expected sequence of lifecycle states. e.g., ['DECLARED','REGISTERED','RUNNING','RETIRED']"
    )
    expected_tool_calls: list[str] = Field(
        default_factory=list,
        description="Expected tool names in order."
    )
    expected_result_status: str = Field(
        default="success",
        description="Expected AgentResult.status."
    )

    model_config = {"extra": "forbid"}


class InteroperabilityTestResult(BaseModel):
    """Result of running one InteroperabilityTestCase on one runtime."""
    test_id: str
    runtime: str = Field(..., description="'langgraph' or 'maf'.")
    passed: bool
    actual_lifecycle_sequence: list[str]
    actual_tool_calls: list[str]
    actual_result_status: str
    divergences: list[str] = Field(
        default_factory=list,
        description="List of ways this result diverges from expected."
    )
    duration_ms: int

    model_config = {"extra": "forbid"}
```

### 5.3 RuntimeAdapterSpec

```python
class RuntimeAdapterSpec(BaseModel):
    """
    Specification that a runtime adapter must implement to be AgentPave-conformant.
    A new runtime (e.g., a hypothetical future framework) implements this to join the ecosystem.
    """
    runtime_id: str = Field(..., description="Unique identifier. e.g., 'langgraph', 'maf'.")
    runtime_version: str = Field(..., description="SemVer of the runtime this adapter targets.")
    agentpave_spec_version: str = Field(..., description="AgentPave Spec version this adapter targets.")
    required_interfaces: list[str] = Field(
        ...,
        description=(
            "List of AgentPave abstract interfaces this adapter must implement. "
            "All 6 MVP dimension interfaces are required for conformance."
        )
    )
    optional_interfaces: list[str] = Field(
        default_factory=list,
        description="Post-MVP dimension interfaces optionally implemented."
    )

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Interoperability Test Case

```python
# The canonical interoperability test case
# Both LangGraph and MAF must produce identical results for this case

canonical_test = InteroperabilityTestCase(
    test_id="interop-canonical-001",
    name="Basic agent lifecycle with one tool call",
    description=(
        "The simplest possible agent: declares, registers, runs, "
        "invokes web_search once, completes, retires."
    ),
    agent_definition={
        "name": "interop-test-agent",
        "version": "1.0.0",
        "owner": "test@agentpave.io",
        "purpose": "Canonical interoperability test agent for AgentPave spec validation",
        "domain": "testing",
        "llm_interface_version": "1.0.0",
        "reliability_contract": {
            "hallucination_tolerance": "medium",
            "grounding_required": False,
            "quality_threshold": 0.5
        },
        "token_budget": {"per_task": 500, "per_day": 10000}
    },
    task_input="Search for AgentPave documentation",
    expected_lifecycle_sequence=["DECLARED", "REGISTERED", "RUNNING", "RETIRED"],
    expected_tool_calls=["web_search"],
    expected_result_status="success"
)
```

### 6.2 Portability Check Output

```
$ agentpave portability-check

AgentPave Portability Check v1.1.1
====================================
Checking: AgentPave-LangGraph v0.1.0

✓ No runtime-specific imports in core
✓ No cloud provider imports in core
✓ All inter-system calls use MCP or A2A
✓ Data models consistent with spec schema

Running interoperability test suite...
  ✓ interop-canonical-001: PASS (LangGraph: 342ms, MAF: 289ms)
  ✓ interop-lifecycle-002: PASS
  ✓ interop-error-003: PASS
  ✓ interop-checkpoint-004: PASS

Result: PORTABLE ✓
4/4 interoperability tests passed
0 critical violations, 0 warnings
```

---

## 7. Interfaces & Contracts

### 7.1 PortabilityChecker

```python
from abc import ABC, abstractmethod


class PortabilityChecker(ABC):
    """
    Validates that an AgentPave implementation is portable.
    Used by agentpave portability-check CLI command.
    """

    @abstractmethod
    def check_imports(self, implementation_path: str) -> list[PortabilityViolation]:
        """
        Scan implementation code for:
        - Runtime-specific imports in core modules
        - Cloud provider SDK imports in core modules
        - Non-MCP/A2A protocol usage
        Returns list of violations. Empty = no violations.
        """
        ...

    @abstractmethod
    def check_schema_consistency(self) -> list[PortabilityViolation]:
        """
        Verify data models are consistent with agentpave-schema.json.
        Returns list of divergences.
        """
        ...

    @abstractmethod
    def run_interoperability_suite(
        self,
        runtime_a: str,
        runtime_b: str
    ) -> list[InteroperabilityTestResult]:
        """
        Run the full interoperability test suite against both runtimes.
        Returns results for every test case on both runtimes.
        """
        ...

    @abstractmethod
    def full_check(self, implementation_path: str) -> PortabilityCheckResult:
        """
        Run all portability checks and return combined result.
        Called by agentpave portability-check command.
        """
        ...
```

### 7.2 RuntimeAdapter (interface that new runtimes must implement)

```python
class RuntimeAdapter(ABC):
    """
    The interface a new runtime must implement to be AgentPave-conformant.
    Provides the bridge between AgentPave Core abstractions and the runtime.

    Any runtime that implements all required methods and passes the
    conformance test suite may claim 'AgentPave Conformant Runtime'.
    """

    @abstractmethod
    def get_runtime_id(self) -> str:
        """Return the runtime identifier. e.g., 'langgraph', 'maf'."""
        ...

    @abstractmethod
    def get_agent_registry(self) -> "AgentRegistry":
        """Return the runtime's AgentRegistry implementation."""
        ...

    @abstractmethod
    def get_lifecycle_manager(self) -> "LifecycleManager":
        """Return the runtime's LifecycleManager implementation."""
        ...

    @abstractmethod
    def get_tool_caller(self) -> "ToolCaller":
        """Return the runtime's ToolCaller implementation."""
        ...

    @abstractmethod
    def get_memory_stores(self) -> dict:
        """
        Return all memory store implementations.
        Keys: 'working', 'episodic', 'semantic', 'procedural'
        """
        ...

    @abstractmethod
    def get_agent_observer(self) -> "AgentObserver":
        """Return the runtime's AgentObserver implementation."""
        ...

    @abstractmethod
    def get_checkpoint_store(self) -> "CheckpointStore":
        """Return the runtime's CheckpointStore implementation."""
        ...
```

---

## 8. Behaviour Specification

### 8.1 Runtime Portability

**THE SYSTEM SHALL** produce structurally identical AgentRecords for the same AgentDefinition on any conformant runtime.

**THE SYSTEM SHALL** produce identical lifecycle state sequences for the same task on any conformant runtime.

**THE SYSTEM SHALL** produce structurally identical AgentResult objects for the same tool call on any conformant runtime.

**THE SYSTEM SHALL** raise the same AgentPave error types for the same failure conditions on any conformant runtime.

### 8.2 Language Portability

**THE SYSTEM SHALL** define all spec requirements in language-agnostic terms (EARS notation, abstract interfaces, JSON Schema).

**THE SYSTEM SHALL NOT** include any Python-specific syntax, types, or idioms in MUST requirements.

**WHEN** the spec uses Python for examples, it MUST include a note: "This Python example is illustrative. Implementations in other languages must fulfil the same contract."

### 8.3 Cloud Portability

**THE SYSTEM SHALL** ensure no cloud provider SDK is imported in AgentPave Core.

**THE SYSTEM SHALL** verify this with an automated linting rule that scans core for imports of `boto3`, `azure-*`, `google-cloud-*`, and similar cloud SDK packages.

### 8.4 Protocol Portability

**THE SYSTEM SHALL** use only MCP for agent-to-tool communication.

**THE SYSTEM SHALL** use only A2A for agent-to-agent communication.

**THE SYSTEM SHALL NOT** introduce any proprietary protocol for inter-system communication.

---

## 9. Three-Tier Boundary System

### ALWAYS
- Same AgentDefinition must run on any conformant runtime without modification
- Same AgentResult schema on all runtimes
- Same error types for same failure conditions on all runtimes
- No cloud provider imports in core
- Only MCP and A2A for inter-system communication
- Run interoperability test suite on every release

### ASK FIRST
- Adding a new inter-system protocol beyond MCP/A2A — requires spec update
- Adding a cloud-specific optimisation path in core — confirm portability is preserved

### NEVER
- Import cloud provider SDKs in core modules
- Use a proprietary protocol for inter-system communication
- Include Python-specific MUST requirements in the spec
- Release a new runtime without passing the interoperability test suite

---

## 10. Error Handling

Portability violations are reported as `PortabilityViolation` objects — not exceptions. The `agentpave portability-check` command exits with non-zero code when critical violations are found.

| Violation Type | Severity | Fix |
|---|---|---|
| Runtime-specific import in core | critical | Move to runtime adapter |
| Cloud SDK import in core | critical | Move to pluggable backend |
| Non-MCP/A2A protocol in core | critical | Replace with MCP or A2A |
| Schema divergence from spec | critical | Fix data model to match spec |
| Behaviour divergence between runtimes | critical | Fix the diverging runtime |

---

## 11. Acceptance Criteria

```
Criteria ID:  D13-001
Stability:    Alpha
Given:        The canonical interoperability test case (Section 6.1)
When:         Run on AgentPave-LangGraph AND AgentPave-MAF with mock tools
Then:         Both runtimes produce:
              - Identical lifecycle state sequence: [DECLARED, REGISTERED, RUNNING, RETIRED]
              - Identical tool call sequence: [web_search]
              - AgentResult.status == "success" on both
              - AgentResult.trace_id format matches (UUID v4) on both
Pass:         All conditions identical on both runtimes
Fail:         Any divergence between runtimes
Error raised: None

---

Criteria ID:  D13-002
Stability:    Alpha
Given:        AgentPave-LangGraph core source code
When:         agentpave portability-check runs import scan
Then:         Zero cloud provider SDK imports found in core modules
              Zero runtime-specific (LangGraph) imports in core interfaces
Pass:         portability_check.violations is empty for import checks
Fail:         Any cloud or runtime-specific import found in core
Error raised: None (SystemExit non-zero if violations found)

---

Criteria ID:  D13-003
Stability:    Alpha
Given:        The same AgentDefinition (canonical example from Section 6.1)
When:         AgentPave.InvalidAgentDefinitionError is triggered on BOTH runtimes
              (by submitting an invalid version string)
Then:         Both runtimes raise AgentPave.InvalidAgentDefinitionError
              Both error contexts contain validation_errors list
              Both exit with the same error_code: AGENTKIT_INVALID_AGENT_DEFINITION
Pass:         Same error type and error_code on both runtimes
Fail:         Different error types or codes on different runtimes
Error raised: AgentPave.InvalidAgentDefinitionError (on both runtimes)

---

Criteria ID:  D13-004
Stability:    Alpha
Given:        The AgentPave spec MUST requirements (from all 13 dimensions)
When:         Each MUST requirement is inspected for Python-specific syntax
Then:         Zero MUST requirements contain Python-specific constructs
              (Python-specific constructs include: list comprehensions in requirements,
               Python type hints in MUST statements, Python import syntax in requirements)
Pass:         Zero Python-specific MUST requirements found
Fail:         Any MUST requirement uses Python-specific syntax
Error raised: None (spec audit finding, not runtime error)
```

---

## 12. Definition of Done

- [ ] `PortabilityViolationType`, `PortabilityViolation`, `PortabilityCheckResult` models implemented
- [ ] `InteroperabilityTestCase`, `InteroperabilityTestResult` models implemented
- [ ] `RuntimeAdapterSpec` model implemented
- [ ] `PortabilityChecker` abstract + concrete implemented
- [ ] `RuntimeAdapter` abstract interface implemented
- [ ] Canonical interoperability test suite (minimum 4 test cases) implemented
- [ ] `agentpave portability-check` CLI command implemented
- [ ] CI linting rule: no cloud SDK imports in core
- [ ] CI linting rule: no runtime-specific imports in core
- [ ] Interoperability suite runs on every release (CI integration)
- [ ] All 4 acceptance criteria pass: D13-001 through D13-004
- [ ] Both AgentPave-LangGraph and AgentPave-MAF pass full interoperability suite

---

## 13. Task Breakdown

```
Task D13-T1: Implement data models (PortabilityViolation, PortabilityCheckResult,
             InteroperabilityTestCase, InteroperabilityTestResult, RuntimeAdapterSpec)
Task D13-T2: Implement PortabilityChecker abstract + concrete
Task D13-T3: Implement RuntimeAdapter abstract interface
Task D13-T4: Build canonical interoperability test suite (4+ test cases)
Task D13-T5: Implement agentpave portability-check CLI command
Task D13-T6: Add CI linting rules (cloud SDK imports, runtime-specific imports)
Task D13-T7: Run interoperability suite against both runtimes
Task D13-T8: Run all 4 acceptance criteria
Task D13-T9: Spec audit: verify zero Python-specific MUST requirements
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — "Portable by convention"**
Claiming portability without a test suite is a hope, not a guarantee. The interoperability test suite is the only credible portability claim. Run it on every release.

**Anti-Pattern 2 — Runtime-specific types in agent definitions**
An AgentDefinition field that uses a LangGraph-specific type (e.g., `StateGraph`) is a portability violation. Agent definitions must be pure Python dicts or JSON — no runtime types.

**Anti-Pattern 3 — Cloud SDK imports in core**
An AgentPave core module that imports `boto3` for S3 checkpoint storage means AgentPave requires AWS. This violates the cloud-agnostic principle and excludes developers on Azure or GCP.

**Anti-Pattern 4 — Proprietary protocols**
Using a proprietary RPC protocol instead of MCP or A2A for tool communication creates a wall that prevents tool reuse across frameworks. MCP is already the universal standard — use it.

**Anti-Pattern 5 — Python-specific spec requirements**
A spec requirement that says "must be a Python `@dataclass`" cannot be implemented in Java or Go. Every MUST requirement must be expressible in any language.

---

## 15. Reference Implementation Notes

### 15.1 Interoperability Test Runner
- Run both runtimes in Docker containers to ensure environment isolation
- Use MockToolServer (D9) to ensure identical tool responses on both runtimes
- Compare results using deep equality on AgentResult schema fields

### 15.2 Language Implementation Guide
Future implementations in Java, Go, or TypeScript must:
1. Implement all abstract interfaces defined in D1–D6
2. Use agentpave-schema.json for data model validation
3. Pass the full conformance test suite
4. Pass the interoperability test suite against at least one existing runtime
5. Publish an `agentpave-conformance.yaml` declaring conformance level

### 15.3 Known Portability Constraints
| Constraint | LangGraph | MAF | Spec Impact |
|---|---|---|---|
| Checkpoint format | LangGraph-native | MAF-native | CheckpointData schema is portable; storage format is not |
| Trust domain config | Env var | App setting | Config mechanism differs; value format identical |
| MCP client library | `mcp` Python | MAF built-in | Both use same MCP protocol; library differs |

All constraints are implementation details — observable behaviour is identical.

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should the interoperability test suite be published as a standalone package? | Yes — enables community runtime implementations |
| OQ-2 | Should AgentPave publish a Java reference implementation? | No — community effort |
| OQ-3 | Should checkpoint format be standardised for cross-runtime checkpoint portability? | Yes — major feature, requires spec update |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec — interoperability test suite, RuntimeAdapter interface, portability checker |

---

*AgentPave Dimension 13 — Portability — v1.1*
