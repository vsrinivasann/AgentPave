# AgentPave Dimension 11 — Extensibility

**Spec version:** 1.1  
**Stability:** Alpha  
**Depends on:** D1–D6 (MVP dimensions)  
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

A framework that cannot be extended is a framework with a ceiling. SpringBoot succeeded because its extension model (starters, auto-configuration, beans) allowed the community to build on top of it without forking it. AgentPave needs the same.

D11 defines the extension model that governs how capabilities are added to AgentPave Core without modifying it. The core design principle — **core never depends on extensions; extensions always depend on core** — is enforced structurally, not by convention.

### 1.2 Production Consequence of Getting This Wrong

Extensibility that is not governed becomes a source of instability. Extensions that modify core behaviour, create circular dependencies, or attach to undeclared extension points are the leading cause of framework version hell. Every extension must have a defined contract, a declared attachment point, and an independent version.

---

## 2. Scope Boundary

### 2.1 In Scope

- Extension point declarations (8 declared points from SPEC.md)
- Extension manifest schema
- Extension loading, validation, and registration
- Extension versioning and compatibility checking
- Extension isolation — extensions cannot modify core
- Extension dependency declaration
- Community extension registry (discovery, not hosting)

### 2.2 Out of Scope

- Building specific extensions (AgentPave-Observe, AgentPave-Eval etc.) — those are separate projects
- Extension hosting or package publishing — that is registry infrastructure
- Extension sandboxing/security isolation — that is D7 concern

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **Eight declared extension points, no others** | Open-ended extension points lead to undocumented hooks that break across versions. Eight named points with defined interfaces are testable, documented, and stable. |
| **Extensions are independently versioned** | An extension that shares core's version number cannot be updated independently. Each extension must declare its own SemVer and minimum required core version. |
| **Core never imports extensions** | Enforced via Python's import system and a linting rule. Any import of an extension module in core code fails CI. |
| **Extensions declare their attachment point** | An extension that doesn't declare its attachment point cannot be loaded. Implicit attachment is not supported. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **Extension Point** | A named, versioned hook in AgentPave Core where an extension may attach behaviour. Eight are declared. No others exist until formally added to the spec. |
| **Extension** | An optional, versioned module that adds capabilities to AgentPave Core via a single declared extension point. |
| **Extension Manifest** | A YAML/JSON file declaring: extension identity, version, minimum core version, attachment point, and configuration schema. |
| **Extension Registry** | The discovery mechanism for published AgentPave extensions. Not a hosting service — a searchable index. |
| **Dependency Conflict** | A state where two extensions require incompatible versions of AgentPave Core or of a shared dependency. |
| **Extension Isolation** | The property that an extension cannot modify core behaviour — it can only add behaviour at its declared extension point. |

---

## 5. Data Models

### 5.1 ExtensionManifest

```python
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum


class ExtensionPointName(str, Enum):
    OBSERVABILITY_BACKEND  = "observability.backend"
    EVALUATION_RUNNER      = "evaluation.runner"
    HITL_APPROVAL          = "hitl.approval"
    MULTIAGENT_COORDINATOR = "multiagent.coordinator"
    CACHE_SEMANTIC         = "cache.semantic"
    MEMORY_CONSOLIDATION   = "memory.consolidation"
    GOVERNANCE_POLICY      = "governance.policy"
    AGENT_REGISTRY         = "agent.registry"


class ExtensionManifest(BaseModel):
    """
    The manifest file for an AgentPave extension.
    Every extension must include this file as agentpave-extension.yaml.
    """
    extension_id: str = Field(..., description="Unique identifier. e.g., 'agentpave-observe'.")
    extension_name: str = Field(..., description="Human-readable name.")
    version: str = Field(..., description="SemVer. Independent of AgentPave Core version.")
    agentpave_core_min_version: str = Field(
        ...,
        description="Minimum AgentPave Core SemVer this extension requires."
    )
    agentpave_core_max_version: Optional[str] = Field(
        None,
        description="Maximum AgentPave Core SemVer (exclusive). None = no upper bound."
    )
    attachment_point: ExtensionPointName = Field(
        ...,
        description="Which extension point this extension attaches to. Exactly one."
    )
    entry_point: str = Field(
        ...,
        description=(
            "Python import path to the class implementing the extension point interface. "
            "Example: 'agentpave_observe.backend.LangSmithBackend'"
        )
    )
    description: str = Field(..., min_length=10)
    author: str = Field(...)
    license: str = Field(default="Apache-2.0")
    config_schema: Optional[dict] = Field(
        None,
        description="JSON Schema for extension configuration. None = no configuration required."
    )
    dependencies: list[str] = Field(
        default_factory=list,
        description="Python package dependencies required by this extension."
    )

    model_config = {"extra": "forbid"}
```

