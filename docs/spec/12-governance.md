# AgentKit Dimension 12 — Governance

**Spec version:** 1.1  
**Stability:** Alpha  
**Depends on:** D1–D7 (MVP + Security)  
**Required by:** Nothing in Release 3  
**Owner:** AgentKit Core  
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

Gartner predicts 40%+ of agentic AI projects will be cancelled by end of 2027 due to governance and ROI failures. D12 exists to prevent AgentKit from being one of them.

Governance is the operating system that decides which agents are approved, which data they can touch, which models they can call, where the workload runs, and how every action is logged so the organisation can answer regulator and auditor questions on demand.

D12 delivers four capabilities:
1. **Agent approval workflows** — no agent deploys without the right approvals
2. **Compliance adapters** — EU AI Act, SOC2, HIPAA, GDPR, ISO 42001
3. **Human oversight interface** — mandatory for high-risk agent actions (EU AI Act Article 14)
4. **Governance audit evidence** — machine-readable compliance evidence for auditors

### 1.2 Production Consequence of Getting This Wrong

EU AI Act high-risk obligations — Articles 8–17, 26, 27, and 73 — activate August 2, 2026. Non-compliance can result in fines up to €35 million or 7% of global annual turnover. Retrofitting governance after deployment averages $2–4M per high-risk system. Build it right first.

---

## 2. Scope Boundary

### 2.1 In Scope

- Agent risk classification (EU AI Act risk tiers)
- Agent approval workflow (declaration → review → approved → deployed)
- Compliance adapter interface (EU AI Act, SOC2, HIPAA, GDPR, ISO 42001)
- Human oversight interface for high-risk agent actions
- Governance audit evidence generation
- Data residency declaration and enforcement hooks
- Incident reporting hooks (EU AI Act Article 73: 15-day serious incident reporting)

### 2.2 Out of Scope

- GRC platform integration (OneTrust, ServiceNow etc.) — these are pluggable adapters
- Legal interpretation of regulations — that is the organisation's legal team
- Data encryption at rest — that is D7 (Security)
- Billing and financial reporting — that is Economics (D8)

---

## 3. Prior Decisions

| Decision | Rationale |
|---|---|
| **Risk classification is mandatory** | EU AI Act uses a risk-based approach. Every AgentKit agent must declare its risk tier. Unclassified agents cannot be deployed. |
| **Approval workflow is pre-deployment, not post** | Retrofitting governance costs 4–5x more. Pre-deployment approval gates prevent costly remediation. |
| **Compliance adapters are pluggable** | No organisation faces a single regulatory framework. Adapters allow mixing EU AI Act + GDPR + HIPAA without writing bespoke code. |
| **Human oversight is Article 14 compliant** | EU AI Act Article 14 mandates human oversight for high-risk AI systems. AgentKit provides the interface; organisations configure the threshold. |
| **Audit evidence is machine-readable** | Human-readable compliance documents satisfy auditors; machine-readable evidence enables automated compliance reporting and continuous monitoring. |

---

## 4. Definitions

| Term | Definition |
|---|---|
| **EU AI Act Risk Tier** | Classification of an AI system under EU AI Act: UNACCEPTABLE (prohibited), HIGH_RISK (Articles 8–17 apply), LIMITED_RISK (transparency obligations), MINIMAL_RISK (no specific obligations). |
| **Agent Approval Workflow** | The formal process for reviewing and approving an agent definition before deployment: DRAFT → UNDER_REVIEW → APPROVED → DEPLOYED or REJECTED. |
| **Compliance Adapter** | A module that maps AgentKit governance data to the evidence format required by a specific regulation (EU AI Act, SOC2, HIPAA, GDPR, ISO 42001). |
| **Human Oversight Interface** | A structured request for human review of a high-risk agent action before it executes. Required for HIGH_RISK agents under EU AI Act Article 14. |
| **Governance Audit Evidence** | Machine-readable records demonstrating compliance: agent risk classification, approval history, oversight invocations, incident reports. |
| **Data Residency** | The requirement that data be processed and stored within a specific geographic region. |
| **Serious Incident** | Under EU AI Act Article 73: an incident causing death, serious harm, property damage, or fundamental rights violations. Must be reported within 15 days. |

---

## 5. Data Models

### 5.1 AgentRiskClassification

