# OpenSML — CiA402 SoftMotion Light: Code Analysis

**Project:** Open Source SoftMotion Light for CiA402 Servo Drivers  
**Source:** https://github.com/feecat/opensml  
**Analysis date:** 2026-04-28  

---

## 1. Critical Bugs (compilation errors / wrong runtime behavior)

### 1.1 `OpenSML_SyncPosition` — Undefined variable `xSimulation`
**Severity: CRITICAL — Compilation Error**

In `OpenSML_SyncVelocity` the input `xSimulation: BOOL` is correctly declared.  
In `OpenSML_SyncPosition` the same variable is used in state 0 (`ELSIF xSimulation THEN`) but is **never declared** in `VAR_INPUT` or any other VAR block.  
This causes a compilation error and prevents the entire library from building.

**Fix applied:** Added `xSimulation: BOOL` to `VAR_INPUT` in `OpenSML_SyncPosition`.

---

### 1.2 `OpenSML_ProfilePosition` — `Target_Position := 0` on disable
**Severity: CRITICAL — Potential Machine Crash**

In state 20 (movement), when `NOT xEnable` is detected, the original code wrote:
```
Axis.Profile_Velocity := 0;
Axis.Target_Position  := 0;   // <-- BUG
```
Writing `Target_Position = 0` sends the drive to **absolute position zero** at the drive's maximum allowed deceleration. On a real machine this causes an uncontrolled, unintended motion toward the machine origin.

The same bug was present in state 999 (error handling).

**Fix applied:** Replaced `0` with `Axis.Position_Actual_Value` in both locations, so the drive holds the current position when the FB is disabled or in error.

---

### 1.3 `OpenSML_AxisController` — Division by zero on `Control.Scale`
**Severity: CRITICAL — Runtime Fault**

The line:
```
Control.lrActualPosition := DINT_TO_LREAL(Axis.Position_Actual_Value) / Control.Scale;
```
is executed every PLC cycle with no guard. If `Control.Scale = 0.0` (e.g. uninitialized struct, misconfiguration), the result is `+INF` or a runtime divide-by-zero exception, stopping the PLC task.

**Fix applied:** Added a guard before the division:
```
IF Control.Scale = 0.0 THEN Control.Scale := 1.0; END_IF
```
The fallback value of 1.0 is intentionally wrong (so the operator notices the misconfiguration) but safe (no crash).

---

### 1.4 `OpenSML_SyncPosition` / `OpenSML_SyncVelocity` — Division by zero on `lrScale`
**Severity: HIGH**

Both FBs divide by `lrScale` to seed the OTG generator:
```
otg.CurrentPosition := DINT_TO_LREAL(Axis.Position_Actual_Value) / lrScale;
```
Default value is `1.0` so the common case is safe, but an explicit assignment of `0.0` from the application would crash the task.

**Fix applied:** Added guard `IF lrScale = 0.0 THEN lrScale := 1.0; END_IF` at the top of both FBs.

---

## 2. Safety Issues

### 2.1 `OpenSML_Power` — Incomplete shutdown sequence
**Severity: HIGH**

When `xEnable` is dropped in state 50 (running), the original code only cleared bit 3 (Enable Operation):
```
Axis.ControlWord.3 := FALSE;
iState := 0;
```
According to CiA402 §8.2.1, the proper **Shutdown** command requires:
- Bit 0 (Switch On) = 0  
- Bit 1 (Enable Voltage) = 1  
- Bit 3 (Enable Operation) = 0  

Dropping only bit 3 leaves the drive in the `SwitchedOn` state with voltage still applied and the power stage potentially energized, depending on the drive implementation.

**Fix applied:** Added `Axis.ControlWord.0 := FALSE` alongside `ControlWord.3 := FALSE`.

---

### 2.2 `OpenSML_Power` — Control bits not cleared in error state
**Severity: HIGH**

In state 999 (error), bits 0 and 3 of `ControlWord` were left in whatever state they were when the error occurred. If the drive was in OperationEnabled when the error was latched, the drive could remain in OperationEnabled indefinitely.

**Fix applied:** Added explicit clearing of bits 0 and 3 in state 999:
```
Axis.ControlWord.0 := FALSE;
Axis.ControlWord.3 := FALSE;
```

---

### 2.3 `OpenSML_Home` — Stall watchdog fires immediately at position zero
**Severity: MEDIUM**

`Pos_Old` is a `DINT` local variable, so it initializes to `0` on FB instantiation.  
The stall watchdog in state 20 is:
```
tonTimeout(IN := (Axis.Position_Actual_Value = Pos_Old), PT := T#10S);
Pos_Old := Axis.Position_Actual_Value;
```
If the axis is physically at position 0 when state 20 is entered, `Position_Actual_Value = 0 = Pos_Old` on the **first cycle**, so the timeout timer starts immediately. If homing completes within 10 seconds this is harmless, but it is fragile and misleading.

