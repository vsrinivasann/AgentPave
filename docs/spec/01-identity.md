# AgentPave Dimension 1 — Identity

**Spec version:** 1.1  
**Stability:** Beta  
**Depends on:** None — this is the root dimension  
**Required by:** All other dimensions  
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

Identity is the root of everything in AgentPave. Every agent must have a stable, unique, cryptographically verifiable identity before it can exist in the system. Every other dimension — Lifecycle, Communication, Memory, Observability, Reliability — references an agent by its Identity. Without a well-defined identity model, no operation in AgentPave has a reliable anchor.

Identity in AgentPave serves three distinct purposes:

1. **Addressability** — every agent can be uniquely and unambiguously referenced across all dimensions, all runtimes, and all time
2. **Traceability** — every action, log entry, span, checkpoint, and error is linked to a specific agent identity
3. **Trustworthiness** — every agent presents a cryptographically verifiable runtime identity that proves it is who it claims to be

### 1.2 Production Consequence of Getting This Wrong

Establishing a verifiable, cryptographic identity for every agent and preserving the full chain of delegation from user to agent to tool is the identity and trust foundation layer — without it, dynamic access control and unified policy enforcement cannot be built.

If Identity is implemented incorrectly:
- Two agents with colliding IDs will corrupt each other's state, logs, and checkpoints silently
- An agent with no cryptographic identity cannot be authenticated in a Zero Trust system — it becomes a rogue agent
- An unversioned agent definition cannot be safely updated in a multi-agent system without silent protocol mismatches
- An agent with no declared owner or domain cannot be governed, audited, or retired by the right party

---

## 2. Scope Boundary

### 2.1 In Scope

- Declaring a new agent and assigning it a UUID v4 Agent ID
- Defining and validating the complete Agent Definition schema
- Generating and associating a cryptographic runtime identity (SPIFFE ID + JWT) with every agent instance
- Storing agent metadata in the Agent Registry
- Looking up an agent by Agent ID
- Validating an agent definition against agentpave-schema.json
- Retiring (deactivating) an agent identity

### 2.2 Out of Scope

- Lifecycle state management — that is Dimension 2
- Authentication of external users invoking agents — that is Dimension 6 (Security)
- Tool registration — that is Dimension 3
- Memory initialisation — that is Dimension 4
- Cost budget enforcement — that is Dimension 8 (Economics)
- Agent-to-agent trust chains beyond identity issuance — that is Dimension 6 (Security)

---

## 3. Prior Decisions

These decisions are closed. Implementations MUST NOT re-open them.

| Decision | Rationale |
|---|---|
| **Agent ID format is UUID v4** | UUID v4 is universally unique, requires no central coordinator, is runtime-agnostic, and is natively supported in Python, Java, Go, and .NET |
| **Cryptographic identity uses SPIFFE** | Every agent instance gets a unique, cryptographically-verifiable SPIFFE identity at spawn, encoding which orchestrator created it, which task it serves, and which instance it is — one identity per instance, never reused |
| **Runtime tokens are JWTs with 5-minute default TTL** | JWTs scoped to exactly the resource and action the task requires. Default TTL: 5 minutes. Max 11 renewals (1 hour total). Key-bound: without the agent's private key, a stolen JWT is useless — eliminates replay attacks |
| **Agent ID is immutable** | Mutable identities create traceability gaps — a renamed agent cannot be reliably linked to its prior actions in logs, checkpoints, or audit trails |
| **Agent definitions are validated against JSON Schema** | Machine-readable schema enables programmatic validation by tooling and autonomous agents without parsing prose |
| **Agent Registry is append-only** | Append-only or cryptographically verifiable log stores prevent silent modifications or deletions — providing Proof of Action for every agent |
| **Canonical JSON uses RFC 8785** | RFC 8785 (JSON Canonicalization Scheme) is the standard for deterministic JSON serialisation — ensures definition_hash is reproducible across languages and runtimes |

---

## 4. Definitions

Terms specific to this dimension. All terms from the master spec Glossary also apply.

| Term | Definition |
|---|---|
| **Agent Declaration** | The act of submitting a valid Agent Definition to AgentPave, resulting in an Agent ID being assigned. Declaration is the first step in the agent lifecycle. |
| **Agent Definition** | The complete, runtime-agnostic, declarative description of an agent. Must conform to agentpave-schema.json. Immutable after registration. |
| **Agent ID** | A UUID v4 string. Globally unique. Assigned at declaration time. Immutable. Never reused. The primary key for every agent in the Agent Registry. |
| **Agent Registry** | The authoritative, append-only store of all declared agents and their metadata. The single source of truth for agent identity in AgentPave. |
| **Agent Metadata** | The descriptive fields associated with an agent: `name`, `version`, `owner`, `purpose`, `domain`, `tags`, `llm_interface_version`, `created_at`. |
| **SPIFFE ID** | A Secure Production Identity Framework For Everyone identifier. Format: `spiffe://<trust-domain>/agentpave/<agent_id>/<instance_id>`. Issued at agent spawn. Cryptographically verifiable. Never reused across instances. |
| **Instance ID** | A UUID v4 generated fresh at every agent spawn (every transition to RUNNING). Included in the SPIFFE ID path to ensure two concurrent instances of the same agent have distinct cryptographic identities. |
| **Runtime Token** | A short-lived JWT bound to the agent's SPIFFE ID. Default TTL: 5 minutes. Scoped to a specific task. Used for Zero Trust authentication in all inter-system calls. |
| **Trust Domain** | The SPIFFE trust domain for the AgentPave deployment. Example: `agentpave.example.com`. Configured at deployment time via `AGENTKIT_TRUST_DOMAIN` environment variable. |
| **Agent Version** | A SemVer string (e.g. `1.2.0`) declaring the version of the agent's definition. Must be incremented whenever the agent definition changes in any way. |
| **Owner** | The entity responsible for this agent. Format: email address (user@example.com) or team path (eng/platform/agents). Used for governance, alerts, and retirement authorisation. |
| **Domain** | The functional domain this agent operates in. Format: lowercase, alphanumeric, hyphen-separated, max 64 chars. Examples: `finance`, `customer-support`, `data-engineering`. |
| **Declaration Time** | The exact UTC timestamp (ISO 8601) at which the AgentRecord was persisted to the registry. Immutable. |
| **definition_hash** | The SHA-256 hash of the RFC 8785 canonical JSON serialisation of the AgentDefinition. Used to detect tampering. Immutable after declaration. |

