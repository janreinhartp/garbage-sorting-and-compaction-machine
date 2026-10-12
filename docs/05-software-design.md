# 05 - Software Design
## Garbage Sorting and Compaction Machine

**Document ID:** SW-001  
**Version:** 1.0  
**Status:** Prototype baseline

## 1. Technology Stack

| Area | Technology |
|---|---|
| Operating system | Raspberry Pi OS 64-bit |
| Language | Python |
| Machine control | Explicit state machine |
| Modbus | `pymodbus` |
| Vision | OpenCV and a trained classifier |
| Web API | FastAPI |
| Web interface | HTML, CSS, JavaScript |
| Database | SQLite |
| Export | Python CSV module |
| Testing | pytest |
| Service management | systemd |

Pin and test dependency versions on the target Raspberry Pi.

## 2. Project Structure

```text
garbage-sorting-system/
├── app/
│   ├── main.py
│   ├── config.py
│   ├── controller/
│   │   ├── state_machine.py
│   │   ├── command_handler.py
│   │   ├── safety_monitor.py
│   │   ├── recovery.py
│   │   └── interlocks.py
│   ├── hardware/
│   │   ├── modbus_client.py
│   │   ├── io_mapping.py
│   │   ├── conveyor.py
│   │   ├── compactor.py
│   │   ├── pneumatics.py
│   │   └── sensors.py
│   ├── vision/
│   │   ├── camera.py
│   │   ├── classifier.py
│   │   └── tracking.py
│   ├── database/
│   │   ├── models.py
│   │   ├── repository.py
│   │   └── migrations/
│   ├── services/
│   │   ├── analytics.py
│   │   ├── event_logger.py
│   │   └── export_service.py
│   ├── lcd/
│   │   └── display.py
│   └── web/
│       ├── api.py
│       ├── templates/
│       └── static/
├── config/
│   └── config.example.yaml
├── data/
│   ├── database/
│   ├── images/
│   └── exports/
├── logs/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── simulation/
│   └── acceptance/
├── docs/
├── requirements.txt
├── .gitignore
└── README.md
```

## 3. Design Principles

- One state machine owns automatic sequencing.
- One Modbus service owns serial-bus access.
- Hardware drivers expose commands and feedback.
- The web API and LCD send requests to a command handler.
- Commands must pass state and interlock validation.
- UI handlers must not write output coils directly.
- Every movement uses a timeout and defined fault response.
- Sensor data must be checked for freshness.
- Hardware interfaces must support simulated implementations for testing.
- Database writes use a repository layer and transactions.
- Faults and operator actions must be logged.

## 4. State Machine

| State | Purpose |
|---|---|
| BOOT | Load configuration and initialize services |
| RECOVERY_CHECK | Inspect saved state, I/O health, and interrupted cycles |
| IDLE | Wait for an authorized start command |
| PRECHECK | Check safety status, communication, sensors, pressure, and bins |
| ITEM_DETECT | Wait for an item at the camera station |
| CLASSIFY | Capture image and classify item |
| ROUTE_SELECT | Select destination and check availability |
| CONVEYOR_MOVE | Move item to selected station |
| SORT_ITEM | Operate the matching sorting plunger |
| VERIFY_SORT | Confirm feedback and save the result |
| CHECK_COMPACTION | Check pending compaction work |
| MOVE_COMPACTOR | Move carriage to target bin |
| COMPACT | Operate press and verify retraction |
| RETURN_HOME | Return carriage to Home |
| BIN_SERVICE | Execute the approved bin-service sequence |
| PAUSED | Hold normal sequencing |
| STOPPING | Perform the defined controlled stop |
| FAULT | Inhibit automatic progression pending recovery |

Reject commands that are invalid for the current state.

## 5. Interlock Rules

Implement these software checks as additional protection:

1. Do not move the carriage unless press-retracted feedback is valid.
2. Do not extend the press unless the correct carriage position is confirmed.
3. Do not command forward and reverse simultaneously.
4. Do not start another cycle until required actuator return feedback is valid.
5. Do not route items to full or unavailable bins.
6. Do not run pneumatic cycles when air pressure is below the validated threshold.
7. Do not progress using stale, missing, or contradictory sensor readings.
8. Do not automatically resume hazardous motion after a fault or restart.
9. Restrict maintenance commands when required conditions are not met.
10. Treat safety-circuit feedback as diagnostic information. The safety hardware must perform the safety function directly.

