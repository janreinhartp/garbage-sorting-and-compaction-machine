# 04 - Hardware Design
## Garbage Sorting and Compaction Machine

**Document ID:** HW-001  
**Version:** 1.0  
**Status:** Preliminary design

## 1. Design Summary

The machine consists of a camera-equipped conveyor, three classified sorting stations, an unidentified drop point, a lead-screw compactor carriage, a pneumatic press, and bin ejectors.

The Raspberry Pi coordinates normal operation through RS485 Modbus RTU digital I/O modules.

Final wiring requires selected part numbers, manufacturer manuals, electrical calculations, pneumatic schematics, and a machine-specific risk assessment.

## 2. Main Assemblies

| Assembly | Components | Function |
|---|---|---|
| Controller | Raspberry Pi 4, power supply, storage, cooling | Runs control software |
| Vision station | Camera, mount, lighting | Captures waste images |
| Conveyor | Belt, frame, motor, drive/starter, station sensors | Moves and positions items |
| Sorting stations | Three cylinders, brackets, valves | Push items into classified bins |
| Compactor carriage | Linear rails, lead screw, coupler, AC motor | Positions the press |
| Compaction press | Cylinder, press plate, valve, feedback | Compresses waste |
| Bin ejectors | Three cylinders, valves, mechanical interfaces | Perform the defined bin-service function |
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
- Use timeouts for expected movement.
- Treat missing or contradictory feedback as a fault.
- Define the response to loss of air, loss of power, trapped air, or a broken return spring.
- Do not use valve output state as proof of cylinder position.

## 4. Conveyor and Motor Design

- Select the motor and drive/starter based on belt load, speed, acceleration, duty cycle, and stopping accuracy.
- Use station sensors or encoder feedback if timed movement is not repeatable enough.
- Provide overload and short-circuit protection.
- Prevent unexpected restart after power restoration.
- Guard belt nip points and other moving parts.
- Validate stop-and-sort repeatability with representative items.

## 5. Lead-Screw Carriage

### 5.1 Mechanical Assembly

The carriage uses linear rails and a lead screw to position the compaction press over each classified bin.

Requirements:

- The press must be retracted before carriage movement.
- Mechanical end stops must limit travel.
- Position sensors must confirm Home, Plastic, Can, and Glass positions.
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

## 6. Preliminary Digital Input Schedule

| DI | Signal | Purpose |
|---|---|---|
| 01 | Item present | Camera station detection |
| 02 | Camera station | Conveyor station confirmation |
| 03 | Plastic station | Conveyor station confirmation |
| 04 | Can station | Conveyor station confirmation |
| 05 | Glass station | Conveyor station confirmation |
| 06 | Unidentified drop station | Optional end-position confirmation |
| 07 | Plastic bin full | Laser sensor |
| 08 | Can bin full | Laser sensor |
| 09 | Glass bin full | Laser sensor |
| 10 | Carriage Home | Magnetic position sensor |
| 11 | Carriage Plastic | Magnetic position sensor |
| 12 | Carriage Can | Magnetic position sensor |
| 13 | Carriage Glass | Magnetic position sensor |
| 14 | Press retracted | Critical feedback |
| 15 | Press extended | Recommended feedback |
| 16 | Plastic plunger home | Return feedback |
| 17 | Can plunger home | Return feedback |
| 18 | Glass plunger home | Return feedback |
| 19 | Plastic ejector home | Return feedback |
| 20 | Can ejector home | Return feedback |
| 21 | Glass ejector home | Return feedback |
| 22 | Air pressure OK | Pneumatic permissive |
| 23 | Safety circuit healthy | Auxiliary status only |
| 24 | Guard closed | Auxiliary status only |
| 25 | Conveyor drive healthy | Recommended |
| 26 | Motor overload healthy | Recommended |
| 27 | Bin service/door status | If required |
| 28 | Ejection complete | If required |

A safety circuit status signal is for diagnostics. It does not replace the safety circuit's direct safety function.

## 7. Preliminary Digital Output Schedule

| DO | Signal | Purpose |
|---|---|---|
| 01 | Plastic sorting valve | Extend plastic plunger |
| 02 | Can sorting valve | Extend can plunger |
| 03 | Glass sorting valve | Extend glass plunger |
| 04 | Press valve | Extend compaction press |
| 05 | Plastic ejector valve | Extend plastic ejector |
| 06 | Can ejector valve | Extend can ejector |
| 07 | Glass ejector valve | Extend glass ejector |
| 08 | Conveyor run | Command motor interface |
| 09 | Carriage forward | Command forward movement |
| 10 | Carriage reverse | Command reverse movement |
| 11 | Alarm lamp | Fault indication |
| 12 | Buzzer | Audible alarm |
| 13 | Ready lamp | Ready indication |

Preliminary estimate: 28 DI and 13 DO. Select modules after confirming the final sensor list, voltage levels, output type, isolation, and spare-channel needs.

## 8. Electrical Design Requirements

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

## 9. RS485 Wiring

- Use suitable twisted-pair cable.
- Prefer a trunk/daisy-chain layout.
- Terminate the two physical ends of the bus when required by the devices and cable.
- Do not place a termination resistor at every device.
- Verify A/B polarity from each manufacturer's documentation.
- Follow recommendations for shield and signal-reference wiring.
- Assign unique Modbus addresses.
- Route signal cables away from motor and switching-power cables where practical.
- Test communication while motors and valves switch.

## 10. Safety Checklist

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

## 11. Commissioning Checklist

1. Inspect the mechanical assembly and guards.
2. Verify wiring against the released schematics.
3. Check power supplies and protective earth.
4. Test each digital input individually.
5. Test each output with a safe test load.
6. Verify valve direction and cylinder spring return under controlled conditions.
7. Test the E-stop and guard interlocks independently of the Pi.
8. Test motor direction interlocks and overload response.
9. Test carriage sensors and timeout behavior.
10. Test RS485 communication with all devices connected.
11. Run manual actuator tests with the machine in a controlled condition.
12. Run state-machine simulation tests.
13. Perform guarded automatic tests.
14. Record results and approved operating settings before release.