**Fix applied:** In state 0, before transitioning to state 10, capture the actual position:
```
Pos_Old := Axis.Position_Actual_Value;
```
This ensures `Pos_Old` is valid when state 20 is first entered.

---

## 3. Logic and Behavioral Issues

### 3.1 `OpenSML_AxisController` — `Control.xBusy` never written
**Severity: MEDIUM**

`OpenSML_Control` declares `xBusy: BOOL` as an output, and the field is visible to the application. However `OpenSML_AxisController` never writes to it, leaving it permanently `FALSE`. An application checking `xBusy` for interlock logic would behave incorrectly.

**Fix applied:** Added at the top of `AxisController`:
```
Control.xBusy := (iState <> 0 AND iState <> 999);
```

---

### 3.2 `OpenSML_ProfileVelocity` — Missing following-error check
**Severity: MEDIUM**

State 20 only checked `StatusWord.3` (Fault). `StatusWord.13` (Following Error) was not monitored. A following error (drive cannot keep up with the commanded velocity) would go undetected, leaving the FB running with a degraded axis.

**Fix applied:** Added `ELSIF Axis.StatusWord.13 THEN iState := 999`.

---

### 3.3 `OpenSML_ProfileVelocity` — No `xAtVelocity` output
**Severity: LOW**

The FB had no way to report that the drive had reached the commanded target velocity. This is standard information in PLCopen-style motion FBs and is readily available from `StatusWord.10` (Target Velocity Reached in PV mode).

**Fix applied:** Added output `xAtVelocity: BOOL := Axis.StatusWord.10` in state 20.

---

### 3.4 `OpenSML_Stop` — No `xStopped` output
**Severity: LOW**

The Halt FB wrote the stop bit but provided no confirmation that the axis had actually reached standstill. An application that needs to wait for a full stop before switching modes had no reliable signal to wait on.

**Fix applied:** Added output `xStopped: BOOL := xStop AND Axis.StatusWord.14`.  
Note: `StatusWord.14` meaning is drive-mode-dependent. Verify against your drive documentation.

---

### 3.5 `OpenSML_SyncPosition` / `OpenSML_SyncVelocity` — No `xBusy` output
**Severity: LOW**

Neither CSP/CSV FB exposed a busy flag. Applications that chain motion phases or display status had no reliable indicator that the FB was actively generating trajectories.

**Fix applied:** Added `xBusy: BOOL := (iState = 10) OR (iState = 20)` to both FBs.

---

### 3.6 `OpenSML_AxisController` — Jog `InterfaceType` always set to `TRUE`
**Severity: LOW (design clarity)**

In the jog block, all three branches (JogPos, JogNeg, and the else/hold branch) set `InterfaceType := 1` (velocity interface). The separate `xFollowEnable` block then conditionally overrides it to `0` (position interface). The logic is correct but the structure suggests the else branch might accidentally override a prior follow-mode assignment if the conditions change mid-scan. Reordered the blocks so follow-enable is evaluated first.

---

## 4. Code Quality and Maintainability Issues

### 4.1 Magic numbers for `StatusWord` bit indices
Every bit access like `Axis.StatusWord.13` appears without explanation throughout all FBs. Without the CiA402 specification in hand, it is impossible to understand what condition is being checked.

**Fix applied:** Added inline comments on every bit access across all files, e.g.:
```
IF Axis.StatusWord.13 THEN  // Following Error: actual position deviates too far
```

---

### 4.2 Missing CiA402 Object Dictionary references in `OpenSML_Axis`
The struct fields had no documentation linking them to actual CiA402 object indices. This makes PDO configuration error-prone.

**Fix applied:** Added a full header table in `OpenSML_Axis.txt` mapping each field to its CiA402 Object Dictionary index (e.g. `Position_Actual_Value → 0x6064`).

---

### 4.3 `OpenSML_Control.Scale` — Hex literal default, no units
```
Scale: LREAL := 16#1000;  // original: 4096 in hex, no explanation
```
The hex literal is non-obvious and there was no documentation of units or a usage example.

**Fix applied:** Changed to `4096.0` (decimal) and added comment:
```
// Scale [counts/unit]: e.g. 4096 counts/mm → Scale = 4096.0
// Position_Actual_Value [counts] / Scale = lrActualPosition [user units]
```

---

### 4.4 `OpenSML_Control.xEmergencyStop` — Misleading default name
`xEmergencyStop: BOOL := TRUE` reads as "emergency stop is TRUE by default", which sounds dangerous. In fact `TRUE` means **no emergency stop** (CiA402 bit 2 = 1 = normal run, 0 = activate quick-stop). The default is correct but the name is confusing.