### 5.2 ExtensionRegistration

```python
class ExtensionRegistration(BaseModel):
    """Record of a loaded and registered extension."""
    registration_id: str = Field(..., description="UUID v4.")
    manifest: ExtensionManifest = Field(...)
    instance: object = Field(
        ...,
        exclude=True,
        description="The instantiated extension object. Excluded from serialisation."
    )
    registered_at: str = Field(..., description="ISO 8601 UTC.")
    config: dict = Field(default_factory=dict)
    status: str = Field(default="active", description="'active' or 'disabled'.")

    model_config = {"extra": "forbid", "arbitrary_types_allowed": True}
```

### 5.3 CompatibilityCheckResult

```python
class CompatibilityCheckResult(BaseModel):
    """Result of checking extension compatibility with current core version."""
    compatible: bool
    extension_id: str
    core_version: str
    required_min: str
    required_max: Optional[str]
    issues: list[str] = Field(default_factory=list)

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Extension Manifest (agentpave-extension.yaml)

```yaml
extension_id: agentpave-observe-langsmith
extension_name: AgentPave LangSmith Observability Backend
version: 1.0.0
agentpave_core_min_version: "1.1.0"
agentpave_core_max_version: null
attachment_point: observability.backend
entry_point: agentpave_observe_langsmith.backend.LangSmithBackend
description: Routes AgentPave observability events to LangSmith for tracing and evaluation.
author: AgentPave Contributors
license: Apache-2.0
config_schema:
  type: object
  required: [api_key, project_name]
  properties:
    api_key:
      type: string
      description: LangSmith API key (use secret_ref:// for production)
    project_name:
      type: string
      description: LangSmith project name
dependencies:
  - langsmith>=0.2.0
```

### 6.2 Loading an Extension

```python
# Load and register an extension
loader = ExtensionLoader()
registration = loader.load(
    manifest_path="agentpave-extension.yaml",
    config={"api_key": "secret_ref://vault/langsmith/api_key", "project_name": "my-agents"}
)

# Extension is now active at its attachment point
# AgentObserver will route events to LangSmithBackend
assert registration.status == "active"
assert registration.manifest.attachment_point == ExtensionPointName.OBSERVABILITY_BACKEND
```

---

## 7. Interfaces & Contracts

### 7.1 ExtensionLoader

```python
from abc import ABC, abstractmethod


class ExtensionLoader(ABC):
    """
    Loads, validates, and registers AgentPave extensions.
    """

    @abstractmethod
    def load(
        self,
        manifest_path: str,
        config: dict = None
    ) -> ExtensionRegistration:
        """
        Load an extension from its manifest file.

        Steps:
        1. Parse and validate agentpave-extension.yaml against ExtensionManifest schema.
        2. Check compatibility with current AgentPave Core version.
        3. Validate config against manifest.config_schema (if defined).
        4. Import entry_point class.
        5. Verify class implements the interface for attachment_point.
        6. Instantiate the class with config.
        7. Register at attachment_point.
        8. Return ExtensionRegistration.

        Raises:
            AgentPave.ExtensionLoadError: any step fails.
            AgentPave.ExtensionCompatibilityError: version incompatible.
            AgentPave.ExtensionConflictError: another extension already registered at this point.
        """
        ...

    @abstractmethod
    def check_compatibility(
        self,
        manifest: ExtensionManifest,
        core_version: str
    ) -> CompatibilityCheckResult:
        """Check if an extension is compatible with core_version."""
        ...

    @abstractmethod
    def unload(self, extension_id: str) -> None:
        """
        Unload and deregister an extension.
        The attachment point reverts to its default implementation.
        """
        ...

    @abstractmethod
    def list_registered(self) -> list[ExtensionRegistration]:
        """List all currently registered extensions."""
        ...
```

---

## 8. Behaviour Specification

### 8.1 Extension Loading

**THE SYSTEM SHALL** validate every extension manifest against the `ExtensionManifest` schema before loading.

**THE SYSTEM SHALL** check core version compatibility before loading any extension.

**WHEN** core version is outside `[agentpave_core_min_version, agentpave_core_max_version)` **THE SYSTEM SHALL** raise `AgentPave.ExtensionCompatibilityError`.

**THE SYSTEM SHALL** verify the extension's entry point class implements the correct interface for its declared `attachment_point`.

**WHEN** an extension attempts to attach at an undeclared extension point **THE SYSTEM SHALL** raise `AgentPave.ExtensionLoadError`.

### 8.2 Extension Isolation

**THE SYSTEM SHALL** enforce that AgentPave Core never imports extension modules.

**THE SYSTEM SHALL** enforce this via a CI linting rule: any import of a known extension package in core code fails the build.

**WHEN** an extension attempts to modify core data (AgentRegistry, LifecycleManager) directly **THE SYSTEM SHALL** raise `AttributeError` — core objects are not modifiable by extensions.

### 8.3 Single Registration Per Point

**THE SYSTEM SHALL** allow only one extension registered per extension point at a time.

**WHEN** a second extension attempts to register at an already-occupied extension point **THE SYSTEM SHALL** raise `AgentPave.ExtensionConflictError`.

---

## 9. Three-Tier Boundary System

### ALWAYS
- Validate extension manifest against schema before loading
- Check core version compatibility before loading
- Verify entry point implements correct interface
- Allow only one extension per extension point
- Never allow core to import extension modules

### ASK FIRST
- Unloading an active extension mid-operation — confirm no in-flight tasks depend on it
- Loading an extension with no declared max core version — note it may break on core upgrades

### NEVER
- Allow extension to attach at undeclared extension point
- Allow extension to modify core objects (AgentRegistry, LifecycleManager, etc.)
- Load an extension that is incompatible with current core version
- Allow two extensions at the same extension point simultaneously

---

## 10. Error Handling

| Scenario | Error Type | Recoverable | Required context |
|---|---|---|---|
| Manifest schema invalid | `AgentPave.ExtensionLoadError` | No — fix manifest | `extension_id, validation_errors` |
| Core version incompatible | `AgentPave.ExtensionCompatibilityError` | No — update extension or core | `extension_id, core_version, required_min` |
| Extension point already occupied | `AgentPave.ExtensionConflictError` | No — unload existing first | `extension_id, attachment_point, current_extension` |
| Entry point class not found | `AgentPave.ExtensionLoadError` | No — fix entry_point path | `extension_id, entry_point` |
| Entry point wrong interface | `AgentPave.ExtensionLoadError` | No — fix implementation | `extension_id, expected_interface` |

---

## 11. Acceptance Criteria

```
Criteria ID:  D11-001
Stability:    Alpha
Given:        A valid extension manifest with attachment_point=observability.backend
When:         ExtensionLoader.load(manifest_path, config) is called
Then:         Extension loaded and registered at observability.backend
              ExtensionRegistration returned with status="active"
Pass:         Registration status=="active",
              manifest.attachment_point==observability.backend
Fail:         Error raised despite valid manifest, or status not "active"
Error raised: None

---

Criteria ID:  D11-002
Stability:    Alpha
Given:        Extension manifest declaring agentpave_core_min_version="2.0.0"
              Current AgentPave Core version is "1.1.1"
When:         ExtensionLoader.load() is called
Then:         AgentPave.ExtensionCompatibilityError raised
              Extension is NOT loaded
Pass:         ExtensionCompatibilityError raised,
              context contains version information
Fail:         Extension loaded despite version incompatibility
Error raised: AgentPave.ExtensionCompatibilityError

---

Criteria ID:  D11-003
Stability:    Alpha
Given:        Extension A already registered at observability.backend
When:         Extension B attempts to register at observability.backend
Then:         AgentPave.ExtensionConflictError raised
              Extension A remains active
Pass:         ExtensionConflictError raised,
              Extension A still registered (not replaced)
Fail:         Extension B replaces Extension A silently
Error raised: AgentPave.ExtensionConflictError

---

Criteria ID:  D11-004
Stability:    Alpha
Given:        Extension manifest with attachment_point="custom.nonexistent"
              (not in ExtensionPointName enum)
When:         ExtensionLoader.load() is called
Then:         AgentPave.ExtensionLoadError raised
Pass:         ExtensionLoadError raised mentioning invalid attachment point
Fail:         Extension loaded at undeclared point
Error raised: AgentPave.ExtensionLoadError
```

---

## 12. Definition of Done

- [ ] `ExtensionPointName` enum with all 8 points implemented
- [ ] `ExtensionManifest` Pydantic model implemented
- [ ] `ExtensionRegistration` model implemented
- [ ] `CompatibilityCheckResult` model implemented
- [ ] `ExtensionLoader` abstract + concrete implemented
- [ ] Core version compatibility checking implemented
- [ ] Interface verification for each attachment point
- [ ] Single-registration-per-point enforcement
- [ ] CI linting rule: core never imports extensions
- [ ] All 4 acceptance criteria pass: D11-001 through D11-004

---

## 13. Task Breakdown

```
Task D11-T1: Implement ExtensionPointName enum, ExtensionManifest, ExtensionRegistration models
Task D11-T2: Implement ExtensionLoader abstract + concrete
Task D11-T3: Implement core version compatibility checking (SemVer range evaluation)
Task D11-T4: Implement interface verification for each of 8 extension points
Task D11-T5: Implement single-registration enforcement
Task D11-T6: Add CI linting rule (pylint/ruff) blocking core imports of extensions
Task D11-T7: Run all 4 acceptance criteria
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Undeclared extension points**
Allowing extensions to hook into arbitrary parts of core creates invisible coupling. When core changes, all extensions that hooked into undeclared points break silently.

**Anti-Pattern 2 — Extensions modifying core objects**
An extension that reaches into AgentRegistry and changes records is a core integrity violation. Extensions add behaviour; they never modify core state.

**Anti-Pattern 3 — No version compatibility checking**
Loading an extension built for core v1.0 on core v2.0 (or vice versa) produces unpredictable behaviour. Compatibility checking is not optional.

**Anti-Pattern 4 — Multiple extensions at one point**
Two observability backends both trying to receive every event creates ordering ambiguity, duplicate events, and debugging nightmares.

---

## 15. Reference Implementation Notes

### 15.1 AgentPave-LangGraph and AgentPave-MAF
- Extension manifest discovery: scan for `agentpave-extension.yaml` in installed packages
- Entry point loading: Python `importlib.import_module()` + `getattr()`
- Interface verification: `isinstance(instance, expected_abstract_class)`
- SemVer comparison: `packaging.version.Version` (pip: `packaging`)

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should multiple extensions be allowed at one point via a chain/composite pattern? | Yes — impacts design |
| OQ-2 | Should extensions be loadable at runtime (hot-loading) or only at startup? | No — startup-only is simpler |
| OQ-3 | Should there be a community extension registry with search? | No — nice to have |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec |

---

*AgentPave Dimension 11 — Extensibility — v1.1*
