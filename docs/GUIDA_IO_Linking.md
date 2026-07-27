**English** | [Italiano](GUIDA_IO_Linking.it.md)

# SML — Physical-axis I/O linking guide (CoDeSys and TwinCAT)

**Version:** SML_v9 · **Date:** 2026-07-23

How to hook `GVL_AXIS.Axis[n]` (the `OpenSML_Axis` struct, i.e. the CiA402 PDO image) to the real PDOs
of the EtherCAT drive — on **CoDeSys** and on **TwinCAT**.

---

## 1. The key idea

`GVL_AXIS.Axis[n]` is a **software struct** (without `AT %I*/%Q*`): this keeps the code **portable**
and **testable in simulation**. The library code (`MAIN`, `FB_AxisCtrl`, `GVL_AXIS`) is **identical**
on CoDeSys and TwinCAT. Only the **thin I/O-hookup layer** changes.

The fields of `OpenSML_Axis` are **already ordered outputs-then-inputs**:
- first 10 fields = **PLC → Drive** (RxPDO)
- remaining 12 fields = **Drive → PLC** (TxPDO)

This ordering is what keeps both the direct link and the bridge clean.

Two hookup strategies:
- **(A) Direct link** — each PDO object is bound to a struct member. No extra code. Idiomatic on
  **CoDeSys**.
- **(B) Bridge** — the PDOs are mapped onto two `AT %Q*`/`AT %I*` images
  (`ST_DriveOut`/`ST_DriveIn`) and an FB copies image↔struct every cycle. Idiomatic on **TwinCAT**
  (this is what `SML_TC3Link` used to do).

Step common to both: in the drive, make sure the objects are in the **PDO assignment**
(RxPDO 0x160x, TxPDO 0x1A0x). If an object isn't in the PDO, it won't appear for linking.

---

## 2. Reference table (field ↔ CiA402 object)

| Field `Axis[n].` | Object | Dir. | PDO |
|---|---|---|---|
| ControlWord | 0x6040 | →Drive | RxPDO |
| Modes_of_operation | 0x6060 | →Drive | RxPDO |
| Target_Position | 0x607A | →Drive | RxPDO |
| Profile_Velocity | 0x6081 | →Drive | RxPDO |
| Target_Velocity | 0x60FF | →Drive | RxPDO |
| Target_Torque | 0x6071 | →Drive | RxPDO |
| Profile_Acceleration | 0x6083 | →Drive | RxPDO |
| Profile_Deceleration | 0x6084 | →Drive | RxPDO |
| Home_Offset | 0x607C | →Drive | RxPDO |
| TouchProbe_ControlWord | 0x60B8 | →Drive | RxPDO |
| StatusWord | 0x6041 | ←Drive | TxPDO |
| Modes_of_operation_display | 0x6061 | ←Drive | TxPDO |
| Torque_Actual_Value | 0x6077 | ←Drive | TxPDO |
| Position_Actual_Value | 0x6064 | ←Drive | TxPDO |
| Velocity_Actual_Value | 0x606C | ←Drive | TxPDO |
| Current_Actual_Value | 0x6078 | ←Drive | TxPDO |
| Following_Error_Actual_Value | 0x60F4 | ←Drive | TxPDO |
| Error_Code | 0x603F | ←Drive | TxPDO |
| Digital_Inputs | 0x60FD | ←Drive | TxPDO |
| TouchProbe_StatusWord | 0x60B9 | ←Drive | TxPDO |
| TouchProbe_Rising_Value | 0x60BA | ←Drive | TxPDO |
| TouchProbe_Falling_Value | 0x60BB | ←Drive | TxPDO |

---

## 3. CoDeSys — direct link in the I/O Mapping tab (recommended)

No bridge code, no `AT`: you bind each PDO channel to a struct member.

1. **Devices** tree → EtherCAT Master → drive → **`<Slave> I/O Mapping`** tab
   (or *EtherCAT I/O Mapping*).
2. For each row (PDO object), the **Variable** column → double-click → **Browse** → pick the member,
   e.g. `GVL_AXIS.Axis[1].ControlWord`, `GVL_AXIS.Axis[1].StatusWord`, … (**1-based** index:
   `[1]`, `[2]`, …).
3. Repeat for axis 2 → `GVL_AXIS.Axis[2].…`, and so on.
4. On the module, set **"Always update variables" = Enabled 2 (always in bus cycle task)**, so the
   fields update even if not used explicitly in the code.
5. If a PDO object is missing: enable **Expert settings** on the drive and add it to the RxPDO/TxPDO.

> Alternatively you can use `%IW/%QW` addresses with `AT`, but binding the struct in the I/O Mapping
> tab is the correct, portable CoDeSys idiom.

### 3b. CoDeSys — bridge variant (optional, like TwinCAT)
If you prefer a single mapping point on CoDeSys too, use the two DUTs `ST_DriveOut`/`ST_DriveIn` as in
§4: declare the two images, map them in the I/O tab and use the same bridge code as in §5.

