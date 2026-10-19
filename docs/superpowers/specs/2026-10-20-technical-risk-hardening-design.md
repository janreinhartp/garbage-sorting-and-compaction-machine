# Technical Risk Hardening Design

**Date:** 2026-10-20  
**Status:** Approved for documentation update  
**Scope:** Keep the full product vision; reduce delivery risk without changing selected components  
**Related docs:** `docs/01`–`06`, `README.md` (updated to v1.4)

## 1. Goal

Keep the Garbage Sorting and Compaction Machine as specified (camera classify, stop-and-sort, three plungers, unidentified drop, lead-screw compaction, ejectors, LCD, role-based LAN dashboard). Change **how** the project is built and gated so the weakest failure modes are caught early.

Non-goals:

- Cutting features for an MVP
- Replacing the already-chosen conveyor motor/drive
- Adding a second camera for positioning
- Switching the primary sequence to gate-while-running plungers
- Expanding the DI/DO map in the default build

## 2. Decisions Locked

| Topic | Decision |
|---|---|
| Optimization | Technical risk reduction, full feature set retained |
| Components | No component changes in the default plan |
| Conveyor sequence | Stop-and-sort remains primary (stop at station, then plunger) |
| Second camera | Out of scope for positioning and for verify-after-stop in v1 |
| Gate-while-running | Not the primary sequence; not scheduled as a parallel experiment |
| Station sensors | Not in the default BOM; last-resort only if Milestone 5 fails after recalibration |
| Press feedback | Remain retracted-sensor-only; strengthen software checks and commissioning review |
| Persistence | Pull core SQLite schema and repositories earlier than the old Milestone 9 blob |
| Dashboard work | Split auth/persistence from full UI polish so recovery and counters are testable earlier |

## 3. Conveyor Positioning Strategy

### 3.1 Primary method (unchanged hardware)

1. Item stops at the camera station (DI-01).
2. Classify and reserve destination.
3. Run the conveyor for the calibrated travel time of the selected station.
4. Stop.
5. Fire only the matching sorting plunger.
6. Confirm plunger home before the next conveyor move.

### 3.2 Risk controls without new parts

- Size the plunger contact zone for the measured stop scatter of the chosen drive.
- Treat Milestone 5 as a **hard gate**: automatic sorting (Milestone 7) must not start until the repeatability test passes.
- Keep software guards: DI-01 must clear after start; maximum travel time; reject zero/missing station travel times before auto start.
- Enforce one tracked item at a time.
- Recalibrate after any belt speed, tension, drive setting, or layout change.

### 3.3 Last-resort contingency (documented only)

If Milestone 5 fails after recalibration and layout tolerance adjustments, the project may later add station presence sensors and expand the DI map. That change requires a new BOM revision and is **not** part of the default build.

## 4. Press and Compaction Hardening

No new sensors. Keep:

- Press retracted sensor must clear after extend command (FLT-018).
- Configured press extend time, then retract with timeout.
- Carriage must not move unless press retracted.
- Press must not extend unless carriage position is confirmed.

Add process discipline:

- During commissioning, review successful and failed compaction records against physical results.
- Treat incomplete compaction as a commissioning defect even if software reported success, and tighten timers/thresholds from those observations.

## 5. Revised Milestone Order

| Milestone | Focus | Gate |
|---|---|---|
| 0 | Requirements freeze, BOM, schematics, risk assessment | No unresolved hardware decision blocks bench testing |
| 1 | Pi skeleton, config, logging, pytest, systemd, health endpoint, **core SQLite schema + repositories** (items, bins, events, faults, settings) | App starts/stops cleanly; DB migrations apply |
| 2 | RS485 and every DI/DO | Every I/O point recorded |
| 3 | Safety circuit and manual actuator validation | Safety tests pass before auto sequencing |
| 4 | State machine, interlocks, faults, category mapping, **recovery using persisted state**, simulation tests | Simulated cycles match defined transitions |
| 5 | Conveyor timed positioning and item tracking | **Hard gate:** ≥20 consecutive in-zone stops per station |
| 6 | Vision dataset, TFLite model, retention, evaluation report | ≥90% held-out accuracy or documented limitations |
| 7 | Automatic sorting integration | Guarded routing tests pass; requires M5 pass |
| 8 | Compaction and bin service | Guarded cycles and fault injection pass |
| 9 | Auth, users/sessions, LCD, web skeleton, role-aware dashboard, analytics, CSV, `gscm.local` | Every route enforces role; UI matches machine state |
| 10 | Commissioning and acceptance | Critical functional and safety tests pass |

Rationale for moving persistence earlier: interrupted-cycle recovery, counters, and category history must not wait until the full dashboard lands.

## 6. Documentation Updates

This design is reflected in:

- `README.md` — development phases and positioning notes
- `docs/01-product-requirement.md` — resolved decisions
- `docs/02-technical-requirement.md` — conveyor gate and contingency note
- `docs/03-system-architecture.md` — sequence and non-goals for alternate positioning
- `docs/04-hardware-design.md` — contact-zone and calibration expectations
- `docs/05-software-design.md` — early persistence, M5 validation rules
- `docs/06-implementation-plan.md` — reordered milestones and hard gates

## 7. Out of Scope Alternatives (Rejected for Default Plan)

- Second camera as primary or verify-after-stop positioner
- Continuous belt with plunger-as-gate as the main sorting sequence
- Immediate addition of station photoeyes without an M5 failure
- Changing the chosen conveyor drive

## 8. Next Step

After user review of this spec and the v1.4 doc set, create a detailed implementation plan with the writing-plans skill, then begin Milestone 1 implementation.
