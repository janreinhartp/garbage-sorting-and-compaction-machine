# 01 - Product Requirements
## Garbage Sorting and Compaction Machine

**Document ID:** PRD-001  
**Version:** 1.0  
**Status:** Prototype baseline

## 1. Purpose

The Garbage Sorting and Compaction Machine identifies waste items using a camera, routes them to the appropriate bin, and compacts classified waste when a configurable item-count threshold is reached.

The machine supports four destinations:

- Plastic
- Can
- Glass
- Unidentified

Plastic, can, and glass use dedicated sorting plungers. Unidentified items continue to the end of the conveyor and drop into a separate bin.

## 2. Project Objectives

1. Automate waste identification and sorting.
2. Reduce manual sorting effort.
3. Compact waste in the three classified bins.
4. Detect full bins and prevent additional routing to unavailable bins.
5. Provide local physical controls and a local-network dashboard.
6. Store machine history and export records to CSV.
7. Detect faults and prevent unsafe automatic operation.

## 3. Project Scope

### 3.1 Included

- Raspberry Pi 4 Model B controller.
- Camera and classification software.
- Stop-and-sort conveyor.
- Three classified bins and one unidentified bin.
- Three single-acting, spring-return sorting plungers.
- One lead-screw-driven compactor carriage on linear rails.
- AC motor with a suitable forward/reverse control interface.
- Four carriage position sensors: Home, Plastic, Can, and Glass.
- One single-acting, spring-return compaction press.
- Three single-acting, spring-return bin ejectors.
- Three digital laser bin-full sensors.
- RS485 Modbus RTU digital input/output modules.
- Basic LCD and physical buttons.
- Local-network web dashboard.
- SQLite database and CSV export.
- Event logging, fault handling, and basic analytics.

### 3.2 Excluded

- Compaction of unidentified waste.
- Ejector mechanism for the unidentified bin.
- Weight measurement.
- Exact fill-percentage measurement.
- Cloud access and public internet control.
- Multiple overlapping items on the conveyor.
- Automatic handling of hazardous, medical, chemical, or electronic waste.
- Industrial certification or unattended commercial deployment.

## 4. Operating Workflow

1. The operator checks the machine, guards, emergency stop, air supply, and bins.
2. The operator starts the machine.
3. The conveyor moves an item to the camera station and stops.
4. The camera captures an image.
5. The classifier predicts plastic, can, glass, or unidentified.
6. If confidence is below the configured threshold, the destination becomes unidentified.
7. For a classified item, the controller checks whether the destination bin is available.
8. The conveyor moves the item to the selected station and stops.
9. The matching pneumatic plunger extends and pushes the item into the bin.
10. The plunger returns by spring force when its valve is de-energized.
11. The controller verifies required feedback and records the result.
12. When a classified bin reaches its configured item-count threshold, compaction becomes pending.
13. The carriage moves to the selected bin, verifies its position, and runs the compaction press.
14. The press retracts before the carriage returns Home.
15. A full bin becomes unavailable until its service procedure is completed and availability is confirmed.
16. Unidentified items continue to the conveyor end and drop into the unidentified bin.

