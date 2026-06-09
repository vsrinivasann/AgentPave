# AgentKit Dimension 10 — Developer Experience

**Spec version:** 1.1  
**Stability:** Alpha  
**Depends on:** D1–D9  
**Required by:** Nothing in Release 2  
**Owner:** AgentKit Core  
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

A framework that is technically complete but painful to use will not be adopted. Developer Experience is a first-class concern in AgentKit — not an afterthought. The most sophisticated agent framework in the world is worthless if developers can't run their first agent in 5 minutes.

D10 delivers five capabilities:
1. **CLI** — scaffold, run, debug, and deploy agents from the terminal
2. **Local runner** — run agents locally with full production-equivalent behaviour
3. **Local trace visualiser** — inspect every step without a cloud dependency
4. **Mock tool server** — develop offline without real API keys
5. **Agent templates** — scaffold common agent patterns in seconds

### 1.2 Production Consequence of Getting This Wrong

The framework with the most production case studies has the fewest stars of the mature options — meaning real-world adoption is undercounted by vanity metrics. The reason is developer experience friction. Every unnecessary step between "I want to build an agent" and "I have a running agent" is an adoption barrier. The "hello world" agent must run in under 5 minutes with zero external dependencies.

---

## 2. Scope Boundary

### 2.1 In Scope

- `agentkit` CLI with commands: init, run, invoke, debug, deploy, validate
- Local runner that mirrors production behaviour (checkpoints, observability, lifecycle)
- Local trace visualiser (terminal UI — no browser required)
- Agent template system (built-in templates + community templates)
- `agentkit validate` — validates AgentDefinition against spec before deployment
- Shell completion (bash, zsh, fish)
- VS Code extension spec (not the extension itself — interface contract only)

### 2.2 Out of Scope

- The VS Code extension implementation — that is a separate project
- Cloud deployment infrastructure — that is runtime-specific
- Package publishing/distribution — that is release engineering
- IDE plugins beyond VS Code spec — community implementations

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **5-minute hello world is a hard requirement** | SpringBoot succeeded partly because `spring init` + run was trivially fast. AgentKit must match this. Any setup that takes longer than 5 minutes loses developers. |
| **Local runner is production-equivalent** | A local runner that behaves differently from production is a liability — it creates bugs that only surface after deployment. Local must be identical to production. |
| **Terminal UI for trace visualiser** | Browser-based trace UIs require a running server, a port, and a browser. Terminal UI has zero external dependencies — it works in CI, SSH sessions, and Docker containers. |
| **Templates are YAML-declared agent skeletons** | Code-generated templates become outdated. YAML-declared templates that render from the current spec are always up to date. |
| **CLI is a Python package installable via pip** | Maximum reach. Developers who don't know Python can still install via `pip install agentkit-cli`. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **CLI** | Command-line interface. The `agentkit` command installed via pip. |
| **Local Runner** | The component that executes an agent locally with full AgentKit behaviour. |
| **Local Trace Visualiser** | A terminal UI that renders agent execution traces in real time. |
| **Agent Template** | A YAML-declared skeleton for a common agent pattern. Rendered into a working agent project by `agentkit init`. |
| **Project** | A directory containing an AgentDefinition, tools, handlers, and configuration. Created by `agentkit init`. |
| **AGENTS.md** | A markdown file at the project root that provides context for AI coding agents (Claude Code, Copilot etc.) about the AgentKit project structure. |
| **Hello World Agent** | The simplest possible AgentKit agent — one tool, one task, completes in under 5 minutes from `pip install` to first successful run. |
| **Validation** | The process of checking an AgentDefinition against the spec before deployment. Performed by `agentkit validate`. |

---

## 5. Data Models

### 5.1 CLICommand

```python
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum


class CLICommandName(str, Enum):
    INIT     = "init"      # Scaffold a new AgentKit project
    RUN      = "run"       # Run an agent locally (long-running)
    INVOKE   = "invoke"    # Send a single task to a running agent
    DEBUG    = "debug"     # Run with verbose trace output
    VALIDATE = "validate"  # Validate AgentDefinition against spec
    DEPLOY   = "deploy"    # Deploy to configured runtime
    EVAL     = "eval"      # Run evaluation suite
    TRACE    = "trace"     # View trace for a completed task


class CLICommandSpec(BaseModel):
    """Specification for a CLI command."""
    name: CLICommandName
    description: str
    required_args: list[str] = Field(default_factory=list)
    optional_flags: list[str] = Field(default_factory=list)
    example: str = Field(..., description="Example invocation shown in help text.")
    exit_codes: dict = Field(
        default_factory=dict,
        description="Map of exit code to meaning. e.g., {0: 'success', 1: 'validation error'}"
    )

    model_config = {"extra": "forbid"}
```