---

## 4. TwinCAT — bridge with `AT %I*/%Q*` image (idiomatic)

TwinCAT doesn't like linking a mixed-direction struct. The clean approach (that of `SML_TC3Link`):
**two separate images** + cyclic copy.

1. The two halves are already in `GVL_IO` as **plain arrays** (DUTs `ST_DriveOut`/`ST_DriveIn`, same
   order/types as the struct):
   ```pascal
   VAR_GLOBAL
       DriveOut : ARRAY[1..GVL_App.MAX_AXIS] OF ST_DriveOut; // -> RxPDO
       DriveIn  : ARRAY[1..GVL_App.MAX_AXIS] OF ST_DriveIn;   // <- TxPDO
   END_VAR
   ```
   > **No `AT %Q*/%I*`**: in a GVL an incomplete address requires a VAR_CONFIG/mapping, otherwise
   > CoDeSys throws error **C0128**. With plain arrays the code compiles and you map it in the
   > configurator. On TwinCAT you can re-add `AT %Q*`/`AT %I*` if you want the explicit process image.
2. In the EtherCAT drive: **Change Link** on each PDO object → link to `DriveOut[1].ControlWord`,
   `DriveIn[1].StatusWord`, … (contiguous blocks → quick link; or "one-to-one" if the types match).
3. Cyclic copy (see §5), with the correct ordering relative to `MAIN`.

> TwinCAT alternative without the bridge: directly **Change Link** each PDO to
> `GVL_AXIS.Axis[1].<member>`. It works, but it's ~22 links × N axes by hand.

---

## 5. The ready-made bridge (files already in the project)

Supplied objects (import from `.txt`):
- **`ST_DriveOut`** = first 10 fields of `OpenSML_Axis` (outputs), same order/types.
- **`ST_DriveIn`**  = remaining 12 fields (inputs), same order/types.
- **`GVL_IO`** — `DriveOut` / `DriveIn` (`ARRAY[1..GVL_App.MAX_AXIS]`, plain arrays; see the C0128
  note in §4) + flag `IO_LINK_ENABLE : BOOL`.
- **`FB_IoLink_In`** — library FB: copies `DriveIn[] -> Axis[]` (input fields).
- **`FB_IoLink_Out`** — library FB: copies `Axis[] -> DriveOut[]` (output fields).
- Both are **instantiated in `MAIN`** and receive the arrays via `VAR_IN_OUT` (`ARRAY[*]`, variable length),
  so they carry no dependency on `MAX_AXIS`.

**Correct timing (zero delay):** `MAIN` already calls the `FB_IoLink_In` instance **before** the axis loop
and `FB_IoLink_Out` **after** (next to `FB_Mapping_In/Out`). There's no 1-cycle bridge delay. Both
auto-bypass when `xEnable := GVL_IO.IO_LINK_ENABLE` is FALSE.

**Activation:** set `GVL_IO.IO_LINK_ENABLE := TRUE` and, in the configurator, link
`GVL_IO.DriveOut[n]` to the RxPDOs and `GVL_IO.DriveIn[n]` to the TxPDOs of the drive.

> **CoDeSys with direct link (§3):** do NOT use the bridge — leave `IO_LINK_ENABLE = FALSE` and map
> `GVL_AXIS.Axis[n]` directly. The two approaches are mutually exclusive (the flag prevents conflicts).

### Note: field-by-field copy vs block copy
Because `ST_DriveOut`/`ST_DriveIn` have the **same types and the same order** as the respective halves
of `OpenSML_Axis`, the two programs copy **field by field**: the safest choice (padding-independent).
Avoid a block `memcpy` between the mixed struct and the two halves: the padding might not match.

---

## 6. Checks and precautions (both platforms)

- **Distributed Clocks (DC)**: enable them on the drive if you use CSP (`AXIS_MOVE_CSP`) or touch-probe
  (synchronized latch).
- **Supported modes**: CSP (mode 8) for the OTG path; PP(1)/PV(3)/HM(6) for the other commands.
- **`Ctrl.Scale`** = encoder counts/unit; **`Ctrl.CycleTime`** = the real task period (for the OTG).
- **Touch-probe PDOs** (0x60B8/0x60B9/0x60BA/0x60BB): often not in the default PDO → add them if you
  use the touch-probe.
- **"Always update variables"** (CoDeSys) or full mapping (TwinCAT): ensures all fields update.

---

## 7. Summary

| | CoDeSys | TwinCAT |
|---|---|---|
| Idiomatic strategy | Direct link (I/O Mapping tab) | Bridge (`ST_DriveOut`/`ST_DriveIn` + copy) |
| Extra code | none | instantiate the library FBs `FB_IoLink_In`/`_Out` in MAIN |
| `AT %I*/%Q*` | no (struct binding) | yes (on the two images) |
| Bridge DUTs | optional (§3b variant) | required |
| Library code (MAIN/FB/GVL) | identical | identical |
