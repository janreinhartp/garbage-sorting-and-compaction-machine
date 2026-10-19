# 04 - Hardware Design
## Garbage Sorting and Compaction Machine

**Document ID:** HW-001  
**Version:** 1.3  
**Status:** Final (design baseline)

## 1. Design Summary

The machine consists of a camera-equipped conveyor, three classified sorting stations, an unidentified drop point, a lead-screw compactor carriage, a pneumatic press, and bin ejectors.

The classified bins are named by position: Bin 1, Bin 2, and Bin 3, in order along the conveyor from the camera station. Their waste categories are software settings, so all hardware, wiring labels, and I/O names use the bin number. Each physical bin should carry a removable or rewritable category label so it can be relabeled when an Admin changes its category.

The Raspberry Pi coordinates normal operation through RS485 Modbus RTU digital I/O modules.

Final wiring requires selected part numbers, manufacturer manuals, electrical calculations, pneumatic schematics, and a machine-specific risk assessment.

## 2. Main Assemblies

| Assembly | Components | Function |
|---|---|---|
| Controller | Raspberry Pi 4, power supply, storage, cooling | Runs control software |
| Vision station | Camera, mount, lighting | Captures waste images |
| Conveyor | Belt, frame, motor, drive/starter, camera-station item sensor | Moves and positions items by calibrated travel time |
| Sorting stations | Three cylinders, brackets, valves | Push items into classified bins |
| Compactor carriage | Linear rails, lead screw, coupler, AC motor | Positions the press |
| Compaction press | Cylinder, press plate, valve, retracted sensor | Compresses waste |
| Bin ejectors | Three cylinders, valves, push plates, Home sensors, collection tray or bag | Push the compacted block out of each classified bin |
| Bin sensing | Three digital laser sensors | Detect full/not-full status |
| Field I/O | RS485 DI/DO modules and suitable drivers | Interfaces with sensors and actuators |
| Safety | E-stop, guard switches, safety controller/relay | Inhibits hazardous motion |
| Operator interface | LCD, buttons, lamps, buzzer | Local status and commands |

## 3. Pneumatic Design

### 3.1 Cylinder Configuration

All seven cylinders are specified as single-acting, spring-return cylinders:

- Three sorting plungers.
- One compaction press.
- Three bin ejectors.

Each cylinder is assumed to use one suitable 3/2 solenoid valve.

Expected operation:

1. Energize the valve to supply air and extend the cylinder.
2. De-energize the valve to exhaust the actuator port.
3. The cylinder spring returns the piston.

Verify the selected valve's port arrangement and the cylinder's required operating conditions. A spring-return cylinder is not guaranteed to return safely under every load, orientation, pressure, or mechanical failure.

### 3.2 Pneumatic Components

Select and verify:

- Cylinder bore, stroke, force, mounting, and duty cycle.
- Valve voltage, flow, port size, pressure rating, and manual override.
- Air filter and regulator.
- Pressure gauge and pressure switch.
- Tubing, fittings, flow controls, and silencers.
- Manifold capacity.
- Isolation and exhaust provisions.
- Mechanical guarding and pinch-point protection.

Calculate available cylinder force from pressure and effective piston area. Account for friction, linkage geometry, material resistance, and the loads imposed on the frame, press plate, and bin.

### 3.3 Pneumatic Feedback

- Verify critical cylinder return positions using suitable sensors.
- Every cylinder has a Home or retracted sensor. No cylinder has an extended sensor in the default build.
- The press is extended for a configured time. The retracted sensor must clear after the extend command and return after release.
- Use timeouts for expected movement.
- Treat missing or contradictory feedback as a fault.
- During commissioning, compare compaction cycle records with the physical compacted result and tighten press timers if incomplete compaction is observed.
- Define the response to loss of air, loss of power, trapped air, or a broken return spring.
- Do not use valve output state as proof of cylinder position.

## 4. Conveyor and Motor Design