### 5.2 ProjectTemplate

```python
class ProjectTemplate(BaseModel):
    """An agent project template."""
    template_id: str = Field(..., description="Unique identifier. e.g., 'basic', 'rag-agent', 'router'.")
    name: str = Field(...)
    description: str = Field(...)
    files: list[str] = Field(..., description="List of files generated by this template.")
    runtime: str = Field(default="langgraph", description="'langgraph' or 'maf'.")
    tags: list[str] = Field(default_factory=list)

    model_config = {"extra": "forbid"}
```

### 5.3 LocalRunConfig

```python
class LocalRunConfig(BaseModel):
    """Configuration for local agent execution."""
    agent_definition_path: str = Field(..., description="Path to agent definition YAML/JSON file.")
    runtime: str = Field(default="langgraph")
    mock_tools: bool = Field(default=True, description="Use MockToolServer instead of real tools.")
    trace_output: str = Field(
        default="terminal",
        description="Where to write traces: 'terminal', 'file', or 'none'."
    )
    checkpoint_dir: str = Field(
        default=".agentkit/checkpoints",
        description="Local directory for checkpoint storage."
    )
    log_level: str = Field(default="INFO", description="Log verbosity: DEBUG, INFO, WARNING, ERROR.")

    model_config = {"extra": "forbid"}
```

### 5.4 ValidationResult

```python
class ValidationSeverity(str, Enum):
    ERROR   = "error"    # Spec violation — must be fixed before deployment
    WARNING = "warning"  # Best practice violation — should be fixed
    INFO    = "info"     # Informational — no action required


class ValidationIssue(BaseModel):
    """A single validation issue found in an AgentDefinition."""
    severity: ValidationSeverity
    dimension: str = Field(..., description="Which spec dimension this issue relates to.")
    field: str = Field(..., description="The AgentDefinition field with the issue.")
    message: str = Field(..., description="Human-readable description of the issue.")
    fix: str = Field(..., description="How to fix this issue.")

    model_config = {"extra": "forbid"}


class ValidationResult(BaseModel):
    """Result of validating an AgentDefinition."""
    valid: bool = Field(..., description="True only if zero ERROR severity issues.")
    errors: list[ValidationIssue] = Field(default_factory=list)
    warnings: list[ValidationIssue] = Field(default_factory=list)
    info: list[ValidationIssue] = Field(default_factory=list)
    spec_version_checked: str = Field(..., description="The spec version this was validated against.")

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Hello World — Complete 5-Minute Flow

```bash
# Step 1: Install (< 30 seconds)
pip install agentkit-cli

# Step 2: Scaffold a new project (< 10 seconds)
agentkit init my-first-agent --template basic --runtime langgraph

# Step 3: Run locally (< 30 seconds)
cd my-first-agent
agentkit run

# Step 4: Invoke with a task (< 5 seconds)
agentkit invoke "Search for information about AgentKit"

# Step 5: View the trace
agentkit trace --last
```

**Total: < 5 minutes from pip install to first successful agent run.**

### 6.2 Generated Project Structure

```
my-first-agent/
├── AGENTS.md                    ← AI coding agent context file
├── agentkit.yaml                ← AgentDefinition + runtime config
├── handlers/
│   └── deterministic.py         ← Deterministic task handlers
├── tools/
│   └── integration_contracts.py ← Tool Integration Contracts
├── tests/
│   ├── fixtures/                ← Evaluation fixtures
│   └── test_agent.py            ← Test file
└── .agentkit/
    ├── checkpoints/             ← Local checkpoint storage
    └── traces/                  ← Local trace storage
```

### 6.3 AGENTS.md Template

```markdown
# AgentKit Project: my-first-agent

## Project Structure
This is an AgentKit project. Key files:
- `agentkit.yaml`: Agent definition and configuration
- `handlers/`: Deterministic task handler functions
- `tools/`: MCP tool Integration Contracts
- `tests/`: Evaluation fixtures and test files

