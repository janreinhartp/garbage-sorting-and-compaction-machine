# 02 - Technical Requirements
## Garbage Sorting and Compaction Machine

**Document ID:** TR-001  
**Version:** 1.0  
**Status:** Prototype baseline

## 1. System Overview

The Raspberry Pi 4 Model B coordinates vision, conveyor movement, sorting, compaction, bin management, operator interfaces, and data storage.

Field sensors and actuator interfaces communicate through RS485 Modbus RTU digital I/O modules.

The Raspberry Pi is not a safety-rated controller. Hazardous-motion protection must be implemented independently through suitable safety hardware.

## 2. Hardware Requirements

| ID | Component | Requirement |
|---|---|---|
| HW-001 | Main controller | Raspberry Pi 4 Model B |
| HW-002 | Camera | Fixed, supported camera with stable lighting |
| HW-003 | Vision software | OpenCV-compatible capture and a versioned classifier |
| HW-004 | Conveyor | Motor and drive/starter suited to stop-and-sort operation |
| HW-005 | Sorting plungers | Three single-acting, spring-return cylinders |
| HW-006 | Compactor carriage | Linear rails and lead screw driven by a reversible AC motor |
| HW-007 | Position feedback | Home, Plastic, Can, and Glass sensors |
| HW-008 | Compaction press | One single-acting, spring-return cylinder |
| HW-009 | Bin ejectors | Three single-acting, spring-return cylinders |
| HW-010 | Bin-full sensors | Three digital laser sensors |
| HW-011 | Pneumatic valves | Suitable 3/2 valves, subject to circuit validation |
| HW-012 | Remote I/O | RS485 Modbus RTU DI/DO modules |
| HW-013 | Local interface | LCD and physical buttons |
| HW-014 | Web interface | Local-network dashboard |
| HW-015 | Database | SQLite |
| HW-016 | Safety circuit | Emergency stop, guard interlocks, and required motion inhibits |
| HW-017 | Motor protection | Suitable overload and short-circuit protection |
| HW-018 | Pneumatic safety | Air preparation, pressure monitoring, isolation, and exhaust as required |

## 3. Software Requirements

| ID | Requirement |
|---|---|
| SW-001 | Implement a deterministic machine state machine. |
| SW-002 | Separate hardware drivers, control logic, vision, database, and user interfaces. |
| SW-003 | Use Python on Raspberry Pi OS 64-bit. |
| SW-004 | Use a tested, version-pinned Modbus library such as `pymodbus`. |
| SW-005 | Use OpenCV or a compatible camera library. |
| SW-006 | Store category, confidence, model version, destination, and result per item. |
| SW-007 | Use SQLite transactions for item and counter updates. |
| SW-008 | Provide a local API, for example FastAPI. |
| SW-009 | Support CSV exports. |
| SW-010 | Log state transitions, operator commands, faults, and timeouts. |
| SW-011 | Detect interrupted cycles after restart. |
| SW-012 | Prevent UI handlers from writing output coils directly. |
| SW-013 | Provide unit, simulation, integration, and acceptance tests. |
| SW-014 | Support managed startup and controlled shutdown. |

## 4. Preliminary Digital Input Map

This list is preliminary. Confirm signal voltage, polarity, isolation, module input type, and terminal assignments before wiring.

| Input | Signal | Purpose |
|---|---|---|
| DI-01 | Item present | Detect an item at the camera station |
| DI-02 | Camera station | Confirm camera-station position |
| DI-03 | Plastic station | Confirm plastic sorting position |
| DI-04 | Can station | Confirm can sorting position |
| DI-05 | Glass station | Confirm glass sorting position |
| DI-06 | Unidentified drop station | Optional end-of-conveyor confirmation |
| DI-07 | Plastic bin full | Digital laser sensor |
| DI-08 | Can bin full | Digital laser sensor |
| DI-09 | Glass bin full | Digital laser sensor |
| DI-10 | Carriage Home | Confirm Home position |
| DI-11 | Carriage Plastic | Confirm Plastic position |
| DI-12 | Carriage Can | Confirm Can position |
| DI-13 | Carriage Glass | Confirm Glass position |
| DI-14 | Press retracted | Critical movement interlock |
| DI-15 | Press extended | Recommended feedback |
| DI-16 | Plastic plunger home | Confirm return |
| DI-17 | Can plunger home | Confirm return |
| DI-18 | Glass plunger home | Confirm return |
| DI-19 | Plastic ejector home | Confirm return |
| DI-20 | Can ejector home | Confirm return |
| DI-21 | Glass ejector home | Confirm return |
| DI-22 | Air pressure OK | Pneumatic permissive |
| DI-23 | Safety circuit healthy | Auxiliary status feedback |
| DI-24 | Guard closed | Auxiliary status feedback |
| DI-25 | Conveyor drive healthy | Recommended diagnostic input |
| DI-26 | Motor overload healthy | Recommended diagnostic input |
| DI-27 | Bin service/door status | If required by the mechanism |
| DI-28 | Ejection complete | If required by the mechanism |

The final input count depends on the selected hardware and required feedback. Do not remove critical feedback only to fit a smaller module.