**Fix applied:** Added clarifying comment:
```
xEmergencyStop: BOOL := TRUE; // TRUE = normal run, FALSE = activate quick-stop
                               // Maps directly to ControlWord bit 2 (active-LOW)
```

---

### 4.5 `OpenSML_Axis.Target_torque` — Inconsistent capitalization
`Target_torque` used a lowercase `t` while all other fields used PascalCase (`Target_Position`, `Target_Velocity`). This inconsistency can cause confusion and typo errors.

**Fix applied:** Renamed to `Target_Torque` (consistent PascalCase) across all files.

---

### 4.6 `OpenSML_TC3Link` — Call order constraint only in informal comment
The requirement to call `link()` after all motion FBs was mentioned in a casual comment (`// Make sure link() call after other instance`). If called in the wrong order, outputs are delayed by one cycle and feedback is one cycle stale.

**Fix applied:** Replaced with a structured explanation of the reason, consequences, and a correct usage example in the FB header.

---

### 4.7 `OpenSML_SyncVelocity` — CSV native mode undocumented
The FB uses CSP mode (8) to emulate velocity control by integrating velocity in the PLC. If the drive natively supports CSV mode (9) with a `Target_Velocity` PDO, using it would reduce following error. The FB provided no guidance on how to switch.

**Fix applied:** Added a commented-out line showing how to use `otg.NewVelocity` for a native CSV setup:
```
// Axis.Target_Velocity := LREAL_TO_DINT(otg.NewVelocity * lrScale);
```

---

## 5. Summary Table

| # | File | Severity | Category | Status |
|---|------|----------|----------|--------|
| 1.1 | SyncPosition | **CRITICAL** | Compilation error (`xSimulation` undefined) | Fixed |
| 1.2 | ProfilePosition | **CRITICAL** | Crash move on disable (Target_Position = 0) | Fixed |
| 1.3 | AxisController | **CRITICAL** | Division by zero (Scale = 0) | Fixed |
| 1.4 | SyncPosition/Velocity | HIGH | Division by zero (lrScale = 0) | Fixed |
| 2.1 | Power | HIGH | Incomplete CiA402 shutdown sequence | Fixed |
| 2.2 | Power | HIGH | Control bits not cleared in error state | Fixed |
| 2.3 | Home | MEDIUM | Stall watchdog fires immediately at position 0 | Fixed |
| 3.1 | AxisController | MEDIUM | `xBusy` never written | Fixed |
| 3.2 | ProfileVelocity | MEDIUM | Following error not monitored | Fixed |
| 3.3 | ProfileVelocity | LOW | No `xAtVelocity` output | Fixed |
| 3.4 | Stop | LOW | No `xStopped` output | Fixed |
| 3.5 | SyncPosition/Velocity | LOW | No `xBusy` output | Fixed |
| 3.6 | AxisController | LOW | InterfaceType logic ordering unclear | Fixed |
| 4.1 | All files | LOW | Magic numbers on StatusWord bits | Fixed |
| 4.2 | Axis | LOW | Missing CiA402 OD index references | Fixed |
| 4.3 | Control | LOW | Hex default for Scale, no units | Fixed |
| 4.4 | Control | LOW | Misleading xEmergencyStop name | Fixed |
| 4.5 | Axis | LOW | Inconsistent capitalization Target_torque | Fixed |
| 4.6 | TC3Link | LOW | Call-order constraint undocumented | Fixed |
| 4.7 | SyncVelocity | LOW | CSV native mode path undocumented | Fixed |

---

## 6. Recommendations Before Deployment

1. **Verify `ProfilePosition` hold-position behavior** with your specific drive.  
   When `xEnable` drops in PP mode, `Target_Position` is now set to `Position_Actual_Value`. Some drives require a rising edge on ControlWord bit 4 (New Setpoint) to latch a new position target. Test that the drive actually holds position and does not fault.

2. **Confirm `Target_Torque` rename** (from `Target_torque`).  
   Any existing application code referencing `Axis.Target_torque` must be updated to `Axis.Target_Torque`.

3. **Validate `xStopped` (StatusWord bit 14)** against your drive datasheet.  
   Bit 14 meaning is operating-mode-dependent. In CSP it may indicate "speed = 0"; in other modes it may be manufacturer-specific.

4. **Confirm OTG library is licensed and linked** (`FB_S7RTT_OTG` from StruckigLight).  
   All CSP/CSV FBs depend on it. Without it none of the trajectory-generating FBs will compile.

5. **Set `CycleTime` to match the actual PLC task period exactly.**  
   A mismatch between `CycleTime` and the real scan time causes the OTG to generate incorrect velocity and acceleration profiles, leading to target-overshoot or sluggish response.
