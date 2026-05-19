# MIG Architecture

## System Overview

MIG is a stateless command validation engine. It receives a command, evaluates it against policy, returns a decision, and logs the result. The engine is deterministic — the same command under the same conditions always produces the same decision.

## Deployment Position

MIG deploys at **Purdue Level 3.5** — the Industrial DMZ between the enterprise IT network and the OT control network.

In this position, MIG acts as a controlled broker: every command attempting to cross from IT to OT passes through MIG's validation pipeline before reaching any controller, SCADA system, or safety instrumented system.

```
Enterprise / IT (Level 4-5)
        │
   [ MIG Engine ]  ← Level 3.5 — Industrial DMZ
        │
   ┌────┴────┐
   │         │
 ALLOW     DENY
   │         │
   ▼         ✕
OT Control    Command
Network       Terminated
(Level 0-3)
```

## Validation Pipeline

Every command passes through a sequential validation pipeline. Each step is independent and produces its own result. The final decision is computed from the combined results.

```
Command Input
    │
    ├── Step 1: Sensitivity Scan
    │   Detect PII, credentials, or sensitive data patterns
    │
    ├── Step 2: Action Classification
    │   Infer the operational intent of the command
    │
    ├── Step 3: Payload Inspection
    │   Analyze what data is being carried and where it's going
    │   Assign data sensitivity and destination risk
    │
    ├── Step 4: Policy Evaluation
    │   Match command against operational safety policies
    │   Traverse policy graph for the most specific match
    │
    ├── Step 5: Operational Mode Check
    │   Verify command is valid for current plant state
    │   (startup, running, shutdown, maintenance, emergency)
    │
    ├── Step 6: Behavioral Analysis
    │   Flag anomalous patterns compared to baseline behavior
    │
    ├── Step 7: Override Evaluation
    │   Apply override logic where higher-risk checks
    │   supersede lower-risk approvals
    │
    ├── Step 8: Decision Computation
    │   Aggregate all check results into final decision
    │   ALLOW, DENY, or APPROVAL
    │
    └── Step 9: Audit Logging
        Log complete decision trace with timestamp,
        policy matched, flags, risk score, and decision ID
```

## Decision Types

| Decision | Meaning | Action |
|----------|---------|--------|
| **ALLOW** | Command passed all checks | Forward to OT network for execution |
| **DENY** | Command violated policy | Block completely. Controller never receives it |
| **APPROVAL** | Command requires human confirmation | Hold until operator approves or rejects |

## Fail-Closed Architecture

MIG defaults to DENY in every ambiguous scenario:

- No matching policy → **DENY**
- MIG engine unreachable → **DENY** (proxy-level enforcement)
- Conflicting signals between checks → higher-risk decision wins
- Unknown command type → **DENY**

The system never fails open. An attacker who finds a way to confuse the validation pipeline gets denied, not allowed.

## Policy Structure

Policies are declarative rules that define what commands are permitted, denied, or require approval. Each policy specifies:

- **Identifier** — unique policy ID (e.g., POL-OT-DENY-002)
- **Action type** — what kind of command this policy covers
- **Direction** — ALLOW, DENY, or APPROVAL
- **Enforcement level** — standard, elevated, or critical
- **Notification list** — who gets notified when this policy fires
- **Description** — human-readable explanation of the rule

Policies are organized by operational domain. The same engine can enforce OT safety policies, HR governance policies, financial compliance rules, or any other domain — by loading different policy sets.

## Zone-Conduit Model

MIG supports zone-based policy enforcement aligned with IEC 62443:

- **Zones** define security boundaries (control zone, safety zone, supervisory zone, external)
- **Conduits** define allowed communication paths between zones
- **Policies** are scoped to zone-conduit pairs

A command that is valid within the control zone may be denied if it attempts to cross into the safety zone or export data to an external zone.

## Payload Inspection

MIG inspects not just the command type but the actual content:

- **Data types detected** — sensor readings, register writes, configuration data, firmware, safety commands
- **Sensitivity classification** — low, medium, high, critical
- **Destination analysis** — where is this data going? Internal, supervisory, external?
- **Risk scoring** — computed from data sensitivity × destination risk

A command to "read sensor value from control zone" has low risk. A command to "export PLC configuration to external server" has critical risk — regardless of whether the source is authorized.

## Operational Mode Awareness

MIG understands that the same command may be safe in one operational mode and dangerous in another:

| Mode | Example allowed | Example denied |
|------|----------------|---------------|
| **Running** | Read sensor values, minor setpoint adjustments | Firmware uploads, safety system writes |
| **Maintenance** | Firmware updates, configuration changes | Process commands to running equipment |
| **Startup** | Sequential startup commands | Full-speed commands before warmup |
| **Emergency** | Emergency shutdown commands | Non-safety commands |

## Audit Trail

Every decision is logged with:

- Decision ID (unique identifier)
- Timestamp
- Original command text
- Inferred action type
- Policy matched (or fail-closed reason)
- All flags raised during validation
- Risk score
- Final decision (ALLOW / DENY / APPROVAL)
- Notification list
- Enforcement level

Audit logs are immutable and exportable for compliance reporting.

## Integration Patterns

### API Mode
MIG exposes a REST API. External systems call `/validate` before executing commands. Suitable for software-defined environments where the calling system respects MIG's decisions.

### Proxy Mode
MIG operates as a network proxy intercepting OT protocol traffic (Modbus TCP, OPC-UA). Commands are inspected in transit. Only allowed commands are forwarded. Suitable for environments requiring inline enforcement.

### Agent Mode
MIG operates as an agent within multi-agent orchestration frameworks. Other agents propose actions; the MIG agent validates and either executes or blocks. Suitable for agentic AI deployments where agents must not bypass governance.

## Performance

- Decision latency: sub-100ms per command
- Stateless design: horizontally scalable
- No GPU required
- Runs on standard Linux with 1GB+ RAM

## Technology

- Engine: Python
- API: FastAPI
- Policy store: Neo4j graph database (Pro) / JSON (Core)
- Protocol: REST API, Modbus TCP proxy
- Deployment: Docker

---

*MIG is proprietary software built by House of Galatine. Patent pending Nova : USPTO Provisional #63/821,489.*

*This document describes the system architecture. The engine source code is not publicly available.*