---

## 5. Data Models

All data models are canonical. The JSON Schema at `/schema/agentpave-schema.json` is authoritative — this prose describes it. Where conflict exists, the JSON Schema wins.

### 5.1 ReliabilityContract

```python
from pydantic import BaseModel, Field
from typing import Optional
from typing import Literal


class ReliabilityContract(BaseModel):
    """
    Declared per-agent reliability requirements.
    Mandatory in every AgentDefinition. Enforced at runtime by Dimension 6.
    """

    hallucination_tolerance: Literal["zero", "low", "medium", "high"] = Field(
        ...,
        description=(
            "Tolerance level for hallucination.\n"
            "  zero   — zero tolerance; every output must be grounded. "
                       "Use for medical, legal, financial domains.\n"
            "  low    — rare hallucination tolerated; grounding strongly recommended.\n"
            "  medium — occasional hallucination tolerated; grounding optional.\n"
            "  high   — hallucination tolerated; creative/exploratory tasks only."
        )
    )
    grounding_required: bool = Field(
        ...,
        description=(
            "Whether factual grounding (RAG or tool-based verification) is required "
            "for every LLM output. MUST be True when hallucination_tolerance is 'zero'."
        )
    )
    quality_threshold: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description=(
            "Minimum acceptable quality score [0.0–1.0]. "
            "Outputs scoring below this threshold raise "
            "AgentPave.ReliabilityContractViolationError."
        )
    )
    domain_notes: Optional[str] = Field(
        None,
        max_length=500,
        description="Optional human-readable domain-specific quality notes."
    )

    def model_post_init(self, __context) -> None:
        # Enforce: hallucination_tolerance=zero requires grounding_required=True
        if self.hallucination_tolerance == "zero" and not self.grounding_required:
            raise ValueError(
                "grounding_required must be True when hallucination_tolerance is 'zero'. "
                "Zero tolerance means every output must be grounded."
            )
```

### 5.2 TokenBudget

```python
class TokenBudget(BaseModel):
    """
    Declared token budget. All three fields must be consistent.
    Mandatory in every AgentDefinition. Enforced at runtime by Dimension 8.
    """

    per_task: int = Field(
        ...,
        gt=0,
        description="Maximum tokens per single task invocation."
    )
    per_day: int = Field(
        ...,
        gt=0,
        description="Maximum tokens consumed in a rolling 24-hour window."
    )
    per_lifetime: Optional[int] = Field(
        None,
        gt=0,
        description=(
            "Maximum tokens for the agent's entire lifetime. "
            "None means no lifetime cap — per_day is the binding constraint."
        )
    )

    def model_post_init(self, __context) -> None:
        # Cross-field invariant: per_task must not exceed per_day
        if self.per_task > self.per_day:
            raise ValueError(
                f"per_task ({self.per_task}) must not exceed per_day ({self.per_day}). "
                "A single task cannot consume more tokens than the daily budget."
            )
        # Cross-field invariant: per_day must not exceed per_lifetime if set
        if self.per_lifetime is not None and self.per_day > self.per_lifetime:
            raise ValueError(
                f"per_day ({self.per_day}) must not exceed per_lifetime ({self.per_lifetime})."
            )
```

### 5.3 AgentDefinition

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Annotated, Optional
import re

SEMVER_PATTERN = re.compile(
    r'^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)'
    r'(?:-((?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*)'
    r'(?:\.(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?' 
    r'(?:\+([0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?$'
)

OWNER_PATTERN = re.compile(
    r'^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$'   # email
    r'|^[a-zA-Z0-9_-]+(\/[a-zA-Z0-9_-]+)*$'                # team path
)

DOMAIN_PATTERN = re.compile(r'^[a-z0-9][a-z0-9-]{0,62}[a-z0-9]$|^[a-z0-9]$')

TAG_PATTERN = re.compile(r'^[a-z0-9][a-z0-9-]{0,30}[a-z0-9]$|^[a-z0-9]$')


class AgentDefinition(BaseModel):
    """
    The complete, declarative description of an agent.
    Runtime-agnostic. Validated against agentpave-schema.json.
    Immutable after registration.

    No undeclared fields are permitted (extra = 'forbid').
    """

    # --- Required identity fields ---
    name: str = Field(
        ...,
        min_length=1,
        max_length=128,
        description=(
            "Human-readable agent name. Unique within an owner+domain namespace. "
            "Does not need to be globally unique — Agent ID provides global uniqueness."
        )
    )
    version: str = Field(
        ...,
        description=(
            "SemVer version of this agent definition (e.g. '1.0.0'). "
            "Must be incremented on every change to the definition."
        )
    )
    owner: str = Field(
        ...,
        description=(
            "Email address or team path of the responsible party. "
            "Email format: user@example.com. "
            "Team path format: eng/platform/agents (slash-separated, no spaces)."
        )
    )
    purpose: str = Field(
        ...,
        min_length=10,
        max_length=1000,
        description=(
            "Plain-language description of what this agent does and why it exists. "
            "Must contain at least 5 words. Must be specific enough for a "
            "human reviewer to understand the agent's intent and scope."
        )
    )
    domain: str = Field(
        ...,
        description=(
            "Functional domain. Lowercase, alphanumeric, hyphen-separated, 1–64 chars. "
            "Must start and end with alphanumeric character. "
            "Examples: 'finance', 'customer-support', 'data-engineering'."
        )
    )
    llm_interface_version: str = Field(
        ...,
        description=(
            "SemVer version of the LLM interface contract this agent uses. "
            "Incrementing this version signals a potentially breaking change "
            "in expected LLM behaviour (e.g. a model upgrade)."
        )
    )

    # --- Optional descriptive fields ---
    tags: Annotated[list[str], Field(max_length=20)] = Field(
        default_factory=list,
        description=(
            "Optional list of tags for filtering and discovery. "
            "Maximum 20 tags. Each tag: lowercase, alphanumeric + hyphens, "
            "must start and end with alphanumeric character, max 32 chars."
        )
    )

    # --- Required contracts ---
    reliability_contract: ReliabilityContract = Field(
        ...,
        description="Declared reliability requirements. Mandatory. Enforced at runtime."
    )
    token_budget: TokenBudget = Field(
        ...,
        description="Declared token budget. Mandatory. No agent runs without a budget."
    )

    # --- Validators ---
    @field_validator("version", "llm_interface_version")
    @classmethod
    def validate_semver(cls, v: str) -> str:
        if not SEMVER_PATTERN.match(v):
            raise ValueError(
                f"'{v}' is not a valid SemVer string. "
                "Expected format: MAJOR.MINOR.PATCH (e.g. '1.0.0')."
            )
        return v

    @field_validator("owner")
    @classmethod
    def validate_owner(cls, v: str) -> str:
        if not OWNER_PATTERN.match(v):
            raise ValueError(
                f"'{v}' is not a valid owner. "
                "Must be an email (user@example.com) or team path (eng/platform/agents)."
            )
        return v

    @field_validator("domain")
    @classmethod
    def validate_domain(cls, v: str) -> str:
        if not DOMAIN_PATTERN.match(v):
            raise ValueError(
                f"'{v}' is not a valid domain. "
                "Must be lowercase, alphanumeric + hyphens, 1–64 chars, "
                "starting and ending with alphanumeric character."
            )
        return v

    @field_validator("tags")
    @classmethod
    def validate_tags(cls, v: list[str]) -> list[str]:
        for tag in v:
            if not TAG_PATTERN.match(tag):
                raise ValueError(
                    f"Tag '{tag}' is invalid. Tags must be lowercase, "
                    "alphanumeric with hyphens only, max 32 characters, "
                    "starting and ending with alphanumeric character."
                )
        return v

    @field_validator("purpose")
    @classmethod
    def validate_purpose(cls, v: str) -> str:
        word_count = len(v.split())
        if word_count < 5:
            raise ValueError(
                f"'purpose' must contain at least 5 words (got {word_count}). "
                "The purpose must be specific enough for a reviewer to understand "
                "the agent's intent."
            )
        return v

    model_config = {"extra": "forbid"}
