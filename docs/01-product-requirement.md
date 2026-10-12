# 01 - Product Requirements
## Garbage Sorting and Compaction Machine

**Document ID:** PRD-001  
**Version:** 1.2  
**Status:** Final

## 1. Purpose

The Garbage Sorting and Compaction Machine identifies waste items using a camera, routes them to the appropriate bin, and compacts classified waste when a configurable item-count threshold is reached.

The machine has four destinations:

- Bin 1
- Bin 2
- Bin 3
- Unidentified

Bins 1 to 3 are classified bins. Each has a waste category assigned by an Admin in the settings, chosen from the categories the active classifier can recognize. The default assignment is:

| Bin | Default category |
|---|---|
| Bin 1 | Plastic |
| Bin 2 | Can |
| Bin 3 | Glass |

Each classified bin has a dedicated sorting plunger. Unidentified items continue to the end of the conveyor and drop into a separate bin with a configurable item-count limit.

## 2. Project Objectives

1. Automate waste identification and sorting.
2. Reduce manual sorting effort.
3. Compact waste in the three classified bins.
4. Allow an Admin to change the waste category of each classified bin.
5. Detect full bins and prevent additional routing to unavailable bins.
6. Provide local physical controls and a login-protected local-network dashboard.
7. Store machine history and export records to CSV.
8. Detect faults and prevent unsafe automatic operation.

## 3. Project Scope

### 3.1 Included

- Raspberry Pi 4 Model B controller.
- Camera and classification software.
- Stop-and-sort conveyor with calibrated, timed station positioning.
- Three classified bins with Admin-assignable categories, and one unidentified bin.
- Three single-acting, spring-return sorting plungers.
- One lead-screw-driven compactor carriage on linear rails.
- AC motor with a suitable forward/reverse control interface.
- Four carriage position sensors: Home, Bin 1, Bin 2, and Bin 3.
- One single-acting, spring-return compaction press.
- Three single-acting, spring-return bin ejectors that push the compacted block out of each classified bin into a collection tray or bag.
- Three digital laser bin-full sensors.
- RS485 Modbus RTU digital input/output modules.
- Basic LCD and physical buttons.
- Local-network web dashboard with user login and Viewer, Operator, and Admin roles.
- SQLite database and CSV export.
- Event logging, fault handling, and basic analytics.

### 3.2 Excluded

- Compaction of unidentified waste.
- Ejector mechanism and full sensor for the unidentified bin.
- Conveyor station-position sensors and encoders.
- Weight measurement.
- Exact fill-percentage measurement.
- Cloud access and public internet control.
- Self-registration of dashboard users.
- Multiple overlapping items on the conveyor.
- Automatic handling of hazardous, medical, chemical, or electronic waste.
- Industrial certification or unattended commercial deployment.

## 4. Operating Workflow

1. The operator checks the machine, guards, emergency stop, air supply, and bins.
2. The operator starts the machine.
3. The conveyor moves an item to the camera station and stops.
4. The camera captures an image.
5. The classifier predicts one of the categories in the active model's label file.
6. If confidence is below the configured threshold, or no bin is assigned to the predicted category, the destination becomes unidentified.
7. Otherwise, the destination is the bin assigned to the predicted category. The controller checks whether that bin is available. If it is, the destination is reserved for the item. If it is not, the machine pauses and alerts the operator.
8. The conveyor runs for the calibrated travel time of the selected station and stops.
9. The matching pneumatic plunger extends and pushes the item into the bin.
10. The plunger returns by spring force when its valve is de-energized.
11. The controller verifies required feedback and records the result.
12. When a classified bin reaches its configured item-count threshold, compaction becomes pending.
13. The carriage moves to the selected bin, verifies its position, and runs the compaction press.
14. The press retracts before the carriage returns Home.
15. A full bin becomes unavailable. If it becomes full while an item is already travelling to it, the reserved sort completes first.
16. During bin service, the ejector pushes the compacted block into the collection tray. The bin becomes available only after the ejector returns Home, the full sensor reads clear, and the operator acknowledges.
17. Unidentified items continue to the conveyor end and drop into the unidentified bin. When its item-count limit is reached, the machine pauses until the operator empties the bin and resets the count.

