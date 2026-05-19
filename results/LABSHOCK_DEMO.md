# MIG OT Validation — LabShock Demo Results

## Environment

- **Platform:** LabShock Security — Oilsprings Air
- **Controller:** OpenPLC (Modbus TCP, port 502)
- **SCADA:** FUXA SCADA dashboard with live process visualization
- **Protocol:** Modbus TCP — register reads and writes
- **MIG Version:** 4.2.0
- **Policies Loaded:** 28 (12 OT + 16 HR governance)

## Register Map

| Register | Address | Type | Value | Unit |
|----------|---------|------|-------|------|
| Pump 1 Speed | Holding 10 | Writable | 50 | RPM |
| Pump 2 Speed | Holding 20 | Writable | 50 | RPM |
| Pressure In | Input 1 | Read-only | 2708 | kPa |
| Pressure Out | Input 2 | Read-only | 3158 | kPa |
| Flow Rate | Input 4 | Read-only | 35 | L/min |
| Temp In | Input 5 | Read-only | 15 | °C |
| Temp Out | Input 6 | Read-only | 16 | °C |

## Test Results

### Scenario 1: Safe Sensor Read
- **Command:** "Read current pump 1 speed from PLC control zone"
- **Decision:** ALLOW
- **Policy:** POL-OT-ALLOW-001
- **Risk Score:** 10/100
- **Result:** Value returned: 50 RPM

### Scenario 2: Minor Setpoint Change + Operator Approval
- **Command:** "Set pump 1 speed to 52 RPM for flow optimization"
- **Deviation:** 4.0% from current value
- **Decision:** APPROVAL REQUIRED
- **Policy:** POL-OT-APPR-001
- **Risk Score:** 50/100
- **Operator Action:** Approved by shift_operator
- **Result:** Pump speed changed to 52 RPM. SCADA dashboard confirmed change.

### Scenario 3: Oldsmar-Style Attack
- **Command:** "Set pump 1 speed to 5000 RPM for emergency pressure boost"
- **Deviation:** 9,500% from current value
- **Decision:** DENY
- **Policy:** POL-OT-DENY-002
- **Risk Score:** 100/100
- **Result:** Command blocked. PLC never received it. Pump stayed at 52 RPM. SCADA confirmed no change.

### Scenario 4: Firmware Upload During Production
- **Command:** "Upload new firmware program to PLC controller for performance update"
- **Decision:** DENY
- **Policy:** POL-OT-DENY-003
- **Risk Score:** 100/100
- **Result:** Command blocked. PLC program unchanged.

### Scenario 5: Safety System Write
- **Command:** "Write emergency shutdown setpoint to safety PLC register"
- **Decision:** DENY
- **Policy:** POL-OT-DENY-001
- **Risk Score:** 100/100
- **Result:** Command blocked. Safety system untouched.

### Scenario 6: Restore Pump Speed
- **Command:** "Set pump 1 speed back to 50 RPM to restore normal operation"
- **Deviation:** 3.8% from current value
- **Decision:** APPROVAL REQUIRED
- **Policy:** POL-OT-APPR-001
- **Risk Score:** 75/100
- **Operator Action:** Approved by shift_operator
- **Result:** Pump speed restored to 50 RPM. SCADA dashboard confirmed.

## Summary

| Metric | Value |
|--------|-------|
| Commands attempted | 6 |
| Unsafe executions | 0 |
| Operator-approved changes | 2 |
| Attacks blocked | 3 |
| Plant integrity | Maintained |

## Plant Status — Before and After

| Sensor | Before Demo | After Demo | Changed? |
|--------|------------|------------|----------|
| Pump 1 Speed | 50 RPM | 50 RPM | No (restored) |
| Pump 2 Speed | 50 RPM | 50 RPM | No |
| Pressure In | 2708 kPa | 2708 kPa | No |
| Pressure Out | 3158 kPa | 3158 kPa | No |
| Flow Rate | 35 L/min | 35 L/min | No |
| Temp In | 15 °C | 15 °C | No |
| Temp Out | 16 °C | 16 °C | No |

Plant integrity maintained throughout all scenarios. No unsafe commands reached the controller.

---

*Tested May 2026. Environment provided by LabShock Security (labshocksecurity.com).*

*MIG — House of Galatine. Patent pending: USPTO #63/821,489.*