```

### 5.4 LifecycleState

```python
from enum import Enum

class LifecycleState(str, Enum):
    """
    The six mutually exclusive states an agent can occupy.
    An agent occupies exactly one state at any point in time.
    State transitions are governed exclusively by Dimension 2 (Lifecycle).
    """
    DECLARED    = "DECLARED"    # Agent definition accepted, ID assigned, not yet registered
    REGISTERED  = "REGISTERED"  # Agent is in the registry and ready to be invoked
    RUNNING     = "RUNNING"     # Agent is actively executing a task
    PAUSED      = "PAUSED"      # Agent execution suspended, state preserved
    RESUMED     = "RESUMED"     # Transient — immediately transitions to RUNNING
    RETIRED     = "RETIRED"     # Terminal — agent will never run again
```

### 5.5 AgentRecord

```python
from pydantic import BaseModel, Field
from typing import Optional


class AgentRecord(BaseModel):
    """
    The complete record stored in the Agent Registry for a declared agent.

    Immutability rules:
      - agent_id: IMMUTABLE — never changes after creation
      - spiffe_id: IMMUTABLE — never changes after creation
      - definition: IMMUTABLE — never changes after creation
      - definition_hash: IMMUTABLE — never changes after creation
      - created_at: IMMUTABLE — never changes after creation
      - lifecycle_state: MUTABLE — updated only by Dimension 2 via AgentRegistry.update_state()
      - updated_at: MUTABLE — updated on every lifecycle_state change

    Implementation note: Use Pydantic's model_config with a custom __setattr__
    to enforce field-level immutability rather than model-level frozen=True,
    since lifecycle_state must remain mutable.
    """

    # --- Immutable fields ---
    agent_id: str = Field(
        ...,
        description="UUID v4. Assigned at declaration. Globally unique. IMMUTABLE."
    )
    spiffe_id: str = Field(
        ...,
        description=(
            "SPIFFE ID. Format: spiffe://<trust_domain>/agentpave/<agent_id>/<instance_id>. "
            "IMMUTABLE."
        )
    )
    definition: AgentDefinition = Field(
        ...,
        description="Complete Agent Definition as submitted at declaration. IMMUTABLE."
    )
    definition_hash: str = Field(
        ...,
        description=(
            "SHA-256 hash of the RFC 8785 canonical JSON serialisation of the definition. "
            "Used to detect tampering. IMMUTABLE. "
            "Hex-encoded lowercase string, 64 characters."
        )
    )
    created_at: str = Field(
        ...,
        description=(
            "ISO 8601 UTC timestamp of the moment AgentRecord was persisted. "
            "Format: '2026-06-09T12:00:00.000Z'. IMMUTABLE."
        )
    )

    # --- Mutable fields ---
    lifecycle_state: LifecycleState = Field(
        default=LifecycleState.DECLARED,
        description=(
            "Current lifecycle state. Updated only by Dimension 2 via "
            "AgentRegistry.update_state(). Initial value: DECLARED."
        )
    )
    updated_at: str = Field(
        ...,
        description=(
            "ISO 8601 UTC timestamp of the most recent lifecycle_state change. "
            "Set to created_at on initial record creation."
        )
    )

    # Immutability enforcement for identity fields
    _IMMUTABLE_FIELDS = frozenset({
        "agent_id", "spiffe_id", "definition", "definition_hash", "created_at"
    })

    def __setattr__(self, name: str, value) -> None:
        if name in self._IMMUTABLE_FIELDS and hasattr(self, name):
            raise AttributeError(
                f"Field '{name}' is immutable and cannot be modified after creation."
            )
        super().__setattr__(name, value)

    model_config = {"extra": "forbid"}
```

### 5.6 RuntimeToken

```python
from typing import Annotated


