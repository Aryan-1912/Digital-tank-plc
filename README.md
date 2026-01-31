# Digital Tank Fill/Drain Control System

**Platform:** RSLogix Micro Starter Lite (MicroLogix 1000/1100)  
**Author:** Aryan  
**Application:** Automated Tank Level Control with Fill and Drain Modes

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [System Architecture](#system-architecture)
3. [Fluid Mechanics in Tanks](#fluid-mechanics-in-tanks)
4. [Timer Logic Fundamentals](#timer-logic-fundamentals)
5. [Ladder Logic Structure](#ladder-logic-structure)
6. [I/O Mapping](#io-mapping)
7. [Operational Modes](#operational-modes)
8. [Troubleshooting](#troubleshooting)

---

## Project Overview

This PLC program implements an automated tank level control system that manages the filling and draining of a process tank. The system uses digital level sensors (high/low limit switches) combined with timer-based control logic to prevent overflow, run-dry conditions, and pump damage.

### Key Features

- Automatic fill/drain mode selection
- High and low level protection
- Timer-controlled pump cycling
- Mode switching with interlock logic

---

## System Architecture

The system consists of a process tank with two digital level sensors (high and low), a fill valve connected to the supply, and a drain valve connected to the process output. The PLC monitors sensor states and controls valve/pump outputs based on the selected operational mode.

---

## Fluid Mechanics in Tanks

Understanding the physics behind tank filling and draining is essential for proper timer calibration and system design.

### Conservation of Mass Principle

The fundamental equation governing tank level dynamics is the **mass balance equation**:

```
Rate of Accumulation = Rate of Inflow - Rate of Outflow

        dV/dt = Q_in - Q_out
```

Where:
- `dV/dt` = Rate of change of volume in tank (m³/s or L/s)
- `Q_in` = Volumetric flow rate entering the tank
- `Q_out` = Volumetric flow rate leaving the tank

### Tank Level Dynamics

For a cylindrical tank with constant cross-sectional area `A`:

```
        A × (dh/dt) = Q_in - Q_out
```

Where:
- `A` = Cross-sectional area of tank (m²)
- `h` = Liquid height/level (m)
- `dh/dt` = Rate of level change (m/s)

#### Fill Mode (Q_out = 0)

When only filling:
```
        dh/dt = Q_in / A

        Time to fill = (h_high - h_low) × A / Q_in
```

**Example Calculation:**
- Tank diameter: 1 meter → A = π(0.5)² = 0.785 m²
- Fill flow rate: 0.01 m³/s (10 L/s)
- Level change needed: 0.5 m (from low to high sensor)

```
        Fill Time = (0.5 m × 0.785 m²) / 0.01 m³/s
                  = 39.25 seconds
```

#### Drain Mode (Q_in = 0)

Draining behavior depends on whether the outlet is gravity-fed or pump-assisted.

**Gravity Drain (Torricelli's Law):**
```
        Q_out = Cd × A_outlet × √(2gh)
```

Where:
- `Cd` = Discharge coefficient (typically 0.6-0.65)
- `A_outlet` = Outlet area
- `g` = Gravitational acceleration (9.81 m/s²)
- `h` = Liquid height above outlet

This creates **non-linear drain behavior** — draining slows as the tank empties because head pressure decreases.

**Pump-Assisted Drain:**
```
        Q_out = constant (pump rated flow)
        Drain Time = V_liquid / Q_pump
```

### Why This Matters for Timer Logic

| Scenario | Timer Consideration |
|----------|---------------------|
| Fill cycle | Timer preset = calculated fill time + safety margin |
| Drain cycle | Timer preset accounts for variable flow (gravity) or constant flow (pump) |
| Pump protection | Minimum OFF time prevents motor overheating |
| Anti-cavitation | Delay before drain ensures pump is primed |

---

## Timer Logic Fundamentals

RSLogix 500 provides three primary timer instructions critical for tank control.

### TON (Timer On-Delay)

**Purpose:** Delays turning ON an output after input goes true. When the input (enable) goes true, the timer begins accumulating. Once the accumulated value reaches the preset, the Done (DN) bit energizes. When the input goes false, the timer resets (accumulator returns to zero) and the DN bit de-energizes.

**Tank Application:**
- **Fill Timeout:** If level doesn't reach high sensor within expected time, fault condition triggers
- **Pump Run Timer:** Track total pump runtime for maintenance scheduling

### TOF (Timer Off-Delay)

**Purpose:** Delays turning OFF an output after input goes false. When the input is true, the output is immediately ON and the timer is held at zero. When the input goes false, the timer begins counting. The output remains ON until the timer reaches preset, then the DN bit energizes and output turns OFF.

**Tank Application:**
- **Drain Delay:** Keep drain valve open briefly after pump stops to clear lines
- **Anti-Short-Cycle:** Prevent rapid pump cycling

### RTO (Retentive Timer On-Delay)

**Purpose:** Accumulates time across multiple enable cycles. Unlike TON, the RTO does not reset when the input goes false — it retains the accumulated value. For example, if the input is ON for 5 seconds, then OFF, then ON for 3 more seconds, the accumulator will show 8 seconds. A separate RES (reset) instruction is required to clear the accumulator.

**Tank Application:**
- **Total Fill Time Tracking:** Measure cumulative fill time across interrupted cycles
- **Batch Processing:** Track total volume transferred over multiple fill cycles

### Timer Resolution in MicroLogix

| Time Base | Resolution | Max Preset |
|-----------|------------|------------|
| 0.01 sec  | 10 ms      | 327.67 sec |
| 0.1 sec   | 100 ms     | 3276.7 sec |
| 1.0 sec   | 1 sec      | 32767 sec  |

---

## Ladder Logic Structure

### Rung-by-Rung Breakdown

#### Rung 0: Mode Selection

A selector switch (I:0/0) determines operational mode. When Mode_Select is ON, the Fill_Mode bit (B3:0/0) is energized. When Mode_Select is OFF, the Drain_Mode bit (B3:0/1) is energized via an XIO (examine if open) instruction.

#### Rung 1: Fill Mode Control

This rung controls the Fill_Valve (O:0/0) using the following logic: Fill_Mode (B3:0/0) must be active AND Low_Level sensor (I:0/1) must be true AND High_Level sensor (I:0/2) must be false. A parallel seal-in branch using the Fill_Valve output keeps the valve energized once activated, allowing it to remain on even if the Low_Level sensor state changes during filling. The valve de-energizes when either Fill_Mode goes false OR High_Level becomes true.

#### Rung 2: Fill Timer (Timeout Protection)

A TON timer (T4:0) is enabled whenever Fill_Valve (O:0/0) is energized, with a preset of 6000 (60 seconds at 0.01s time base). If the timer's Done bit (T4:0/DN) becomes true before the High_Level sensor stops the fill, a Fill_Fault output (O:0/3) is energized. This detects blocked supply lines, faulty level sensors, or tank leaks.

#### Rung 3: Drain Mode Control

This rung controls the Drain_Valve (O:0/1) using similar logic to fill mode: Drain_Mode (B3:0/1) must be active AND High_Level sensor (I:0/2) must be true AND Low_Level sensor (I:0/1) must be false. A seal-in branch maintains drain operation during the level transition zone. Draining stops when the Low_Level sensor de-energizes (tank empty).

#### Rung 4: Pump Interlock

The Pump_Run output (O:0/2) energizes when either Fill_Valve OR Drain_Valve is open, but includes interlock logic to prevent both valves from being open simultaneously. This is achieved by examining if the opposite valve is closed (XIO instruction) before allowing either valve to run the pump. This prevents hydraulic conflict and protects the pump from damage.

#### Rung 5: Anti-Short-Cycle Timer

A TOF timer (T4:1) is triggered when Pump_Run (O:0/2) goes false, with a preset of 300 (3 seconds). The Enable_Change bit (B3:0/2) only becomes true when both the timer's Done bit is active AND a valid mode selection exists. This prevents rapid mode switching that could damage the pump motor — industry standard recommends minimum 3-5 second delay between motor starts.

---

## I/O Mapping

### Inputs (I:0)

| Address | Description | Type | Normal State |
|---------|-------------|------|--------------|
| I:0/0 | Mode Select Switch | Digital | NO (Fill when closed) |
| I:0/1 | Low Level Sensor (LS_L) | Digital | NO (Closed when submerged) |
| I:0/2 | High Level Sensor (LS_H) | Digital | NO (Closed when submerged) |
| I:0/3 | Emergency Stop | Digital | NC (Opens to stop) |

### Outputs (O:0)

| Address | Description | Type | Action |
|---------|-------------|------|--------|
| O:0/0 | Fill Valve/Pump | Digital | Energize to fill |
| O:0/1 | Drain Valve/Pump | Digital | Energize to drain |
| O:0/2 | Main Pump Contactor | Digital | Master pump control |
| O:0/3 | Fault Indicator | Digital | Illuminates on fault |

### Internal Bits (B3:0)

| Address | Description |
|---------|-------------|
| B3:0/0 | Fill Mode Active |
| B3:0/1 | Drain Mode Active |
| B3:0/2 | Mode Change Enabled |

### Timers (T4)

| Address | Function | Preset | Time Base |
|---------|----------|--------|-----------|
| T4:0 | Fill Timeout | 6000 | 0.01 sec |
| T4:1 | Anti-Short-Cycle | 300 | 0.01 sec |

---

## Operational Modes

### Fill Mode Sequence

The operator sets Mode_Select to FILL position. The system checks the Low_Level sensor — if active AND High_Level is inactive, the Fill_Valve energizes and Pump_Run activates. The Fill_Timer begins counting as a timeout safeguard. The tank fills until the High_Level sensor activates, which de-energizes the Fill_Valve and completes the cycle.

### Drain Mode Sequence

The operator sets Mode_Select to DRAIN position. The system checks the High_Level sensor — if active, the Drain_Valve energizes and Pump_Run activates. The tank drains until the Low_Level sensor de-activates (indicating empty), which de-energizes the Drain_Valve and completes the cycle.

### Fault Conditions

| Fault | Cause | Response |
|-------|-------|----------|
| Fill Timeout | Fill timer expires before high level | Stop fill, activate fault indicator |
| Sensor Conflict | Both high and low active simultaneously | Stop all operations, fault |
| E-Stop | Emergency stop pressed | Immediate shutdown |

---

## Troubleshooting

### Timer Not Resetting

**Symptom:** Timer accumulator doesn't reset to zero after cycle completes.

**Cause:** Using RTO instead of TON, or enable bit staying true.

**Fix:** 
- Ensure timer enable rung goes false at end of cycle
- Use TON for non-retentive timing
- Add explicit RES (reset) instruction if using RTO

### Pump Short-Cycling

**Symptom:** Pump turns on and off rapidly.

**Cause:** 
- Level sensor "chattering" near setpoint
- No anti-short-cycle timer implemented
- Timer preset too short

**Fix:**
- Add hysteresis between high/low setpoints
- Implement TOF timer (minimum 3 seconds)
- Increase physical distance between level sensors

### Fill Never Completes

**Symptom:** Fill mode activates but times out before reaching high level.

**Cause:**
- Inlet flow rate lower than expected
- Leak in tank
- High level sensor malfunction

**Fix:**
- Verify Q_in matches design specification
- Inspect tank for leaks
- Test high level sensor manually
- Adjust timer preset based on actual fill time

---

## References

- RSLogix 500 Programming Manual (1766-RM001)
- Allen-Bradley MicroLogix 1000/1100 User Manual
- ISA-5.1 Instrumentation Symbols and Identification
- Fluid Mechanics: Fundamentals and Applications (Çengel & Cimbala)

---

*Document Version: 1.0*  
*Last Updated: January 2026*