## AgentKit Spec
This project conforms to AgentKit Spec v1.1.1.
Dimension specs: docs.agentkit.io/spec

## Running Locally
agentkit run          # Start the agent
agentkit invoke "..."  # Send a task
agentkit trace --last  # View last trace

## Testing
agentkit eval         # Run evaluation suite
```

---

## 7. Interfaces & Contracts

### 7.1 AgentKitCLI

```python
# CLI command specifications (all commands)

CLI_COMMANDS = [
    CLICommandSpec(
        name=CLICommandName.INIT,
        description="Scaffold a new AgentKit project from a template.",
        required_args=["project_name"],
        optional_flags=["--template (default: basic)", "--runtime (default: langgraph)", "--dir"],
        example="agentkit init my-agent --template rag-agent --runtime langgraph",
        exit_codes={0: "success", 1: "invalid template", 2: "directory already exists"}
    ),
    CLICommandSpec(
        name=CLICommandName.RUN,
        description="Start the agent locally. Mirrors production behaviour.",
        required_args=[],
        optional_flags=["--config (default: agentkit.yaml)", "--mock-tools", "--port"],
        example="agentkit run --mock-tools",
        exit_codes={0: "agent retired cleanly", 1: "startup error", 2: "validation failed"}
    ),
    CLICommandSpec(
        name=CLICommandName.INVOKE,
        description="Send a single task to a running agent and wait for result.",
        required_args=["task"],
        optional_flags=["--agent-id", "--timeout"],
        example='agentkit invoke "Search for AgentKit documentation"',
        exit_codes={0: "task completed", 1: "task failed", 2: "agent not running"}
    ),
    CLICommandSpec(
        name=CLICommandName.DEBUG,
        description="Run agent with verbose step-by-step trace output.",
        required_args=[],
        optional_flags=["--breakpoint (pause at specific action types)"],
        example="agentkit debug --breakpoint tool_call",
        exit_codes={0: "success", 1: "error"}
    ),
    CLICommandSpec(
        name=CLICommandName.VALIDATE,
        description="Validate AgentDefinition against AgentKit Spec before deployment.",
        required_args=[],
        optional_flags=["--config", "--strict (treat warnings as errors)"],
        example="agentkit validate --strict",
        exit_codes={0: "valid", 1: "errors found", 2: "warnings found (with --strict)"}
    ),
    CLICommandSpec(
        name=CLICommandName.EVAL,
        description="Run evaluation suite against the agent.",
        required_args=[],
        optional_flags=["--dataset", "--threshold (default: 0.9)", "--mock-tools"],
        example="agentkit eval --threshold 0.85",
        exit_codes={0: "all pass", 1: "below threshold", 2: "eval error"}
    ),
    CLICommandSpec(
        name=CLICommandName.TRACE,
        description="View execution trace in terminal UI.",
        required_args=[],
        optional_flags=["--last", "--task-id", "--format (terminal|json)"],
        example="agentkit trace --last",
        exit_codes={0: "success", 1: "trace not found"}
    ),
]
```

### 7.2 LocalRunner

```python
from abc import ABC, abstractmethod


class LocalRunner(ABC):
    """
    Runs an AgentKit agent locally with full production-equivalent behaviour.
    Uses the same lifecycle, checkpointing, observability, and security as production.
    The ONLY difference from production: mock tools are available.
    """

    @abstractmethod
    def start(self, config: LocalRunConfig) -> None:
        """
        Start the agent. Blocks until the agent is REGISTERED and ready to receive tasks.
        Raises AgentKit.InvalidAgentDefinitionError if agentkit.yaml is invalid.
        """
        ...

    @abstractmethod
    def invoke(self, task: str, timeout_seconds: int = 300) -> "AgentResult":
        """
        Send a task to the running agent and wait for result.
        Returns AgentResult. Never raises — returns error AgentResult on failure.
        """
        ...

    @abstractmethod
    def stop(self) -> None:
        """Gracefully stop the agent. Transitions to RETIRED."""
        ...

    @abstractmethod
    def get_trace(self, task_id: str) -> list["StructuredLogEntry"]:
        """Retrieve the complete trace for a task."""
        ...