```python
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum


class EUAIActRiskTier(str, Enum):
    UNACCEPTABLE = "unacceptable"   # Prohibited — cannot be deployed
    HIGH_RISK    = "high_risk"      # Articles 8-17 apply — approval required
    LIMITED_RISK = "limited_risk"   # Transparency obligations only
    MINIMAL_RISK = "minimal_risk"   # No specific EU AI Act obligations


class AgentRiskClassification(BaseModel):
    """Risk classification for an agent. Required before deployment."""
    agent_id: str = Field(...)
    eu_ai_act_tier: EUAIActRiskTier = Field(...)
    classification_rationale: str = Field(
        ...,
        min_length=50,
        description="Detailed rationale for the risk classification. Minimum 50 characters."
    )
    high_risk_category: Optional[str] = Field(
        None,
        description=(
            "If eu_ai_act_tier=HIGH_RISK: the specific Annex III category. "
            "Examples: 'biometric_identification', 'critical_infrastructure', "
            "'employment_decisions', 'essential_services', 'law_enforcement'."
        )
    )
    classified_by: str = Field(..., description="Email of person who performed classification.")
    classified_at: str = Field(..., description="ISO 8601 UTC.")
    review_required_by: Optional[str] = Field(
        None,
        description="ISO 8601 date by which this classification must be reviewed."
    )

    model_config = {"extra": "forbid"}
```

### 5.2 ApprovalWorkflow

```python
class ApprovalStatus(str, Enum):
    DRAFT        = "draft"
    UNDER_REVIEW = "under_review"
    APPROVED     = "approved"
    REJECTED     = "rejected"
    DEPLOYED     = "deployed"


class ApprovalRecord(BaseModel):
    """A single step in the approval workflow."""
    record_id: str = Field(..., description="UUID v4.")
    agent_id: str = Field(...)
    status: ApprovalStatus = Field(...)
    reviewer: str = Field(..., description="Email of reviewer.")
    decision: Optional[str] = Field(None, description="APPROVED or REJECTED.")
    comments: Optional[str] = Field(None, description="Reviewer comments.")
    conditions: list[str] = Field(
        default_factory=list,
        description="Conditions attached to approval (if any)."
    )
    decided_at: str = Field(..., description="ISO 8601 UTC.")

    model_config = {"extra": "forbid"}
```

### 5.3 ComplianceEvidence

```python
class ComplianceFramework(str, Enum):
    EU_AI_ACT  = "eu_ai_act"
    SOC2       = "soc2"
    HIPAA      = "hipaa"
    GDPR       = "gdpr"
    ISO_42001  = "iso_42001"
    NIST_AI_RMF = "nist_ai_rmf"


class ComplianceEvidenceRecord(BaseModel):
    """A single piece of compliance evidence."""
    record_id: str = Field(..., description="UUID v4.")
    agent_id: str = Field(...)
    framework: ComplianceFramework = Field(...)
    control_id: str = Field(
        ...,
        description=(
            "The specific control this evidence satisfies. "
            "Examples: 'EU_AI_ACT.Article14', 'SOC2.CC6.1', 'HIPAA.164.312'"
        )
    )
    evidence_type: str = Field(
        ...,
        description="Type of evidence: 'audit_log', 'approval_record', 'test_result', 'policy'."
    )
    evidence_payload: dict = Field(...)
    generated_at: str = Field(..., description="ISO 8601 UTC.")
    valid_until: Optional[str] = Field(None, description="ISO 8601 UTC. When this evidence expires.")

    model_config = {"extra": "forbid"}
```

### 5.4 IncidentReport

```python
class IncidentSeverity(str, Enum):
    MINOR    = "minor"
    MODERATE = "moderate"
    SERIOUS  = "serious"   # EU AI Act Article 73: 15-day reporting required
    CRITICAL = "critical"  # Immediate notification required


class IncidentReport(BaseModel):
    """Incident report for EU AI Act Article 73 compliance."""
    report_id: str = Field(..., description="UUID v4.")
    agent_id: str = Field(...)
    severity: IncidentSeverity = Field(...)
    description: str = Field(..., min_length=50)
    affected_users: Optional[int] = Field(None, ge=0)
    harm_type: Optional[str] = Field(
        None,
        description=(
            "Type of harm: 'personal_injury', 'property_damage', "
            "'fundamental_rights_violation', 'other'."
        )
    )
    detected_at: str = Field(..., description="ISO 8601 UTC.")
    reported_to_authority_at: Optional[str] = Field(
        None,
        description="ISO 8601 UTC. When reported to regulatory authority (if required)."
    )
    reporting_deadline: Optional[str] = Field(
        None,
        description=(
            "ISO 8601 date. For SERIOUS incidents: detected_at + 15 days. "
            "Computed automatically."
        )
    )
    resolution_at: Optional[str] = Field(None, description="ISO 8601 UTC. When resolved.")

    model_config = {"extra": "forbid"}
```

