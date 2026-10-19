# 02 - Technical Requirements
## Garbage Sorting and Compaction Machine

**Document ID:** TR-001  
**Version:** 1.4  
**Status:** Final

## 1. System Overview

The Raspberry Pi 4 Model B coordinates vision, conveyor movement, sorting, compaction, bin management, operator interfaces, and data storage.

Field sensors and actuator interfaces communicate through RS485 Modbus RTU digital I/O modules.

The Raspberry Pi is not a safety-rated controller. Hazardous-motion protection must be implemented independently through suitable safety hardware.

The three classified bins are identified by position: Bin 1, Bin 2, and Bin 3. Hardware signals, carriage positions, and sorting stations are named by bin position. The waste category of each bin is a setting, so no hardware name contains a category.

## 2. Hardware Requirements

| ID | Component | Requirement |
|---|---|---|
| HW-001 | Main controller | Raspberry Pi 4 Model B |
| HW-002 | Camera | Fixed, supported camera with stable lighting |
| HW-003 | Vision software | OpenCV-compatible capture and a versioned TensorFlow Lite MobileNet classifier with a label file |
| HW-004 | Conveyor | Motor and drive/starter with repeatable start/stop behavior suited to timed stop-and-sort positioning |
| HW-005 | Sorting plungers | Three single-acting, spring-return cylinders |
| HW-006 | Compactor carriage | Linear rails and lead screw driven by a reversible AC motor |
| HW-007 | Position feedback | Home, Bin 1, Bin 2, and Bin 3 sensors |
| HW-008 | Compaction press | One single-acting, spring-return cylinder |
| HW-009 | Bin ejectors | Three single-acting, spring-return cylinders that push the compacted block into a collection tray or bag |
| HW-010 | Bin-full sensors | Three digital laser sensors |
| HW-011 | Pneumatic valves | Suitable 3/2 valves, subject to circuit validation |
| HW-012 | Remote I/O | RS485 Modbus RTU DI/DO modules |
| HW-013 | Local interface | LCD and physical buttons |
| HW-014 | Web interface | Login-protected local-network dashboard at `http://gscm.local` |
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
| SW-005 | Use OpenCV or a compatible camera library, and run the classifier with TensorFlow Lite. |
| SW-006 | Store predicted category, confidence, model version, destination bin, the category assigned to that bin at sort time, and result per item. |
| SW-007 | Use SQLite transactions for item and counter updates. |
| SW-008 | Provide a local API, for example FastAPI. |
| SW-009 | Support CSV exports. |
| SW-010 | Log state transitions, operator commands, faults, and timeouts. |
| SW-011 | Detect interrupted cycles after restart. |
| SW-012 | Prevent UI handlers from writing output coils directly. |
| SW-013 | Provide unit, simulation, integration, and acceptance tests. |
| SW-014 | Support managed startup and controlled shutdown. |
| SW-015 | Position the conveyor using calibrated per-station travel times referenced to the camera-station item sensor. |
| SW-016 | Reserve the destination bin at route selection and track the unidentified-bin item limit. |
| SW-017 | Save only low-confidence and unidentified images, and enforce total-size and age limits. |
| SW-018 | Load the list of available categories from the active model's label file. |
| SW-019 | Route by looking up the bin assigned to the predicted category. A category with no assigned bin routes to unidentified. |
| SW-020 | Allow bin category changes only in IDLE, only for an empty bin, only to a category in the active label file, and with each category assigned to at most one bin. |
| SW-021 | Validate at startup that every assigned category exists in the active label file. |
| SW-022 | Require login for all dashboard pages and API endpoints except the login page, and enforce Viewer, Operator, and Admin roles on the server. |
| SW-025 | Publish the dashboard on the LAN as `gscm.local` using mDNS (Avahi), with a fixed DHCP reservation for the Pi. | |
| SW-023 | Store passwords as argon2 hashes, use server-side sessions with an idle timeout, protect state-changing requests with CSRF tokens, and lock accounts temporarily after repeated failed logins. |
| SW-024 | Record the user ID with every command, settings change, and user-management action. |

## 4. Digital Input Map

Confirm signal voltage, polarity, isolation, module input type, and terminal assignments against the selected parts before wiring.

