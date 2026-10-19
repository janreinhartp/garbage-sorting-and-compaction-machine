# 05 - Software Design
## Garbage Sorting and Compaction Machine

**Document ID:** SW-001  
**Version:** 1.3  
**Status:** Final

## 1. Technology Stack

| Area | Technology |
|---|---|
| Operating system | Raspberry Pi OS 64-bit |
| Language | Python |
| Machine control | Explicit state machine |
| Modbus | `pymodbus` |
| Vision capture | OpenCV |
| Classifier | Transfer-learned MobileNet on TensorFlow Lite (`tflite-runtime`) with a label file |
| Model training | TensorFlow/Keras on a development PC, exported to `.tflite` |
| Web API | FastAPI |
| Web pages | Server-rendered Jinja2 templates, CSS, and JavaScript |
| Authentication | Server-side sessions, `argon2-cffi` password hashing, CSRF tokens |
| Database | SQLite |
| Export | Python CSV module |
| Testing | pytest |
| Service management | systemd |

Pin and test dependency versions on the target Raspberry Pi.

## 2. Project Structure

```text
garbage-sorting-and-compaction-machine/
├── app/
│   ├── main.py
│   ├── config.py
│   ├── cli.py
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
│   │   ├── labels.py
│   │   ├── image_store.py
│   │   └── tracking.py
│   ├── database/
│   │   ├── models.py
│   │   ├── repository.py
│   │   └── migrations/
│   ├── services/
│   │   ├── bin_manager.py
│   │   ├── category_mapping.py
│   │   ├── user_service.py
│   │   ├── analytics.py
│   │   ├── event_logger.py
│   │   └── export_service.py
│   ├── lcd/
│   │   └── display.py
│   └── web/
│       ├── api.py
│       ├── auth.py
│       ├── dependencies.py
│       ├── pages.py
│       ├── admin.py
│       ├── templates/
│       │   ├── base.html
│       │   ├── login.html
│       │   ├── dashboard.html
│       │   ├── bins.html
│       │   ├── items.html
│       │   ├── events.html
│       │   ├── faults.html
│       │   ├── analytics.html
│       │   ├── account.html
│       │   └── admin/
│       │       ├── settings.html
│       │       ├── bin_categories.html
│       │       ├── users.html
│       │       └── maintenance.html
│       └── static/
├── config/
│   └── config.example.yaml
├── data/
│   ├── database/
│   ├── images/
│   └── exports/
├── models/
│   ├── waste_classifier.tflite
│   └── labels.txt
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
- Hardware, I/O signals, and carriage positions are named by bin position (`bin1`, `bin2`, `bin3`), never by waste category.
- The web API and LCD send requests to a command handler.
- Commands must pass authentication, role, state, and interlock validation.
- UI handlers must not write output coils directly.
- Every movement uses a timeout and defined fault response.
- Sensor data must be checked for freshness.
- Hardware interfaces must support simulated implementations for testing.
- Database writes use a repository layer and transactions.
- Faults and user actions must be logged with the user ID where one exists.

## 4. State Machine

| State | Purpose |
|---|---|
| BOOT | Load configuration, the label file, and the bin category map, and initialize services |
| RECOVERY_CHECK | Inspect saved state, I/O health, and interrupted cycles |
| IDLE | Wait for an authorized start command; Admin category changes are allowed here |
| PRECHECK | Check communication, sensors, bins, and the category map |
| ITEM_DETECT | Wait for an item at the camera station |
| CLASSIFY | Capture image and classify item |
| ROUTE_SELECT | Look up the bin for the predicted category, check availability, and reserve the destination |
| CONVEYOR_MOVE | Run the conveyor for the selected station's calibrated travel time |
| SORT_ITEM | Operate the matching sorting plunger |
| VERIFY_SORT | Confirm feedback, save the result, and update counts |
| CHECK_COMPACTION | Check pending compaction work and the unidentified-bin limit |
| MOVE_COMPACTOR | Move carriage to target bin |
| COMPACT | Operate press and verify retraction |
| RETURN_HOME | Return carriage to Home |
| BIN_SERVICE | Run the push-out ejector sequence and wait for operator acknowledgement |
| PAUSED | Hold normal sequencing |
| STOPPING | Perform the defined controlled stop |
| FAULT | Inhibit automatic progression pending recovery |

Reject commands that are invalid for the current state. The permitted transitions are defined in `03-system-architecture.md` section 5.

## 5. Interlock Rules

Implement these software checks as additional protection:

1. Do not move the carriage unless press-retracted feedback is valid.
2. Do not extend the press unless the correct carriage position is confirmed.
3. Do not command forward and reverse simultaneously.
4. Do not start another cycle until required actuator return feedback is valid.
5. Do not route items to full or unavailable bins. A destination reserved before the bin became full may complete its sort.
6. Do not rely on software for air-pressure or safety-circuit permissives; those are hardwired.
7. Do not progress using stale, missing, or contradictory sensor readings.
8. Do not automatically resume hazardous motion after a fault or restart.
9. Restrict maintenance commands to the Admin role and to states where required conditions are met.
10. Do not run an ejector unless the press is retracted and the carriage is at Home.
11. Do not accept a new item while the unidentified bin is at its item-count limit.
12. Do not change a bin's category unless the machine is IDLE and the bin's `total_count` is 0.
13. Do not start if the category map is invalid (FLT-020).
14. Do not expect Modbus inputs for air pressure, E-stop/guards, drive healthy, or motor overload. Safety and protection remain hardwired.

## 6. Vision Pipeline

### 6.1 Classifier

- Model: MobileNet (for example MobileNetV2 or V3) pretrained on ImageNet and fine-tuned by transfer learning on the project dataset.
- Classes: the categories listed in `models/labels.txt`, one per line, in the model's output order. The initial model has `plastic`, `can`, and `glass`. Unidentified is not a trained class; it is assigned by the confidence threshold, validation rules, and category map.
- Adding a category, such as `paper`, requires retraining with labeled images of that category and releasing a new model and label file together.
- Training: TensorFlow/Keras on a development PC. Export to TensorFlow Lite, optionally with integer quantization, and record the model version.
- Inference: `tflite-runtime` on the Raspberry Pi. Measure inference time on the target.
- The model file and label file are versioned together. A model is rejected at load time if its output size does not match the number of labels.

### 6.2 Runtime Steps

1. Confirm the item is stationary and visible.
2. Capture an image under consistent lighting.
3. Apply the validated preprocessing steps (crop, resize to the model input size, normalize).
4. Run the TensorFlow Lite classifier.
5. Validate the returned category and confidence.
6. Map invalid categories to unidentified.
7. Map predictions below the configured threshold to unidentified.
8. If the item is low-confidence or unidentified, save the image and record its path.
9. Store category, confidence, and model version.
10. Return the result to the state machine. Route selection then looks up the bin assigned to the category; a category with no bin routes to unidentified.

### 6.3 Image Retention

- Save only low-confidence and unidentified images for review and retraining.
- Enforce `max_mb` and `max_days` limits by deleting the oldest images first.
- Clear the image path in the database when its file is deleted.

Vision validation must include representative samples of every category in the label file under realistic conditions. Test transparent, reflective, crushed, dirty, partially obscured, and differently lit items.

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

Use named signals by bin position, such as:

- `press_retracted`
- `carriage_home`
- `carriage_bin1`
- `bin1_full`
- `bin1_sort_valve`
- `bin1_plunger_home`
- `bin1_ejector_valve`
- `carriage_forward`

Do not scatter raw channel numbers throughout application logic, and do not name signals by waste category.

### 7.3 Conveyor Interface

Recommended methods:

- `start()`
- `stop()`
- `move_to_station(station)`
- `get_feedback()`

`station` is one of `bin1`, `bin2`, `bin3`, or `unidentified`. `move_to_station(station)` requires the item at the camera station (DI-01). It starts the conveyor, checks that DI-01 clears, runs for the configured `conveyor.station_travel_ms[station]`, then stops. It returns a failure if DI-01 does not clear, the drive reports a fault, or the travel time exceeds the configured maximum.

Timed positioning is permitted only for the conveyor and only with travel times calibrated and validated in Milestone 5. Carriage and cylinder positions must always be confirmed by sensors.

### 7.4 Compactor Interface

Recommended methods:

- `move_to(position)` where `position` is `home`, `bin1`, `bin2`, or `bin3`
- `stop_motion()`
- `press_extend()`
- `press_release()`
- `return_home()`
- `get_position_feedback()`

The state machine must enforce press-retracted-before-motion and position-confirmed-before-press rules.

`press_extend()` must confirm that the press-retracted sensor clears, then hold for `timeouts.press_extend_ms`. Press extension is not sensed. `press_release()` must confirm the retracted sensor within `timeouts.press_cycle_ms`.

### 7.5 Pneumatic Interface

Each actuator method must:

- Check the allowed state and interlocks.
- Command only the required valve.
- Wait for required feedback with a timeout.
- Return an explicit success/failure result.
- Trigger the defined fault path on failure.

Output state alone does not prove cylinder movement.

## 8. Category Mapping

`services/category_mapping.py` owns the category-to-bin map.

- Loads available categories from the active label file through `vision/labels.py`.
- Reads each bin's assigned category from the `bins` table.
- `bin_for(category)` returns the assigned bin position, or `unidentified` if none.
- `assign(position, category, user)` checks the rules below, then updates the bin and writes a `bin_category_changes` record in one transaction.
- `validate()` runs at BOOT and PRECHECK and raises FLT-020 if any assigned category is missing from the label file or assigned to more than one bin.

Assignment rules:

- The machine is IDLE.
- The target bin is a classified bin (`bin1`, `bin2`, or `bin3`) and its `total_count` is 0.
- The category exists in the active label file.
- No other bin already has that category. To swap two bins, the Admin clears one first (sets it to no category), assigns the other, then assigns the first.
- The user has the Admin role.

A classified bin with no category receives no items until a category is assigned.

## 9. Authentication and Authorization

### 9.1 Roles

| Role | Permitted |
|---|---|
| Viewer | Read pages and read-only API endpoints, CSV export, own password change |
| Operator | Viewer permissions, plus start, pause, stop, fault acknowledgement, bin service and acknowledgement, and unidentified-bin reset |
| Admin | Operator permissions, plus settings, bin categories, active model, users, maintenance mode, I/O tests, and recovery |

Roles are ordered Viewer < Operator < Admin. `web/dependencies.py` provides `require_role(role)` for every route. A route without an explicit role requirement is a test failure.

### 9.2 Login and Sessions

- `POST /login` checks the username and password against the argon2 hash.
- On success, create a session with a random 256-bit token. Store only the token's SHA-256 hash in the `sessions` table. Set the cookie `session` with `HttpOnly`, `SameSite=Strict`, and `Secure` when HTTPS is enabled.
- Regenerate the session at login and delete it at logout.
- Expire a session after `web.session_idle_minutes` of inactivity and after `web.session_max_hours` in total.
- Disabling a user or changing their role ends all of their sessions.
- Use the same generic error message for an unknown user and a wrong password.

### 9.3 Lockout

- Record every login attempt in `login_attempts`.
- After `web.lockout_threshold` consecutive failures, lock the account for `web.lockout_minutes`.
- A successful login resets the failure count.
- An Admin can unlock an account.

### 9.4 CSRF

- Each session has a CSRF token.
- Forms include it as a hidden field. JavaScript requests send it in the `X-CSRF-Token` header.
- Reject any POST, PATCH, PUT, or DELETE request without a matching token.

### 9.5 Account Management

- The first Admin is created on the Raspberry Pi with `python -m app.cli create-admin`, which prompts for a username and password. There is no default account.
- Admins create users, set roles, disable or enable accounts, unlock accounts, and reset passwords.
- An Admin cannot disable or demote the last active Admin.
- Passwords must be at least 10 characters.
- Users can change their own password after entering the current one.

## 10. Database Design

Use SQLite with foreign keys enabled and transactional updates.

### 10.1 `items`

| Column | Purpose |
|---|---|
| `id` | Primary key |
| `detected_at` | UTC detection timestamp |
| `classified_at` | UTC classification timestamp |
| `completed_at` | UTC completion timestamp |
| `predicted_category` | Label from the active label file, or unidentified |
| `confidence` | Classifier score |
| `model_version` | Classifier version |
| `destination_bin_id` | Selected bin (Bin 1 to 3 or unidentified) |
| `bin_category_at_sort` | Category assigned to the destination bin when the item was sorted |
| `sort_status` | Pending, sorted, unidentified, failed |
| `image_path` | Image path for low-confidence or unidentified items; null otherwise or after retention cleanup |
| `fault_code` | Related failure code |

### 10.2 `bins`

| Column | Purpose |
|---|---|
| `id` | Primary key |
| `position` | Unique physical position: `bin1`, `bin2`, `bin3`, `unidentified` |
| `category` | Assigned category from the label file; unique when not null; null for unidentified or an unassigned bin |
| `compaction_count` | Items since the last successful compaction; classified bins only |
| `total_count` | Items since the last successful ejection or unidentified-bin reset |
| `compaction_threshold` | Threshold for classified bins; null for unidentified |
| `item_limit` | Item-count limit for the unidentified bin; null for classified bins |
| `is_full` | Last confirmed full state; for unidentified, `total_count >= item_limit` |
| `is_available` | Whether routing is permitted |
| `reserved_item_id` | Item currently reserved for this bin, if any |
| `compaction_pending` | Whether compaction is due |
| `last_compaction_at` | Last successful compaction |
| `last_service_at` | Last successful ejection or unidentified-bin reset |

Count rules:

- A confirmed sort increments `compaction_count` and `total_count` in the same transaction as the item result.
- A successful compaction resets `compaction_count` and clears `compaction_pending`.
- A successful ejection resets `total_count` and `compaction_count`, and sets `is_available` after operator acknowledgement.
- An operator reset of the unidentified bin resets its `total_count`.

### 10.3 `bin_category_changes`

Fields:

- `id`
- `bin_id`
- `old_category`
- `new_category`
- `changed_at`
- `user_id`

### 10.4 `compaction_cycles`

Fields:

- `id`
- `bin_id`
- `category_at_cycle`
- `started_at`
- `completed_at`
- `result`
- `duration_ms`
- `fault_code`
- `compaction_count_before`
- `total_count_at_cycle`

### 10.5 `service_cycles`

Fields:

- `id`
- `bin_id`
- `category_at_cycle`
- `started_at`
- `completed_at`
- `acknowledged_at`
- `result`
- `full_before` (boolean)
- `full_after` (boolean)
- `total_count_before`
- `fault_code`
- `user_id`

Unidentified-bin resets are also recorded here with no ejector cycle.

### 10.6 `users`

| Column | Purpose |
|---|---|
| `id` | Primary key |
| `username` | Unique login name |
| `password_hash` | argon2 hash |
| `role` | `viewer`, `operator`, or `admin` |
| `is_active` | Disabled accounts cannot log in |
| `failed_login_count` | Consecutive failures |
| `locked_until` | UTC time the lockout ends, if locked |
| `created_at` | UTC creation time |
| `created_by` | Admin user ID, or null for the first Admin |
| `last_login_at` | UTC time of last successful login |
| `password_changed_at` | UTC time of last password change |

### 10.7 `sessions`

| Column | Purpose |
|---|---|
| `id` | Primary key |
| `token_hash` | SHA-256 hash of the session token |
| `user_id` | Session owner |
| `csrf_token` | CSRF token for the session |
| `created_at` | UTC creation time |
| `last_seen_at` | UTC time of last request |
| `expires_at` | UTC absolute expiry |
| `ip_address` | Client address |
| `user_agent` | Client browser |

### 10.8 `login_attempts`

Fields:

- `id`
- `timestamp`
- `username`
- `user_id`, if the username exists
- `ip_address`
- `success`

### 10.9 `machine_events`

Fields:

- `id`
- `timestamp`
- `event_type`
- `state_from`
- `state_to`
- `details_json`
- `user_id`, if the event was caused by a user

### 10.10 `fault_events`

Fields:

- `id`
- `fault_code`
- `severity`
- `started_at`
- `acknowledged_at`
- `acknowledged_by`
- `cleared_at`
- `description`
- `recovery_action`

### 10.11 `settings`

Fields:

- `key`
- `value_json`
- `updated_at`
- `updated_by`
- `version`

### 10.12 `sensor_snapshots`

Fields:

- `id`
- `timestamp`
- `input_states_json`
- `output_states_json`
- `carriage_position`
- `communication_status`

### 10.13 Database Rules

- Store timestamps in UTC.
- Enable foreign keys.
- Use parameterized SQL.
- Use transactions for item completion and counter updates, and for category changes with their audit record.
- Index timestamps, categories, bin IDs, fault codes, results, usernames, and session token hashes.
- Store images as files, not database blobs.
- Preserve failed and interrupted cycle records.
- Define retention and backup procedures, including expired-session and old login-attempt cleanup.
- Do not reset counts after failed compaction or failed service.
- Never store plain-text passwords or session tokens.

## 11. Web Interface and API

### 11.1 Pages

| Page | Path | Minimum role |
|---|---|---|
| Login | `/login` | None |
| Dashboard | `/` | Viewer |
| Bins | `/bins` | Viewer |
| Items | `/items` | Viewer |
| Events | `/events` | Viewer |
| Faults | `/faults` | Viewer |
| Analytics | `/analytics` | Viewer |
| My account | `/account` | Viewer |
| Settings | `/admin/settings` | Admin |
| Bin categories | `/admin/bins` | Admin |
| Users | `/admin/users` | Admin |
| Maintenance | `/admin/maintenance` | Admin |

All pages share `base.html` with navigation that shows only the links the user's role allows, the logged-in username, the current machine state, and a logout button. Unauthenticated requests to a page redirect to `/login`. Unauthenticated API requests return 401, and requests from a role that is too low return 403.

### 11.2 Endpoints

| Method | Endpoint | Minimum role | Purpose |
|---|---|---|---|
| POST | `/login` | None | Log in |
| POST | `/logout` | Viewer | Log out |
| GET | `/api/auth/me` | Viewer | Current user and role |
| POST | `/api/auth/change-password` | Viewer | Change own password |
| GET | `/api/status` | Viewer | Current machine state and health |
| GET | `/api/counts` | Viewer | Counts by bin and category |
| GET | `/api/bins` | Viewer | Bin positions, categories, status, and availability |
| GET | `/api/categories` | Viewer | Categories in the active label file |
| GET | `/api/items` | Viewer | Filtered item history |
| GET | `/api/events` | Viewer | Recent machine events |
| GET | `/api/faults` | Viewer | Active and historical faults |
| GET | `/api/analytics/summary` | Viewer | Summary metrics |
| GET | `/api/export/items.csv` | Viewer | Export item records |
| GET | `/api/export/events.csv` | Viewer | Export event records |
| POST | `/api/commands/start` | Operator | Request start |
| POST | `/api/commands/pause` | Operator | Request pause |
| POST | `/api/commands/stop` | Operator | Request controlled stop |
| POST | `/api/commands/acknowledge` | Operator | Acknowledge eligible fault |
| POST | `/api/commands/service/{position}` | Operator | Request the ejector sequence for a classified bin |
| POST | `/api/commands/service/{position}/acknowledge` | Operator | Confirm service completion |
| POST | `/api/commands/reset-unidentified` | Operator | Reset the unidentified-bin count after emptying |
| POST | `/api/commands/recover` | Admin | Request recovery checks |
| PATCH | `/api/settings` | Admin | Update validated settings |
| PATCH | `/api/settings/bins/{position}` | Admin | Assign or clear a bin's category |
| PATCH | `/api/settings/model` | Admin | Select the active model and label file |
| GET | `/api/admin/users` | Admin | List users |
| POST | `/api/admin/users` | Admin | Create a user |
| PATCH | `/api/admin/users/{id}` | Admin | Change role, enable, disable, or unlock |
| POST | `/api/admin/users/{id}/reset-password` | Admin | Set a new password |
| POST | `/api/admin/maintenance/{action}` | Admin | Gated maintenance and I/O test actions |

Validate request bodies, audit state-changing requests with the user ID, require a CSRF token on state-changing requests, and prevent API requests from bypassing the state machine.

The application stop endpoint is not an emergency-stop substitute.

## 12. LCD and Physical Buttons

The LCD needs no login. It offers Operator-level functions only.

Suggested pages:

- Home: machine state and counts per bin with each bin's category.
- Bin status: position, category, full/available, compaction pending, and unidentified count versus limit.
- Service: request ejection, acknowledge completion, and reset the unidentified count.
- Faults: code, description, and required action.

Bin category assignment, settings, user management, and maintenance are web-Admin only.

Suggested controls:

- Start/Confirm.
- Pause/Back.
- Stop.
- Navigation controls.

Button behavior must be consistent and visible. A physical stop button connected only to the Raspberry Pi is not an emergency stop. LCD actions are logged with the source `lcd` instead of a user ID.

## 13. Analytics

Calculate:

- Total completed items.
- Count and share by category and by bin.
- Unidentified rate.
- Items processed per hour.
- Successful and failed compaction cycles.
- Cycle duration.
- Ejector/service success rate.
- Fault frequency by code.
- Running, idle, paused, and fault duration from state logs.

Use `bin_category_at_sort` for category statistics so results stay correct after a category change. Report vision performance separately using labeled test results.

## 14. Example Configuration

Create `config/config.example.yaml`:

```yaml
machine:
  mode: prototype
  one_item_at_a_time: true

