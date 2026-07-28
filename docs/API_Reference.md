**English** | [Italiano](API_Reference.it.md)

# SML — Axis API reference (data contract)

All variables to command and monitor one axis `n`. Access: `GVL_AXIS.<struct>[n].<field>`.

Convention: **Ctrl** and **Data** are **written by you** (app -> layer); **State** and **Info** are
**read** (layer -> app: the FB rewrites them every cycle, do not write them).

---

## Writable

### `GVL_AXIS.Ctrl[n]` — command + configuration
| Variable | Type | Default | Meaning |
|---|---|---|---|
| `eCmd` | `E_AXIS_CTRL` | AXIS_NULL | **the command** (see eCmd table below) |
| `xEmergencyStop` | BOOL | TRUE | **TRUE = run**, FALSE = quick-stop (CiA402 bit 2). Keep it TRUE |
| `xSimulation` | BOOL | FALSE | mirror Target -> Actual (test without a drive) |
| `Scale` | LREAL | 4096.0 | counts per user unit (**> 0**) |
| `CycleTime` | LREAL | 0.001 | task period [s] — must match (OTG/CSP) |
| `PositionWindow` | LREAL | 0.01 | "in position" deadband [units] (CSP) |
| `HomeOffset` | DINT | 0 | homing offset [encoder counts -> 0x607C] |
| `xTouchProbeEnable` | BOOL | FALSE | arm the TouchProbe hardware |
| `xTouchProbeRising` | BOOL | FALSE | arm the latch on the rising edge |
| `xTouchProbeFalling` | BOOL | FALSE | arm the latch on the falling edge |

### `GVL_AXIS.Data[n]` — motion setpoints (user units)
| Variable | Type | Default | Meaning |
|---|---|---|---|
| `lrTargetPosition` | LREAL | 0 | target [units] for MOVE_ABS/REL/CSP |
| `lrVelocity` | LREAL | 10.0 | velocity [units/s]; **signed** for MOVE_VELOCITY |
| `lrAcceleration` | LREAL | 1000.0 | acceleration [units/s^2] |
| `lrDeceleration` | LREAL | 1000.0 | deceleration [units/s^2] |
| `lrJerk` | LREAL | 10000.0 | jerk [units/s^3] — CSP/OTG only |

---

## Readable

### `GVL_AXIS.State[n]` — command result
| Variable | Type | Meaning |
|---|---|---|
| `eProgress` | `E_PROGRESS` | progress (INVALID...DONE/ERROR — table below) |
| `eState` | INT | combined state = functional state + eProgress |
| `eCmdActive` | `E_AXIS_CTRL` | command currently executing |
| `xDone` | BOOL | current command complete (= eProgress DONE) |
| `xError` | BOOL | aggregated motion error |
| `DiagCode` | `SML_DiagCode` | live diagnostic condition |
| `DiagText` | STRING(80) | text of the live condition |
| `FirstFaultCode` | `SML_DiagCode` | first latched fault (root cause) |
| `FirstFaultText` | STRING(80) | text of the first fault |

### `GVL_AXIS.Info[n]` — actual values
| Variable | Type | Meaning |
|---|---|---|
| `lrActualPosition` | LREAL | actual position [units] |
| `lrActualVelocity` | LREAL | actual velocity [units/s] |
| `xEnabled` | BOOL | drive in OperationEnabled |
| `xHomed` | BOOL | homing done at least once (latch) |
| `xMoving` | BOOL | axis is moving |
| `xInPosition` | BOOL | within PositionWindow (CSP path) |
| `Status` | `ST_CiA402_Status` | full CiA402 decoder (below) |
| `xTouchProbeDoneR` | BOOL | rising-edge latch captured |
| `xTouchProbeDoneF` | BOOL | falling-edge latch captured |
| `xTouchProbeBusy` | BOOL | TouchProbe armed / waiting |
| `xTouchProbeError` | BOOL | TouchProbe error |
| `TouchProbeRisingValue` | DINT | position latched on rising [encoder counts] |
| `TouchProbeFallingValue` | DINT | position latched on falling [encoder counts] |

