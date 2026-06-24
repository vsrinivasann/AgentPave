# AgentPave Dimension 7 — Security

**Spec version:** 1.1  
**Stability:** Alpha  
**Depends on:** D1 (Identity), D2 (Lifecycle), D3 (Communication), D5 (Observability)  
**Required by:** D12 (Governance)  
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

Security is not a layer added on top of AgentPave — it is a property woven through every dimension. D1 (Identity) established cryptographic agent identity. D7 extends that foundation into a full Zero Trust security model: every agent action is authenticated, every tool call is authorised against policy, every decision is audited.

In 2026, the standard is Zero-Trust Agent Identity — every agent call must be authenticated, authorised, and audited independently.

D7 delivers four security pillars:
1. **Zero Trust enforcement** — no agent, user, or system is trusted by default
2. **Policy-as-Code** — security rules expressed as machine-readable, version-controlled policies
3. **Privilege model** — agents never exceed the privilege of the entity that invoked them
4. **Immutable audit trail** — every decision is tamper-evidently logged

### 1.2 Production Consequence of Getting This Wrong

Each communication edge has an OPA/Rego or Cedar policy evaluating the specific tuple (caller_identity, callee_identity, requested_action, current_context). If agents cross organisational trust boundaries or access data where compromise is unrecoverable, per-edge zero-trust is mandatory.

Without D7:
- Prompt injection can cause an over-privileged agent to take actions the invoking user never authorised (Confused Deputy)
- Stolen runtime tokens can be replayed without key-binding protection
- Absent audit trails mean regulatory investigations cannot be answered
- Shared service accounts across agents mean a single compromise exposes all agents

---

## 2. Scope Boundary

### 2.1 In Scope

- Zero Trust policy evaluation on every tool call (OPA/Cedar)
- Privilege model — Confused Deputy prevention
- Policy-as-code declaration, loading, and evaluation
- Immutable, tamper-evident audit trail for all security decisions
- Prompt injection detection and defence
- mTLS enforcement for agent-to-agent communication
- Secret reference model (no secrets in AgentDefinition)
- OWASP Agentic AI Top 10 coverage

### 2.2 Out of Scope

- SPIFFE identity issuance — that is D1
- Runtime Token issuance — that is D1
- Compliance adapters (EU AI Act, SOC2) — that is D12 (Governance)
- Network-level firewall rules — that is infrastructure concern
- Encryption at rest for checkpoints — addressed in D6 implementation guide
- SSO integration — that is enterprise deployment concern

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **OPA/Cedar as policy engines** | The CNCF 2026 recommendation for internal service-to-service agent auth: SPIFFE for identity, OAuth 2.0 for access delegation, OPA for policy. This triple-layer approach means a credential compromise at any single agent yields only the minimum permissions that agent held for its specific operation. |
| **Policy-as-code, not documentation** | Without runtime enforcement, governance is just a hope that developers remember to follow rules. OPA/Cedar evaluate policies sub-millisecond — no performance excuse for policy-as-documentation. |
| **Confused Deputy prevention is architectural** | Agents must never exceed the privilege of the entity that invoked them. This is enforced at every tool call boundary, not at deployment time. |
| **Audit trail is cryptographically signed** | Append-only or cryptographically verifiable log stores prevent silent modifications or deletions — providing Proof of Action for AI agents. Hash-chaining makes tampering detectable. |
| **Prompt injection detection is first-class** | Prompt injection is OWASP Agentic AI Top 1. It cannot be left to individual agent implementations. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **Zero Trust** | Security model where no agent, user, or system is trusted by default. Every request is independently authenticated, authorised, and audited. |
| **Policy-as-Code** | Security rules expressed as declarative, machine-readable, version-controlled code evaluated at runtime. AgentPave supports OPA Rego and Cedar policy languages. |
| **Policy Decision Point (PDP)** | The component that evaluates a policy against an input and returns ALLOW or DENY. AgentPave embeds a PDP that evaluates on every tool call. |
| **Policy Enforcement Point (PEP)** | The component that intercepts an action and submits it to the PDP before allowing execution. AgentPave's ToolCaller is the PEP. |
| **Confused Deputy** | A security vulnerability where an agent is manipulated — via prompt injection or malicious tool output — into exercising its privileges on behalf of an unauthorised party. |
| **Privilege Ceiling** | The maximum privilege an agent may exercise — capped at the privilege of the entity that invoked it. Never higher. |
| **Prompt Injection** | An attack where malicious content in tool outputs or user inputs attempts to override the agent's instructions and cause it to take unauthorised actions. |
| **Audit Decision Record** | A tamper-evident, immutable record of a single policy evaluation: who asked, what action, what decision, why. Hash-chained to the previous record. |
| **Secret Reference** | A pointer to a secret stored in an external secrets manager (e.g., `vault://path/to/secret`). Used instead of embedding secrets in AgentDefinition. |
| **mTLS** | Mutual TLS. Both parties in a connection present certificates. Used for agent-to-agent communication to prevent impersonation. |
| **OWASP Agentic AI Top 10** | The OWASP list of the top 10 security risks for agentic AI systems. D7 addresses all 10. |