### 4.1 Changing a Bin's Category

1. An Admin logs in to the dashboard and opens the bin settings page.
2. The machine must be IDLE, and the bin must be empty (total count 0 after service).
3. The Admin selects a category from the active model's label file. A category can be assigned to at most one bin.
4. The change is saved and recorded in the audit log with the Admin's user ID.
5. The operator relabels the physical bin to match.

Adding a new category, such as Paper, requires a classifier trained with that category first.

## 5. Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-001 | Classify items into the categories of the active model's label file, or unidentified. | Must |
| FR-002 | Store the predicted category and confidence score. | Must |
| FR-003 | Provide a configurable classification threshold. | Must |
| FR-004 | Route low-confidence items, and items whose category has no assigned bin, to the unidentified bin. | Must |
| FR-005 | Stop the conveyor at the camera station. | Must |
| FR-006 | Position each item at its selected sorting station using calibrated, timed conveyor movement referenced to the camera-station item sensor. | Must |
| FR-007 | Provide a dedicated sorting plunger for each classified bin. | Must |
| FR-008 | Send unidentified items to the conveyor-end destination. | Must |
| FR-009 | Verify camera-station item detection and required actuator feedback. | Must |
| FR-010 | Maintain separate item counts for all four destinations. | Must |
| FR-011 | Provide a configurable compaction threshold for each classified bin. | Must |
| FR-012 | Compact only when the required interlocks and feedback are valid. | Must |
| FR-013 | Confirm carriage position before press actuation. | Must |
| FR-014 | Confirm press retraction before carriage movement. | Must |
| FR-015 | Detect full/not-full status for each classified bin. | Must |
| FR-016 | Block routing to a full or unavailable bin. | Must |
| FR-017 | Provide a push-out ejector/service sequence for each classified bin. | Must |
| FR-018 | Confirm bin availability before routing resumes using ejector Home feedback, a clear full sensor, and operator acknowledgement. | Must |
| FR-019 | Display machine state, counts, bin categories, and faults on the LCD. | Must |
| FR-020 | Provide physical start, pause, stop, and navigation controls. | Must |
| FR-021 | Display status, counts, bin categories, bin status, and faults on the dashboard. | Must |
| FR-022 | Export selected records to CSV. | Must |
| FR-023 | Store items, settings, events, compaction cycles, and faults. | Must |
| FR-024 | Detect communication failures and actuator timeouts. | Must |
| FR-025 | Require deliberate recovery after a hazardous-motion fault. | Must |
| FR-026 | Record manual commands and configuration changes with the user ID of the person who made them. | Must |
| FR-027 | Provide throughput, category distribution, and fault summaries. | Should |
| FR-028 | Record the classifier version used for each prediction. | Should |
| FR-029 | Pause when the unidentified bin reaches its configurable item-count limit, and require an operator count reset. | Must |
| FR-030 | Reserve the destination at route selection and complete the reserved sort if the bin becomes full in transit. | Must |
| FR-031 | Reset the compaction count after successful compaction and the total bin count after successful ejection. | Must |
| FR-032 | Allow an Admin to assign a category from the active label file to each classified bin, only while the machine is IDLE and the bin is empty, with each category assigned to at most one bin. | Must |
| FR-033 | Store the bin and the category assigned at the time of each sort, so history stays correct after a category change. | Must |
| FR-034 | Require login for all dashboard pages and API endpoints except the login page. | Must |
| FR-035 | Enforce Viewer, Operator, and Admin roles for dashboard pages and commands. | Must |
| FR-036 | Allow an Admin to create, disable, and change the role of user accounts and reset their passwords. | Must |

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

### Security

- Every dashboard user has an individual account. There is no default password and no self-registration.
- Passwords are stored only as salted hashes.
- Sessions expire after a period of inactivity.
- Repeated failed logins lock the account temporarily.
- State-changing requests are protected against cross-site request forgery.
- The dashboard is reachable only on the local network.

