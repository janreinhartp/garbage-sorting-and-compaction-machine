# 06 - Implementation Plan
## Garbage Sorting and Compaction Machine

**Document ID:** PLAN-001  
**Version:** 1.3  
**Status:** Final

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

- Review product and technical requirements, including the resolved decisions in `01-product-requirement.md` section 9.
- Confirm the mechanical layout and dimensions, including the plunger contact width at each station and the ejector collection tray.
- Select the conveyor motor, carriage motor, drives/starters, cylinders, valves, sensors, I/O modules, power supplies, and safety components. Prefer a conveyor drive with repeatable start/stop behavior.
- Prepare wiring diagrams, terminal plans, and pneumatic schematics.
- Complete the machine-specific risk assessment.
- Confirm the 15 DI / 13 DO list and spare-channel requirements, with all bin-related signals named by bin position. Air pressure, safety, guards, drive healthy, and motor overload remain hardwired and are not Modbus DI.
- Size the bins, press, and ejectors for the hardest category that may be assigned to any bin.

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
- Add timeouts for conveyor, cylinders, carriage, press, and ejectors, plus the press extend timer.
- Implement destination reservation, the unidentified-bin limit, and the BIN_SERVICE state.
- Implement the category mapping service with a test label file, including assignment rules and FLT-020 validation at BOOT and PRECHECK.
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
- Validate the camera-station item sensor (DI-01) as the timing reference.
- Measure and configure the travel time from the camera station to each sorting station and to the unidentified drop.
- Run the repeatability test: at least 20 consecutive runs per station with representative items of every category in the label file. Record the stopping position of each run.
- Recalibrate if belt speed, tension, motor, drive settings, or layout change.
- Implement one-item tracking and unique item IDs.
- Add travel fault handling (DI-01 not clearing, drive fault, maximum travel time).
- Define a response to unexpected or overlapping items.

Deliverables:

- Conveyor driver with timed `move_to_station`.
- Calibrated travel-time settings.
- Repeatability test record.
- Item tracking module.

Exit criteria: every repeatability run stops the item within the plunger contact zone of its station under guarded test conditions.

### Milestone 6: Vision Classification

Tasks:

- Mount the camera and establish consistent lighting.
- Build and label a representative image dataset for each category (initially plastic, can, and glass).
- Separate training, validation, and held-out test data.
- Fine-tune a pretrained MobileNet by transfer learning on a development PC.
- Export the model to TensorFlow Lite with a matching `labels.txt`, and run it on the Raspberry Pi with `tflite-runtime`.
- Check at load time that the model output size matches the label count.
- Document the retraining procedure for adding a category, such as paper.
- Measure inference time on the Raspberry Pi.
- Implement confidence thresholding and unidentified fallback.
- Implement image retention for low-confidence and unidentified items with size and age limits.
- Store model version and classification metadata.
- Measure per-class precision, recall, and confusion matrix, and review misclassifications.

Deliverables:

- Camera module.
- Versioned `.tflite` classifier and matching label file.
- Retraining procedure.
- Image retention service.
- Vision evaluation report.
- Documented model limitations.

Exit criteria: the classifier achieves at least 90% correct classification on the held-out test set covering every label, or its limitations are formally documented before integration.

### Milestone 7: Sorting Integration

Tasks:

- Connect classification to destination selection through the category-to-bin map.
- Route categories with no assigned bin to unidentified.
- Record the destination bin and the category assigned to it at sort time.
- Check bin availability and reserve the destination before routing.
- Pause and alert the operator when the destination is unavailable.
- Move to the selected station using its calibrated travel time.
- Operate only the matching sorting plunger.
- Verify return feedback before restarting the conveyor.
- Complete a reserved sort when the bin-full sensor activates in transit, then mark the bin unavailable.
- Route unidentified items to the conveyor end and pause at the unidentified-bin limit.
- Save sort results and update counters transactionally.

Deliverables:

- Automatic single-item sorting sequence.
- Item history with per-bin and per-category counters.
- Sorting acceptance test report.

Exit criteria: repeated guarded tests demonstrate correct routing, including unknown items, a category swap between two empty bins, and fault cases.

### Milestone 8: Compaction and Bin Service

Tasks:

- Implement separate thresholds for the classified bins.
- Move the carriage to the selected bin.
- Verify position before extending the press.
- Confirm the retracted sensor clears after the extend command, then hold for the press extend time.
- Retract the press and confirm its retracted sensor.
- Return the carriage to Home.
- Reset the compaction count after successful compaction.
- Preserve pending status and counters after failed cycles.
- Implement bin-full handling.
- Implement the push-out ejector sequence: press retracted and carriage Home, extend ejector, confirm ejector Home, confirm full sensor clear, operator acknowledgement.
- Reset the total and compaction counts after successful ejection.
- Implement the unidentified-bin count reset command.
- Require verified bin availability before routing resumes.

Deliverables:

- Compaction state machine.
- Bin-service sequence.
- Compaction and service cycle records.
- Fault-injection test results.

Exit criteria: guarded cycles complete successfully and missing or contradictory feedback produces a fault.

### Milestone 9: LCD, Dashboard, Login, Database, and Export

Tasks:

- Implement the SQLite schema and migrations, including `users`, `sessions`, `login_attempts`, and `bin_category_changes`.
- Implement LCD pages and button navigation, including bin categories and the service page. Keep the LCD to Operator-level functions.
- Build the web skeleton: `base.html` layout, role-aware navigation, login and logout pages, error pages for 401 and 403, and empty placeholder pages for every route.
- Implement authentication: argon2 password hashing, server-side sessions with idle and absolute expiry, CSRF tokens, login lockout, and the `create-admin` command.
- Implement `require_role` and apply it to every page and API endpoint.
- Build the Viewer pages: dashboard status, counts per bin and category, bin status, items, events, faults, and analytics.
- Add Operator commands: start, pause, stop, fault acknowledgement, bin service and acknowledgement, and unidentified-bin reset.
- Build the Admin pages: settings, bin categories, active model, users, and maintenance.
- Add configuration validation and audit logging with user IDs.
- Implement CSV exports and analytics.
- Restrict dashboard access to the local network.
- Set the Pi hostname to `gscm`, enable Avahi/mDNS, and reserve a fixed DHCP lease so the dashboard opens as `http://gscm.local`.
- Optionally add a reverse proxy on port 80 so the URL needs no port number. Optionally enable HTTPS with a self-signed certificate.
- Do not use public-domain-style names such as `gscm.local.com`.
- Verify UI handlers cannot bypass the state machine.

Deliverables:

- LCD interface.
- Login-protected local web dashboard with Viewer, Operator, and Admin roles.
- User management and bin category settings pages.
- Database and export services.
- Analytics page.
- API, authentication, authorization, and CSRF tests.

Exit criteria: UI status matches the machine state, all commands pass validation, every route enforces its minimum role, and an Admin can change a bin's category only under the defined conditions.

### Milestone 10: Commissioning and Acceptance

Tasks:

- Inspect mechanical assembly and guarding.
- Verify wiring, terminal labels, and electrical protection.
- Test every I/O point.
- Re-verify conveyor travel-time calibration on the assembled machine.
- Test safety functions independently of the Pi.
- Test all routing outcomes.
- Swap the categories of two empty bins from the Admin page and confirm items route to the new bins.
- Test low-confidence and unrecognized objects.
- Test login, roles, and lockout on the deployed machine.
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
7. Conveyor travel-time calibration and item tracking.
8. Vision classification.
9. Automatic sorting.
10. Compaction and bin service.
11. Database, LCD, web skeleton and login, dashboard, analytics, and CSV.
12. Commissioning and acceptance.

Safety design must not be postponed until the final stage.

## 5. Test Plan

### Functional Tests

- Items of each assigned category route to their assigned bins.
- After a category swap between two empty bins, items route to the newly assigned bins.
- A category change is rejected when the machine is not IDLE, the bin is not empty, the category is not in the label file, or the category is already assigned to another bin.
- Every category change creates an audit record with the Admin user ID.
- History and analytics keep the category recorded at sort time after a category change.
- Low-confidence items, unknown items, and items whose category has no assigned bin route to unidentified.
- Each sorting plunger extends and returns.
- Counters update only after the sort result is confirmed.
- Each classified bin has an independent threshold.
- The carriage reaches Home and each bin position.
- The press operates only at the selected bin.
- A full bin becomes unavailable.
- A bin that becomes full while an item is in transit completes the reserved sort, then becomes unavailable.
- An item whose destination is unavailable at route selection stays at the camera station and the machine pauses.
- The unidentified bin pauses the machine at its item-count limit, and the operator reset clears the count.
- Timed conveyor moves stop items within the plunger contact zone of each station.
- The compaction count resets after successful compaction; the total count resets after successful ejection.
- Each ejector pushes the compacted block into the collection tray.
- A serviced bin becomes available only after ejector Home, full sensor clear, and operator acknowledgement.
- Only low-confidence and unidentified images are saved, and retention limits delete the oldest images.
- CSV exports contain the expected records.