### `GVL_AXIS.Info[n].Status` — CiA402 bits (full decoder)
| Variable | Type | | Variable | Type |
|---|---|---|---|---|
| `NotReadyToSwitchOn` | BOOL | | `TorqueProfileMode` | BOOL |
| `SwitchOnDisabled` | BOOL | | `HomingMode` | BOOL |
| `ReadyToSwitchOn` | BOOL | | `SyncPositionMode` (CSP) | BOOL |
| `SwitchedOn` | BOOL | | `SyncVelocityMode` (CSV) | BOOL |
| `OperationEnabled` | BOOL | | `Moving` | BOOL |
| `QuickStopActive` | BOOL | | `StandStill` | BOOL |
| `FaultReactionActive` | BOOL | | `Homed` | BOOL |
| `Fault` | BOOL | | `Err` | BOOL (Error_Code != 0) |
| `Warning` | BOOL | | `ErrId` | UINT (0x603F) |
| `TargetReached` | BOOL | | `ActTorque` | INT (0.1% of rated) |
| `Halted` | BOOL | | `FollowingError` | DINT (encoder counts) |
| `VoltageEnabled` | BOOL | | | |
| `InternalLimit` | BOOL | | | |
| `ProfilePositionMode` | BOOL | | | |
| `ProfileVelocityMode` | BOOL | | | |
| `VelocityMode` (legacy) | BOOL | | | |

---

## `eCmd` values (`E_AXIS_CTRL`)
`AXIS_NULL`(0) · `AXIS_ENABLE`(1) · `AXIS_DISABLE`(2) · `AXIS_RESET`(3) · `AXIS_HOME`(4) ·
`AXIS_MOVE_ABS`(5) · `AXIS_MOVE_REL`(6) · `AXIS_MOVE_VELOCITY`(7) · `AXIS_JOG_POS`(8) ·
`AXIS_JOG_NEG`(9) · `AXIS_MOVE_CSP`(10) · `AXIS_STOP`(11)

See [`MANUALE_SML.md`](MANUALE_SML.md) §3 for the full per-command reference (Data used, DONE-when).

## `eProgress` values (`E_PROGRESS`)
`PROGRESS_INVALID`(0) · `INIT`(1) · `BUSY`(2) · `PREPARE`(3) · `STARTUP`(4) · `CHECK`(5) ·
**`DONE`**(6) · **`ERROR`**(7)

---

## Combined state `eState` (INT)

`eState` packs "what the axis is doing" and "how far along" into a single number:

```
eState = functional_state + progress
         (hundreds: WHAT)    (units: HOW FAR)
```

- **hundreds** = functional state (`E_AXIS_STATE`, multiples of 100)
- **units** = progress (`E_PROGRESS`, the values above)

Example: `502` = `500` (MOVING) + `2` (BUSY) -> "moving, in progress"; `306` = `300` (IDLE) + `6` (DONE)
-> "enabled and still, ready".

### Values actually produced
| `eState` | Meaning | When |
|---|---|---|
| `0` | not yet executed | before the first cycle |
| `200` | disabled, stopped | `AXIS_NULL` with drive OFF |
| `202` / `206` | disabling / disabled (done) | `AXIS_DISABLE` |
| `300` | **IDLE**: enabled and stopped, ready | `AXIS_NULL` with drive ON |
| `302` / `306` | enabling / **enabled (done)** | `AXIS_ENABLE` |
| `102` / `106` | reset in progress / done | `AXIS_RESET` |
| `402` / `406` | homing / **homed (done)** | `AXIS_HOME` |
| `502` / `506` | move in progress / **at target** | `AXIS_MOVE_ABS/REL/CSP` |
| `602` | ramping / jog held | `AXIS_MOVE_VELOCITY`, `AXIS_JOG_*` |
| `606` | **target velocity reached** | `AXIS_MOVE_VELOCITY` at steady state |
| `702` / `706` | stopping / **stopped** | `AXIS_STOP` |
| `907` | **error** | any fault |

Units mnemonic: **0** = idle/no request · **2** = working · **6** = done · **7** = error. (Jog stays at
`x02`: continuous motion has no discrete "DONE".) The combined state uses only progress 0/2/6/7.

### Decompose it
```pascal
base := GVL_AXIS.State[n].eState / 100;                 // 5 = MOVING
prog := GVL_AXIS.State[n].eState MOD 100;               // 2 = BUSY, 6 = DONE
// or with the library functions:
base := SML.f_GetState(GVL_AXIS.State[n].eState);       // 500
prog := SML.f_GetProgress(GVL_AXIS.State[n].eState);    // 2 (SML.PROGRESS_BUSY)
```