## 6. Vision Pipeline

1. Confirm the item is stationary and visible.
2. Capture an image under consistent lighting.
3. Apply the validated preprocessing steps.
4. Run the classifier.
5. Validate the returned category and confidence.
6. Map invalid categories to unidentified.
7. Map predictions below the configured threshold to unidentified.
8. Store category, confidence, model version, and image path if image retention is enabled.
9. Return the result to the state machine.

Vision validation must include representative plastic, can, and glass samples under realistic conditions. Test transparent, reflective, crushed, dirty, partially obscured, and differently lit items.

Measure per-class precision, recall, and confusion matrix using labeled test data. Do not treat confidence as measured accuracy.

If the camera fails, do not continue moving an item whose destination is unknown unless a defined safe recovery path is available.

## 7. Hardware Interfaces

### 7.1 Modbus Client

Responsibilities:

- Open and maintain the serial connection.
- Read configured input registers or coils.
- Write approved output states.
- Read back outputs where supported.
- Detect timeouts, CRC errors, stale data, and bus loss.
- Serialize requests.
- Expose communication health and last-success timestamps.
- Apply a documented reconnection policy.

The Modbus client does not decide when to sort or compact.

### 7.2 I/O Mapping

Keep physical channel assignments in one configuration module.

Use named signals such as:

- `press_retracted`
- `carriage_home`
- `plastic_bin_full`
- `plastic_sort_valve`
- `carriage_forward`

Do not scatter raw channel numbers throughout application logic.

### 7.3 Conveyor Interface

Recommended methods:

- `start()`
- `stop()`
- `move_to_station(station)`
- `get_feedback()`

Use station sensors or validated position feedback. A fixed delay alone is not reliable proof of position.

### 7.4 Compactor Interface

Recommended methods:

- `move_to(position)`
- `stop_motion()`
- `press_extend()`
- `press_release()`
- `return_home()`
- `get_position_feedback()`

The state machine must enforce press-retracted-before-motion and position-confirmed-before-press rules.

### 7.5 Pneumatic Interface

Each actuator method must:

- Check the allowed state and interlocks.
- Command only the required valve.
- Wait for required feedback with a timeout.
- Return an explicit success/failure result.
- Trigger the defined fault path on failure.

Output state alone does not prove cylinder movement.

## 8. Database Design

Use SQLite with foreign keys enabled and transactional updates.

### 8.1 `items`

| Column | Purpose |
|---|---|
| `id` | Primary key |
| `detected_at` | UTC detection timestamp |
| `classified_at` | UTC classification timestamp |
| `completed_at` | UTC completion timestamp |
| `predicted_category` | Plastic, can, glass, unidentified |
| `confidence` | Classifier score |
| `model_version` | Classifier version |
| `destination` | Selected destination |
| `sort_status` | Pending, sorted, unidentified, failed |
| `image_path` | Optional image path |
| `fault_code` | Related failure code |

### 8.2 `bins`

| Column | Purpose |
|---|---|
| `id` | Primary key |
| `category` | Unique bin category |
| `item_count` | Count since the defined reset point |
| `compaction_threshold` | Threshold for classified bins |
| `is_full` | Last confirmed full state |
| `is_available` | Whether routing is permitted |
| `compaction_pending` | Whether compaction is due |
| `last_compaction_at` | Last successful compaction |
| `last_discharge_at` | Last successful service cycle |

### 8.3 `compaction_cycles`

Fields:

- `id`
- `bin_id`
- `started_at`
- `completed_at`
- `result`
- `duration_ms`
- `fault_code`
- `items_before_cycle`
- `items_after_cycle`

### 8.4 `discharge_cycles`

Fields:

- `id`
- `bin_id`
- `started_at`
- `completed_at`
- `result`
- `fill_before`
- `fill_after`
- `fault_code`
- `operator_id`, if available

### 8.5 `machine_events`

Fields:

- `id`
- `timestamp`
- `event_type`
- `state_from`
- `state_to`
- `details_json`
- `user_id`, if available

