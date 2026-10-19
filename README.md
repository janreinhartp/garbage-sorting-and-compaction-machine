# Garbage Sorting and Compaction Machine

A prototype waste sorting machine using camera-based classification, a conveyor, pneumatic sorting plungers, a lead-screw compactor, and a Raspberry Pi controller.

## 1. Project Overview

The machine sorts waste into four destinations: three classified bins (Bin 1, Bin 2, Bin 3) and one unidentified bin.

Each classified bin's waste category is a setting that an Admin can change from the dashboard, chosen from the categories the classifier was trained on. The defaults are:

| Bin | Default category |
|---|---|
| Bin 1 | Plastic |
| Bin 2 | Can |
| Bin 3 | Glass |

For example, an Admin can swap Bin 1 and Bin 2, or assign Paper to a bin once the classifier has been retrained with Paper. Categories can be changed only while the machine is idle and the bin is empty.

The conveyor stops at the camera station for classification. Classified items move to their assigned sorting station using calibrated, timed conveyor movement, and a pneumatic plunger pushes the item into its bin. Unidentified items continue to the end of the conveyor into a bin with a configurable item-count limit.

A lead-screw carriage positions one pneumatic press over the three classified bins. Each classified bin has a configurable compaction threshold, a digital laser full/not-full sensor, and an ejector that pushes the compacted block into a collection tray or bag.

## 2. Main Features

- Camera-based waste classification with a TensorFlow Lite MobileNet model and label file.
- Admin-assignable waste category for each classified bin.
- Stop-and-sort conveyor control with calibrated travel times.
- Three dedicated pneumatic sorting plungers.
- Unidentified waste routing.
- Lead-screw-driven compaction carriage.
- Four carriage position sensors.
- Pneumatic compaction press.
- Three push-out bin ejectors.
- Full-bin detection and an unidentified-bin item limit.
- RS485 Modbus RTU digital I/O.
- LCD and physical controls.
- Local-network dashboard with user login and Viewer, Operator, and Admin roles at `http://gscm.local`.
- SQLite database and CSV export.
- Analytics and fault history.
- Timeouts, interlocks, and interrupted-cycle recovery.

## 3. Hardware

- Raspberry Pi 4 Model B.
- Camera and suitable lighting.
- Conveyor, motor, drive/starter, and one camera-station item sensor.
- Three single-acting, spring-return sorting cylinders, one per bin.
- One AC motor-driven lead-screw carriage.
- Four carriage position sensors: Home, Bin 1, Bin 2, and Bin 3.
- One single-acting, spring-return compaction press with a retracted sensor.
- Three single-acting, spring-return bin ejectors with Home sensors and a collection tray or bag.
- Seven suitable 3/2 pneumatic valves, subject to circuit validation.
- Three digital laser bin-full sensors.
- RS485 Modbus RTU DI/DO modules.
- LCD and physical buttons.
- Suitable motor protection, emergency stop, guard interlocks, and pneumatic isolation.

The final bill of materials depends on mechanical loads, cylinder dimensions, motor ratings, valve voltage, sensor electrical type, and safety requirements.

## 4. Software Stack

| Component | Technology |
|---|---|
| Operating system | Raspberry Pi OS 64-bit |
| Main language | Python |
| Camera | OpenCV-compatible camera interface |
| Classifier | Transfer-learned MobileNet on TensorFlow Lite |
| Machine control | State machine |
| Field communication | Modbus RTU |
| Web API | FastAPI |
| Dashboard | Jinja2 templates, HTML, CSS, JavaScript |
| Authentication | Server-side sessions, argon2 password hashing, CSRF tokens |
| Database | SQLite |
| Export | CSV |
| Testing | pytest |
| Service management | systemd |

## 5. Repository Structure

```text
garbage-sorting-and-compaction-machine/
├── app/
│   ├── main.py
│   ├── config.py
│   ├── cli.py
│   ├── controller/
│   ├── hardware/
│   ├── vision/
│   ├── database/
│   ├── services/
│   ├── lcd/
│   └── web/
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
│   ├── 01-product-requirement.md
│   ├── 02-technical-requirement.md
│   ├── 03-system-architecture.md
│   ├── 04-hardware-design.md
│   ├── 05-software-design.md
│   └── 06-implementation-plan.md
├── requirements.txt
├── .gitignore
└── README.md
```

## 6. Documentation

Read the documents in this order:

1. `01-product-requirement.md`: goals, scope, workflow, and acceptance criteria.
2. `02-technical-requirement.md`: hardware/software requirements, I/O mapping, and fault handling.
3. `03-system-architecture.md`: software layers, state machine, communication, and safety boundaries.
4. `04-hardware-design.md`: mechanical, electrical, pneumatic, and I/O design.
5. `05-software-design.md`: code structure, database, API, configuration, and testing.
6. `06-implementation-plan.md`: development milestones, tests, and risks.

## 7. I/O Count

| Type | Count |
|---|---:|
| Digital inputs | 15 |
| Digital outputs | 13 |

A 16-channel output module is a possible starting point. Input capacity must cover 15 inputs plus spare channels. Air pressure, E-stop/guards, drive protection, and motor overload are hardwired and are not counted as Modbus DI. Confirm module electrical ratings, isolation, watchdog behavior, register maps, and spare channels before purchasing.

## 8. Initial Prototype Settings

These are starting values for testing. They are not validated operating limits.

