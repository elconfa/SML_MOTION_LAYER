[English](API_Reference.md) | **Italiano**

# SML — Riferimento API asse (contratto dati)

Tutte le variabili per comandare e monitorare un asse `n`. Accesso: `GVL_AXIS.<struttura>[n].<campo>`.

Convenzione: **Ctrl** e **Data** le **scrivi tu** (app -> layer); **State** e **Info** le **leggi**
(layer -> app: il FB le riscrive ogni ciclo, non scriverle).

---

## Scrivibili

### `GVL_AXIS.Ctrl[n]` — comando + configurazione
| Variabile | Tipo | Default | Significato |
|---|---|---|---|
| `eCmd` | `E_AXIS_CTRL` | AXIS_NULL | **il comando** (vedi tabella eCmd sotto) |
| `xEmergencyStop` | BOOL | TRUE | **TRUE = marcia**, FALSE = quick-stop (CiA402 bit 2). Tienilo TRUE |
| `xSimulation` | BOOL | FALSE | specchia Target -> Actual (test senza drive) |
| `Scale` | LREAL | 4096.0 | counts per unita' utente (**> 0**) |
| `CycleTime` | LREAL | 0.001 | periodo task [s] — deve combaciare (OTG/CSP) |
| `PositionWindow` | LREAL | 0.01 | deadband "in posizione" [unita'] (CSP) |
| `HomeOffset` | DINT | 0 | offset homing [encoder counts -> 0x607C] |
| `xTouchProbeEnable` | BOOL | FALSE | arma l'hardware TouchProbe |
| `xTouchProbeRising` | BOOL | FALSE | arma il latch sul fronte di salita |
| `xTouchProbeFalling` | BOOL | FALSE | arma il latch sul fronte di discesa |

### `GVL_AXIS.Data[n]` — setpoint di moto (unita' utente)
| Variabile | Tipo | Default | Significato |
|---|---|---|---|
| `lrTargetPosition` | LREAL | 0 | target [unita'] per MOVE_ABS/REL/CSP |
| `lrVelocity` | LREAL | 10.0 | velocita' [unita'/s]; **con segno** per MOVE_VELOCITY |
| `lrAcceleration` | LREAL | 1000.0 | accelerazione [unita'/s^2] |
| `lrDeceleration` | LREAL | 1000.0 | decelerazione [unita'/s^2] |
| `lrJerk` | LREAL | 10000.0 | jerk [unita'/s^3] — solo CSP/OTG |

---

## Leggibili

### `GVL_AXIS.State[n]` — esito del comando
| Variabile | Tipo | Significato |
|---|---|---|
| `eProgress` | `E_PROGRESS` | avanzamento (INVALID...DONE/ERROR — tabella sotto) |
| `eState` | INT | stato combinato = stato funzionale + eProgress |
| `eCmdActive` | `E_AXIS_CTRL` | comando attualmente in esecuzione |
| `xDone` | BOOL | comando corrente completato (= eProgress DONE) |
| `xError` | BOOL | errore di moto aggregato |
| `DiagCode` | `SML_DiagCode` | condizione diagnostica live |
| `DiagText` | STRING(80) | testo della condizione live |
| `FirstFaultCode` | `SML_DiagCode` | primo guasto latchato (root cause) |
| `FirstFaultText` | STRING(80) | testo del primo guasto |

### `GVL_AXIS.Info[n]` — valori attuali
| Variabile | Tipo | Significato |
|---|---|---|
| `lrActualPosition` | LREAL | posizione attuale [unita'] |
| `lrActualVelocity` | LREAL | velocita' attuale [unita'/s] |
| `xEnabled` | BOOL | drive in OperationEnabled |
| `xHomed` | BOOL | homing eseguito almeno una volta (latch) |
| `xMoving` | BOOL | asse in movimento |
| `xInPosition` | BOOL | entro PositionWindow (percorso CSP) |
| `Status` | `ST_CiA402_Status` | decoder CiA402 completo (sotto) |
| `xTouchProbeDoneR` | BOOL | latch fronte salita acquisito |
| `xTouchProbeDoneF` | BOOL | latch fronte discesa acquisito |
| `xTouchProbeBusy` | BOOL | TouchProbe armato / in attesa |
| `xTouchProbeError` | BOOL | errore TouchProbe |
| `TouchProbeRisingValue` | DINT | posizione latchata su salita [encoder counts] |
| `TouchProbeFallingValue` | DINT | posizione latchata su discesa [encoder counts] |

### `GVL_AXIS.Info[n].Status` — bit CiA402 (decoder completo)
| Variabile | Tipo | | Variabile | Tipo |
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
| `TargetReached` | BOOL | | `ActTorque` | INT (0.1% del nominale) |
| `Halted` | BOOL | | `FollowingError` | DINT (encoder counts) |
| `VoltageEnabled` | BOOL | | | |
| `InternalLimit` | BOOL | | | |
| `ProfilePositionMode` | BOOL | | | |
| `ProfileVelocityMode` | BOOL | | | |
| `VelocityMode` (legacy) | BOOL | | | |

---

## Valori di `eCmd` (`E_AXIS_CTRL`)
`AXIS_NULL`(0) · `AXIS_ENABLE`(1) · `AXIS_DISABLE`(2) · `AXIS_RESET`(3) · `AXIS_HOME`(4) ·
`AXIS_MOVE_ABS`(5) · `AXIS_MOVE_REL`(6) · `AXIS_MOVE_VELOCITY`(7) · `AXIS_JOG_POS`(8) ·
`AXIS_JOG_NEG`(9) · `AXIS_MOVE_CSP`(10) · `AXIS_STOP`(11)

Vedi [`MANUALE_SML.it.md`](MANUALE_SML.it.md) §3 per il riferimento completo per comando (Data usati, DONE-quando).

## Valori di `eProgress` (`E_PROGRESS`)
`PROGRESS_INVALID`(0) · `INIT`(1) · `BUSY`(2) · `PREPARE`(3) · `STARTUP`(4) · `CHECK`(5) ·
**`DONE`**(6) · **`ERROR`**(7)

---

## Stato combinato `eState` (INT)

`eState` impacchetta "cosa fa l'asse" e "a che punto e'" in un unico numero:

```
eState = stato_funzionale + avanzamento
         (centinaia: COSA)   (unita': A CHE PUNTO)
```

- **centinaia** = stato funzionale (`E_AXIS_STATE`, multipli di 100)
- **unita'** = avanzamento (`E_PROGRESS`, i valori sopra)

Esempio: `502` = `500` (MOVING) + `2` (BUSY) -> "in movimento, in corso"; `306` = `300` (IDLE) + `6`
(DONE) -> "abilitato e fermo, pronto".

### Valori che ottieni davvero
| `eState` | Significato | Quando |
|---|---|---|
| `0` | non ancora eseguito | prima del primo ciclo |
| `200` | disabilitato, fermo | `AXIS_NULL` con drive OFF |
| `202` / `206` | disabilitazione / disabilitato (fatto) | `AXIS_DISABLE` |
| `300` | **IDLE**: abilitato e fermo, pronto | `AXIS_NULL` con drive ON |
| `302` / `306` | abilitazione / **abilitato (fatto)** | `AXIS_ENABLE` |
| `102` / `106` | reset in corso / fatto | `AXIS_RESET` |
| `402` / `406` | homing / **azzerato (fatto)** | `AXIS_HOME` |
| `502` / `506` | moto in corso / **al target** | `AXIS_MOVE_ABS/REL/CSP` |
| `602` | in rampa / jog tenuto | `AXIS_MOVE_VELOCITY`, `AXIS_JOG_*` |
| `606` | **velocita' raggiunta** | `AXIS_MOVE_VELOCITY` a regime |
| `702` / `706` | in arresto / **fermo** | `AXIS_STOP` |
| `907` | **errore** | qualsiasi fault |

Regola sulle unita': **0** = fermo/nessuna richiesta · **2** = sta lavorando · **6** = finito · **7** =
errore. (Il jog resta `x02`: moto continuo, nessun "DONE" discreto.) Lo stato combinato usa solo gli
avanzamenti 0/2/6/7.

### Come scomporlo
```pascal
base := GVL_AXIS.State[n].eState / 100;                 // 5 = MOVING
prog := GVL_AXIS.State[n].eState MOD 100;               // 2 = BUSY, 6 = DONE
// oppure con le funzioni della libreria:
base := SML.f_GetState(GVL_AXIS.State[n].eState);       // 500
prog := SML.f_GetProgress(GVL_AXIS.State[n].eState);    // 2 (SML.PROGRESS_BUSY)
```

Per la logica applicativa di solito basta leggere `State[n].eProgress` (gia' la parte "unita'") piu'
`State[n].xDone` / `xError`. `eState` e' comodo come singolo numero per un display HMI o un log
(es. "502" = in movimento).

---

## Diagnostica: capire se e dove l'asse è andato in errore

| Domanda | Dove leggere |
|---|---|
| C'è un errore di moto? | `State[n].xError` = TRUE (o `eProgress = PROGRESS_ERROR`, o `eState = 907`) |
| Di che tipo / quale FB? | `State[n].DiagCode` + `DiagText` |
| Qual è la causa prima? | `State[n].FirstFaultCode` / `FirstFaultText` (root cause latchata) |
| Quale comando era attivo? | `State[n].eCmdActive` (es. `AXIS_HOME`) |
| Fault grezzo del drive? | `Info[n].Status.Fault` / `Status.ErrId` (0x603F) |

Il **TouchProbe** è ortogonale: un suo errore **non** setta `xError`; leggi
`Info[n].xTouchProbeError`.

### Valori di `DiagCode` (`SML_DiagCode`)
| Codice | Val | Categoria | Origine | Comando tipico |
|---|---|---|---|---|
| `DIAG_OK` | 0 | ok | nessun fault/warning | — |
| `DIAG_WARNING` | 1 | warning | StatusWord bit 7 | qualsiasi |
| `DIAG_INTERNAL_LIMIT` | 2 | warning | StatusWord bit 11 | qualsiasi |
| `DIAG_DRIVE_FAULT` | 10 | drive | StatusWord bit 3 (Fault) | qualsiasi |
| `DIAG_DRIVE_ERRORCODE` | 11 | drive | Fault + Error_Code (0x603F) | qualsiasi |
| `DIAG_QUICK_STOP` | 12 | drive | QuickStop (bit 5 = 0) | qualsiasi |
| `DIAG_FOLLOWING_ERROR` | 13 | drive | StatusWord bit 13 | move/CSP |
| `DIAG_POWER_ERROR` | 20 | FB | `SML_Power` | `AXIS_ENABLE` |
| `DIAG_HOME_ERROR` | 21 | FB | `SML_Home` | `AXIS_HOME` |
| `DIAG_MOVE_ERROR` | 22 | FB | `SML_ProfilePosition` | `AXIS_MOVE_ABS/REL` |
| `DIAG_VEL_ERROR` | 23 | FB | `SML_ProfileVelocity` | `AXIS_MOVE_VELOCITY` |
| `DIAG_JOG_ERROR` | 24 | FB | `SML_ProfileVelocity_Jog` | `AXIS_JOG_POS/NEG` |
| `DIAG_TOUCHPROBE_ERROR` | 25 | FB | `SML_TouchProbe` (non ferma il moto) | TouchProbe |
| `DIAG_OTG_ERROR` | 26 | FB | `FB_S7RTT_OTG` (limiti/CycleTime) | `AXIS_MOVE_CSP` |

Fasce: **1..9** warning (moto continua) · **10..19** fault drive · **20..29** fault FB.
Reset con `AXIS_RESET`.

```pascal
IF GVL_AXIS.State[1].xError THEN
    CASE GVL_AXIS.State[1].DiagCode OF
        SML.DIAG_HOME_ERROR: (* homing fallito *) ;
        SML.DIAG_MOVE_ERROR: (* move fallito *) ;
    END_CASE
    GVL_AXIS.Ctrl[1].eCmd := SML.AXIS_RESET;
END_IF
```

---

## Note

- **Namespace:** con libreria pubblicata come `SML`, prefissa *tipi* ed *enum*
  (`SML.AXIS_MOVE_ABS`, `SML.PROGRESS_DONE`); i **campi** delle struct sono variabili tue e non si
  prefissano (`GVL_AXIS.Ctrl[n].eCmd`).
- **Alternativa via interfaccia** (`GVL_AXIS.ItfSmlAxis[n]`), al posto di scrivere `eCmd`: metodi
  `Enable()`, `Disable()`, `Reset()`, `Home()`, `MoveAbsolute(lrPosition, lrVelocity)`,
  `MoveRelative(lrDistance, lrVelocity)`, `MoveVelocity(lrVelocity)`,
  `Jog(xForward, xReverse, lrVelocity)`, `MoveFollow(lrPosition, lrVelocity)`, `Stop()`; piu' le
  proprieta' in lettura `Position` (LREAL), `Enabled` (BOOL).
- **Dettaglio diagnostica** (categorie `SML_DiagCode`, reset): vedi [`MANUALE_SML.it.md`](MANUALE_SML.it.md) §12.