### Usability

- The LCD must show concise status and fault messages.
- The dashboard must distinguish idle, running, paused, warning, and fault states.
- The dashboard must show the category currently assigned to each bin.
- Manual actuator testing must require maintenance mode, the Admin role, and valid interlocks.

### Performance

Initial prototype objectives:

- Classification target: at least 90% correct on a representative held-out test set covering every category in the active label file.
- Initial confidence threshold: 0.80, configurable.
- Process one tracked item at a time.
- Display current state and active faults without a manual page refresh.

Confidence is not measured accuracy. Validate accuracy using labeled test data.

## 7. Analytics

The system should report:

- Total processed items.
- Counts by category and by bin.
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

1. Test items of each assigned category reach their correct bins under agreed test conditions.
2. After an Admin swaps the categories of two empty bins, test items reach the newly assigned bins.
3. Low-confidence items, unrecognized items, and items whose category has no assigned bin reach the unidentified bin.
4. Full bins are blocked from receiving new items, and the unidentified bin pauses the machine at its item-count limit.
5. Sorting plungers return to their required positions within validated time limits.
6. The carriage reaches Home and each classified-bin position.
7. Timed conveyor movement stops items within the plunger contact zone of each station for the agreed number of consecutive test runs.
8. Compaction is scheduled at the configured threshold.
9. Failed feedback and communication loss produce visible faults.
10. Safety functions operate independently of the Raspberry Pi application.
11. LCD and dashboard counts agree with database records.
12. Records persist after restart.
13. CSV exports contain the expected records and columns.
14. Each role can use only its permitted pages and commands, and unauthenticated requests are rejected.
15. Documented functional, security, and safety tests pass.

## 9. Resolved Decisions

| Topic | Decision |
|---|---|
| Conveyor positioning | Timed movement referenced to the camera-station item sensor (DI-01). Travel time is calibrated per station and validated by repeatability tests. No station sensors. |
| Press feedback | Retracted sensor only. Extension is verified by a timer. |
| Bin ejector function | Pushes the compacted block out of the bin into a collection tray or bag. |
| Ejector completion | Ejector Home feedback, bin-full sensor clear, then operator acknowledgement on the LCD or dashboard. |
| Bin full while item in transit | The destination is reserved at route selection. The reserved sort completes, then the bin is marked unavailable. The full sensor is mounted with enough margin for one more item. |
| Destination unavailable at route selection | The machine pauses and alerts the operator. |
| Unidentified bin | Configurable item-count limit. The machine pauses at the limit until the operator empties the bin and resets the count. |
| Count resets | Compaction count resets after successful compaction. Total bin count resets after successful ejection. |
| Classifier | Transfer-learned MobileNet image classifier running on TensorFlow Lite, with a versioned label file. |
| Image retention | Save only low-confidence and unidentified images, with total-size and age limits. |
| Bin categories | Bins are identified by position (Bin 1 to 3). An Admin assigns each bin one category from the active label file, only while IDLE and the bin is empty. A category is assigned to at most one bin. Defaults: Plastic, Can, Glass. |
| Dashboard access | Login required for all pages. Roles: Viewer (read-only and CSV export), Operator (machine commands and bin service), Admin (settings, bin categories, users, model, maintenance). |
| Login implementation | Server-side sessions, argon2 password hashes, CSRF protection, temporary lockout after repeated failures, idle timeout. First Admin created from the command line. |
| LCD access | No login at the LCD. It offers Operator-level functions only. Bin category assignment and user management are web-Admin only. |

## 10. Remaining Hardware Selections

These are bill-of-materials tasks in Milestone 0 and do not change the functional design:

- Conveyor motor, drive/starter, and carriage reversing interface.
- Cylinder bore, stroke, valve voltage, and valve models.
- RS485 I/O module models and electrical ratings.
- Sensor models, polarity, and mounting.