### Fault Tests

- Camera disconnected or capture timeout.
- RS485 communication lost.
- Sensor stuck active or inactive.
- Contradictory carriage position sensors.
- Item still detected at the camera station after the conveyor starts.
- Conveyor timing drift (simulated slower belt) causes a missed station.
- Missing or out-of-range station travel time in configuration.
- Plunger fails to return.
- Press does not leave the retracted position after the extend command.
- Press fails to retract.
- Ejector fails to return Home.
- Bin-full sensor still active after ejection.
- Carriage movement timeout.
- Database write failure.
- Hardwired air-pressure, safety, guard, drive, and overload trips (verify hardware inhibits motion; software may only see secondary timeouts).
- Power loss during a cycle.
- Restart with an unresolved interrupted cycle.
- Assigned category missing from a newly selected label file (FLT-020 blocks start).
- Model output size does not match the label count.

### Security Tests

- Every page and API endpoint rejects unauthenticated requests.
- Each role is rejected from endpoints above its level (Viewer cannot start the machine, Operator cannot change settings or bin categories).
- State-changing requests without a valid CSRF token are rejected.
- The account locks after 5 failed logins and unlocks after the lockout period or by an Admin.
- Sessions expire after the idle timeout and end at logout, on account disable, and on role change.
- The last active Admin cannot be disabled or demoted.
- Passwords and session tokens never appear in the database or logs in plain text.
- The login error message is the same for an unknown user and a wrong password.

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
| Item overlap | Incorrect tracking | One-item feed and camera-station item sensing |
| Bin fills during transit | Overfilled bin or jam | Destination reserved at route selection; full sensor mounted with one-item margin; bin marked unavailable after the reserved sort |
| Conveyor timing drift | Item stops outside the plunger zone and is mis-sorted or jams | Repeatable drive, wide plunger face, calibration and repeatability tests, recalibration after changes, maximum travel limit |
| Press extension not sensed | Incomplete compaction goes undetected | Retracted sensor must clear after extend; extend timer; compaction results reviewed during commissioning |
| Unidentified bin overflow | Spill at conveyor end | Configurable item-count limit with pause and operator reset |
| Ejected block not cleared | Bin stays blocked or jams | Ejector Home feedback, full sensor clear, operator acknowledgement |
| Carriage position not confirmed | Collision or damage | Position feedback, limits, timeout |
| Press remains extended | Carriage collision | Retraction sensor and movement interlock |
| Motor direction conflict | Equipment damage or injury | Hardware-interlocked reversing interface |
| Modbus loss | Stale commands or feedback | Watchdog and tested output behavior |
| Cylinder fails to return | Jam or collision | Return feedback and timeout |
| Power loss mid-cycle | Unknown machine state | Startup recovery and inspection |
| Database failure | Lost history | Transactions, disk monitoring, backups |
| Inadequate guarding | Injury | Risk assessment and independent safety validation |
| Wrong bin category assignment | Items sorted into a bin labeled for another category | Changes only in IDLE with an empty bin, dashboard and LCD show each bin's category, physical bin relabeling step, audit log |
| New category unsuitable for the bin or press | Jam, damage, or unsafe compaction | Size hardware for the hardest assignable category; only categories in a validated model are selectable |
| Unauthorized dashboard access | Unsafe commands or changed settings | Login on every route, server-side roles, CSRF tokens, lockout, local-network binding (`gscm.local` only), audit log |
| Lost Admin password | No one can change settings | Last-Admin protection; `create-admin` command on the Raspberry Pi for recovery |

## 7. Definition of Done

A milestone is complete when:

- Deliverables are committed to version control.
- Relevant tests pass.
- Hardware changes are reflected in the I/O map and wiring documents.
- Known limitations and faults are documented.
- No unresolved critical safety issue remains before automatic operation.