For application logic you usually just read `State[n].eProgress` (already the units part) plus
`State[n].xDone` / `xError`. `eState` is handy as a single number for an HMI display or a log
(e.g. "502" = moving).

---

## Diagnostics: telling whether and where the axis faulted

| Question | Where to read |
|---|---|
| Is there a motion error? | `State[n].xError` = TRUE (or `eProgress = PROGRESS_ERROR`, or `eState = 907`) |
| What type / which FB? | `State[n].DiagCode` + `DiagText` |
| What is the root cause? | `State[n].FirstFaultCode` / `FirstFaultText` (latched root cause) |
| Which command was active? | `State[n].eCmdActive` (e.g. `AXIS_HOME`) |
| Raw drive fault? | `Info[n].Status.Fault` / `Status.ErrId` (0x603F) |

**TouchProbe** is orthogonal: its error does **not** set `xError`; read
`Info[n].xTouchProbeError`.

### `DiagCode` values (`SML_DiagCode`)
| Code | Val | Category | Origin | Typical command |
|---|---|---|---|---|
| `DIAG_OK` | 0 | ok | no fault/warning | — |
| `DIAG_WARNING` | 1 | warning | StatusWord bit 7 | any |
| `DIAG_INTERNAL_LIMIT` | 2 | warning | StatusWord bit 11 | any |
| `DIAG_DRIVE_FAULT` | 10 | drive | StatusWord bit 3 (Fault) | any |
| `DIAG_DRIVE_ERRORCODE` | 11 | drive | Fault + Error_Code (0x603F) | any |
| `DIAG_QUICK_STOP` | 12 | drive | QuickStop (bit 5 = 0) | any |
| `DIAG_FOLLOWING_ERROR` | 13 | drive | StatusWord bit 13 | move/CSP |
| `DIAG_POWER_ERROR` | 20 | FB | `SML_Power` | `AXIS_ENABLE` |
| `DIAG_HOME_ERROR` | 21 | FB | `SML_Home` | `AXIS_HOME` |
| `DIAG_MOVE_ERROR` | 22 | FB | `SML_ProfilePosition` | `AXIS_MOVE_ABS/REL` |
| `DIAG_VEL_ERROR` | 23 | FB | `SML_ProfileVelocity` | `AXIS_MOVE_VELOCITY` |
| `DIAG_JOG_ERROR` | 24 | FB | `SML_ProfileVelocity_Jog` | `AXIS_JOG_POS/NEG` |
| `DIAG_TOUCHPROBE_ERROR` | 25 | FB | `SML_TouchProbe` (does not stop motion) | TouchProbe |
| `DIAG_OTG_ERROR` | 26 | FB | `FB_S7RTT_OTG` (limits/CycleTime) | `AXIS_MOVE_CSP` |

Ranges: **1..9** warning (motion continues) · **10..19** drive fault · **20..29** FB fault.
Reset with `AXIS_RESET`.

```pascal
IF GVL_AXIS.State[1].xError THEN
    CASE GVL_AXIS.State[1].DiagCode OF
        SML.DIAG_HOME_ERROR: (* homing failed *) ;
        SML.DIAG_MOVE_ERROR: (* move failed *) ;
    END_CASE
    GVL_AXIS.Ctrl[1].eCmd := SML.AXIS_RESET;
END_IF
```

---

## Notes

- **Namespace:** with the library published as `SML`, prefix *types* and *enum values*
  (`SML.AXIS_MOVE_ABS`, `SML.PROGRESS_DONE`); struct **fields** are your own variables and are not
  prefixed (`GVL_AXIS.Ctrl[n].eCmd`).
- **Interface alternative** (`GVL_AXIS.ItfSmlAxis[n]`), instead of writing `eCmd`: methods
  `Enable()`, `Disable()`, `Reset()`, `Home()`, `MoveAbsolute(lrPosition, lrVelocity)`,
  `MoveRelative(lrDistance, lrVelocity)`, `MoveVelocity(lrVelocity)`,
  `Jog(xForward, xReverse, lrVelocity)`, `MoveFollow(lrPosition, lrVelocity)`, `Stop()`; plus read-only
  properties `Position` (LREAL), `Enabled` (BOOL).
- **Diagnostics detail** (`SML_DiagCode` categories, reset): see [`MANUALE_SML.md`](MANUALE_SML.md) §12.
