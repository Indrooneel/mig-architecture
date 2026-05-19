<p align="center">
  <strong>MIG — Memory Intelligence Graph</strong><br>
  <em>The Execution Control Layer for Critical Infrastructure</em><br><br>
  Every command validated before it touches a controller. Fail-closed by default.<br><br>
  <a href="https://houseofgalatine.com">Website</a> · 
  <a href="https://houseofgalatine.com/playground">Try the Playground</a> · 
  <a href="https://houseofgalatine.com#contact">Contact</a>
</p>

---

## What is MIG?

MIG is a deterministic command validation layer that sits at the IT-OT boundary.

Before any command — human, automated, or adversarial — reaches a control system, MIG intercepts it, validates it against operational safety policy, and decides:

- **ALLOW** — command is safe, proceed to controller
- **DENY** — command is blocked, controller never sees it, OT network isolated
- **APPROVAL** — command is held until an operator confirms

MIG does not monitor what happened. MIG controls what is allowed to happen.

---

## Why MIG Exists

OT protocols like Modbus have no built-in authentication, authorization, or logging. Any device on the network can read or write to any PLC register. The protocol trusts the network completely.

Current OT security tools (Claroty, Dragos, Nozomi) monitor network traffic and alert after a command executes. By the time the alert fires, the command has already reached the controller.

MIG adds the missing layer: **pre-execution command validation**. Every command is inspected for operational context — not just network compliance — before it crosses the boundary into the OT network.

---

## Where MIG Sits

```
IT Network / Enterprise Zone
        │
        ▼
┌──────────────────────────────┐
│     MIG — Level 3.5          │
│     Industrial DMZ           │
│                              │
│  ┌─────────────────────────┐ │
│  │   Validation Pipeline   │ │
│  │                         │ │
│  │  Payload Inspection     │ │
│  │  Policy Matching        │ │
│  │  Authorization Check    │ │
│  │  Operational Mode       │ │
│  │  Risk Scoring           │ │
│  │  Decision + Audit       │ │
│  └─────────────────────────┘ │
│                              │
│  ALLOW → forward to OT      │
│  DENY  → block + isolate    │
│  APPROVAL → hold for human  │
└──────────────────────────────┘
        │
        ▼
OT Network / Control Zone
  PLCs · SCADA · DCS · SIS
        │
        ▼
Physical Process
  Pumps · Valves · Turbines · Sensors
```

MIG operates at Purdue Level 3.5 — the Industrial DMZ. This is the controlled broker zone between IT and OT where every command crossing the boundary is validated against operational safety policy.

---

## Key Properties

| Property | Description |
|----------|-------------|
| **Pre-execution** | Commands validated before reaching the controller, not after |
| **Fail-closed** | No policy match → DENY. MIG unreachable → DENY. Default is never ALLOW |
| **Source-agnostic** | Validates the command, not the source. Works against human error, compromised insiders, rogue automation, and adversarial commands |
| **Deterministic** | Same command → same decision. No probabilistic scoring. Policy in, decision out |
| **Auditable** | Every decision logged with full trace — command, policy matched, checks performed, risk score, timestamp |
| **Operator-in-the-loop** | APPROVAL decisions require human confirmation before execution proceeds |
| **Domain-agnostic** | Same engine validates OT commands, HR workflows, financial actions. Different policies, same boundary |

---

## Validated Results

Tested on LabShock Security's Oilsprings Air environment — a simulated oil processing facility with live OpenPLC, SCADA dashboard, and Modbus communication.

| Scenario | Command | Decision | Risk |
|----------|---------|----------|------|
| Safe read | Read pump speed from PLC | ALLOW | 10 |
| Minor change | Set pump to 52 RPM (4% deviation) | APPROVAL | 50 |
| Oldsmar-style attack | Set pump to 5000 RPM (9500% deviation) | DENY | 100 |
| Firmware during production | Upload firmware to running PLC | DENY | 100 |
| Safety system write | Write to SIS register | DENY | 100 |
| Config export | Export PLC config to external network | DENY | 90 |

**Result: 6 commands attempted. 0 unsafe executions. 2 operator-approved changes executed safely. Plant integrity maintained.**

The SCADA dashboard confirmed: pump speed changed only after operator approval. The Oldsmar-style attack (9500% setpoint deviation) was blocked before the command reached the PLC. The controller never processed the malicious command.

---

## API Overview

MIG exposes a REST API for command validation:

```
POST /validate
```

Request:
```json
{
  "text": "Set pump 1 speed to 5000 RPM",
  "action_type": "write_setpoint_major",
  "stage": "running"
}
```

Response:
```json
{
  "decision": "DENY",
  "policy_id": "POL-OT-DENY-002",
  "risk_score": 100,
  "matched_policy": "Setpoint changes exceeding 5% of current operating value are blocked",
  "flags": ["PAYLOAD_HIGH_RISK"],
  "decision_id": "dec_1778318441588",
  "timestamp": "2026-05-09T09:20:48Z"
}
```

Full API documentation: [API.md](API.md)

---

## Try It Live

**MIG Playground** — type any OT command and watch the validation pipeline process it in real time:

→ [houseofgalatine.com/playground](https://houseofgalatine.com/playground)

No signup. No install. No server required.

---

## Architecture

For detailed system architecture, validation pipeline, and integration patterns:

→ [ARCHITECTURE.md](ARCHITECTURE.md)

---

## Policy Format

MIG uses declarative policy rules. Example OT policy:

```json
{
  "id": "POL-OT-DENY-002",
  "description": "Block dangerous setpoint deviations",
  "action_type": "write_setpoint_major",
  "direction": "DENY",
  "enforcement": "critical",
  "notify": ["safety_officer", "plant_supervisor"]
}
```

Sample policies: [examples/](examples/)

---

## About

MIG is built by **House of Galatine** — an AI cybersecurity company focused on execution control for critical infrastructure.

- **Patent Pending:** USPTO Provisional #63/821,489
- **Founded:** 2024
- **Founder:** Indrooneel Panday
- **Website:** [houseofgalatine.com](https://houseofgalatine.com)

---

## License

MIG is proprietary software. This repository contains architecture documentation, API specifications, and example configurations only. The MIG engine source code is not included.

© 2026 House of Galatine. All rights reserved.
