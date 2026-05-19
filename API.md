# MIG API Documentation

## Base URL

```
https://mig.houseofgalatine.com
```

## Authentication

API access requires authentication in production deployments. Contact House of Galatine for API credentials.

---

## Endpoints

### POST /validate

Validate a command against operational safety policy.

**Request:**

```json
{
  "text": "Set pump 1 speed to 5000 RPM for emergency pressure boost",
  "action_type": "write_setpoint_major",
  "stage": "running",
  "context": "AI agent attempting setpoint change on pump controller"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| text | string | yes | The command to validate |
| action_type | string | no | Explicit action type (auto-inferred if omitted) |
| stage | string | no | Current operational mode (default: "running") |
| context | string | no | Additional context about the command source |

**Response:**

```json
{
  "decision": "DENY",
  "decision_id": "dec_1778318441588",
  "policy_id": "POL-OT-DENY-002",
  "matched_policy": "Setpoint changes exceeding 5% of current operating value are blocked as they risk process instability",
  "risk_score": 100,
  "flags": ["PAYLOAD_HIGH_RISK"],
  "enforcement": {
    "level": "critical",
    "notify": ["safety_officer", "plant_supervisor", "control_engineer"],
    "requires": []
  },
  "timestamp": "2026-05-09T09:20:48.238Z"
}
```

| Field | Description |
|-------|-------------|
| decision | ALLOW, DENY, or APPROVAL |
| decision_id | Unique identifier for this decision |
| policy_id | ID of the matched policy |
| matched_policy | Human-readable description of why this decision was made |
| risk_score | 0-100 risk assessment |
| flags | Array of flags raised during validation |
| enforcement | Notification and authorization requirements |
| timestamp | ISO 8601 timestamp |

---

### POST /approve

Approve a pending APPROVAL decision.

**Request:**

```json
{
  "decision_id": "dec_1778318441588",
  "approved_by": "shift_operator"
}
```

**Response:**

```json
{
  "status": "approved",
  "decision_id": "dec_1778318441588",
  "approved_by": "shift_operator",
  "timestamp": "2026-05-09T09:21:02Z"
}
```

---

### POST /reject

Reject a pending APPROVAL decision.

**Request:**

```json
{
  "decision_id": "dec_1778318441588",
  "rejected_by": "shift_operator",
  "reason": "Setpoint change not authorized for current shift"
}
```

---

### GET /health

Check system status.

**Response:**

```json
{
  "status": "healthy",
  "version": "4.2.0",
  "policy_count": 28,
  "memory_count": 28,
  "neo4j_connected": true
}
```

---

### GET /audit

Retrieve decision history.

**Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| limit | int | 50 | Number of decisions to return |

**Response:**

```json
{
  "count": 1,
  "decisions": [
    {
      "id": "dec_1778318441588",
      "input": "Set pump 1 speed to 5000 RPM",
      "decision": "DENY",
      "reason": "Policy: POL-OT-DENY-002 | Flags: PAYLOAD_HIGH_RISK",
      "matched_policy": "Setpoint changes exceeding 5% are blocked",
      "flags": ["PAYLOAD_HIGH_RISK"],
      "stage": "running",
      "timestamp": "2026-05-09T09:20:48Z"
    }
  ]
}
```

---

## Decision Logic

```
Command received
    → No matching policy?      → DENY (fail-closed)
    → MIG unreachable?         → DENY (fail-closed)
    → Policy says DENY?        → DENY
    → Payload risk override?   → DENY or APPROVAL
    → Policy says APPROVAL?    → APPROVAL (wait for operator)
    → All checks pass?         → ALLOW
```

## Error Handling

All errors result in DENY. MIG never fails open.

```json
{
  "decision": "DENY",
  "reason": "MIG unreachable — fail-closed",
  "flags": ["MIG_UNREACHABLE"],
  "policy_id": "FAIL_CLOSED"
}
```

---

*© 2026 House of Galatine. All rights reserved.*