| Input | Signal | Purpose |
|---|---|---|
| DI-01 | Item present | Detect an item at the camera station; timing reference for conveyor travel |
| DI-02 | Bin 1 full | Digital laser sensor |
| DI-03 | Bin 2 full | Digital laser sensor |
| DI-04 | Bin 3 full | Digital laser sensor |
| DI-05 | Carriage Home | Confirm Home position |
| DI-06 | Carriage Bin 1 | Confirm Bin 1 position |
| DI-07 | Carriage Bin 2 | Confirm Bin 2 position |
| DI-08 | Carriage Bin 3 | Confirm Bin 3 position |
| DI-09 | Press retracted | Critical movement interlock |
| DI-10 | Bin 1 plunger home | Confirm return |
| DI-11 | Bin 2 plunger home | Confirm return |
| DI-12 | Bin 3 plunger home | Confirm return |
| DI-13 | Bin 1 ejector home | Confirm return and service completion |
| DI-14 | Bin 2 ejector home | Confirm return and service completion |
| DI-15 | Bin 3 ejector home | Confirm return and service completion |

**Total: 15 digital inputs.**

The following are hardwired and are **not** Modbus digital inputs to the Raspberry Pi:

- Air-pressure permissive and pneumatic isolation.
- Emergency stop and guard interlocks (safety circuit).
- Conveyor drive protection.
- Motor overload and short-circuit protection.

They inhibit hazardous motion or energy in hardware. The Pi does not use them for sequencing. Keep spare Modbus channels for later additions.

## 5. Digital Output Map

All outputs must match the selected module's voltage, current rating, isolation, and electrical output type.

| Output | Signal | Purpose |
|---|---|---|
| DO-01 | Bin 1 sorting valve | Extend Bin 1 plunger |
| DO-02 | Bin 2 sorting valve | Extend Bin 2 plunger |
| DO-03 | Bin 3 sorting valve | Extend Bin 3 plunger |
| DO-04 | Press valve | Extend compaction press |
| DO-05 | Bin 1 ejector valve | Extend Bin 1 ejector |
| DO-06 | Bin 2 ejector valve | Extend Bin 2 ejector |
| DO-07 | Bin 3 ejector valve | Extend Bin 3 ejector |
| DO-08 | Conveyor run | Command conveyor motor interface |
| DO-09 | Carriage forward | Command one travel direction |
| DO-10 | Carriage reverse | Command opposite travel direction |
| DO-11 | Alarm lamp | Fault indication |
| DO-12 | Buzzer | Audible alarm |
| DO-13 | Ready lamp | Ready indication |

**Total: 13 digital outputs.** A 16-channel output module is a possible starting point, subject to electrical validation and spare-channel requirements.

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
| Bin 1 category | Plastic |
| Bin 2 category | Can |
| Bin 3 category | Glass |
| Bin 1 compaction threshold | 10 items |
| Bin 2 compaction threshold | 10 items |
| Bin 3 compaction threshold | 10 items |
| Unidentified bin item limit | 30 items |
| Sensor debounce | 50 ms |
| Modbus timeout | 1000 ms |
| Conveyor travel time to Bin 1 station | Calibrate in Milestone 5 |
| Conveyor travel time to Bin 2 station | Calibrate in Milestone 5 |
| Conveyor travel time to Bin 3 station | Calibrate in Milestone 5 |
| Conveyor travel time to unidentified drop | Calibrate in Milestone 5 |
| Conveyor maximum travel time | 10 seconds |
| Carriage movement timeout | 15 seconds |
| Sorting plunger timeout | 3 seconds |
| Press extend time | 3 seconds |
| Press cycle timeout | 10 seconds |
| Ejector cycle timeout | 10 seconds |
| Session idle timeout | 30 minutes |
| Failed logins before lockout | 5 |
| Login lockout duration | 15 minutes |

Replace these values with measured and validated settings during commissioning. A timeout must trigger a fault and block unsafe progression.

### 7.1 Timed Conveyor Positioning (Stop-and-Sort)

