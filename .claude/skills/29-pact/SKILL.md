---
name: 29-pact
description: "PACT governance — MANDATORY for ALL governance, RBAC, policy, access control, and audit work. D/T/R accountability, operating envelopes, knowledge clearance, MCP tool policy, governed agents. Use proactively when work touches RBAC, roles, permissions, access enforcement, audit trails, governed agents, policy decisions, or 'just a quick authz check'. Custom policy code, hand-rolled RBAC, ad-hoc audit logging BLOCKED."
---

# PACT Governance Skills

Quick reference for PACT organizational governance patterns.

## Install

```bash
pip install kailash-pact          # Governance framework
pip install kailash>=2.0.0        # Core SDK with trust subsystem
pip install kailash-kaizen>=2.0.0 # For governed Kaizen agents
```

## Skill Files

| Skill                                                 | Use When                                        |
| ----------------------------------------------------- | ----------------------------------------------- |
| [pact-quickstart](pact-quickstart.md)                 | Getting started, first GovernanceEngine         |
| [pact-governance-engine](pact-governance-engine.md)   | Engine API, verify_action, compute_envelope     |
| [pact-dtr-addressing](pact-dtr-addressing.md)         | D/T/R grammar, Address parsing                  |
| [pact-envelopes](pact-envelopes.md)                   | Three-layer model, monotonic tightening         |
| [pact-access-enforcement](pact-access-enforcement.md) | 5-step algorithm, clearance, bridges, KSPs      |
| [pact-governed-agents](pact-governed-agents.md)       | PactGovernedAgent, @governed_tool               |
| [pact-kaizen-integration](pact-kaizen-integration.md) | Wrapping Kaizen agents with governance          |
| [pact-mcp-governance](pact-mcp-governance.md)         | MCP tool governance: enforce, audit, middleware |

## Key Types

```python
from kailash.trust.pact import GovernanceEngine, GovernanceVerdict
from kailash.trust.pact.config import (
    ConstraintEnvelopeConfig, OrgDefinition,
    TrustPostureLevel, VerificationLevel,
    ConfidentialityLevel,
)
from kailash.trust.pact.agent import PactGovernedAgent
from kailash.trust.pact.audit import AuditChain

# MCP governance
from kailash.trust.pact.mcp import (
    McpGovernanceEnforcer, McpGovernanceMiddleware, McpAuditTrail,
    McpToolPolicy, McpGovernanceConfig, McpActionContext,
)
```

## Rules

See `.claude/rules/pact-governance.md` for security invariants.