class RuntimeToken(BaseModel):
    """
    A short-lived JWT issued to an agent instance for Zero Trust authentication.
    Bound to the agent's SPIFFE ID and scoped to a specific task.
    Key-bound: the JWT is tied to the agent's private key.
    Without the key, a stolen JWT cannot be used (eliminates replay attacks).
    """
    jwt: str = Field(
        ...,
        description=(
            "Signed JWT string. Bound to agent's private key via RS256. "
            "Verified using the AgentPave trust domain's public key. "
            "Never log or transmit this value in plaintext."
        )
    )
    spiffe_id: str = Field(
        ...,
        description="The SPIFFE ID this token is bound to."
    )
    agent_id: str = Field(
        ...,
        description="UUID v4. The Agent ID this token is issued for."
    )
    task_id: str = Field(
        ...,
        description="UUID v4. The specific task this token is scoped to."
    )
    issued_at: str = Field(
        ...,
        description="ISO 8601 UTC. Token issuance time."
    )
    expires_at: str = Field(
        ...,
        description=(
            "ISO 8601 UTC. Token expiry time. "
            "Default TTL: 5 minutes from issued_at. "
            "Maximum TTL: 60 minutes. Minimum TTL: 1 minute."
        )
    )
    scope: list[str] = Field(
        ...,
        min_length=1,
        description=(
            "List of permitted actions. Narrowly scoped to this task's requirements. "
            "Format: '<resource_type>:<resource_name>:<permission>' "
            "Examples: ['tool:web_search:read', 'memory:episodic:write']"
        )
    )
    renewable: bool = Field(
        default=True,
        description=(
            "Whether this token may be renewed before expiry. "
            "Renewal produces: same identity, fresh TTL, same or narrower scope."
        )
    )
    renewal_count: Annotated[int, Field(ge=0, le=11)] = Field(
        default=0,
        description=(
            "Number of times this token has been renewed. "
            "Maximum: 11 renewals (1 hour total at 5-minute TTL). "
            "Renewal is refused when renewal_count == 11."
        )
    )

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

This is a complete, valid AgentDefinition and its resulting AgentRecord. Implementations MUST produce structurally identical output for this input.

### 6.1 Valid AgentDefinition (Python dict)

```python
valid_agent_definition = {
    "name": "customer-support-router",
    "version": "1.0.0",
    "owner": "platform-agents@example.com",
    "purpose": (
        "Routes incoming customer support requests to the correct specialised agent "
        "based on request type, urgency, and customer tier. "
        "Does not resolve requests directly."
    ),
    "domain": "customer-support",
    "llm_interface_version": "1.0.0",
    "tags": ["routing", "customer-support", "tier-aware"],
    "reliability_contract": {
        "hallucination_tolerance": "low",
        "grounding_required": False,
        "quality_threshold": 0.85,
        "domain_notes": "Routing decisions must match correct department 85%+ of the time."
    },
    "token_budget": {
        "per_task": 2000,
        "per_day": 500000,
        "per_lifetime": None
    }
}
```

### 6.2 Expected AgentRecord (structure)

```python
expected_agent_record_structure = {
    "agent_id": "<UUID v4>",                    # e.g. "f47ac10b-58cc-4372-a567-0e02b2c3d479"
    "spiffe_id": "spiffe://agentpave.local/agentpave/<agent_id>/<instance_id>",
    "definition": valid_agent_definition,        # exact copy of submitted definition
    "definition_hash": "<64-char lowercase hex SHA-256>",
    "lifecycle_state": "DECLARED",
    "created_at": "<ISO 8601 UTC>",             # e.g. "2026-06-09T12:00:00.000Z"
    "updated_at": "<ISO 8601 UTC>",             # same as created_at at declaration
}
```

### 6.3 Invalid AgentDefinition Examples (each must raise InvalidAgentDefinitionError)

```python
# Invalid: version not SemVer
{"name": "test", "version": "v1", ...}

# Invalid: per_task > per_day
{"token_budget": {"per_task": 10000, "per_day": 5000}, ...}

# Invalid: undeclared extra field
{"name": "test", "version": "1.0.0", "unknown_field": "value", ...}

# Invalid: domain contains uppercase
{"domain": "CustomerSupport", ...}

# Invalid: purpose too short (< 5 words)
{"purpose": "Does stuff", ...}

# Invalid: hallucination_tolerance=zero with grounding_required=False
{"reliability_contract": {"hallucination_tolerance": "zero", "grounding_required": False, ...}}
```

---

## 7. Interfaces & Contracts

### 7.1 AgentRegistry

```python
from abc import ABC, abstractmethod


class AgentRegistry(ABC):
    """
    The authoritative, append-only store of all AgentPave agents.

    Every conformant AgentPave implementation MUST provide a concrete
    implementation of this interface.

    Contract rules:
    - register() is the only operation that creates records.
      Records are never deleted.
    - update_state() is the only operation that modifies records.
      It modifies only lifecycle_state and updated_at.
    - All reads are consistent — they reflect the latest committed write.
    - The registry MUST be durable: data survives process restart,
      memory failure, and infrastructure fault.
    - Concurrent register() calls MUST be safe: no two records
      may receive the same agent_id under any concurrent load.
    """

    @abstractmethod
    def register(self, definition: AgentDefinition) -> AgentRecord:
        """
        Declare a new agent and assign it a UUID v4 Agent ID.

        Steps performed atomically (in order):
        1. Validate definition against AgentDefinition schema.
           Raise InvalidAgentDefinitionError on failure.
        2. Generate UUID v4 agent_id.
        3. Check uniqueness: if agent_id exists, generate a new one and retry once.
           If collision persists after retry, raise DuplicateAgentIDError.
        4. Generate instance_id (UUID v4).
        5. Construct SPIFFE ID:
           spiffe://<AGENTKIT_TRUST_DOMAIN>/agentpave/<agent_id>/<instance_id>
        6. Compute definition_hash:
           sha256(rfc8785_canonicalize(definition.model_dump())).hexdigest()
        7. Set created_at and updated_at to current UTC time (ISO 8601).
        8. Create AgentRecord with lifecycle_state=DECLARED.
        9. Persist AgentRecord to durable store under exclusive lock.
        10. Return the created AgentRecord.

        Raises:
            AgentPave.InvalidAgentDefinitionError: Definition fails schema validation.
                context = {"validation_errors": [list of str]}
            AgentPave.DuplicateAgentIDError: UUID v4 collision after one retry.
                context = {"agent_id": str}
                Note: Probability ~1 in 5.3×10^36 per attempt.
                      Retry once automatically before raising.
        """
        ...

    @abstractmethod
    def get(self, agent_id: str) -> AgentRecord:
        """
        Retrieve an agent record by Agent ID.

        Raises:
            AgentPave.AgentNotFoundError: agent_id not in registry.
                context = {"agent_id": agent_id}
            ValueError: agent_id is not a valid UUID v4 string.
        """
        ...

    @abstractmethod
    def exists(self, agent_id: str) -> bool:
        """
        Check whether an Agent ID exists. Returns False if not found.
        Never raises AgentNotFoundError.
        """
        ...

    @abstractmethod
    def update_state(
        self,
        agent_id: str,
        new_state: LifecycleState,
        reason: str
    ) -> AgentRecord:
        """
        Update the lifecycle_state of an agent.
        Called exclusively by Dimension 2 (Lifecycle).
        Not part of the public developer API.

        Raises:
            AgentPave.AgentNotFoundError: agent_id not in registry.
            AgentPave.RetiredAgentError: agent is already RETIRED.
                context = {"agent_id": str, "retired_at": str}
            AgentPave.InvalidStateTransitionError: transition is invalid per D2 rules.
                context = {"from_state": str, "to_state": str}
        """
        ...

    @abstractmethod
    def list_by_owner(self, owner: str) -> list[AgentRecord]:
        """
        List all agents for a given owner. Returns empty list if none found.
        Never raises AgentNotFoundError.
        """
        ...

    @abstractmethod
    def list_by_domain(self, domain: str) -> list[AgentRecord]:
        """
        List all agents in a given domain. Returns empty list if none found.
        Never raises AgentNotFoundError.
        """
        ...
```

