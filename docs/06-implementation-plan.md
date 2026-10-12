# 06 - Implementation Plan
## Garbage Sorting and Compaction Machine

**Document ID:** PLAN-001  
**Version:** 1.0  
**Status:** Prototype baseline

## 1. Objective

Build and validate the prototype in stages. Verify I/O, actuator behavior, safety functions, and control sequences before enabling automatic sorting and compaction.

## 2. Development Principles

- Finalize the mechanical layout and I/O list before wiring.
- Test every hardware interface independently.
- Use simulated I/O while developing the state machine.
- Keep safety functions independent of the Raspberry Pi application.
- Process one item at a time in the initial prototype.
- Record test evidence and unresolved risks.
- Do not enable automatic operation before the safety review and commissioning checks are complete.

## 3. Milestones

### Milestone 0: Requirements and Design Freeze

Tasks:

- Review product and technical requirements.
- Confirm the mechanical layout and dimensions.
- Define the physical function of each bin ejector.
- Decide how to handle a bin becoming full while an item is in transit.
- Select the conveyor motor, carriage motor, drives/starters, cylinders, valves, sensors, I/O modules, power supplies, and safety components.
- Prepare wiring diagrams, terminal plans, and pneumatic schematics.
- Complete the machine-specific risk assessment.
- Confirm the final DI/DO list and spare-channel requirements.

Deliverables:

- Approved requirements.
- Bill of materials.
- Electrical and pneumatic schematics.
- Final I/O map.
- Risk assessment and safety test plan.

Exit criteria: no unresolved hardware decision blocks safe bench testing.

### Milestone 1: Controller and Repository Setup

Tasks:

- Install Raspberry Pi OS 64-bit.
- Configure stable power, cooling, storage, and network access.
- Create the Python project structure and virtual environment.
- Add configuration loading and structured logging.
- Set up Git, `.gitignore`, dependency pinning, and pytest.
- Create the systemd service definition.
- Implement a basic health/status endpoint.

Deliverables:

- Working controller environment.
- Version-controlled application skeleton.
- Configuration example.
- Initial unit-test workflow.

Exit criteria: the application starts, reports health, and shuts down cleanly.

### Milestone 2: RS485 and Digital I/O

Tasks:

- Wire the RS485 adapter and digital I/O modules.
- Verify baud rate, parity, slave addresses, register map, and A/B polarity.
- Implement Modbus diagnostics.
- Test every input using a known signal.
- Test every output with a safe test load.
- Verify output watchdog behavior during bus loss.
- Confirm communication-timeout and stale-data detection.
- Finalize named I/O mappings.

Deliverables:

- Modbus client.
- I/O diagnostic script or page.
- Verified I/O test table.
- Updated wiring documentation.

Exit criteria: every configured I/O point has a recorded test result.

### Milestone 3: Safety Circuit and Manual Hardware Validation

Tasks:

- Build and test the E-stop and guard-interlock circuit.
- Validate motor protection and forward/reverse interlocking.
- Verify pneumatic isolation and exhaust procedures.
- Test each pneumatic valve and spring-return cylinder individually.
- Test carriage movement at low speed.
- Confirm each carriage position sensor.
- Verify press-retracted and position-confirmed interlocks.
- Test behavior during power loss and air loss.

Deliverables:

- Safety test record.
- Manual I/O test mode.
- Verified sensor polarity and actuator behavior.

Exit criteria: required safety functions pass their documented tests before automatic sequencing is enabled.

### Milestone 4: State Machine and Fault Handling

Tasks:

- Implement states and valid transitions.
- Add start, pause, stop, acknowledgement, and recovery requests.
- Implement interlocks and sensor freshness checks.
- Add timeouts for conveyor, cylinders, carriage, press, and ejectors.
- Add interrupted-cycle detection at startup.
- Create simulated hardware interfaces.
- Write unit and simulation tests.

Deliverables:

- Tested state machine.
- Fault and recovery rules.
- Automated tests for normal and failure paths.

Exit criteria: simulated cycles and expected faults follow the defined transitions.

### Milestone 5: Conveyor and Item Tracking

Tasks:

- Implement conveyor commands and feedback.
- Validate camera and destination station sensing.
- Test stop-and-sort repeatability.
- Implement one-item tracking and unique item IDs.
- Add station timeout handling.
- Define a response to unexpected or overlapping items.

Deliverables:

- Conveyor driver.
- Station detection logic.
- Item tracking module.

Exit criteria: a single item is positioned repeatably under guarded test conditions.

### Milestone 6: Vision Classification

Tasks:

- Mount the camera and establish consistent lighting.
- Build and label a representative image dataset.
- Separate training, validation, and held-out test data.
- Train or integrate a suitable classifier.
- Implement confidence thresholding and unidentified fallback.
- Store model version and classification metadata.
- Measure per-class performance and review misclassifications.

Deliverables:

- Camera module.
- Versioned classifier.
- Vision evaluation report.
- Documented model limitations.

Exit criteria: the classifier meets the agreed prototype objective on held-out data, or its limitations are formally documented before integration.

### Milestone 7: Sorting Integration

Tasks:

- Connect classification to destination selection.
- Check bin availability before routing.
- Move to the selected station and verify position.
- Operate only the matching sorting plunger.
- Verify return feedback before restarting the conveyor.
- Route unidentified items to the conveyor end.
- Save sort results and update counters transactionally.