- The conveyor motor and drive are already selected for this prototype. Work within their measured start/stop behavior; do not assume a drive change for positioning accuracy.
- Primary sequence is stop-and-sort: the belt stops at the sorting station before the plunger extends. Do not design the plungers as continuous-belt gates for the default build.
- Position items by timed movement. A single item-present sensor at the camera station (DI-01) is the timing reference. There are no station sensors in the default I/O map.
- Do not add a second camera for conveyor positioning.
- Make each plunger face wide enough along the belt to tolerate the measured stopping variation of the chosen drive.
- Calibrate the travel time to each station and the unidentified drop, and recalibrate after mechanical or drive changes.
- Milestone 5 is a hard gate: at least 20 consecutive guarded runs per station must stop inside the plunger contact zone before automatic sorting is enabled. If that gate fails after recalibration and contact-zone adjustments, station presence sensors may be considered as a later BOM revision only.
- Provide overload and short-circuit protection.
- Prevent unexpected restart after power restoration.
- Guard belt nip points and other moving parts.
- Validate stop-and-sort repeatability with representative items of each category.

## 5. Lead-Screw Carriage

### 5.1 Mechanical Assembly

The carriage uses linear rails and a lead screw to position the compaction press over each classified bin.

Requirements:

- The press must be retracted before carriage movement.
- Mechanical end stops must limit travel.
- Position sensors must confirm Home, Bin 1, Bin 2, and Bin 3 positions.
- Design the bins, press plate, and compaction force for the hardest waste category that may be assigned to any bin, because categories can be swapped.
- Use additional limit switches or drive limits where required.
- The press and carriage must not collide with bins, guards, or other structures.
- The frame and lead screw must withstand the selected motor and press loads.

### 5.2 AC Motor Reversing Interface

Use a suitable reversing starter or drive with:

- Correct motor voltage, phase, current, and duty rating.
- Electrical and mechanical interlocking where contactors are used.
- Overload and short-circuit protection.
- A defined stop/dead-time sequence before reversing.
- Drive fault feedback where available.
- A suitable safety function selected through risk assessment.

Ordinary relays controlled by a Raspberry Pi are not sufficient as the sole means of safely reversing an AC motor.

### 5.3 Position Logic

- Confirm the expected target sensor before press actuation.
- Treat unexpected combinations of position sensors as a fault.
- Stop carriage motion when the target position is confirmed.
- Use a timeout if the target is not reached.
- Do not use elapsed time alone as proof of carriage position.

## 6. Bin Ejectors

- Each classified bin has one ejector cylinder with a push plate.
- Extending the ejector pushes the compacted block out of the bin into a collection tray or bag.
- The ejector returns by spring force and is confirmed by its Home sensor.
- The bin-full sensor must read clear after ejection.
- The operator acknowledges completion before the bin becomes available.
- Run the ejector only with the press retracted and the carriage at Home.
- Guard the discharge opening and collection area against crush and ejection hazards.

## 7. Digital Input Schedule

| DI | Signal | Purpose |
|---|---|---|
| 01 | Item present | Camera station detection and conveyor timing reference |
| 02 | Bin 1 full | Laser sensor |
| 03 | Bin 2 full | Laser sensor |
| 04 | Bin 3 full | Laser sensor |
| 05 | Carriage Home | Magnetic position sensor |
| 06 | Carriage Bin 1 | Magnetic position sensor |
| 07 | Carriage Bin 2 | Magnetic position sensor |
| 08 | Carriage Bin 3 | Magnetic position sensor |
| 09 | Press retracted | Critical feedback |
| 10 | Bin 1 plunger home | Return feedback |
| 11 | Bin 2 plunger home | Return feedback |
| 12 | Bin 3 plunger home | Return feedback |
| 13 | Bin 1 ejector home | Return and service feedback |
| 14 | Bin 2 ejector home | Return and service feedback |
| 15 | Bin 3 ejector home | Return and service feedback |

Mount each bin-full laser sensor with enough margin for one more item, so a reserved sort can finish after the sensor activates.