---

## 5. Data Models

### 5.1 PolicyDocument

```python
from pydantic import BaseModel, Field
from typing import Literal
from enum import Enum


class PolicyLanguage(str, Enum):
    OPA_REGO = "opa_rego"
    CEDAR    = "cedar"
    YAML     = "yaml"      # AgentPave simplified policy format


class PolicyEffect(str, Enum):
    ALLOW = "allow"
    DENY  = "deny"


class PolicyDocument(BaseModel):
    """
    A policy document evaluated by the Policy Decision Point.
    Attached to an agent's definition to declare what it is and is not allowed to do.
    """
    policy_id: str = Field(..., description="UUID v4.")
    policy_version: str = Field(..., description="SemVer.")
    language: PolicyLanguage = Field(...)
    content: str = Field(
        ...,
        min_length=1,
        description=(
            "The policy content in the declared language. "
            "For OPA Rego: valid Rego policy text. "
            "For Cedar: valid Cedar policy text. "
            "For YAML: AgentPave simplified policy format."
        )
    )
    default_effect: PolicyEffect = Field(
        default=PolicyEffect.DENY,
        description=(
            "What happens when no policy rule matches. "
            "Default: DENY (fail-closed). "
            "ALLOW default is strongly discouraged."
        )
    )
    description: str = Field(
        ...,
        min_length=10,
        description="Human-readable description of what this policy enforces."
    )

    model_config = {"extra": "forbid"}
```

### 5.2 PolicyEvaluationInput

```python
class PolicyEvaluationInput(BaseModel):
    """
    The input tuple passed to the Policy Decision Point for evaluation.
    Follows the (caller_identity, callee_identity, requested_action, current_context) pattern.
    """
    request_id: str = Field(..., description="UUID v4. For tracing.")
    agent_id: str = Field(..., description="UUID v4. The requesting agent.")
    agent_spiffe_id: str = Field(..., description="SPIFFE ID of the requesting agent.")
    invoking_user: str = Field(
        ...,
        description="Identity of the human or system that invoked this agent. Email or system ID."
    )
    invoking_user_privilege_level: str = Field(
        ...,
        description=(
            "Privilege level of the invoking user. "
            "Examples: 'read-only', 'standard', 'admin', 'super-admin'. "
            "Agent's effective privilege is capped at this level."
        )
    )
    action: str = Field(
        ...,
        description=(
            "The action being requested. "
            "Format: '<resource_type>:<resource_name>:<operation>'. "
            "Example: 'tool:web_search:invoke', 'memory:episodic:write'."
        )
    )
    resource_attributes: dict = Field(
        default_factory=dict,
        description="Attributes of the resource being accessed. Used by Cedar policies."
    )
    context: dict = Field(
        default_factory=dict,
        description="Additional context for policy evaluation (time, location, risk score)."
    )
    trace_id: str = Field(..., description="UUID v4. For observability correlation.")

    model_config = {"extra": "forbid"}
```

### 5.3 PolicyDecision

