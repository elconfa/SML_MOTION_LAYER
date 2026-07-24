# SML v6 — Changelog

**Base:** SML_v5
**Date:** 2026-07-22
**Scope:** primo slice del **Livello B** del pattern Motion Layer
(PLC_MOTION_LAYER) su SML. Obiettivo strategico: **SML al posto delle
librerie `Tc2_*`/NC come livello di esecuzione**.

Decisioni (concordate): **FB foglia diretti**, **core mono-asse**,
**con CSP+OTG**. Rimandati: orchestrazione `ARRAY[1..MAX_AXIS]` in un
`MAIN`, layer MAPPING/UNION.

Riferimenti: `../NOTA_Pattern_MotionLayer.md`, `BOZZA_LevelA_CtrlState.md`,
`CHANGELOG_v5.md` (Livello A: `E_PROGRESS`).

---

## Novità v6

### Contratto dati (4 struct piatte, serializzabili)
- `ST_AXIS_CTRL.txt` — comando `eCmd` + config (Scale/CycleTime/PositionWindow/
  HomeOffset/xEmergencyStop/xSimulation).  [app -> layer]
- `ST_MOVE_DATA.txt` — setpoint in **unita' utente** (pos/vel/acc/dec/jerk). [app -> layer]
- `ST_AXIS_STATE.txt` — esito: `eState` (combinato), `eProgress`, `eCmdActive`,
  `xDone`, `xError`, diagnostica. [layer -> app]
- `ST_AXIS_INFO.txt` — valori attuali + bit (pos/vel/enabled/homed/moving/inPos). [layer -> app]

### Enum
- `E_AXIS_CTRL.txt` — comando asse (eCmd): NULL/ENABLE/DISABLE/RESET/HOME/
  MOVE_ABS/MOVE_REL/MOVE_VELOCITY/JOG_POS/JOG_NEG/MOVE_CSP/STOP.
- `E_AXIS_STATE.txt` — stato funzionale (multipli di 100) per lo **stato
  combinato** `eState = E_AXIS_STATE + E_PROGRESS`. Qui il Livello B **usa**
  `f_GetState`/`f_GetProgress` del Livello A.

### Interfaccia
- `I_Axis.txt` — equivalente di `I_McAxis`: metodi (Enable/Disable/Reset/Home/
  MoveAbsolute/MoveRelative/MoveVelocity/Jog/MoveFollow/Stop) + proprieta'
  (Position/Enabled). I metodi sono **setter** (impostano `Ctrl.eCmd`+`Data`,
  ritornano `State.eProgress`).

### FB di controllo
- `FB_AxisCtrl.txt` — `IMPLEMENTS I_Axis`. Macchina a comandi `eCmd` che
  pilota **direttamente** gli FB foglia CiA402 di SML (Power/Reset/Home/
  ProfilePosition/ProfileVelocity/Jog/Stop) + percorso **CSP con FB_S7RTT_OTG**,
  con diagnostica aggregata (SML_Diagnostics). Popola State/Info e lo stato
  combinato. Un solo `eCmd` per ciclo => nessun arbiter esplicito.
  Converte unita' utente -> grezze del drive via `Ctrl.Scale`.
- `FB_AxisCtrl_METHODS.txt` — codice reale dei metodi/proprieta' di I_Axis
  (da creare come oggetti METHOD/PROPERTY sotto il FB in CoDeSys).

### Test
- `PRG_LevelB_Test.txt` — guida `FB_AxisCtrl` in `xSimulation` con emulatore
  CiA402 inline: ENABLE->CSP->JOG->STOP->DISABLE; verifica `eProgress` e la
  decomposizione dello stato combinato.

---

## Mappatura MC_* Beckhoff -> FB SML (livello esecuzione)

| PLC_MOTION_LAYER | SML v6 |
|---|---|
| `FB_McAxisCtrl` (eCmd, wrappa MC_*) | `FB_AxisCtrl` (eCmd, wrappa FB foglia SML) |
| `I_McAxis` | `I_Axis` |
| `MC_Power` | `SML_Power` |
| `MC_Reset` | `SML_Reset` |
| `MC_Home` | `SML_Home` |
| `MC_MoveAbsolute` | `SML_ProfilePosition` |
| `MC_MoveVelocity` | `SML_ProfileVelocity` |
| `MC_Jog` | `SML_ProfileVelocity_Jog` |
| `MC_Halt`/`MC_Stop` | `SML_Stop` |
| NCI/CAM su AXIS_REF | (fuori scope; futuro) |
| Ctrl/State/Info/Data + E_PROGRESS combinato | idem (ST_AXIS_*) |

---

## Note tecniche / rischi (CoDeSys)
- Enum **non-strict** (nessun `{attribute 'strict'}`): letterali non qualificati e
  `TO_INT(enum)` per lo stato combinato. Se il progetto forza enum strict, va
  adattato (qui e in `f_GetProgress`).
- **Scaling**: `ST_MOVE_DATA` in unita' utente; conversione con `Ctrl.Scale`
  assume scala velocita'/accel = scala posizione. Per drive con unita' 0x6081/
  0x60FF diverse, raffinare in `FB_AxisCtrl`.
- **Metodi vs struct**: i metodi sono setter che accedono ai VAR_IN_OUT del FB.
  Chiamare il **corpo del FB ogni ciclo** (che lega i riferimenti IN_OUT); usare
  O i metodi O la struct `Ctrl`, non entrambi.
- **CSP + PP/PV nello stesso FB**: gli FB foglia inattivi (xExecute=FALSE) non
  forzano il modo (stesso principio di FB_SML); il blocco OTG gira solo con
  `eCmd = AXIS_MOVE_CSP`. Da validare su drive reale il passaggio di modo
  PP/PV <-> CSP (modo 8).
- Sorgenti `.txt` da importare in CoDeSys; i metodi di `I_Axis` vanno creati
  come oggetti METHOD/PROPERTY sotto `FB_AxisCtrl`.

## Verifica
1. Import + Build: `E_AXIS_CTRL`/`E_AXIS_STATE`/ST_*/`I_Axis` risolti;
   `FB_AxisCtrl` implementa tutti i metodi di `I_Axis`.
2. `PRG_LevelB_Test` in un task: `xTestPassed` -> TRUE; osservare `State.eState`
   (es. 306 = IDLE+DONE, 502 = MOVING+BUSY) e la coerenza `eStProg = eProgress`.
3. Confermare che v6 non modifichi gli FB foglia di v5 (solo aggiunte).
