# Garbage Sorting and Compaction Machine

A prototype waste sorting machine using camera-based classification, a conveyor, pneumatic sorting plungers, a lead-screw compactor, and a Raspberry Pi controller.

## 1. Project Overview

The machine classifies waste into four destinations:

- Plastic
- Can
- Glass
- Unidentified

The conveyor stops at the camera station for classification. Classified items move to their assigned sorting station, where a pneumatic plunger pushes the item into its bin. Unidentified items continue to the end of the conveyor.

A lead-screw carriage positions one pneumatic press over the three classified bins. Each classified bin has a configurable compaction threshold and a digital laser full/not-full sensor.

## 2. Main Features

- Camera-based waste classification.
- Stop-and-sort conveyor control.
- Three dedicated pneumatic sorting plungers.
- Unidentified waste routing.
- Lead-screw-driven compaction carriage.
- Four carriage position sensors.
- Pneumatic compaction press.
- Three bin ejectors.
- Full-bin detection.
- RS485 Modbus RTU digital I/O.
- LCD and physical controls.
- Local-network dashboard.
- SQLite database and CSV export.
- Analytics and fault history.
- Timeouts, interlocks, and interrupted-cycle recovery.

## 3. Hardware

- Raspberry Pi 4 Model B.
- Camera and suitable lighting.
- Conveyor, motor, and drive/starter.
- Three single-acting, spring-return sorting cylinders.
- One AC motor-driven lead-screw carriage.
- Four carriage position sensors: Home, Plastic, Can, and Glass.
- One single-acting, spring-return compaction press.
- Three single-acting, spring-return bin ejectors.
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
| Machine control | State machine |
| Field communication | Modbus RTU |
| Web API | FastAPI |
| Dashboard | HTML, CSS, JavaScript |
| Database | SQLite |
| Export | CSV |
| Testing | pytest |
| Service management | systemd |

## 5. Repository Structure

```text
garbage-sorting-system/
├── app/
│   ├── main.py
│   ├── config.py
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

## 7. Initial I/O Estimate

| Type | Preliminary count |
|---|---:|
| Digital inputs | 28 |
| Digital outputs | 13 |

A 16-channel output module is a possible starting point. Input capacity must cover the final sensor list. Confirm module electrical ratings, isolation, watchdog behavior, register maps, and spare channels before purchasing.

## 8. Initial Prototype Settings

These are starting values for testing. They are not validated operating limits.

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
cd garbage-sorting-system
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

### 9.3 Create `requirements.txt`

Start with the libraries required by your selected hardware and software:

```text
fastapi
uvicorn
pymodbus
opencv-python
PyYAML
pytest
httpx
```

Pin the versions after testing them together on the Raspberry Pi. Additional dependencies depend on the camera, display, and classifier framework.

Install the dependencies:

```bash
pip install -r requirements.txt
```

### 9.4 Configure the machine

1. Copy `config/config.example.yaml` to a local configuration file.
2. Set the correct serial port and Modbus parameters.
3. Verify each DI/DO assignment against the tested I/O map.
4. Configure movement timeouts based on measured hardware behavior.
5. Keep credentials and machine-specific secrets out of source control.

### 9.5 Run tests

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

## 11. Development Phases

1. Finalize requirements, mechanical design, I/O, and risk assessment.
2. Build the Raspberry Pi application skeleton.
3. Validate RS485 and every digital I/O point.
4. Test the independent safety circuit.
5. Validate manual motor and pneumatic operation.
6. Implement and simulate the state machine.
7. Integrate conveyor positioning and item tracking.
8. Integrate camera classification.
9. Integrate automatic sorting.
10. Integrate compaction and bin service.
11. Add the database, LCD, dashboard, analytics, and CSV export.
12. Complete commissioning and acceptance tests.

## 12. Decisions to Finalize

- Conveyor motor and drive/starter.
- AC carriage motor and reversing interface.
- Cylinder bore, stroke, force, and mounting.
- Valve voltage and 3/2 valve configuration.
- RS485 DI/DO module models and electrical output types.
- Sensor models, polarity, and mounting.
- Conveyor positioning method.
- Behavior when a bin fills while an item is in transit.
- Physical function and completion criteria of the ejectors.
- Image retention policy.
- Final classifier and measured test performance.

## 13. Prototype Acceptance

Before accepting the prototype:

- Test all four routing outcomes.
- Test each pneumatic actuator and its required return feedback.
- Test all four carriage positions.
- Test each compaction threshold.
- Test full-bin blocking and the service sequence.
- Test communication loss and sensor faults.
- Test interrupted-cycle recovery.
- Test the emergency stop and guard interlocks independently of the Pi.
- Verify database persistence, analytics, and CSV export.
- Record validated settings and remaining limitations.