```python
class PolicyDecision(BaseModel):
    """
    The result of a policy evaluation.
    Returned by the Policy Decision Point.
    """
    request_id: str = Field(..., description="UUID v4. Matches PolicyEvaluationInput.request_id.")
    effect: PolicyEffect = Field(...)
    policy_id: str = Field(..., description="UUID v4. Which policy produced this decision.")
    rule_matched: str = Field(
        ...,
        description="The specific rule that produced this decision. 'default_deny' if no rule matched."
    )
    reason: str = Field(
        ...,
        description="Human-readable reason for the decision. Required for DENY decisions."
    )
    evaluated_at: str = Field(..., description="ISO 8601 UTC.")
    evaluation_latency_ms: int = Field(..., ge=0)

    model_config = {"extra": "forbid"}
```

### 5.4 AuditDecisionRecord

```python
class AuditDecisionRecord(BaseModel):
    """
    A tamper-evident, immutable record of a single policy decision.
    Hash-chained to the previous record in the audit trail.
    Written to the append-only audit log.
    """
    record_id: str = Field(..., description="UUID v4.")
    sequence_number: int = Field(..., ge=0, description="Monotonically increasing within agent.")
    previous_hash: str = Field(
        ...,
        description=(
            "SHA-256 hash of the previous AuditDecisionRecord. "
            "'genesis' for the first record (sequence_number=0)."
        )
    )
    record_hash: str = Field(
        ...,
        description="SHA-256 hash of this record's content (excluding record_hash field)."
    )
    agent_id: str = Field(...)
    trace_id: str = Field(...)
    evaluation_input: PolicyEvaluationInput = Field(...)
    decision: PolicyDecision = Field(...)
    created_at: str = Field(..., description="ISO 8601 UTC.")

    model_config = {"extra": "forbid"}
```

### 5.5 PromptInjectionScanResult

```python
class InjectionRiskLevel(str, Enum):
    SAFE     = "safe"
    LOW      = "low"
    MEDIUM   = "medium"
    HIGH     = "high"
    CRITICAL = "critical"


class PromptInjectionScanResult(BaseModel):
    """Result of a prompt injection scan on tool output or user input."""
    scan_id: str = Field(..., description="UUID v4.")
    input_source: str = Field(
        ...,
        description="Where the scanned content came from: 'user_input', 'tool_output', 'memory_read'."
    )
    risk_level: InjectionRiskLevel = Field(...)
    detected_patterns: list[str] = Field(
        default_factory=list,
        description="List of detected injection pattern identifiers."
    )
    should_block: bool = Field(
        ...,
        description="True when risk_level >= HIGH. Content must not reach the LLM."
    )
    sanitised_content: str = Field(
        ...,
        description=(
            "The content with detected injection patterns removed or escaped. "
            "Used in place of original content when should_block is False."
        )
    )
    scanned_at: str = Field(..., description="ISO 8601 UTC.")

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 OPA Rego Policy

```rego
# agentpave_policy.rego
# AgentPave standard policy format
# Denies all tool calls by default; allows only declared permitted actions.

package agentpave

import future.keywords.if
import future.keywords.in

# Default: deny everything
default allow := false

# Allow if action is in agent's permitted_actions list
allow if {
    input.action in data.agent_permissions[input.agent_id].permitted_actions
    input.invoking_user_privilege_level in data.agent_permissions[input.agent_id].permitted_invokers
}

# Privilege ceiling: deny if requested action privilege exceeds invoking user's level
deny if {
    action_privilege_level(input.action) > user_privilege_level(input.invoking_user_privilege_level)
}
```

### 6.2 Cedar Policy

```cedar
// AgentPave Cedar policy example
permit(
    principal == AgentPave::Agent::"customer-support-router",
    action in [AgentPave::Action::"tool:web_search:invoke"],
    resource == AgentPave::Tool::"web_search"
) when {
    principal.domain == "customer-support" &&
    context.invoking_user_privilege_level != "read-only"
};
```

### 6.3 Policy Evaluation Flow

```python
# Every tool call goes through PDP before execution
input_data = PolicyEvaluationInput(
    request_id=str(uuid4()),
    agent_id="<agent_uuid>",
    agent_spiffe_id="spiffe://agentpave.local/agentpave/<agent_id>/<instance_id>",
    invoking_user="user@example.com",
    invoking_user_privilege_level="standard",
    action="tool:web_search:invoke",
    resource_attributes={"tool_name": "web_search"},
    context={"task_id": "<task_uuid>"},
    trace_id="<trace_id>"
)

