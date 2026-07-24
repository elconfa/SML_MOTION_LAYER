# SML v7 — Changelog

**Base:** SML_v6
**Date:** 2026-07-22
**Scope:** Livello B — **orchestrazione multi-asse** `ARRAY[1..MAX_AXIS]`.
Aggiunge il `MAIN` che esegue N istanze di `FB_AxisCtrl`, con GVL
degli array di contratto dati e accesso a interfaccia.

Riferimenti: `CHANGELOG_v6.md` (core mono-asse Livello B),
`../NOTA_Pattern_MotionLayer.md`.

---

## Novità v7

### Costanti
- `GVL_SML_CONST.txt` — `{attribute 'qualified_only'}`: `MAX_AXIS` (default 4)
  e `PROGRESS_SPAN` (100). Consolidata qui la costante prima locale nelle
  funzioni.
- `f_GetProgress` / `f_GetState` aggiornate: ora usano
  `GVL_SML_CONST.PROGRESS_SPAN` (rimossa la `VAR CONSTANT` locale).

### Orchestrazione
- `GVL_AXIS.txt` — array `[1..MAX_AXIS]` di: `Axis` (immagini PDO), `Ctrl`,
  `Data`, `State`, `Info`, `Control` (FB_AxisCtrl) e `ItfSmlAxis` (I_Axis).
- `MAIN.txt` — `FOR nAxis := 1 TO MAX_AXIS`: aggiorna `ItfSmlAxis[n] := Control[n]`
  e chiama `Control[n](Axis/Ctrl/Data/State/Info[n])`. Nessuna macchina di INIT
  dedicata (abilitazione per asse via `eCmd = AXIS_ENABLE`).

### Test
- `PRG_MultiAxis_Test.txt` — harness in `xSimulation`: usa gli array reali di
  `GVL_AXIS`, esegue la stessa logica di MAIN + emulatore CiA402, guida l'asse 1
  in CSP e l'asse 2 in JOG **contemporaneamente** e verifica l'indipendenza.

---

## Differenza vs PLC_MOTION_LAYER

PLC_MOTION_LAYER cabla con array di puntatori (`ADR`/`REF=`) e ha una macchina
di INIT asse-per-asse. In SML il contratto e' passato per **VAR_IN_OUT**, quindi
il MAIN e' molto piu' semplice: nessun cablaggio a puntatori, nessun INIT.

| PLC_MOTION_LAYER MAIN | SML v7 MAIN |
|---|---|
| `Control[n].AxisCtrl := ADR(...)` (molti puntatori) | passaggio VAR_IN_OUT nella chiamata |
| `ItfMcAxis[n] := Control[n]` | `ItfSmlAxis[n] := Control[n]` |
| INIT state machine (AXIS_INIT per asse) | nessuna (enable via eCmd) |
| `FOR n ... Control[n]()` | `FOR n ... Control[n](...)` |

---

## Note tecniche / rischi (CoDeSys)
- Bound array da costante globale (`GVL_SML_CONST.MAX_AXIS`): supportato
  (VAR_GLOBAL CONSTANT).
- `ItfSmlAxis[n] := Control[n]`: assegnazione FB->interfaccia (FB implementa
  I_Axis): valida.
- MAIN e PRG_MultiAxis_Test **non** vanno nello stesso task (doppia chiamata
  degli stessi FB). In produzione: MAIN + `SML_TC3Link` per l'I/O; in
  simulazione: il banco.
- Assi non comandati (eCmd = AXIS_NULL) restano idle senza errore.
- Restano i caveat v6 (enum non-strict, scaling velocita'=posizione, creare i
  METHOD/PROPERTY di I_Axis all'import).

## Verifica
1. Build: `GVL_SML_CONST`/`GVL_AXIS`/`MAIN` risolti; array dimensionati da
   `MAX_AXIS`.
2. `PRG_MultiAxis_Test` in un task (senza MAIN): `xTestPassed` -> TRUE;
   osservare `GVL_AXIS.State[1]` e `[2]` con comandi diversi in parallelo
   (`xIndependent`).
3. Cambiare `GVL_SML_CONST.MAX_AXIS` e verificare che gli array e il MAIN
   scalino senza altre modifiche.
