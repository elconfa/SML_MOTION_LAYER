# SML v9 — Changelog

**Base:** SML_v8
**Date:** 2026-07-23
**Scope:** rendere `FB_AxisCtrl` un **superset di `FB_SML`** portando dentro
il layer Livello B i due sottosistemi finora esposti solo dalla faccia legacy:
**decoder Status CiA402 completo** e **TouchProbe** (opzione 3).

Riferimenti: `CHANGELOG_v8.md`, `../NOTA_Pattern_MotionLayer.md`.

---

## Novità v9

### Nuovo tipo
- `ST_CiA402_Status.txt` — tutti i bit di stato CiA402 (macchina di stato, modo
  operativo, moto, errore) + `ErrId`/`ActTorque`/`FollowingError`. Replica gli
  output di `SML_Status`.

### Contratto dati esteso (solo aggiunte)
- `ST_AXIS_CTRL` + `xTouchProbeEnable`, `xTouchProbeRising`, `xTouchProbeFalling`.
- `ST_AXIS_INFO` + `Status : ST_CiA402_Status` e i risultati TouchProbe
  (`xTouchProbeDoneR/F`, `xTouchProbeBusy`, `xTouchProbeError`,
  `TouchProbeRisingValue/FallingValue`).
- `GVL_SML_CONST.MAP_SIZE_INFO` 64 -> 128 (Info ora piu' grande).

### FB_AxisCtrl
- Aggiunte istanze `inst_Status : SML_Status` e `inst_TouchProbe : SML_TouchProbe`.
- `inst_Status` (read-only) -> popola `Info.Status.*`.
- `inst_TouchProbe` comandato da `Ctrl.xTouchProbe*` -> `Info.xTouchProbe*`.
- Errore TouchProbe (`_xTpErr`) alimenta `inst_Diag.xTouchProbeError` e
  `Info.xTouchProbeError`, ma **NON** lo stato di moto combinato (`eState`):
  il TouchProbe e' ortogonale al moto.

---

## Parita' con FB_SML — ora superset

| | FB_SML | FB_AxisCtrl (v9) |
|---|:--:|:--:|
| Power/Reset/Home/PP/PV/Jog/Stop | ✅ | ✅ |
| Diagnostica aggregata | ✅ | ✅ |
| **Status CiA402 completo** | ✅ | ✅ (Info.Status) |
| **TouchProbe** | ✅ | ✅ (Ctrl/Info) |
| **CSP + OTG** | ❌ | ✅ |
| eCmd + Ctrl/State + multi-asse + I_Axis | ❌ | ✅ |

`FB_AxisCtrl` copre ora tutto FB_SML **e in piu'** CSP/OTG, comando eCmd,
contratto dati e orchestrazione multi-asse. Le facce legacy (`FB_SML`,
`SML_AxisController`) restano non cablate; **rimozione opzionale** in una
prossima versione (con i FB non piu' usati: SyncPosition/Velocity, ecc.).

## Note tecniche / rischi
- Nomi output di `SML_Status` replicati 1:1 nella chiamata; se il tuo
  `SML_Status` ha un set di output diverso, allineare i `=>`.
- `ST_AXIS_INFO` con sotto-struct `Status`: accesso `Info.Status.OperationEnabled`.
  In `U_AXIS_INFO` la UNION contiene il nuovo Info piu' grande: `MAP_SIZE_INFO`
  portato a 128 (Check di MAPPING_out lo verifica).
- Restano i caveat precedenti (enum non-strict; scaling velocita'=posizione;
  metodi I_Axis come oggetti; MAPPING via flag runtime AXIS_MAP_ENABLE).

## Verifica
1. Build: `ST_CiA402_Status` risolto; `FB_AxisCtrl` compila con le due nuove
   istanze; nomi output `SML_Status` corretti.
2. Simulazione: `Info.Status.OperationEnabled` segue l'abilitazione; armando
   `Ctrl.xTouchProbeEnable`+`xTouchProbeRising` e forzando i bit TouchProbe del
   drive, `Info.xTouchProbeDoneR` e `TouchProbeRisingValue` si popolano.
3. I test esistenti (Livello A/B/multi-asse) restano validi (solo aggiunte).