### 7.2 IdentityService

```python
class IdentityService(ABC):
    """
    Manages cryptographic runtime identity for agents.
    Issues, renews, and validates SPIFFE IDs and Runtime Tokens.
    """

    @abstractmethod
    def issue_token(
        self,
        agent_id: str,
        task_id: str,
        scope: list[str],
        ttl_seconds: int = 300
    ) -> RuntimeToken:
        """
        Issue a short-lived JWT for a specific agent task.

        Args:
            agent_id: UUID v4. Must exist in registry. Must not be RETIRED.
            task_id: UUID v4. The task this token is scoped to.
            scope: Non-empty list of permitted action strings.
                   Format: '<resource_type>:<resource_name>:<permission>'
            ttl_seconds: Token lifetime. Range: [60, 3600]. Default: 300.

        Raises:
            AgentPave.AgentNotFoundError: agent_id not in registry.
            AgentPave.RetiredAgentError: agent is RETIRED.
            ValueError: scope is empty, or ttl_seconds outside [60, 3600].
        """
        ...

    @abstractmethod
    def renew_token(self, token: RuntimeToken) -> RuntimeToken:
        """
        Renew a Runtime Token before expiry.
        Produces: same SPIFFE ID, fresh TTL, same or narrower scope,
                  renewal_count incremented by 1.

        Raises:
            ValueError: Token expired, not renewable, or renewal_count >= 11.
            AgentPave.RetiredAgentError: Agent RETIRED since token was issued.
        """
        ...

    @abstractmethod
    def validate_token(self, token: RuntimeToken) -> bool:
        """
        Validate a Runtime Token.
        Checks: JWT signature, expiry, scope non-empty, agent still exists and not RETIRED.
        Returns False for all invalid states. Never raises.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 Agent Declaration

**THE SYSTEM SHALL** validate every submitted AgentDefinition against the AgentDefinition Pydantic schema before assigning an Agent ID. An invalid definition MUST NOT receive an Agent ID.

**THE SYSTEM SHALL** assign a UUID v4 Agent ID at declaration time using a cryptographically secure random generator (e.g. `uuid.uuid4()` in Python — which uses `os.urandom()`).

**THE SYSTEM SHALL** compute `definition_hash` as the SHA-256 hash of the RFC 8785 canonical JSON of the definition, hex-encoded in lowercase. This MUST be computed before the record is persisted.

**THE SYSTEM SHALL** set `created_at` and `updated_at` to the UTC timestamp of the moment the AgentRecord is written to the durable store — not the moment the declaration request is received.

**THE SYSTEM SHALL** set `lifecycle_state` to `DECLARED` on every new AgentRecord.

**WHEN** a declaration request contains a `name`+`version`+`owner` combination already present in the registry **THE SYSTEM SHALL** reject the request with `AgentPave.InvalidAgentDefinitionError` with `context = {"reason": "duplicate_name_version_owner"}`.

### 8.2 Agent ID Immutability

**THE SYSTEM SHALL NOT** permit modification of `agent_id`, `definition`, `definition_hash`, or `created_at` on any existing AgentRecord after creation.

**THE SYSTEM SHALL NOT** reuse an Agent ID — even after the agent it belonged to has been RETIRED.

**THE SYSTEM SHALL NOT** accept any operation that replaces a record in the Agent Registry — only append (register) and partial update (update_state) are permitted.

### 8.3 SPIFFE Identity

**THE SYSTEM SHALL** generate a SPIFFE ID at every agent spawn using this exact format:
```
spiffe://<AGENTKIT_TRUST_DOMAIN>/agentpave/<agent_id>/<instance_id>
```
Where `AGENTKIT_TRUST_DOMAIN` is read from the `AGENTKIT_TRUST_DOMAIN` environment variable, and `instance_id` is a UUID v4 generated fresh for each spawn.

**THE SYSTEM SHALL** issue a Runtime Token before an agent transitions to RUNNING state. No agent may enter RUNNING state without a valid, non-expired Runtime Token.

**WHEN** a Runtime Token's TTL expires during a long-running task **THE SYSTEM SHALL** renew the token (if `renewable=True` and `renewal_count < 11`) before the agent proceeds to its next step.

**WHEN** `renewal_count == 11` and the token has expired **THE SYSTEM SHALL** terminate the task, raise `AgentPave.BudgetExceededError` with `context = {"reason": "token_renewal_limit_reached"}`, and transition the agent to PAUSED state pending explicit restart.

### 8.4 Agent Registry Durability

**THE SYSTEM SHALL** hold an exclusive write lock during `register()` to prevent duplicate Agent IDs under concurrent load.

**THE SYSTEM SHALL** make the Agent Registry durable — persisted data MUST survive process restarts, memory failures, and infrastructure faults.

**THE SYSTEM SHALL** respond to `get()` calls with the most recently committed state of the AgentRecord.

---

## 9. Three-Tier Boundary System

### ALWAYS — Must always do these, no exceptions

- Validate every AgentDefinition against agentpave-schema.json before assigning an Agent ID
- Assign a UUID v4 Agent ID — never an integer, slug, or human-readable string
- Generate a SPIFFE ID for every declared agent
- Compute and store SHA-256 RFC-8785-canonical definition_hash at declaration time
- Set `created_at` at the moment of persistence — not request receipt
- Require `token_budget` and `reliability_contract` in every AgentDefinition
- Hold an exclusive write lock during register() for concurrent safety
- Append records to the registry — never overwrite or delete

### ASK FIRST — Must obtain explicit confirmation before doing these

- Retiring an agent that has active RUNNING tasks — confirm with the owner
- Registering an agent whose `name`+`domain` closely matches an existing agent (Levenshtein distance < 3 on name, same domain) — confirm this is not an accidental duplicate
- Issuing a Runtime Token with scope broader than the minimum required for the declared task

### NEVER — Must never do these under any circumstance

- Assign a non-UUID-v4 Agent ID
- Reuse an Agent ID after its agent is RETIRED
- Modify `agent_id`, `definition`, `definition_hash`, or `created_at` after creation
- Accept an AgentDefinition with undeclared extra fields
- Issue a Runtime Token with TTL > 3600 seconds (1 hour)
- Issue a Runtime Token to a RETIRED agent
- Delete or overwrite any record in the Agent Registry

---

## 10. Error Handling

All errors are `AgentPaveError` subclasses as defined in SPEC.md Section 7.

| Scenario | Error Type | Recoverable | Required context fields |
|---|---|---|---|
| Definition fails Pydantic schema validation | `AgentPave.InvalidAgentDefinitionError` | No — fix the definition | `validation_errors: list[str]` |
| name+version+owner combination already exists | `AgentPave.InvalidAgentDefinitionError` | No — bump version | `reason: "duplicate_name_version_owner"` |
| per_task > per_day in token_budget | `AgentPave.InvalidAgentDefinitionError` | No — fix the budget | `reason: "invalid_token_budget"` |
| Generated UUID v4 already exists in registry | `AgentPave.DuplicateAgentIDError` | Yes — retry once automatically | `agent_id: str` |
| agent_id not found in registry | `AgentPave.AgentNotFoundError` | No — check the caller | `agent_id: str` |
| Operation on a RETIRED agent | `AgentPave.RetiredAgentError` | No | `agent_id: str, retired_at: str` |
| Token renewal_count == 11 and token expired | `AgentPave.BudgetExceededError` | No — task must restart | `reason: "token_renewal_limit_reached"` |
| Attempted modification of immutable field | `AttributeError` (not AgentPaveError) | No — fix the caller | — |
| agent_id argument is not a valid UUID v4 | `ValueError` (not AgentPaveError) | No — fix the caller | — |

---

## 11. Acceptance Criteria

All criteria are binary pass/fail. No partial pass. All must pass before this dimension is complete.

```
Criteria ID:  D1-001
Stability:    Beta
Given:        The valid_agent_definition from Section 6.1 (all required fields, correct types)
When:         AgentRegistry.register(AgentDefinition(**valid_agent_definition)) is called
Then:         An AgentRecord is returned where:
              - agent_id is a valid UUID v4 (matches uuid4 regex)
              - lifecycle_state == LifecycleState.DECLARED
              - definition == valid_agent_definition (exact field match)
              - definition_hash == sha256(rfc8785_canonicalize(definition)).hexdigest()
              - created_at is a valid ISO 8601 UTC string
              - spiffe_id matches "spiffe://<trust_domain>/agentpave/<agent_id>/<uuid4>"
