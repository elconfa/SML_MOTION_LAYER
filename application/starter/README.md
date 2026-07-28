# Starter application

The **minimum** application to drive axes with the `SML` library. Three objects you paste into a new
project after referencing the library. Everything else (examples, tests, MAPPING) is optional.

| File | Type | Role |
|---|---|---|
| `GVL_App.txt` | GVL | machine config: `MAX_AXIS` (change to your axis count) |
| `GVL_AXIS.txt` | GVL | the `[1..MAX_AXIS]` arrays + `Control : SML.FB_AxisCtrl` + `ItfSmlAxis` |
| `MAIN.txt` | PROGRAM | runs every axis each cycle |

These are prefixed with `SML.` (library-consumer mode). If you import everything into a **single project
without a forced namespace**, remove the `SML.` prefixes.

## Steps

1. **Reference the library**: *Library Manager -> Add library -> SML* (build it first from `../../library/`
   if you haven't — see `../../library/README.md`).
2. **Create the three objects** above (Add Object -> GVL / Program) and paste the `.txt` contents.
   Set `GVL_App.MAX_AXIS` to your number of axes.
3. **Put `MAIN` in a cyclic task.** Align `GVL_AXIS.Ctrl[n].CycleTime` to the task period (for CSP/OTG).
4. **Link the drive I/O**: map `GVL_AXIS.Axis[n]` to the CiA402 PDOs (ControlWord<->0x6040,
   StatusWord<->0x6041, ...). CoDeSys: direct link in the I/O Mapping tab. See `../../docs/GUIDA_IO_Linking.md`.

## Command an axis (from your own logic)

```pascal
GVL_AXIS.Ctrl[1].Scale := 4096.0;              // counts/unit of your encoder

GVL_AXIS.Ctrl[1].eCmd := SML.AXIS_ENABLE;      // enable
GVL_AXIS.Ctrl[1].eCmd := SML.AXIS_HOME;        // home

GVL_AXIS.Data[1].lrTargetPosition := 250.0;    // move to 250 units at 100 units/s
GVL_AXIS.Data[1].lrVelocity       := 100.0;
GVL_AXIS.Ctrl[1].eCmd := SML.AXIS_MOVE_ABS;

IF GVL_AXIS.State[1].eProgress = SML.PROGRESS_DONE THEN ... END_IF
IF GVL_AXIS.State[1].xError THEN GVL_AXIS.Ctrl[1].eCmd := SML.AXIS_RESET; END_IF
```

Full variable list: `../../docs/API_Reference.md`. Command guide: `../../docs/MANUALE_SML.md`.

## Test without hardware

Do NOT write your own emulator here. Use the ready-made, self-contained bench
`../tests/PRG_LevelB_Test.txt` (it drives the FB + a CiA402 emulator in the right order) — put **that**
bench in the task INSTEAD of `MAIN` and check `xTestPassed = TRUE`. For a quick per-axis check you can also
set `GVL_AXIS.Ctrl[n].xSimulation := TRUE` (mirrors position), but the drive enable still needs the bench's
emulator or real hardware.

## Optional add-ons (only if you need them)

- **I/O bridge** (e.g. TwinCAT, or a single mapping point): add `GVL_IO` and instantiate
  `SML.FB_IoLink_In/_Out` in `MAIN` (snippet in `MAIN.txt`).
- **Bus MAPPING** (expose Ctrl/State over fieldbus/ADS): add `GVL_AXIS_MAP` and instantiate
  `SML.FB_Mapping_In/_Out` in `MAIN`.

See the full template in `../src/` and `../../docs/IMPORT_CHECKLIST.md`.