---

## 6. Canonical Example

### 6.1 Risk Classification

```python
classification = AgentRiskClassification(
    agent_id="<agent_uuid>",
    eu_ai_act_tier=EUAIActRiskTier.HIGH_RISK,
    classification_rationale=(
        "This agent makes employment screening decisions by analysing CVs and "
        "ranking candidates. It falls under EU AI Act Annex III category "
        "'employment, workers management and access to self-employment' "
        "as it is used for recruitment and selection of natural persons."
    ),
    high_risk_category="employment_decisions",
    classified_by="dpo@example.com",
    classified_at="2026-06-09T12:00:00Z",
    review_required_by="2027-06-09"
)
```

### 6.2 Approval Workflow

```python
# Draft → Under Review → Approved → Deployed
# HIGH_RISK agents require approval before deployment

# 1. Submit for review
governance_manager.submit_for_review(agent_id, submitter="developer@example.com")

# 2. Reviewer approves
governance_manager.approve(
    agent_id,
    reviewer="compliance-officer@example.com",
    conditions=["Must use grounded outputs only", "Human oversight enabled"]
)

# 3. Deploy (only possible after APPROVED status)
governance_manager.mark_deployed(agent_id)
```

---

## 7. Interfaces & Contracts

### 7.1 GovernanceManager

```python
from abc import ABC, abstractmethod


class GovernanceManager(ABC):
    """
    Manages the full governance lifecycle for AgentKit agents.
    Attaches at extension point: governance.policy
    """

    @abstractmethod
    def classify_risk(self, classification: AgentRiskClassification) -> None:
        """
        Record risk classification for an agent.
        Required before approval workflow can start for HIGH_RISK agents.

        Raises:
            AgentKit.GovernanceError: agent is UNACCEPTABLE risk tier (prohibited).
        """
        ...

    @abstractmethod
    def submit_for_review(self, agent_id: str, submitter: str) -> ApprovalRecord:
        """
        Submit an agent definition for governance review.
        Transitions agent approval status: DRAFT → UNDER_REVIEW.
        """
        ...

    @abstractmethod
    def approve(
        self,
        agent_id: str,
        reviewer: str,
        conditions: list[str] = None,
        comments: str = None
    ) -> ApprovalRecord:
        """
        Approve an agent for deployment.
        UNDER_REVIEW → APPROVED.
        HIGH_RISK agents must have a risk classification before approval.
        """
        ...

    @abstractmethod
    def reject(self, agent_id: str, reviewer: str, reason: str) -> ApprovalRecord:
        """
        Reject an agent definition.
        UNDER_REVIEW → REJECTED.
        """
        ...

    @abstractmethod
    def can_deploy(self, agent_id: str) -> bool:
        """
        Returns True only if the agent is APPROVED and all conditions are met.
        HIGH_RISK agents: also checks that risk classification is current.
        """
        ...

    @abstractmethod
    def generate_evidence(
        self,
        agent_id: str,
        framework: ComplianceFramework
    ) -> list[ComplianceEvidenceRecord]:
        """
        Generate compliance evidence for a specific framework.
        Assembles evidence from: risk classification, approval records,
        audit trail (D7), observability data (D5).
        """
        ...

    @abstractmethod
    def report_incident(self, report: IncidentReport) -> IncidentReport:
        """
        Record an incident.
        For SERIOUS incidents: automatically computes reporting_deadline = detected_at + 15 days.
        For CRITICAL incidents: raises immediate alert via AgentObserver.
        """
        ...
```

---

## 8. Behaviour Specification

### 8.1 Deployment Gate

**THE SYSTEM SHALL** block deployment of any agent where `can_deploy(agent_id)` returns False.

**WHEN** `eu_ai_act_tier == UNACCEPTABLE` **THE SYSTEM SHALL** raise `AgentKit.GovernanceError` and prevent any lifecycle transition beyond DECLARED.

