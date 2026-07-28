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