### 8.6 `fault_events`

Fields:

- `id`
- `fault_code`
- `severity`
- `started_at`
- `acknowledged_at`
- `cleared_at`
- `description`
- `recovery_action`

### 8.7 `settings`

Fields:

- `key`
- `value_json`
- `updated_at`
- `updated_by`
- `version`

### 8.8 `sensor_snapshots`

Fields:

- `id`
- `timestamp`
- `input_states_json`
- `output_states_json`
- `carriage_position`
- `communication_status`

### 8.9 Database Rules

- Store timestamps in UTC.
- Enable foreign keys.
- Use parameterized SQL.
- Use transactions for item completion and counter updates.
- Index timestamps, categories, bin IDs, fault codes, and results.
- Store images as files, not database blobs.
- Preserve failed and interrupted cycle records.
- Define retention and backup procedures.
- Do not reset counts after failed compaction.

## 9. Web API

Suggested endpoints:

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/status` | Current machine state and health |
| GET | `/api/counts` | Counts by category |
| GET | `/api/bins` | Bin status and availability |
| GET | `/api/items` | Filtered item history |
| GET | `/api/events` | Recent machine events |
| GET | `/api/faults` | Active and historical faults |
| GET | `/api/analytics/summary` | Summary metrics |
| GET | `/api/export/items.csv` | Export item records |
| GET | `/api/export/events.csv` | Export event records |
| POST | `/api/commands/start` | Request start |
| POST | `/api/commands/pause` | Request pause |
| POST | `/api/commands/stop` | Request controlled stop |
| POST | `/api/commands/acknowledge` | Acknowledge eligible fault |
| POST | `/api/commands/recover` | Request recovery checks |
| PATCH | `/api/settings` | Update validated settings |

These endpoints are design proposals. Validate request bodies, restrict access, audit state-changing commands, and prevent API requests from bypassing the state machine.

The application stop endpoint is not an emergency-stop substitute.

## 10. LCD and Physical Buttons

Suggested pages:

- Home: machine state and category counts.
- Bin status: full/available and compaction pending.
- Faults: code, description, and required action.
- Settings: permitted operator settings.
- Maintenance: gated I/O diagnostics and actuator tests.

Suggested controls:

- Start/Confirm.
- Pause/Back.
- Stop.
- Navigation controls.

Button behavior must be consistent and visible. A physical stop button connected only to the Raspberry Pi is not an emergency stop.

## 11. Analytics

Calculate:

- Total completed items.
- Count and share by category.
- Unidentified rate.
- Items processed per hour.
- Successful and failed compaction cycles.
- Cycle duration.
- Ejector/service success rate.
- Fault frequency by code.
- Running, idle, paused, and fault duration from state logs.

Report vision performance separately using labeled test results.

## 12. Example Configuration

Create `config/config.example.yaml`:

```yaml
machine:
  mode: prototype
  one_item_at_a_time: true

vision:
  confidence_threshold: 0.80
  save_images: true
  image_directory: data/images

bins:
  plastic:
    compaction_threshold: 10
  can:
    compaction_threshold: 10
  glass:
    compaction_threshold: 10

timeouts:
  modbus_ms: 1000
  conveyor_move_ms: 10000
  carriage_move_ms: 15000
  sorting_plunger_ms: 3000
  press_cycle_ms: 10000
  ejector_cycle_ms: 10000

sensors:
  debounce_ms: 50

modbus:
  port: /dev/serial0
  baudrate: 9600
  parity: N
  stopbits: 1
  bytesize: 8
  timeout_seconds: 1.0
```

These are initial values only. Confirm serial settings and timings against the selected hardware before operation.

## 13. Testing Strategy

- Unit tests for state transitions, interlocks, configuration, counters, and fault mapping.
- Simulation tests for normal cycles, sensor failures, timeouts, communication loss, and restart recovery.
- Hardware integration tests for every DI/DO point.
- Vision tests using a labeled held-out dataset.
- Database tests for transactions and persistence.
- API tests for invalid commands and access restrictions.
- Guarded machine acceptance tests.
- Independent safety-function tests for E-stop, guard interlocks, motor interlocks, and energy isolation.
