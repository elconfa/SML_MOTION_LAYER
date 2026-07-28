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