## 5. Preliminary Digital Output Map

All outputs must match the selected module's voltage, current rating, isolation, and electrical output type.

| Output | Signal | Purpose |
|---|---|---|
| DO-01 | Plastic sorting valve | Extend plastic plunger |
| DO-02 | Can sorting valve | Extend can plunger |
| DO-03 | Glass sorting valve | Extend glass plunger |
| DO-04 | Press valve | Extend compaction press |
| DO-05 | Plastic ejector valve | Extend plastic ejector |
| DO-06 | Can ejector valve | Extend can ejector |
| DO-07 | Glass ejector valve | Extend glass ejector |
| DO-08 | Conveyor run | Command conveyor motor interface |
| DO-09 | Carriage forward | Command one travel direction |
| DO-10 | Carriage reverse | Command opposite travel direction |
| DO-11 | Alarm lamp | Fault indication |
| DO-12 | Buzzer | Audible alarm |
| DO-13 | Ready lamp | Ready indication |

**Preliminary total: 13 digital outputs.** A 16-channel output module is a possible starting point, subject to electrical validation and spare-channel requirements.

### Output Rules

- DO-09 and DO-10 must never be energized simultaneously.
- The reversing interface must have hardware interlocking.
- A software interlock is supplementary and must not be the only protection against conflicting motor directions.
- Each single-acting cylinder is assumed to extend when its valve is energized and return when de-energized. Verify the selected pneumatic circuit.
- Output de-energization is not proof of a safe mechanical position.
- Verify what the remote I/O module does to its outputs after watchdog timeout or RS485 loss.

## 6. RS485 Modbus RTU Requirements

- Document baud rate, parity, stop bits, slave addresses, and register maps.
- Assign a unique Modbus address to each device.
- Prefer a trunk/daisy-chain topology over long star branches.
- Apply termination at the two physical ends of the bus when required by the selected devices and cable.
- Do not install termination resistors at every device.
- Verify A/B polarity using each manufacturer's documentation.
- Follow manufacturer recommendations for cable type, shielding, grounding, and signal reference.
- Detect timeouts, CRC errors, stale readings, and repeated communication failures.
- Validate register addressing and bit order using a test script.
- Serialize access to a shared serial bus.
- Never treat a successful output write as proof that an actuator moved.

## 7. Initial Timing and Configuration

These values are starting points for bench testing, not validated machine limits.

| Setting | Initial value |
|---|---:|
| Vision confidence threshold | 0.80 |
| Plastic compaction threshold | 10 items |
| Can compaction threshold | 10 items |
| Glass compaction threshold | 10 items |
| Sensor debounce | 50 ms |
| Modbus timeout | 1000 ms |
| Conveyor movement timeout | 10 seconds |
| Carriage movement timeout | 15 seconds |
| Sorting plunger timeout | 3 seconds |
| Press cycle timeout | 10 seconds |
| Ejector cycle timeout | 10 seconds |

Replace these values with measured and validated settings during commissioning. A timeout must trigger a fault and block unsafe progression.

## 8. Data Requirements

Record the following:

- Item ID and timestamps.
- Predicted category, confidence, model version, destination, and sort result.
- Compaction start/end time, bin, result, duration, and fault code.
- Ejector/service start/end time, bin, result, and sensor observations.
- State transitions, faults, acknowledgements, and recovery actions.
- Configuration changes and operator commands.
- Diagnostic snapshots where useful.

Store timestamps in UTC and display local time in the interface. Store image files separately from SQLite and retain their paths in the database if image capture is enabled.

## 9. Fault Requirements

| Fault | Condition | Required response |
|---|---|---|
| FLT-001 | RS485 communication lost | Stop sequence progression and enter a communication fault |
| FLT-002 | Camera unavailable | Prevent new classification and untracked routing |
| FLT-003 | Classification failed | Use unidentified routing only if tracking and route conditions remain valid |
| FLT-004 | Destination timeout | Stop conveyor and raise fault |
| FLT-005 | Plunger failed to return | Inhibit conveyor movement |
| FLT-006 | Carriage position timeout | Stop carriage motion and raise fault |
| FLT-007 | Contradictory position sensors | Inhibit press and carriage motion |
| FLT-008 | Press failed to retract | Inhibit carriage movement |
| FLT-009 | Low air pressure | Inhibit pneumatic cycles |
| FLT-010 | Bin full | Mark bin unavailable |
| FLT-011 | Ejector/service failed | Keep bin unavailable pending inspection |
| FLT-012 | Safety circuit open | Safety hardware inhibits hazardous motion |
| FLT-013 | Interrupted cycle | Require startup recovery checks |
| FLT-014 | Database write failure | Prevent silent data loss and enter defined degraded state |
| FLT-015 | Motor overload or drive fault | Remove motion command and require inspection |
| FLT-016 | Conflicting direction commands | Block motion and report a fault |

## 10. Verification

Verification must include:

- Unit tests.
- State-machine simulation.
- Hardware I/O bench tests.
- Vision validation on labeled samples.
- Modbus communication tests.
- Guarded acceptance tests.
- Independent safety-function tests.

Test every I/O point against the final wiring diagram before enabling automatic operation.