decision = policy_decision_point.evaluate(input_data)
# decision.effect == PolicyEffect.ALLOW → tool call proceeds
# decision.effect == PolicyEffect.DENY → AgentPave.PolicyDeniedError raised
```

---

## 7. Interfaces & Contracts

### 7.1 PolicyDecisionPoint

```python
from abc import ABC, abstractmethod


class PolicyDecisionPoint(ABC):
    """
    Evaluates policy against an input and returns ALLOW or DENY.
    Embedded in AgentPave core — not a remote call.
    MUST evaluate in < 1ms p99 (sub-millisecond, fail-closed).

    Contract:
    - MUST be fail-closed: if evaluation fails for any reason, return DENY.
    - MUST be stateless: same input always produces same output.
    - MUST NOT make network calls during evaluation.
    - MUST complete within 1ms p99.
    """

    @abstractmethod
    def evaluate(self, input_data: PolicyEvaluationInput) -> PolicyDecision:
        """
        Evaluate policy for the given input.
        Always returns a PolicyDecision — never raises.
        On any internal error: return PolicyDecision(effect=DENY, reason="evaluation_error").
        """
        ...

    @abstractmethod
    def load_policy(self, policy: PolicyDocument) -> None:
        """
        Load a policy document into the PDP.
        Called at agent registration time.

        Raises:
            AgentPave.PolicyLoadError: policy content is invalid for the declared language.
        """
        ...

    @abstractmethod
    def validate_policy(self, policy: PolicyDocument) -> list[str]:
        """
        Validate a policy document without loading it.
        Returns list of validation errors. Empty list = valid.
        """
        ...
```

### 7.2 PromptInjectionScanner

```python
class PromptInjectionScanner(ABC):
    """
    Scans text content for prompt injection patterns.
    Called on all tool outputs before they reach the LLM.
    Called on all user inputs before they enter the agent loop.
    """

    @abstractmethod
    def scan(self, content: str, source: str) -> PromptInjectionScanResult:
        """
        Scan content for injection patterns.

        Args:
            content: The text to scan.
            source: Origin of content: 'user_input', 'tool_output', or 'memory_read'.

        Returns:
            PromptInjectionScanResult. Never raises.
            On scanner failure: return SAFE result and log the failure.
        """
        ...