- Primary sequence: stop at the camera, classify, run for the calibrated station travel time, stop, then fire the matching plunger. Do not divert while the belt is running as the primary method.
- Each move starts with the item detected at the camera station (DI-01).
- The conveyor runs for the calibrated travel time of the selected station, then stops.
- DI-01 must clear shortly after the conveyor starts. If the item is still detected, raise FLT-004.
- No configured travel time may exceed the conveyor maximum travel time.
- Configuration validation must reject zero or missing station travel times so automatic operation cannot start before calibration.
- Recalibrate travel times after any change to belt speed, belt tension, motor, drive settings, or station layout.
- Milestone 5 is a hard gate: automatic sorting must not begin until the repeatability test passes (at least 20 consecutive in-zone stops per station under guarded conditions).
- Timed positioning applies only to the conveyor. Carriage and cylinder positions must still be confirmed by sensors.
- Do not use a second camera for conveyor positioning. Station presence sensors are out of the default I/O map; consider them only if Milestone 5 fails after recalibration and contact-zone adjustments.

## 8. Data Requirements

Record the following:

- Item ID and timestamps.
- Predicted category, confidence, model version, destination bin, category assigned to the bin at sort time, and sort result.
- Bin category assignment history: bin, old category, new category, time, and Admin user ID.
- User accounts, roles, login successes and failures, and lockouts.
- Compaction start/end time, bin, result, duration, and fault code.
- Ejector/service start/end time, bin, result, full-sensor state before and after, and operator acknowledgement.
- Compaction count and total count per bin, and the unidentified-bin count and limit.
- State transitions, faults, acknowledgements, and recovery actions.
- Configuration changes and operator commands, with the user ID.
- Diagnostic snapshots where useful.

Store timestamps in UTC and display local time in the interface. Save images only for low-confidence and unidentified items. Store image files separately from SQLite, retain their paths in the database, and delete the oldest images when the configured total-size or age limit is exceeded.

## 9. Fault Requirements

| Fault | Condition | Required response |
|---|---|---|
| FLT-001 | RS485 communication lost | Stop sequence progression and enter a communication fault |
| FLT-002 | Camera unavailable | Prevent new classification and untracked routing |
| FLT-003 | Classification failed | Use unidentified routing only if tracking and route conditions remain valid |
| FLT-004 | Conveyor travel failed (item still at camera station after start, or travel exceeds the maximum) | Stop conveyor and raise fault |
| FLT-005 | Plunger failed to return | Inhibit conveyor movement |
| FLT-006 | Carriage position timeout | Stop carriage motion and raise fault |
| FLT-007 | Contradictory position sensors | Inhibit press and carriage motion |
| FLT-008 | Press failed to retract | Inhibit carriage movement |
| FLT-010 | Bin full | Complete any reserved sort, then mark bin unavailable |
| FLT-011 | Ejector/service failed (ejector not Home or full sensor still active) | Keep bin unavailable pending inspection |
| FLT-013 | Interrupted cycle | Require startup recovery checks |
| FLT-014 | Database write failure | Prevent silent data loss and enter defined degraded state |
| FLT-016 | Conflicting direction commands | Block motion and report a fault |
| FLT-017 | Unidentified bin item limit reached | Pause before the next item and require an operator count reset |
| FLT-018 | Press did not leave retracted position after extend command | De-energize press valve, keep compaction pending, and raise fault |
| FLT-019 | Destination unavailable at route selection | Hold the item at the camera station, pause, and alert the operator |
| FLT-020 | Invalid bin category assignment (category missing from the active label file, or assigned to more than one bin) | Block start, show the affected bin, and require an Admin to correct the assignment |

FLT-009 (low air pressure), FLT-012 (safety circuit open), and FLT-015 (motor overload or drive fault) are reserved and unused. Those conditions are handled by hardwired circuits. Software may still see secondary effects such as plunger, press, ejector, conveyor, or carriage timeouts.

## 10. Verification

Verification must include:

- Unit tests.
- State-machine simulation.
- Hardware I/O bench tests.
- Conveyor travel-time calibration and repeatability tests.
- Vision validation on labeled samples.
- Modbus communication tests.
- Bin category assignment and routing tests, including a category swap.
- Authentication and role-permission tests.
- Guarded acceptance tests.
- Independent safety-function tests.

Test every I/O point against the final wiring diagram before enabling automatic operation.