| Setting | Initial value |
|---|---:|
| Vision confidence threshold | 0.80 |
| Bin 1 / Bin 2 / Bin 3 category | Plastic / Can / Glass |
| Bin 1, 2, and 3 compaction threshold | 10 items each |
| Unidentified bin item limit | 30 items |
| Sensor debounce | 50 ms |
| Modbus timeout | 1000 ms |
| Conveyor travel time per station | Calibrate in Milestone 5 |
| Conveyor maximum travel time | 10 seconds |
| Carriage movement timeout | 15 seconds |
| Sorting plunger timeout | 3 seconds |
| Press extend time | 3 seconds |
| Press cycle timeout | 10 seconds |
| Ejector cycle timeout | 10 seconds |
| Image retention | Low-confidence and unidentified only, 2048 MB / 30 days |
| Session idle timeout | 30 minutes |
| Login lockout | 15 minutes after 5 failed attempts |

## 9. Development Setup

The following setup assumes Raspberry Pi OS 64-bit.

### 9.1 Install system packages

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git
```

Install any additional operating-system packages required by the selected camera, display, and RS485 adapter.

### 9.2 Create the virtual environment

```bash
cd garbage-sorting-and-compaction-machine
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

### 9.3 Create `requirements.txt`

Start with the libraries required by your selected hardware and software:

```text
fastapi
uvicorn
jinja2
python-multipart
itsdangerous
argon2-cffi
pymodbus
opencv-python
tflite-runtime
numpy
PyYAML
pytest
httpx
```

Pin the versions after testing them together on the Raspberry Pi. Additional dependencies depend on the camera and display. Train the classifier with TensorFlow/Keras on a development PC; the Raspberry Pi only needs `tflite-runtime`.

Install the dependencies:

```bash
pip install -r requirements.txt
```

### 9.4 Configure the machine

1. Copy `config/config.example.yaml` to a local configuration file.
2. Set the correct serial port and Modbus parameters.
3. Verify each DI/DO assignment against the tested I/O map.
4. Configure movement timeouts based on measured hardware behavior.
5. Set the calibrated conveyor travel time for each station. Automatic operation is rejected until these are set.
6. Check the initial bin categories. After first start, Admins change them from the dashboard.
7. Set a random session secret in the `GSCM_SESSION_SECRET` environment variable, for example in the systemd service's environment file.
8. Set the Pi hostname to `gscm`, enable Avahi/mDNS, and reserve a fixed DHCP lease on the router so clients open `http://gscm.local` (or `http://gscm.local:8000` without a reverse proxy). Do not use `gscm.local.com`.
9. Keep credentials and machine-specific secrets out of source control.

### 9.5 Create the first Admin account

There is no default login. Create the first Admin on the Raspberry Pi:

```bash
python -m app.cli create-admin
```

The command prompts for a username and a password of at least 10 characters. Open `http://gscm.local`, log in with this account, then create Operator and Viewer accounts from the Users page. Run the same command again to recover access if every Admin password is lost.

The dashboard stays on the local network only. `gscm.local` is a friendly LAN hostname (mDNS), not a public internet site.

### 9.6 Run tests

```bash
pytest
```

Run simulation tests before connecting live actuators. Test outputs with safe loads before connecting motor and pneumatic interfaces.

## 10. Safety Requirements

- Do not operate without appropriate guarding and a completed risk assessment.
- Do not use the Raspberry Pi or dashboard as the only emergency-stop mechanism.
- Do not energize forward and reverse commands simultaneously.
- Do not move the carriage unless press-retracted feedback is valid.
- Do not extend the press unless the intended carriage position is confirmed.
- Do not route an item to a full or unavailable bin.
- Do not automatically resume hazardous motion after a fault or restart.
- Verify remote I/O behavior during RS485 communication loss.
- Isolate electrical and pneumatic energy before maintenance.
- Use qualified personnel for electrical work and safety validation.
- Relabel the physical bin whenever an Admin changes its category.
- Give the Admin role only to people responsible for machine settings.

## 11. Development Phases

1. Finalize requirements, mechanical design, I/O, and risk assessment.
2. Build the Raspberry Pi application skeleton.
3. Validate RS485 and every digital I/O point.
4. Test the independent safety circuit.
5. Validate manual motor and pneumatic operation.
6. Implement and simulate the state machine.
7. Calibrate conveyor travel times and integrate item tracking.
8. Integrate camera classification.
9. Integrate automatic sorting.
10. Integrate compaction and bin service.
11. Add the database, LCD, web skeleton and login, dashboard, analytics, and CSV export.
12. Complete commissioning and acceptance tests.

## 12. Design Decisions

Functional design decisions are resolved and recorded in `docs/01-product-requirement.md` section 9.

The remaining hardware selections are Milestone 0 bill-of-materials tasks:

- Conveyor motor and drive/starter with repeatable start/stop behavior.
- AC carriage motor and reversing interface.
- Cylinder bore, stroke, force, and mounting.
- Valve voltage and 3/2 valve configuration.
- RS485 DI/DO module models and electrical output types.
- Sensor models, polarity, and mounting.

Measured classifier performance is recorded in the Milestone 6 vision evaluation report.

## 13. Prototype Acceptance

Before accepting the prototype:

- Test all four routing outcomes.
- Swap the categories of two empty bins and confirm items follow the new assignment.
- Test login, role permissions, and lockout.
- Test conveyor stopping repeatability at each station.
- Test each pneumatic actuator and its required return feedback.
- Test all four carriage positions.
- Test each compaction threshold.
- Test full-bin blocking, bin full during transit, the unidentified-bin limit, and the service sequence.
- Test communication loss and sensor faults.
- Test interrupted-cycle recovery.
- Test the emergency stop and guard interlocks independently of the Pi.
- Verify database persistence, analytics, and CSV export.
- Record validated settings and remaining limitations.