**WHEN** `eu_ai_act_tier == HIGH_RISK` and no approved risk classification exists **THE SYSTEM SHALL** block deployment with `AgentKit.GovernanceError`.

### 8.2 Human Oversight (Article 14)

**WHEN** an agent with `eu_ai_act_tier == HIGH_RISK` is about to execute an action flagged for human oversight **THE SYSTEM SHALL** invoke the HITL extension (AgentKit-HITL) before proceeding.

**THE SYSTEM SHALL** record every human oversight invocation as a `ComplianceEvidenceRecord` with `framework=EU_AI_ACT` and `control_id='Article14'`.

### 8.3 Incident Reporting

**WHEN** an `IncidentReport` with `severity=SERIOUS` is created **THE SYSTEM SHALL** automatically compute `reporting_deadline = detected_at + 15 days`.

**WHEN** `reporting_deadline` is within 48 hours and `reported_to_authority_at` is null **THE SYSTEM SHALL** emit a critical alert via `AgentObserver`.

### 8.4 Evidence Generation

**THE SYSTEM SHALL** produce machine-readable `ComplianceEvidenceRecord` objects for all declared frameworks.

**THE SYSTEM SHALL** include in EU AI Act evidence: risk classification, approval records, human oversight invocations, incident reports, and accuracy metrics from ReliabilityEnforcer (D6).

---

## 9. Three-Tier Boundary System

### ALWAYS
- Block deployment of UNACCEPTABLE risk agents
- Require risk classification for HIGH_RISK agents before approval
- Record all approval workflow transitions
- Generate ComplianceEvidenceRecord for every human oversight invocation
- Compute 15-day reporting deadline for SERIOUS incidents

### ASK FIRST
- Approving an agent with unresolved approval conditions
- Deploying an agent whose risk classification is past review_required_by date
- Bypassing human oversight for a HIGH_RISK agent action

### NEVER
- Allow UNACCEPTABLE risk agents to transition beyond DECLARED
- Allow HIGH_RISK agents to deploy without approval
- Allow SERIOUS incidents to pass the 15-day reporting deadline without alert
- Delete or modify compliance evidence records

---

## 10. Error Handling

| Scenario | Error Type | Recoverable | Required context |
|---|---|---|---|
| UNACCEPTABLE risk agent attempts any operation | `AgentKit.GovernanceError` | No | `agent_id, eu_ai_act_tier` |
| HIGH_RISK agent attempts deployment without approval | `AgentKit.GovernanceError` | No — get approval first | `agent_id, approval_status` |
| Approval attempted without risk classification | `AgentKit.GovernanceError` | No — classify first | `agent_id` |

---

## 11. Acceptance Criteria

```
Criteria ID:  D12-001
Stability:    Alpha
Given:        Agent classified as UNACCEPTABLE risk tier
When:         Any lifecycle transition beyond DECLARED is attempted
Then:         AgentKit.GovernanceError raised
              Agent remains in DECLARED state
Pass:         GovernanceError raised, lifecycle_state == DECLARED
Fail:         Agent transitions despite UNACCEPTABLE classification
Error raised: AgentKit.GovernanceError

---

Criteria ID:  D12-002
Stability:    Alpha
Given:        HIGH_RISK agent with no risk classification
When:         governance_manager.can_deploy(agent_id) is called
Then:         Returns False
              Deployment blocked
Pass:         can_deploy returns False, deployment raises GovernanceError
Fail:         can_deploy returns True without classification
Error raised: AgentKit.GovernanceError (if deployment attempted)

---

Criteria ID:  D12-003
Stability:    Alpha
Given:        SERIOUS incident detected at "2026-06-09T12:00:00Z"
When:         governance_manager.report_incident(report) is called
Then:         IncidentReport returned with:
              reporting_deadline == "2026-06-24T12:00:00Z" (detected_at + 15 days)
Pass:         reporting_deadline == detected_at + 15 calendar days
Fail:         reporting_deadline not computed or wrong date
Error raised: None

---

Criteria ID:  D12-004
Stability:    Alpha
Given:        Approved HIGH_RISK agent that has executed 3 tasks with human oversight
When:         governance_manager.generate_evidence(agent_id, ComplianceFramework.EU_AI_ACT)
Then:         Returns ComplianceEvidenceRecords including:
              - Risk classification record (control_id: 'Article9')
              - Approval record (control_id: 'Article26')
              - 3 human oversight records (control_id: 'Article14')
Pass:         All 3 evidence types present with correct control_ids
Fail:         Missing evidence records or wrong control_ids
Error raised: None
```