Pass:         All conditions above are True
Fail:         Any condition is False or any field is null/missing
Error raised: None

---

Criteria ID:  D1-002
Stability:    Beta
Given:        Registry pre-seeded with an AgentRecord having agent_id="test-collision-uuid"
              (inject via registry._store["test-collision-uuid"] = mock_record)
              AND the UUID generator is mocked to return "test-collision-uuid" on first call
              and a fresh UUID on the second call
When:         AgentRegistry.register(valid_definition) is called
Then:         The system retries UUID generation and returns a valid AgentRecord
              with the second-generated UUID
Pass:         AgentRecord returned, agent_id != "test-collision-uuid",
              agent_id is a valid UUID v4
Fail:         DuplicateAgentIDError raised (should only raise after 2 collisions),
              or first colliding UUID is used
Error raised: None on first collision (retry). AgentPave.DuplicateAgentIDError on second.

---

Criteria ID:  D1-003
Stability:    Beta
Given:        An AgentDefinition with version="not-semver" (invalid version)
When:         AgentRegistry.register(AgentDefinition(**invalid_definition)) is called
Then:         AgentPave.InvalidAgentDefinitionError is raised. No Agent ID is assigned.
Pass:         InvalidAgentDefinitionError raised,
              context["validation_errors"] is a non-empty list of strings,
              registry.exists() returns False for any new agent_id
Fail:         Agent ID assigned, or different error raised, or validation_errors is empty
Error raised: AgentPave.InvalidAgentDefinitionError

---

Criteria ID:  D1-004
Stability:    Beta
Given:        A registered agent with agent_id=<known_uuid>
When:         AgentRegistry.get(agent_id=<known_uuid>) is called
Then:         The complete AgentRecord is returned
Pass:         record.agent_id == <known_uuid>,
              record.definition_hash matches hash computed at registration
Fail:         Record not returned, or definition_hash differs from registration value
Error raised: None

---

Criteria ID:  D1-005
Stability:    Beta
Given:        No agent exists with agent_id="00000000-0000-0000-0000-000000000000"
When:         AgentRegistry.get(agent_id="00000000-0000-0000-0000-000000000000") is called
Then:         AgentPave.AgentNotFoundError is raised
Pass:         AgentNotFoundError raised,
              context["agent_id"] == "00000000-0000-0000-0000-000000000000"
Fail:         No error raised, None returned, or different error raised
Error raised: AgentPave.AgentNotFoundError

---

Criteria ID:  D1-006
Stability:    Beta
Given:        A registered AgentRecord with agent_id=<known_uuid>
When:         Code attempts: agent_record.agent_id = "different-uuid"
Then:         AttributeError is raised. agent_record.agent_id remains <known_uuid>.
Pass:         AttributeError raised, agent_record.agent_id == <known_uuid> (unchanged)
Fail:         agent_id is modified successfully
Error raised: AttributeError
Note:         AttributeError is the correct error here — immutability is enforced at the
              Python model level, not by AgentPave error handling.