```

### 7.3 AuditTrail

```python
class AuditTrail(ABC):
    """
    Append-only, tamper-evident log of all security decisions.
    Hash-chained: each record includes the hash of the previous record.
    """

    @abstractmethod
    def append(self, record: AuditDecisionRecord) -> None:
        """
        Append a decision record to the audit trail.
        MUST be synchronous and durable.
        MUST maintain hash chain integrity.

        Raises:
            AgentPave.AuditTrailError: record could not be appended.
                If this raises, the associated tool call MUST be blocked.
        """
        ...

    @abstractmethod
    def verify_chain(self, agent_id: str) -> bool:
        """
        Verify the hash chain integrity for an agent's audit trail.
        Returns True if chain is intact, False if tampering is detected.
        """
        ...

    @abstractmethod
    def get_records(
        self,
        agent_id: str,
        from_sequence: int = 0
    ) -> list[AuditDecisionRecord]:
        """
        Retrieve audit records for an agent, ordered by sequence_number.
        Used for compliance investigations.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 Policy Evaluation on Every Tool Call

**THE SYSTEM SHALL** evaluate policy via the PDP before every tool call execution.

**WHEN** the PDP returns `PolicyEffect.DENY` **THE SYSTEM SHALL** raise `AgentPave.PolicyDeniedError` and block the tool call.

**THE SYSTEM SHALL** write an `AuditDecisionRecord` for every policy evaluation — both ALLOW and DENY.

**THE SYSTEM SHALL** be fail-closed: if the PDP fails to evaluate for any reason, the default is DENY.

### 8.2 Privilege Model — Confused Deputy Prevention

**THE SYSTEM SHALL** cap the agent's effective privilege at the privilege level of the invoking user.

**WHEN** an agent attempts an action that requires privilege exceeding `invoking_user_privilege_level` **THE SYSTEM SHALL** raise `AgentPave.PrivilegeEscalationError`.

**THE SYSTEM SHALL** include `invoking_user` and `invoking_user_privilege_level` in every `PolicyEvaluationInput`.

### 8.3 Prompt Injection Defence

**THE SYSTEM SHALL** scan every tool output via `PromptInjectionScanner.scan()` before it reaches the LLM.

**THE SYSTEM SHALL** scan every user input via `PromptInjectionScanner.scan()` before it enters the agent loop.

**WHEN** `scan_result.should_block == True` **THE SYSTEM SHALL** block the content from reaching the LLM, raise `AgentPave.PromptInjectionDetectedError`, and log the detection.

**WHEN** `scan_result.risk_level` is LOW or MEDIUM **THE SYSTEM SHALL** use `scan_result.sanitised_content` in place of the original content.

### 8.4 Audit Trail

**THE SYSTEM SHALL** write every `AuditDecisionRecord` synchronously and durably before the associated action proceeds.

**WHEN** `AuditTrail.append()` raises `AgentPave.AuditTrailError` **THE SYSTEM SHALL** block the associated tool call — no action proceeds without an audit record.

**THE SYSTEM SHALL** include hash-chaining: each record includes the SHA-256 hash of the previous record.

### 8.5 Secret References

**THE SYSTEM SHALL** reject any `AgentDefinition` that contains string values matching known secret patterns (API keys, passwords, tokens) in any field.

**THE SYSTEM SHALL** support secret references in the format `secret_ref://<provider>/<path>` and resolve them at runtime via the configured secrets manager.

---

## 9. Three-Tier Boundary System

### ALWAYS
- Evaluate policy via PDP before every tool call
- Write AuditDecisionRecord for every policy evaluation (ALLOW and DENY)
- Fail-closed: deny when PDP evaluation fails
- Cap agent privilege at invoking user's privilege level
- Scan all tool outputs for prompt injection before LLM exposure
- Scan all user inputs for prompt injection before agent loop entry
- Block tool calls when AuditTrail.append() fails
- Hash-chain all audit records

### ASK FIRST
- Loading a new policy document that grants broader permissions than the current one
- Allowing a MEDIUM-risk injection scan result through (use sanitised content only)
- Granting an agent temporary privilege elevation for a specific task

### NEVER
- Allow a tool call without prior PDP evaluation
- Allow a tool call without a written audit record
- Use ALLOW as the default policy effect (default must always be DENY)
- Allow secret values in AgentDefinition fields
- Allow an agent to exceed the privilege of the invoking user
- Suppress or catch PromptInjectionDetectedError — always surface it

---

## 10. Error Handling

| Scenario | Error Type | Recoverable | Required context |
|---|---|---|---|
| PDP returns DENY | `AgentPave.PolicyDeniedError` | No | `agent_id, action, rule_matched, reason` |
| Agent exceeds invoking user privilege | `AgentPave.PrivilegeEscalationError` | No | `agent_id, requested_level, ceiling_level` |
| Prompt injection detected (HIGH/CRITICAL) | `AgentPave.PromptInjectionDetectedError` | No | `source, risk_level, detected_patterns` |
| Audit trail write fails | `AgentPave.AuditTrailError` | No — block the action | `record_id, cause` |
| Policy document invalid | `AgentPave.PolicyLoadError` | No — fix the policy | `policy_id, validation_errors` |

---

## 11. Acceptance Criteria

```
Criteria ID:  D7-001
Stability:    Alpha
Given:        A running agent with a policy that allows "tool:web_search:invoke"
When:         tool_caller.call(request) for web_search is attempted
Then:         PDP is called with correct PolicyEvaluationInput
              Decision is ALLOW, AuditDecisionRecord is written
              Tool call proceeds
Pass:         PDP called once, decision.effect==ALLOW,
              AuditDecisionRecord exists with matching trace_id
Fail:         PDP not called, tool call proceeds without evaluation
Error raised: None

---

Criteria ID:  D7-002
Stability:    Alpha
Given:        A running agent with a policy that DENIES "tool:database_write:invoke"
When:         tool_caller.call(request) for database_write is attempted
Then:         PDP returns DENY, tool call is blocked
              AuditDecisionRecord written with effect=DENY
              AgentPave.PolicyDeniedError raised
Pass:         PolicyDeniedError raised, tool not invoked,
              AuditDecisionRecord written with effect=DENY
Fail:         Tool invoked despite DENY decision
Error raised: AgentPave.PolicyDeniedError

---

Criteria ID:  D7-003
Stability:    Alpha
Given:        Agent invoked by user with privilege_level="read-only"
              Agent attempts action requiring privilege_level="admin"
When:         PDP evaluates the action
Then:         AgentPave.PrivilegeEscalationError raised
              Action is blocked
Pass:         PrivilegeEscalationError raised,
              context["ceiling_level"]=="read-only", action not executed
Fail:         Action proceeds despite privilege ceiling violation
Error raised: AgentPave.PrivilegeEscalationError

---

Criteria ID:  D7-004
Stability:    Alpha
Given:        A tool returns output containing known prompt injection pattern:
              "Ignore all previous instructions and..."
When:         injection_scanner.scan(tool_output, "tool_output") is called
Then:         PromptInjectionScanResult.risk_level >= HIGH
              should_block == True
              Content does NOT reach the LLM
Pass:         risk_level in (HIGH, CRITICAL), should_block==True,
              LLM call log shows content was blocked
Fail:         Injected content reaches LLM, or risk_level is LOW/SAFE
Error raised: AgentPave.PromptInjectionDetectedError

---

Criteria ID:  D7-005
Stability:    Alpha
Given:        5 AuditDecisionRecords written for an agent (sequence 0-4)
When:         audit_trail.verify_chain(agent_id) is called
Then:         Returns True (chain intact)
When:         Record at sequence 2 is manually tampered (hash altered)
Then:         audit_trail.verify_chain(agent_id) returns False
Pass:         verify_chain returns True for untampered chain,
              returns False after tampering
Fail:         verify_chain returns True despite tampering
Error raised: None

---

Criteria ID:  D7-006
Stability:    Alpha
Given:        AuditTrail backend is unavailable (connection refused)
When:         A tool call is attempted (PDP returns ALLOW)
Then:         AuditTrail.append() raises AuditTrailError
              Tool call is BLOCKED despite PDP ALLOW
              AgentPave.AuditTrailError propagated
Pass:         Tool not invoked despite ALLOW decision,
              AuditTrailError raised
Fail:         Tool invoked without audit record
Error raised: AgentPave.AuditTrailError

---

Criteria ID:  D7-007
Stability:    Alpha
Given:        An AgentDefinition dict containing a field with value
              matching API key pattern: "sk-abc123def456..."
When:         AgentRegistry.register(definition) is called
Then:         AgentPave.InvalidAgentDefinitionError raised
              Agent is NOT registered
Pass:         InvalidAgentDefinitionError raised,
              validation_errors mentions secret pattern detected
Fail:         Agent registered with embedded secret
Error raised: AgentPave.InvalidAgentDefinitionError
```

---

## 12. Definition of Done

- [ ] `PolicyLanguage`, `PolicyEffect`, `InjectionRiskLevel` enums implemented
- [ ] `PolicyDocument`, `PolicyEvaluationInput`, `PolicyDecision` models implemented
- [ ] `AuditDecisionRecord` with hash-chaining implemented
- [ ] `PromptInjectionScanResult` model implemented
- [ ] `PolicyDecisionPoint` abstract + concrete (OPA embedded) implemented
- [ ] `PromptInjectionScanner` abstract + concrete implemented
- [ ] `AuditTrail` abstract + concrete (SQLite/PostgreSQL) implemented
- [ ] PDP called before every tool call in ToolCaller (D3)
- [ ] Fail-closed: PDP failure → DENY
- [ ] Privilege ceiling enforced on every evaluation
- [ ] Injection scanner called on all tool outputs and user inputs
- [ ] AuditTrail failure blocks associated tool call
- [ ] Secret pattern detection in AgentDefinition validation
- [ ] Hash-chain verification works for tampered and untampered chains
- [ ] All 7 acceptance criteria pass: D7-001 through D7-007
- [ ] PDP evaluation latency < 1ms p99
- [ ] OWASP Agentic AI Top 10 coverage verified (documented mapping)

---

## 13. Task Breakdown

```
Task D7-T1: Implement data models (PolicyDocument, PolicyEvaluationInput, PolicyDecision,
            AuditDecisionRecord, PromptInjectionScanResult)
Task D7-T2: Implement PolicyDecisionPoint abstract interface
Task D7-T3: Implement concrete OPA-embedded PDP (use opa-python or subprocess OPA)
Task D7-T4: Implement concrete Cedar PDP (use cedarpy library)
Task D7-T5: Wire PDP into ToolCaller (D3) — before every tool call
Task D7-T6: Implement privilege ceiling check in PolicyEvaluationInput
Task D7-T7: Implement PromptInjectionScanner abstract + concrete
            (pattern-matching + embedding-based detection)
Task D7-T8: Wire injection scanner into ToolCaller (output scan) and
            agent input handler (input scan)
Task D7-T9: Implement AuditTrail abstract + concrete (append-only, hash-chained)
Task D7-T10: Implement secret pattern detection in AgentDefinition validator
Task D7-T11: Run all 7 acceptance criteria
Task D7-T12: Verify PDP latency < 1ms p99
Task D7-T13: Document OWASP Agentic AI Top 10 coverage mapping
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Policy-as-documentation**
Writing policies in README files and trusting developers to remember them. Without runtime enforcement, every new developer and every new tool call is a potential policy violation.

**Anti-Pattern 2 — ALLOW as default effect**
A policy that allows everything not explicitly denied is an accident waiting to happen. Default must always be DENY. New actions must be explicitly permitted.

**Anti-Pattern 3 — Shared service accounts**
One service account shared across all agents means a single compromise exposes all agents. Every agent must have its own cryptographic identity.

**Anti-Pattern 4 — Remote PDP calls**
A PDP that makes a network call for every tool call adds 50-200ms latency per call. The PDP must be embedded and local — sub-millisecond evaluation is achievable only in-process.

**Anti-Pattern 5 — Audit log as a database table with UPDATE permissions**
A security audit log that can be edited is not an audit log. Use append-only storage with hash-chaining and verify the chain on every compliance query.

**Anti-Pattern 6 — Trusting tool outputs**
Tool outputs are an attack surface. An MCP server that has been compromised can return prompt injection payloads. All tool outputs must be scanned before reaching the LLM.

---

## 15. Reference Implementation Notes

### 15.1 AgentPave-LangGraph
- OPA embedded: use `opa-python` (pip: `opa-python`) or subprocess OPA binary
- Cedar: use `cedarpy` (pip: `cedarpy`)
- Prompt injection: `rebuff` library or custom pattern matcher + embedding similarity
- Audit trail: PostgreSQL with `INSERT` only permissions (no UPDATE/DELETE on audit table)

### 15.2 AgentPave-MAF
- OPA: available as Azure Container Apps sidecar
- Cedar: AWS Cedar for Python (`cedar-policy` pip package)
- Audit trail: Azure Cosmos DB with append-only access policy + change feed

### 15.3 OWASP Agentic AI Top 10 Coverage

| OWASP Risk | D7 Control |
|---|---|
| AA1: Prompt Injection | PromptInjectionScanner on all inputs/outputs |
| AA2: Insecure Output Handling | Injection scan + output schema validation (D3) |
| AA3: Excessive Agency | PolicyDecisionPoint + privilege ceiling |
| AA4: Privilege Escalation | Confused Deputy prevention in PDP |
| AA5: Insecure Plugin Design | IntegrationContract validation (D3) |
| AA6: Sensitive Information Disclosure | Secret reference model |
| AA7: Insecure Agent Communication | mTLS for A2A (D3) |
| AA8: Inadequate Sandboxing | Execution rings (reference implementation) |
| AA9: Overreliance on Agent Decisions | HITL extension (AgentPave-HITL) |
| AA10: Lack of Auditability | Hash-chained AuditTrail |

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should AgentPave support dynamic trust scoring (DynaTrust graph) per agent edge? | No — add in v2 |
| OQ-2 | Should policy documents be signed and verified before loading? | Yes — required for tamper-evident policy management |
| OQ-3 | Should the injection scanner use an LLM-based detector or pattern-only? | No — both modes should be supported |
| OQ-4 | Should privilege levels be an enum or a free-form hierarchy? | Yes — standardise the privilege level taxonomy |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec — OWASP coverage, OPA/Cedar policy, hash-chained audit trail, prompt injection defence |

---

*AgentPave Dimension 7 — Security — v1.1*