vision:
  model_path: models/waste_classifier.tflite
  labels_path: models/labels.txt
  confidence_threshold: 0.80
  image_directory: data/images
  image_retention:
    mode: low_confidence_only
    max_mb: 2048
    max_days: 30

conveyor:
  station_travel_ms:     # calibrate in Milestone 5
    bin1: 0
    bin2: 0
    bin3: 0
    unidentified: 0
  max_travel_ms: 10000

bins:                    # initial values; Admin changes are stored in the database
  bin1:
    category: plastic
    compaction_threshold: 10
  bin2:
    category: can
    compaction_threshold: 10
  bin3:
    category: glass
    compaction_threshold: 10
  unidentified:
    item_limit: 30

timeouts:
  modbus_ms: 1000
  carriage_move_ms: 15000
  sorting_plunger_ms: 3000
  press_extend_ms: 3000
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

web:
  host: 0.0.0.0          # bind to the local network interface
  port: 8000
  public_hostname: gscm.local
  session_secret_env: GSCM_SESSION_SECRET
  session_idle_minutes: 30
  session_max_hours: 12
  lockout_threshold: 5
  lockout_minutes: 15
  https:
    enabled: false
    cert_path: ""
    key_path: ""
```

These are initial values only. Confirm serial settings and timings against the selected hardware before operation. Configuration validation must reject a zero or missing station travel time and any travel time above `max_travel_ms`, so automatic operation cannot start before calibration.

The `bins` values seed the database on first start. After that, the database is authoritative for bin categories, thresholds, and the unidentified limit, and Admins change them from the dashboard. The session secret is read from an environment variable and is never stored in the repository.

Operators open the dashboard at `http://gscm.local` (or `http://gscm.local:8000` if no reverse proxy is used). The name is LAN-only via mDNS. It is not a public internet domain.

## 15. Testing Strategy

- Unit tests for state transitions, interlocks, configuration, counters, and fault mapping.
- Unit tests for category mapping: label loading, `bin_for`, assignment rules, duplicate rejection, IDLE and empty-bin checks, and FLT-020 validation.
- Simulation tests for normal cycles, sensor failures, timeouts, communication loss, restart recovery, bin full during transit, unidentified-bin limit, bin service, and a category swap between runs.
- Hardware integration tests for every DI/DO point.
- Conveyor travel-time calibration and repeatability tests.
- Vision tests using a labeled held-out dataset covering every label.
- Database tests for transactions and persistence.
- Authentication tests: login success and failure, generic error message, lockout and unlock, session idle and absolute expiry, logout, session end on disable or role change, and last-Admin protection.
- Authorization tests: every route rejects unauthenticated requests and roles below its minimum, and no route lacks a role requirement.
- CSRF tests: state-changing requests without a valid token are rejected.
- API tests for invalid commands and state restrictions.
- Guarded machine acceptance tests.
- Independent safety-function tests for E-stop, guard interlocks, motor interlocks, and energy isolation.
