# SML v5 — Changelog

**Base:** SML_v4
**Date:** 2026-07-22
**Scope:** adozione del pattern "Motion Layer" (Ctrl/State) da PLC_MOTION_LAYER,
**Livello A** (non-breaking). Solo aggiunte: nessuna modifica alle struct dati
né alle macchine a stati `iState`.

Riferimenti di design: `../NOTA_Pattern_MotionLayer.md`,
`BOZZA_LevelA_CtrlState.md`.

---

## Cosa introduce v5 (Livello A)

Un vocabolario di **avanzamento unificato** `E_PROGRESS`, esposto da ogni FB con
macchina a stati come nuovo output `eProgress`, per dare a app/HMI/diagnostica una
semantica di stato comune al posto degli `iState:INT` locali (che restano interni
e invariati).

### Nuovi file
- `E_PROGRESS.txt` — ENUM `(INVALID, INIT, BUSY, PREPARE, STARTUP, CHECK, DONE,
  ERROR) INT`. Senza `qualified_only` (letterali non qualificati, come
  `SML_DiagCode`).
- `f_GetProgress.txt` — estrae il progress da uno stato combinato (`MOD 100`).
- `f_GetState.txt` — estrae la base di stato funzionale (`/100*100`).
  Le due funzioni sono **infrastruttura per il futuro Livello B** (stato
  combinato); non ancora usate dalla proiezione.

### FB con nuovo output `eProgress` + proiezione a fine ciclo (12)
`SML_Power`, `SML_Home`, `SML_ProfilePosition`, `SML_ProfileVelocity`,
`SML_ProfileVelocity_Jog`, `SML_SyncPosition`, `SML_SyncVelocity`,
`SML_TouchProbe`, `SML_Reset`, `SML_Stop`, `SML_AxisController`, `FB_SML`.

### Convenzione di proiezione (uniforme)
`xError → PROGRESS_ERROR` · `<done-flag> → PROGRESS_DONE` ·
`iState = 0 → PROGRESS_INVALID` · altrimenti `PROGRESS_BUSY`.

Varianti motivate:
- **ProfileVelocity**: DONE = `xAtVelocity` (velocità di regime).
- **ProfileVelocity_Jog / SyncPosition / SyncVelocity**: moto continuo, nessun
  DONE discreto (solo INVALID/BUSY/ERROR).
- **TouchProbe**: DONE = `xDone_R OR xDone_F`.
- **Stop**: level-driven (no `iState`): `NOT xStop → INVALID`,
  `xStopped → DONE`, altrimenti BUSY.
- **SML_AxisController**: DONE = `iState = 20 AND Control.xInPosition`.
- **FB_SML** (facade, no `iState` unico): aggregato —
  `xActiveFault OR PwrError → ERROR`, `NOT PwrStatus → INVALID`,
  moto in corso → BUSY, abilitato e fermo → DONE.

---

## Garanzie (non-breaking)

- **Zero righe rimosse** in tutti i 12 FB (diff v4→v5 solo additivo).
- **Struct invariate**: `OpenSML_Axis` e `OpenSML_Control` identici a v4.
- **Macchine a stati invariate**: nessun `iState` o transizione modificata.
- Aggiungere un `VAR_OUTPUT` è compatibile con i chiamanti a parametri nominati
  (stile già usato in `FB_SML`/`SML_AxisController`).

## Non incluso (rimandato al Livello B)
- Split reale delle struct in `SML_AxisCtrl`/`SML_AxisState`.
- Uso dello stato combinato con `f_GetProgress`/`f_GetState`.
- FB di comando `FB_SmlAxisCtrl` (`eCmd`), interfaccia `I_SmlAxis`,
  orchestrazione `ARRAY[1..MAX_AXIS]`.

## Verifica consigliata
1. Import in CoDeSys + compilazione: nessun errore/warning nuovo, `E_PROGRESS`
   risolto ovunque. Verificare l'assegnazione `INT → E_PROGRESS` in
   `f_GetProgress`.
2. Test su asse reale o `xSimulation`: `eProgress` segue
   INVALID → BUSY → DONE → (ERROR su fault) → INVALID dopo reset.