---

## 12. Definition of Done

- [ ] `EUAIActRiskTier`, `ApprovalStatus`, `ComplianceFramework`, `IncidentSeverity` enums implemented
- [ ] `AgentRiskClassification`, `ApprovalRecord`, `ComplianceEvidenceRecord`, `IncidentReport` models implemented
- [ ] `GovernanceManager` abstract + concrete implemented
- [ ] Deployment gate blocks UNACCEPTABLE and unapproved HIGH_RISK agents
- [ ] 15-day incident reporting deadline computed for SERIOUS incidents
- [ ] EU AI Act evidence generation covers Articles 9, 14, 26, 73
- [ ] All 4 acceptance criteria pass: D12-001 through D12-004

---

## 13. Task Breakdown

```
Task D12-T1: Implement all enums and data models
Task D12-T2: Implement GovernanceManager abstract + concrete
Task D12-T3: Implement deployment gate integration (wire into LifecycleManager D2)
Task D12-T4: Implement human oversight integration (wire into HITL extension)
Task D12-T5: Implement EU AI Act compliance adapter (Articles 9, 14, 26, 73)
Task D12-T6: Implement SOC2, GDPR, HIPAA, ISO 42001 adapter stubs
Task D12-T7: Implement incident reporting with deadline computation
Task D12-T8: Run all 4 acceptance criteria
```

---

## 14. Anti-Patterns

**Anti-Pattern 1 — Retrofitting governance**
Adding governance after deployment is 4–5x more expensive than building it in. Every agent must be classified and approved before deployment.

**Anti-Pattern 2 — Treating compliance as documentation**
A PDF risk assessment that sits on a SharePoint drive does not satisfy regulatory requirements. Compliance evidence must be machine-readable, timestamped, and auditor-accessible on demand.

**Anti-Pattern 3 — Ignoring the UNACCEPTABLE tier**
EU AI Act Article 5 prohibits certain AI practices outright. Agents that perform prohibited practices (subliminal manipulation, social scoring, real-time biometric surveillance in public spaces) must be blocked at the framework level, not flagged in documentation.

**Anti-Pattern 4 — One-time risk classification**
Risk classifications expire. An agent's risk profile changes when its data inputs, capabilities, or deployment context change. Review dates must be enforced.

---

## 15. Reference Implementation Notes

### 15.1 AgentKit-LangGraph and AgentKit-MAF
- Compliance evidence storage: PostgreSQL with append-only audit table
- EU AI Act evidence generation: map to Annex IV documentation requirements
- Incident reporting integration: webhook to notification system
- GRC platform adapters: OneTrust API, ServiceNow GRC API (community extensions)

### 15.2 Five-Framework Crosswalk

| AgentKit Control | EU AI Act | SOC2 | HIPAA | GDPR | ISO 42001 |
|---|---|---|---|---|---|
| Risk classification | Art. 9 | CC3.2 | — | Art. 35 DPIA | 6.1.2 |
| Approval workflow | Art. 26 | CC8.1 | — | — | 8.4 |
| Human oversight | Art. 14 | CC6.1 | — | Art. 22 | 6.1.2 |
| Audit trail (D7) | Art. 12 | CC7.2 | 164.312(b) | Art. 30 | 9.1 |
| Incident reporting | Art. 73 | CC7.3 | 164.308(a)(6) | Art. 33 | 10.2 |

---

## 16. Open Questions

| # | Question | Blocks Stable? |
|---|---|---|
| OQ-1 | Should AgentKit provide pre-built Annex IV technical documentation templates? | No — community extension |
| OQ-2 | Should risk classification be automated using the agent's capability declarations? | Yes — reduces classification burden |
| OQ-3 | Should DORA (Digital Operational Resilience Act) be a supported framework? | No — add in v2 |

---

## 17. Changelog

| Version | Date | Change |
|---|---|---|
| 1.1 | June 2026 | Initial dimension spec — EU AI Act Article 73 deadline computation, five-framework crosswalk |

---

*AgentKit Dimension 12 — Governance — v1.1*