Deliverables:

- Automatic single-item sorting sequence.
- Item history and category counters.
- Sorting acceptance test report.

Exit criteria: repeated guarded tests demonstrate correct routing, including unknown items and fault cases.

### Milestone 8: Compaction and Bin Service

Tasks:

- Implement separate thresholds for the classified bins.
- Move the carriage to the selected bin.
- Verify position before extending the press.
- Retract the press and confirm its retracted sensor.
- Return the carriage to Home.
- Preserve pending status and counters after failed cycles.
- Implement bin-full handling.
- Implement the ejector/service sequence.
- Require verified bin availability before routing resumes.

Deliverables:

- Compaction state machine.
- Bin-service sequence.
- Compaction and service cycle records.
- Fault-injection test results.

Exit criteria: guarded cycles complete successfully and missing or contradictory feedback produces a fault.

### Milestone 9: LCD, Dashboard, Database, and Export

Tasks:

- Implement LCD pages and button navigation.
- Build dashboard status, counts, bin status, events, and fault views.
- Add validated API commands.
- Add configuration validation and audit logging.
- Implement the SQLite schema and migrations.
- Implement CSV exports and analytics.
- Restrict dashboard access to the local network.
- Verify UI handlers cannot bypass the state machine.

Deliverables:

- LCD interface.
- Local web dashboard.
- Database and export services.
- Analytics page.
- API tests.

Exit criteria: UI status matches the machine state and all commands pass validation.

### Milestone 10: Commissioning and Acceptance

Tasks:

- Inspect mechanical assembly and guarding.
- Verify wiring, terminal labels, and electrical protection.
- Test every I/O point.
- Test safety functions independently of the Pi.
- Test all routing outcomes.
- Test low-confidence and unrecognized objects.
- Test compaction thresholds and bin-service sequences.
- Test Modbus loss, sensor failure, actuator timeout, and interrupted restart.
- Verify event logs, database persistence, and CSV exports.
- Record final settings and known limitations.

Deliverables:

- Acceptance checklist.
- Final wiring and pneumatic documentation.
- Test results and issue list.
- Release candidate and backup procedure.

Exit criteria: all critical functional and safety tests pass.

## 4. Suggested Development Order

1. Requirements, risk assessment, and final I/O list.
2. Hardware selection and schematics.
3. Raspberry Pi application skeleton.
4. Modbus I/O diagnostics.
5. Safety circuit and manual actuator validation.
6. Simulated state machine and fault handling.
7. Conveyor positioning and item tracking.
8. Vision classification.
9. Automatic sorting.
10. Compaction and bin service.
11. Database, LCD, dashboard, analytics, and CSV.
12. Commissioning and acceptance.

Safety design must not be postponed until the final stage.

## 5. Test Plan

### Functional Tests

- Plastic, can, and glass route to their correct bins.
- Low-confidence and unknown items route to unidentified.
- Each sorting plunger extends and returns.
- Counters update only after the sort result is confirmed.
- Each classified bin has an independent threshold.
- The carriage reaches Home and each bin position.
- The press operates only at the selected bin.
- A full bin becomes unavailable.
- A serviced bin becomes available only after verification.
- CSV exports contain the expected records.

### Fault Tests

- Camera disconnected or capture timeout.
- RS485 communication lost.
- Sensor stuck active or inactive.
- Contradictory carriage position sensors.
- Plunger fails to return.
- Press fails to retract.
- Low air pressure.
- Carriage movement timeout.
- Motor overload or drive fault.
- Database write failure.
- Power loss during a cycle.
- Restart with an unresolved interrupted cycle.

### Safety Tests

- E-stop during conveyor movement.
- E-stop during compaction movement.
- Guard opened during hazardous motion.
- Forward/reverse conflict request.
- Press actuation requested at an invalid position.
- Carriage movement requested while the press is extended.
- Power restored after a safety trip.
- Pneumatic isolation and stored-energy maintenance procedure.

Perform tests under a controlled plan with appropriate guarding and qualified personnel. Software logs alone do not prove that a safety function passed.

## 6. Risk Register

| Risk | Impact | Mitigation |
|---|---|---|
| Vision misclassification | Wrong bin | Representative data, held-out testing, confidence threshold, unidentified fallback |
| Item overlap | Incorrect tracking | One-item feed and station sensing |
| Bin fills during transit | Jam or invalid routing | Define in-transit policy and reserve destination |
| Carriage position not confirmed | Collision or damage | Position feedback, limits, timeout |
| Press remains extended | Carriage collision | Retraction sensor and movement interlock |
| Motor direction conflict | Equipment damage or injury | Hardware-interlocked reversing interface |
| Modbus loss | Stale commands or feedback | Watchdog and tested output behavior |
| Cylinder fails to return | Jam or collision | Return feedback and timeout |
| Power loss mid-cycle | Unknown machine state | Startup recovery and inspection |
| Database failure | Lost history | Transactions, disk monitoring, backups |
| Inadequate guarding | Injury | Risk assessment and independent safety validation |

## 7. Definition of Done

A milestone is complete when:

- Deliverables are committed to version control.
- Relevant tests pass.
- Hardware changes are reflected in the I/O map and wiring documents.
- Known limitations and faults are documented.
- No unresolved critical safety issue remains before automatic operation.