---

Criteria ID:  D1-007
Stability:    Beta
Given:        A registered agent with agent_id=<known_uuid> and lifecycle_state != RETIRED
When:         IdentityService.issue_token(
                  agent_id=<known_uuid>,
                  task_id=<new_uuid4>,
                  scope=["tool:web_search:read"],
                  ttl_seconds=300
              ) is called
Then:         A RuntimeToken is returned where:
              - jwt is a non-null, non-empty string
              - agent_id == <known_uuid>
              - task_id == <new_uuid4>
              - (expires_at - issued_at) == 300 seconds
              - renewable == True
              - renewal_count == 0
              - scope == ["tool:web_search:read"]
Pass:         All conditions above are True, JWT verifiable with trust domain public key
Fail:         Any condition False, JWT not verifiable, or any field null
Error raised: None

---

Criteria ID:  D1-008
Stability:    Beta
Given:        An AgentDefinition with token_budget.per_task=10000, token_budget.per_day=5000
              (per_task > per_day — invalid)
When:         AgentRegistry.register(AgentDefinition(**invalid_budget_definition)) is called
Then:         AgentPave.InvalidAgentDefinitionError is raised
Pass:         InvalidAgentDefinitionError raised,
              context["reason"] == "invalid_token_budget"
Fail:         Agent registered, or different error raised
Error raised: AgentPave.InvalidAgentDefinitionError

---

Criteria ID:  D1-009
Stability:    Beta
Given:        A registered agent with lifecycle_state=RETIRED
When:         IdentityService.issue_token(agent_id=<retired_agent_id>, ...) is called
Then:         AgentPave.RetiredAgentError is raised
Pass:         RetiredAgentError raised,
              context["agent_id"] == <retired_agent_id>,
              context["retired_at"] is a valid ISO 8601 UTC string
Fail:         Token issued, or different error raised
Error raised: AgentPave.RetiredAgentError

---

Criteria ID:  D1-010
Stability:    Beta
Given:        An AgentDefinition dict with an extra undeclared field: {"unknown_field": "value"}
When:         AgentDefinition(**definition_with_extra_field) is instantiated
Then:         AgentPave.InvalidAgentDefinitionError is raised (wrapping Pydantic ValidationError)
Pass:         InvalidAgentDefinitionError raised,
              context["validation_errors"] mentions the undeclared field
Fail:         AgentDefinition created with extra field, or field silently ignored
Error raised: AgentPave.InvalidAgentDefinitionError

---

Criteria ID:  D1-011
Stability:    Beta
Given:        An AgentRegistry with a single-threaded durable backing store
When:         100 concurrent threads each call AgentRegistry.register(unique_definition)
              simultaneously
Then:         Exactly 100 unique AgentRecords are created, each with a distinct UUID v4
Pass:         len(set(record.agent_id for record in results)) == 100,
              all 100 register() calls return successfully,
              no DuplicateAgentIDError raised
Fail:         Any two records share an agent_id,
              or any register() call raises DuplicateAgentIDError,
              or fewer than 100 records are created
Error raised: None
```

---

## 12. Definition of Done

A dimension is complete if and only if every item below is checked. An autonomous agent MUST NOT proceed to Dimension 2 until all items are satisfied.

- [ ] `LifecycleState` enum implemented with all six states
- [ ] `ReliabilityContract` Pydantic model implemented with all fields and cross-field validator
- [ ] `TokenBudget` Pydantic model implemented with all fields and cross-field validators
- [ ] `AgentDefinition` Pydantic model implemented with all fields, validators, and `extra="forbid"`
- [ ] `AgentRecord` Pydantic model implemented with immutability enforcement for identity fields
- [ ] `RuntimeToken` Pydantic model implemented with `renewal_count` bounded [0, 11]
- [ ] `AgentRegistry` abstract interface implemented with all six method signatures and docstrings
- [ ] Concrete `AgentRegistry` implementation is durable (survives process restart — verified by test)
- [ ] Concurrent `register()` calls produce no duplicate Agent IDs (D1-011 passes)
- [ ] `IdentityService` abstract interface implemented with all three method signatures
- [ ] Concrete `IdentityService` implementation issues verifiable JWTs with correct TTL
- [ ] RFC 8785 canonical JSON used for `definition_hash` computation
- [ ] All 11 acceptance criteria pass: D1-001 through D1-011
- [ ] All relevant invariants verified: INV-001, INV-008, INV-013, INV-014, INV-015
- [ ] Declaration time p99 latency < 100ms (NFR from SPEC.md Section 8.1)
- [ ] All errors raised are `AgentPaveError` subclasses (except AttributeError and ValueError for model-level guards)

---

## 13. Task Breakdown

Ordered implementation tasks. Each independently implementable and testable.

```
Task D1-T1: Implement LifecycleState enum
  Criteria: INV-002
  Test: assert all six values present; assert LifecycleState("DECLARED") works

Task D1-T2: Implement ReliabilityContract Pydantic model
  Criteria: D1-001
  Test: valid contract; invalid tolerance; out-of-range threshold;
        hallucination_tolerance="zero" + grounding_required=False must fail

Task D1-T3: Implement TokenBudget Pydantic model
  Criteria: D1-008
  Test: valid budget; per_task > per_day must fail; per_day > per_lifetime must fail

Task D1-T4: Implement AgentDefinition Pydantic model
  Criteria: D1-001, D1-003, D1-010, INV-014
  Test: valid definition (Section 6.1); invalid version; extra field;
        invalid owner; invalid domain (uppercase); purpose < 5 words

Task D1-T5: Implement AgentRecord Pydantic model with immutability enforcement
  Criteria: D1-006
  Test: creation; attempt mutation of agent_id must raise AttributeError;
        lifecycle_state mutation must succeed

Task D1-T6: Implement RuntimeToken Pydantic model
  Criteria: D1-007
  Test: valid token; renewal_count=12 must fail; empty scope must fail

Task D1-T7: Implement AgentRegistry abstract interface
  Test: AgentRegistry cannot be instantiated directly (TypeError)