```

### 7.3 TraceVisualiser

```python
class TraceVisualiser(ABC):
    """
    Terminal UI for visualising agent execution traces.
    Zero external dependencies — works in any terminal.
    """

    @abstractmethod
    def render(self, trace: list["StructuredLogEntry"]) -> None:
        """
        Render a trace to the terminal.
        Shows: timeline, tool calls, LLM calls, lifecycle transitions, errors.
        """
        ...

    @abstractmethod
    def render_live(self, agent_id: str) -> None:
        """
        Render a live trace as the agent executes.
        Updates in real time. Press Ctrl+C to stop.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 Hello World Constraint

**THE SYSTEM SHALL** support a complete hello world flow (install → scaffold → run → invoke → trace) in under 5 minutes on a standard developer machine with internet access.

**THE SYSTEM SHALL** require zero external service dependencies for local development when `--mock-tools` is used.

### 8.2 CLI Commands

**THE SYSTEM SHALL** provide all 7 CLI commands with the specified exit codes.

**THE SYSTEM SHALL** provide shell completion for bash, zsh, and fish.

**THE SYSTEM SHALL** output human-readable error messages — never raw Python stack traces — when CLI commands fail.

### 8.3 Validation

**THE SYSTEM SHALL** validate AgentDefinition against agentkit-schema.json in `agentkit validate`.

**THE SYSTEM SHALL** report all ERROR and WARNING issues with field name, message, and fix suggestion.

**THE SYSTEM SHALL** exit with code 0 only when zero ERROR issues are found.

### 8.4 Local Runner Equivalence

**THE SYSTEM SHALL** use identical lifecycle management, checkpointing, observability, and security as the production runtime.

**THE SYSTEM SHALL** store checkpoints in `.agentkit/checkpoints/` by default.

**THE SYSTEM SHALL** write traces to `.agentkit/traces/` by default.

### 8.5 AGENTS.md

**THE SYSTEM SHALL** generate an `AGENTS.md` file in every project created by `agentkit init`.

**THE SYSTEM SHALL** include in `AGENTS.md`: project structure, key files, run commands, and spec version reference.

---

## 9. Three-Tier Boundary System

### ALWAYS
- Generate AGENTS.md in every project scaffolded by agentkit init
- Use production-equivalent behaviour in local runner
- Report human-readable errors (never raw stack traces) from CLI
- Exit with documented exit codes
- Validate AgentDefinition against spec in agentkit validate

### ASK FIRST
- Overwriting an existing project directory with agentkit init
- Deploying an agent that has validation warnings (not errors)

### NEVER
- Require internet access for local development with --mock-tools
- Expose sensitive credentials in CLI output or traces
- Use different lifecycle/checkpoint/security behaviour in local vs production

---

## 10. Error Handling

| Scenario | Exit Code | Output |
|---|---|---|
| agentkit init — directory exists | 2 | "Directory already exists. Use --force to overwrite." |
| agentkit validate — errors found | 1 | Formatted list of errors with fix suggestions |
| agentkit run — invalid YAML | 2 | "AgentDefinition is invalid: <error list>" |
| agentkit invoke — agent not running | 2 | "No agent running. Start with: agentkit run" |
| agentkit eval — below threshold | 1 | "Evaluation failed: pass_rate=X.XX < threshold=0.90" |

---

## 11. Acceptance Criteria

```
Criteria ID:  D10-001
Stability:    Alpha
Given:        Clean Python environment with pip available
When:         Developer runs:
              pip install agentkit-cli
              agentkit init my-agent --template basic
              cd my-agent && agentkit run --mock-tools &
              agentkit invoke "Hello, what can you do?"
              agentkit trace --last
Then:         All 5 commands succeed within 5 minutes total
              agentkit invoke returns a non-empty AgentResult
              agentkit trace renders a readable trace to terminal
Pass:         All commands exit 0, total time < 5 minutes, trace visible
Fail:         Any command fails, or total time > 5 minutes
Error raised: None

---

Criteria ID:  D10-002
Stability:    Alpha
Given:        AgentDefinition with version="not-semver" (invalid)
When:         agentkit validate is run
Then:         Exit code 1, error reported:
              field: "version"
              message: describes the SemVer violation
              fix: describes how to correct it
Pass:         Exit code 1, ValidationIssue with severity=ERROR, field="version"
Fail:         Exit code 0, or error not reported
Error raised: None (SystemExit with code 1)

---

Criteria ID:  D10-003
Stability:    Alpha
Given:        Local runner started with --mock-tools
When:         agentkit invoke "Search for something" is called
Then:         MockToolServer handles the tool call (no real API call)
              AgentResult returned with non-empty output
              Trace stored in .agentkit/traces/
Pass:         Mock tool used (no real API call), AgentResult returned,
              trace file exists in .agentkit/traces/
Fail:         Real API called, or AgentResult empty, or no trace stored
Error raised: None

---

Criteria ID:  D10-004
Stability:    Alpha
Given:        agentkit init my-agent --template basic is run
When:         Project directory is inspected
Then:         AGENTS.md exists with project structure, run commands, and spec version
              agentkit.yaml exists with a valid AgentDefinition skeleton
              tests/ directory exists with at least one fixture
Pass:         All required files present, AGENTS.md contains spec version reference,
              agentkit validate returns exit 0 for generated skeleton
Fail:         Any file missing, or generated skeleton fails validation
Error raised: None
```

---

## 12. Definition of Done

- [ ] All 7 CLI commands implemented with correct exit codes
- [ ] Shell completion for bash, zsh, fish
- [ ] `LocalRunner` abstract + concrete implemented
- [ ] Local runner uses production-equivalent behaviour
- [ ] `TraceVisualiser` terminal UI implemented
- [ ] `agentkit validate` validates against agentkit-schema.json
- [ ] Built-in templates: basic, rag-agent, router, multi-agent
- [ ] `AGENTS.md` generated in every `agentkit init` project
- [ ] Hello world flow completes in < 5 minutes (measured)
- [ ] Zero external dependencies for local dev with --mock-tools
- [ ] All 4 acceptance criteria pass: D10-001 through D10-004

---

## 13. Task Breakdown

```
Task D10-T1: Implement CLI framework (Click or Typer based)
Task D10-T2: Implement agentkit init with template system
Task D10-T3: Implement agentkit validate (schema + spec validation)
Task D10-T4: Implement LocalRunner (wraps chosen runtime with production behaviour)
Task D10-T5: Implement agentkit run / invoke / stop
Task D10-T6: Implement TraceVisualiser (terminal UI, Rich library)
Task D10-T7: Implement agentkit trace command
Task D10-T8: Implement agentkit debug (verbose trace mode)
Task D10-T9: Implement agentkit eval (wraps D9 EvaluationRunner)
Task D10-T10: Implement agentkit deploy (runtime-specific, pluggable)
Task D10-T11: Add shell completion (bash/zsh/fish)
Task D10-T12: Create built-in templates (basic, rag-agent, router, multi-agent)
Task D10-T13: AGENTS.md template generation
Task D10-T14: Measure hello world flow end-to-end < 5 minutes
Task D10-T15: Run all 4 acceptance criteria
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Local runner that differs from production**
A "simplified" local runner that doesn't use real checkpointing, real lifecycle management, or real security creates a false sense of correctness. Bugs in these areas only surface in production.

**Anti-Pattern 2 — Hello world requiring external services**
If the simplest possible agent requires a Redis instance, a vector DB, and an LLM API key, most developers will give up before their first success. Zero external dependencies for the 5-minute hello world is non-negotiable.

**Anti-Pattern 3 — Raw stack traces in CLI output**
A Python traceback is not a user-facing error message. Every CLI error must be human-readable with a clear fix suggestion. Stack traces go to a log file, not stdout.

**Anti-Pattern 4 — No AGENTS.md**
AI coding agents (Claude Code, GitHub Copilot) need project context to be useful. Without AGENTS.md, every coding session starts without context. This is a missed productivity opportunity for the exact audience AgentKit targets.

---

## 15. Reference Implementation Notes

### 15.1 AgentKit-LangGraph
- CLI framework: `typer` (pip: `typer[all]`) + `rich` for terminal UI
- TraceVisualiser: `rich` `Tree` and `Panel` components
- LocalRunner: wraps LangGraph's local execution with AgentKit lifecycle

### 15.2 AgentKit-MAF
- CLI: same Python CLI (language-agnostic command interface)
- LocalRunner: wraps MAF's local execution mode

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should agentkit deploy support multiple cloud targets (LangSmith, Azure, AWS)? | No — pluggable deployer interface |
| OQ-2 | Should there be a TUI (terminal UI) dashboard for long-running agents? | No — nice to have |
| OQ-3 | Should agentkit init support interactive prompts (wizard mode)? | No — flags are sufficient |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec |

---

*AgentKit Dimension 10 — Developer Experience — v1.1*