## 5. Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-001 | Classify items as plastic, can, glass, or unidentified. | Must |
| FR-002 | Store the predicted category and confidence score. | Must |
| FR-003 | Provide a configurable classification threshold. | Must |
| FR-004 | Route low-confidence items to the unidentified bin. | Must |
| FR-005 | Stop the conveyor at the camera station. | Must |
| FR-006 | Position each item at its selected sorting station. | Must |
| FR-007 | Provide a dedicated sorting plunger for each classified bin. | Must |
| FR-008 | Send unidentified items to the conveyor-end destination. | Must |
| FR-009 | Verify required station and actuator feedback. | Must |
| FR-010 | Maintain separate item counts for all four destinations. | Must |
| FR-011 | Provide a configurable compaction threshold for each classified bin. | Must |
| FR-012 | Compact only when the required interlocks and feedback are valid. | Must |
| FR-013 | Confirm carriage position before press actuation. | Must |
| FR-014 | Confirm press retraction before carriage movement. | Must |
| FR-015 | Detect full/not-full status for each classified bin. | Must |
| FR-016 | Block routing to a full or unavailable bin. | Must |
| FR-017 | Provide an ejector/service sequence for each classified bin. | Must |
| FR-018 | Confirm bin availability before routing resumes. | Must |
| FR-019 | Display machine state, counts, and faults on the LCD. | Must |
| FR-020 | Provide physical start, pause, stop, and navigation controls. | Must |
| FR-021 | Display status, counts, bin status, and faults on the dashboard. | Must |
| FR-022 | Export selected records to CSV. | Must |
| FR-023 | Store items, settings, events, compaction cycles, and faults. | Must |
| FR-024 | Detect communication failures and actuator timeouts. | Must |
| FR-025 | Require deliberate recovery after a hazardous-motion fault. | Must |
| FR-026 | Record manual commands and configuration changes. | Should |
| FR-027 | Provide throughput, category distribution, and fault summaries. | Should |
| FR-028 | Record the classifier version used for each prediction. | Should |

## 6. Non-Functional Requirements

### Safety

- Emergency-stop and guard-interlock functions must use a suitable hardwired safety circuit or safety controller.
- Raspberry Pi software and ordinary Modbus I/O must not be the only means of stopping hazardous motion.
- The carriage must not move while the press is extended.
- The press must not extend unless the carriage position is confirmed.
- Forward and reverse motor commands must have hardware interlocking.
- A restart must not automatically resume hazardous motion.

### Reliability

- Critical movements must use timeouts.
- Missing or contradictory feedback must prevent sequence progression.
- Database updates for item results and counters must be transactional.
- Interrupted cycles must be detected after restart.

### Usability

- The LCD must show concise status and fault messages.
- The dashboard must distinguish idle, running, paused, warning, and fault states.
- Manual actuator testing must require maintenance mode and valid interlocks.

### Performance

Initial prototype objectives:

- Classification target: at least 90% correct on a representative held-out test set.
- Initial confidence threshold: 0.80, configurable.
- Process one tracked item at a time.
- Display current state and active faults without a manual page refresh.

Confidence is not measured accuracy. Validate accuracy using labeled test data.

## 7. Analytics

The system should report:

- Total processed items.
- Counts by category.
- Unidentified item rate.
- Throughput over a selected period.
- Compaction cycle count, duration, and result.
- Ejector/service cycle count and result.
- Fault frequency by code.
- Running, idle, paused, and fault duration where reliable logs exist.
- Classification metrics from a labeled validation dataset.

Do not report item mass without a weighing sensor. Digital laser sensors provide full/not-full status, not exact fill percentage.

## 8. Acceptance Criteria

The prototype is ready for acceptance when:

1. Plastic, can, and glass test items reach their correct bins under agreed test conditions.
2. Low-confidence and unrecognized items reach the unidentified bin.
3. Full bins are blocked from receiving new items.
4. Sorting plungers return to their required positions within validated time limits.
5. The carriage reaches Home and each classified-bin position.
6. Compaction is scheduled at the configured threshold.
7. Failed feedback and communication loss produce visible faults.
8. Safety functions operate independently of the Raspberry Pi application.
9. LCD and dashboard counts agree with database records.
10. Records persist after restart.
11. CSV exports contain the expected records and columns.
12. Documented functional and safety tests pass.

## 9. Open Decisions

- Final motor, drive/starter, and reversing interface.
- Final cylinder dimensions, valve voltage, and valve models.
- Exact RS485 I/O module models and electrical ratings.
- Conveyor station-position sensing method.
- Whether the press has both extended and retracted feedback.
- Exact mechanical purpose of each bin ejector.
- Behavior when a bin becomes full while an item is already in transit.
- Whether ejector completion is sensed or confirmed manually.