Task D1-T8: Implement in-memory AgentRegistry (for tests)
  Criteria: D1-001, D1-002, D1-003, D1-004, D1-005, D1-008, D1-010
  Test: all D1 acceptance criteria except D1-011

Task D1-T9: Implement durable AgentRegistry (for production)
  Note: backed by SQLite (dev) or external store (prod)
  Test: write record; restart process; read record back — must succeed (durability test)

Task D1-T10: Implement concurrent safety in AgentRegistry.register()
  Criteria: D1-011
  Test: 100 concurrent register() calls produce 100 unique agent_ids

Task D1-T11: Implement IdentityService abstract interface
  Test: IdentityService cannot be instantiated directly (TypeError)

Task D1-T12: Implement concrete IdentityService
  Criteria: D1-007, D1-009
  Test: issue token; validate token; renew token up to limit;
        renewal_count==11 must raise BudgetExceededError;
        RETIRED agent must raise RetiredAgentError;
        expired token must return False from validate_token

Task D1-T13: Implement definition_hash using RFC 8785 canonical JSON + SHA-256
  Criteria: D1-001
  Test: same definition always produces same hash;
        changing any field produces a different hash;
        hash is 64-char lowercase hex string

Task D1-T14: Run all 11 acceptance criteria and record in conformance-results.json
Task D1-T15: Verify all 5 relevant invariants: INV-001, INV-008, INV-013, INV-014, INV-015
Task D1-T16: Verify declaration time p99 < 100ms under 50 concurrent requests
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Sequential integer Agent IDs**
Using integers (1, 2, 3...) as Agent IDs breaks global uniqueness in distributed deployments, makes IDs guessable (security risk), and violates INV-001. Always use UUID v4.

**Anti-Pattern 2 — Mutable Agent IDs**
Allowing `agent_id` to change after declaration severs the traceability link between an agent and its audit trail, logs, and checkpoints. Any prior log entry becomes orphaned.

**Anti-Pattern 3 — Storing credentials in AgentDefinition**
AgentDefinition is stored in the registry and may be serialised and logged. API keys, passwords, and secrets MUST NEVER appear in an AgentDefinition. Reference a secrets manager: `{"secret_ref": "vault://path/to/secret"}`.

**Anti-Pattern 4 — Long-lived Runtime Tokens**
Issuing tokens with TTL > 3600 seconds creates a large attack window. Key-bound JWTs mean a stolen token without the private key is useless — but long TTLs still expand the blast radius of any key compromise.

**Anti-Pattern 5 — Soft-deleting Agent Records**
Adding a `deleted` or `is_active` flag creates ambiguity about what "exists" means. Use `LifecycleState.RETIRED` exclusively. The registry is append-only — records are never deleted.

**Anti-Pattern 6 — Silently accepting extra fields in AgentDefinition**
`extra = "allow"` means an agent definition written for a future version silently passes validation today. This creates invisible behaviour differences between agent versions. Always use `extra = "forbid"`.

**Anti-Pattern 7 — Sharing instance_id across spawns**
Reusing the same `instance_id` in the SPIFFE path across two spawns of the same agent means two concurrent instances share a cryptographic identity — breaking per-instance accountability and enabling privilege escalation.

**Anti-Pattern 8 — Non-canonical JSON for definition_hash**
Using `json.dumps(definition.dict())` without RFC 8785 canonicalisation produces different hashes for identical definitions depending on key order and floating-point representation. The hash will differ across Python versions and across languages — making cross-runtime hash verification impossible.

---

## 15. Reference Implementation Notes

### 15.1 AgentPave-LangGraph

- Use LangGraph's `SqliteSaver` (dev) or `PostgresSaver` (prod) as the durable registry backing store
- SPIFFE ID generation uses the `spiffe` Python library (pip: `spiffe`)
- JWT operations use `python-jose[cryptography]` with RS256 signing
- RFC 8785 canonical JSON uses `canonicaljson` library (pip: `canonicaljson`)
- Trust domain: `AGENTKIT_TRUST_DOMAIN` environment variable (default: `agentpave.local`)
- AgentDefinition validation should occur before any LangGraph graph is constructed

### 15.2 AgentPave-MAF

- Use in-memory store (dev) or Azure Cosmos DB (prod) as the durable registry backing store
- SPIFFE identity generation is independent of MAF's actor model — implement as a standalone service
- JWT operations: `System.IdentityModel.Tokens.Jwt` (.NET) or `python-jose` (Python)
- RFC 8785 canonical JSON: `rfc8785` Python library or `JsonCanonicalization` NuGet package (.NET)
- Trust domain: `AgentPave:TrustDomain` app setting (default: `agentpave.local`)

### 15.3 Known Divergences Between Implementations

| Behaviour | AgentPave-LangGraph | AgentPave-MAF | Conformance impact |
|---|---|---|---|
| Default registry backend | SqliteSaver | In-memory / Cosmos DB | None — observable behaviour identical |
| JWT library | python-jose | System.IdentityModel.Tokens.Jwt | None — JWT format identical |
| Trust domain config | Env var | App setting | None — value identical |
| Schema validation | Pydantic v2 | Pydantic v2 (Python) / System.Text.Json (.NET) | None — validation rules identical |

All divergences are implementation details. Observable behaviour (Agent IDs, records, tokens, errors) MUST be identical across both implementations.

---

## 16. Open Questions

| # | Question | Impact | Blocks Stable? |
|---|---|---|---|
| OQ-1 | Should AgentDefinition support `parent_agent_id` for agent derivation hierarchies? | Would enable agent genealogy — useful for multi-agent extension | No |
| OQ-2 | Should the Agent Registry expose a streaming API for registry change events? | Would enable reactive architectures | No |
| OQ-3 | Should the SPIFFE trust domain be per-deployment or per-agent? | Per-agent trust domains enable multi-tenancy but add complexity | Yes — decision needed before Stable |
| OQ-4 | Should `definition_hash` use SHA-256 or SHA-3-256? | SHA-3 is more collision-resistant; negligible practical difference at this scale | No |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec. Post-validation fixes applied: tags field type corrected, domain format validator added, renewal_count bounded [0,11], AgentRecord immutability pattern fixed, RFC 8785 canonical JSON referenced, D1-011 concurrency criterion added, canonical example added, purpose word count validator added |

---

*AgentPave Dimension 1 — Identity — v1.1*
