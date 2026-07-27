**English** | [Italiano](NOTA_Pattern_MotionLayer.it.md)

# Reusable note — the "Motion Layer" pattern (Ctrl/State)

**Source:** `github.com/haud-ba/PLC_MOTION_LAYER` (TwinCAT 3, author HAUD)
**Abstracted on:** 2026-07-22
**Purpose:** distill the architectural pattern — independent of the TwinCAT code — for a reusable motion
abstraction layer in IEC 61131-3 / CoDeSys projects.

> The **code** of PLC_MOTION_LAYER is not portable (it depends on Beckhoff NC + `Tc2_MC2`). What you
> carry over is the **architectural skeleton** described here.

---

## 1. The core idea

Cleanly separate **three levels**:

```
   Application / HMI / fieldbus
            │  (flat data structures only)
            ▼
   ┌─────────────────────────────┐
   │ COMMAND LEVEL  (Ctrl FB)     │  state machine, translates eCmd → calls
   │  - accepts Ctrl, writes State│
   └─────────────────────────────┘
            │  (interface I_*)
            ▼
   ┌─────────────────────────────┐
   │ EXECUTION LEVEL (Base FB)    │  encapsulates the "raw" motion FBs
   │  ABSTRACT, IMPLEMENTS I_*     │
   └─────────────────────────────┘
            │
            ▼
   Primitive motion (TwinCAT: MC_*; SML: direct CiA402 drive)
```

The application **never calls** the motion function blocks: it only writes a **command structure** and
reads a **status structure**.

---

## 2. The six elements of the pattern

### 2.1 Ctrl/State/Info/Data data contract
For each axis (or function), four flat structures with no logic:

| Struct | Direction | Content |
|--------|-----------|---------|
| `Ctrl`  | app → layer | command (`eCmd`), enable, requests |
| `State` | layer → app | command result, machine state |
| `Data`  | app → layer | setpoints and dynamic limits (pos, vel, acc, jerk) |
| `Info`  | layer → app | act/set values, axis status bits |

Golden rule: **serializable structures** (no pointers/interfaces inside), so they are accessible via
ADS symbol **or** copyable onto any bus.

### 2.2 Abstract base + control FB (inheritance)
- `FB_..Base` = `FUNCTION_BLOCK ABSTRACT IMPLEMENTS I_..` — encapsulates the primitive motion FBs
  (TwinCAT: `MC_Power`, `MC_MoveAbsolute`…), startup, errors.
- `FB_..Ctrl EXTENDS FB_..Base` — adds the **command state machine** driven by an enum (`E_AXIS_CTRL`:
  `AXIS_INIT`, `AXIS_ENABLE`, `AXIS_MOVE…`).

### 2.3 Interface for cross-cutting access
`I_McAxis` (methods `Enable/Disable/Halt/Stop/Reset/Read-WriteParameter`, properties
`Position/AxisRef`). It lets high-level subsystems (CNC, camming) maneuver the axes **without depending**
on the implementation.

### 2.4 Universal state machine `E_PROGRESS`
A single progress enum shared by all subsystems:
```
INVALID → INIT → BUSY → PREPARE → STARTUP → CHECK → DONE
                                                   ↘ ERROR
```
Key trick: the **combined state** = `functional_state + E_PROGRESS` (e.g. `AXIS_INIT + PROGRESS_DONE`).
Two utilities decompose it: `f_GetState()` and `f_GetProgress()`. This makes the orchestration code
identical for axes, CNC, camming, etc.

### 2.5 Multi-axis orchestration in a single PRG
`MAIN` for `nAxis := 1 TO MAX_AXIS`:
1. **wires** the struct pointers/references to the `Control[nAxis]` FB (`ADR(...)` / `REF=`);
2. runs an **INIT sequence** axis-by-axis gated by `E_PROGRESS`;
3. cyclically calls `Control[nAxis]()`.
`MAX_AXIS` is a global constant → it scales without touching the logic.

### 2.6 Optional MAPPING layer (bus-agnostic)
- `PRG_Mapping_In` / `PRG_Mapping_Out`: `memcpy` GVL ↔ bus structures, with **UNIONs** (`U_AXIS_CTRL`…)
  to reinterpret the same bytes as a struct or a raw block.
- Activation via **conditional compilation** (`{IF defined (AXIS_MAP)}`): if the define is missing, the
  mapping is bypassed → you pay only for what you use.
- **Data-size** check at startup (`Check()`).

### 2.7 (bonus) Messaging/logging subsystem
`E_MessageType` (Verbose/Info/Error), a timestamped queue, file writing, per-axis log levels.
Observability without polluting the motion logic.

---

## 3. Pros / costs

**Pros:** app↔motion decoupling; scalable multi-axis; reuse via inheritance+interfaces; bus-agnostic;
built-in observability.

**Cons:** heavy use of `POINTER`/`memcpy` (fragile w.r.t. memory layout — the size `Check()` only
partially mitigates); strong coupling to GVL placement; steep learning curve.

---

## 4. Applying it to SML (CoDeSys / CiA402) — mapping

SML **already has** the execution level: its FBs (`SML_Power`, `SML_Home`, `SML_ProfilePosition`,
`SML_ProfileVelocity(_Jog)`, `SML_SyncPosition/Velocity`, `SML_Stop`, `SML_TouchProbe`) are the
equivalent of the `MC_*` wrappers. `SML_TC3Link` is already an embryo of the MAPPING (struct ↔ PDO
image).

| Pattern element | State in SML | Action to adopt it |
|------------------|--------------|--------------------|
| Execution level | ✅ present (`SML_*` FBs) | none |
| Ctrl/State/Info/Data contract | ⚠️ fused into `OpenSML_Control` | separate command/state |
| Abstract base + `I_Axis` interface | ❌ absent | optional (full level) |
| Command FB with `eCmd` | ⚠️ `FB_SML` uses xExecute+arbiter | introduce `FB_AxisCtrl` with an enum |
| Unified `E_PROGRESS` | ❌ each FB uses `iState:INT` | unify + `f_GetState/f_GetProgress` |
| `ARRAY[1..MAX_AXIS]` orchestration | ❌ single-axis per instance | PRG `MAIN` with an array |
| MAPPING/UNION/define | ⚠️ `SML_TC3Link` partial | generalize (optional) |
| Log/diagnostics | ✅ `SML_Diagnostics`+`DiagCode` | already aligned |

**Conclusion:** the incompatibility (NC/`Tc2_MC2` vs direct CiA402) sits **entirely below the
abstraction line**, so it doesn't block the pattern. Adoption is **additive**, not a rewrite.

### Two adoption levels
- **Level A — lightweight (low risk):** a uniform `E_PROGRESS` enum + `f_GetState/f_GetProgress` + an
  explicit Ctrl/State split on `OpenSML_Control`. No change to execution.
- **Level B — complete (high multi-axis value):** `FB_AxisCtrl` (command `eCmd` + Ctrl/State/Info/Data),
  the `I_Axis` interface, `ARRAY[1..MAX_AXIS]` orchestration in `MAIN`, optional MAPPING.