Air-pressure permissive, emergency stop, guard interlocks, conveyor drive protection, and motor overload protection are hardwired. They are not Modbus digital inputs to the Raspberry Pi.

## 8. Digital Output Schedule

| DO | Signal | Purpose |
|---|---|---|
| 01 | Bin 1 sorting valve | Extend Bin 1 plunger |
| 02 | Bin 2 sorting valve | Extend Bin 2 plunger |
| 03 | Bin 3 sorting valve | Extend Bin 3 plunger |
| 04 | Press valve | Extend compaction press |
| 05 | Bin 1 ejector valve | Extend Bin 1 ejector |
| 06 | Bin 2 ejector valve | Extend Bin 2 ejector |
| 07 | Bin 3 ejector valve | Extend Bin 3 ejector |
| 08 | Conveyor run | Command motor interface |
| 09 | Carriage forward | Command forward movement |
| 10 | Carriage reverse | Command reverse movement |
| 11 | Alarm lamp | Fault indication |
| 12 | Buzzer | Audible alarm |
| 13 | Ready lamp | Ready indication |

Total: 15 DI and 13 DO. Select modules after confirming voltage levels, output type, isolation, and spare-channel needs.

## 9. Electrical Design Requirements

- Never connect field voltage directly to Raspberry Pi GPIO.
- Check whether the remote outputs are relay, sourcing, or sinking type.
- Use suitable interposing relays or drivers when required.
- Separate logic power from noisy motor and actuator circuits where practical.
- Protect circuits with correctly rated fuses or breakers.
- Suppress inductive transients according to manufacturer guidance.
- Label all wires, terminals, modules, sensors, and actuators.
- Provide protective earth, enclosure bonding, cable strain relief, and suitable cable segregation.
- Use terminals and conductors rated for the expected current.
- Provide accessible electrical isolation for maintenance.
- Follow applicable local electrical requirements.

## 10. RS485 Wiring

- Use suitable twisted-pair cable.
- Prefer a trunk/daisy-chain layout.
- Terminate the two physical ends of the bus when required by the devices and cable.
- Do not place a termination resistor at every device.
- Verify A/B polarity from each manufacturer's documentation.
- Follow recommendations for shield and signal-reference wiring.
- Assign unique Modbus addresses.
- Route signal cables away from motor and switching-power cables where practical.
- Test communication while motors and valves switch.

## 11. Safety Checklist

- [ ] Complete a machine-specific risk assessment.
- [ ] Identify crush, shear, pinch, entanglement, and ejection hazards.
- [ ] Install an accessible emergency stop.
- [ ] Guard conveyor, lead screw, carriage, press, and ejector hazard zones.
- [ ] Use guard interlocks where required.
- [ ] Implement suitable safety-rated removal of hazardous motion/energy.
- [ ] Prevent carriage movement unless the press is retracted.
- [ ] Prevent press extension unless the correct carriage position is confirmed.
- [ ] Prevent simultaneous forward/reverse commands in hardware.
- [ ] Provide overload and short-circuit protection.
- [ ] Address stored pneumatic energy and unexpected spring-return movement.
- [ ] Require deliberate reset after a safety trip.
- [ ] Test behavior during power loss, air loss, bus loss, and restart.
- [ ] Document isolation and lockout procedures for maintenance.
- [ ] Validate safety functions before automatic operation.

## 12. Commissioning Checklist

1. Inspect the mechanical assembly and guards.
2. Verify wiring against the released schematics.
3. Check power supplies and protective earth.
4. Test each digital input individually.
5. Test each output with a safe test load.
6. Verify valve direction and cylinder spring return under controlled conditions.
7. Test the E-stop and guard interlocks independently of the Pi.
8. Test motor direction interlocks and overload response.
9. Test carriage sensors and timeout behavior.
10. Calibrate conveyor travel times and run the repeatability test for each station.
11. Test RS485 communication with all devices connected.
12. Run manual actuator tests with the machine in a controlled condition.
13. Run state-machine simulation tests.
14. Perform guarded automatic tests.
15. Record results and approved operating settings before release.
