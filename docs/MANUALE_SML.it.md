[English](MANUALE_SML.md) | **Italiano**

# Manuale SML — Gestione Asse (Motion Layer CiA402)

**Versione:** SML_v9 · **Target:** CoDeSys 3.5 / TwinCAT 3 · CiA402 su EtherCAT

Manuale d'uso per un programmatore che non conosce il progetto ma deve **comandare
uno o più assi**. Se vieni da PLCopen (`MC_Power`, `MC_MoveAbsolute`, …): qui **non**
istanzi un FB per ogni funzione con decine di ingressi; scrivi **un comando** in una
struct e leggi lo **stato** in un'altra. Un solo FB per asse fa tutto.

---

## 1. Concetto in 1 minuto

Ogni asse è gestito da un'istanza di **`FB_AxisCtrl`**. Tu interagisci con
**quattro strutture** (il "contratto dati"):

```
   APPLICAZIONE
      │  scrive Ctrl (comando) + Data (setpoint)
      ▼
   ┌───────────────────────────────┐
   │  FB_AxisCtrl  (un per asse) │  ← esegue, pilota gli FB CiA402
   └───────────────────────────────┘
      │  scrive State (esito) + Info (valori attuali)
      ▼
   APPLICAZIONE legge
```

- **`Ctrl`** — cosa deve fare l'asse (`eCmd`) + configurazione.  *(tu scrivi)*
- **`Data`** — numeri del movimento: posizione, velocità, acc, dec, jerk (unità utente).  *(tu scrivi)*
- **`State`** — esito: avanzamento, fatto/errore, diagnostica.  *(tu leggi)*
- **`Info`** — valori attuali: posizione, velocità, bit di stato, TouchProbe.  *(tu leggi)*

Comandi in **due modi equivalenti**:
1. **Struct**: `GVL_AXIS.Ctrl[1].eCmd := AXIS_MOVE_ABS;`
2. **Metodi** (interfaccia `I_Axis`): `GVL_AXIS.ItfSmlAxis[1].MoveAbsolute(50.0, 100.0);`

Il comando **`eCmd` è "a livello"** (level): resta attivo finché non lo cambi.
Un solo `eCmd` per asse → un solo movimento alla volta (niente conflitti).

---

## 2. Avvio rapido (copia-incolla)

Asse 1, in simulazione o su HW già collegato:

```pascal
// una-tantum / configurazione
GVL_AXIS.Ctrl[1].Scale     := 4096.0;   // counts per unità (mm, giri, …)
GVL_AXIS.Ctrl[1].CycleTime := 0.001;    // = periodo del task [s]

// 1) abilita
GVL_AXIS.Ctrl[1].eCmd := AXIS_ENABLE;
// attendi: GVL_AXIS.Info[1].xEnabled = TRUE  (State.eProgress = PROGRESS_DONE)

// 2) homing
GVL_AXIS.Ctrl[1].eCmd := AXIS_HOME;
// attendi: GVL_AXIS.State[1].xDone = TRUE

// 3) muovi a 50 mm a 100 mm/s
GVL_AXIS.Data[1].lrTargetPosition := 50.0;
GVL_AXIS.Data[1].lrVelocity       := 100.0;
GVL_AXIS.Ctrl[1].eCmd := AXIS_MOVE_ABS;
// attendi: GVL_AXIS.State[1].xDone = TRUE
```

Nota: `MAIN` chiama già `FB_AxisCtrl` per tutti gli assi ogni ciclo. Tu tocchi
solo `GVL_AXIS.Ctrl[]`/`Data[]` e leggi `State[]`/`Info[]`.

---

## 3. Riferimento comandi `eCmd` (come le pagine MC_*)

Imposti `Ctrl.eCmd` al valore voluto. La colonna "DONE quando" dice quando
`State.xDone` diventa TRUE (`State.eProgress = PROGRESS_DONE`).

| eCmd | Cosa fa | Data usati | DONE quando | Note |
|---|---|---|---|---|
| `AXIS_NULL` | nessun comando; mantiene lo stato | — | — | l'asse resta com'è (idle o in errore) |
| `AXIS_ENABLE` | abilita il drive (Power on) | — | `Info.xEnabled` | equivale a MC_Power |
| `AXIS_DISABLE` | disabilita il drive | — | drive non abilitato | Shutdown CiA402 |
| `AXIS_RESET` | reset fault (drive + FB) | — | fault sparito | equivale a MC_Reset |
| `AXIS_HOME` | homing | `HomeOffset` (in Ctrl) | homing completato | MC_Home |
| `AXIS_MOVE_ABS` | move assoluto (PP) | `lrTargetPosition`, `lrVelocity`, `lrAcceleration`, `lrDeceleration` | target raggiunto | MC_MoveAbsolute |
| `AXIS_MOVE_REL` | move relativo (PP) | come sopra (`lrTargetPosition` = distanza) | target raggiunto | MC_MoveRelative |
| `AXIS_MOVE_VELOCITY` | velocità continua (PV) | `lrVelocity` (con **segno**), `lrAcceleration`, `lrDeceleration` | velocità raggiunta | MC_MoveVelocity |
| `AXIS_JOG_POS` | jog positivo (tenuto) | `lrVelocity` (magnitudo) | — (moto continuo) | MC_Jog + |
| `AXIS_JOG_NEG` | jog negativo (tenuto) | `lrVelocity` (magnitudo) | — (moto continuo) | MC_Jog − |
| `AXIS_MOVE_CSP` | inseguimento CSP+OTG jerk-limited | `lrTargetPosition`, `lrVelocity`, `lrAcceleration`, `lrJerk` | entro `PositionWindow` | ri-targettabile **online** |
| `AXIS_STOP` | Halt (decelerazione controllata) | — | asse fermo | MC_Halt |

**Regole d'oro**
- Per **abilitare** basta un qualsiasi comando ≠ `AXIS_NULL`/`AXIS_DISABLE`
  (l'enable è implicito). Consigliato: `AXIS_ENABLE` esplicito prima di muovere.
- **`AXIS_DISABLE`** toglie l'abilitazione. **`AXIS_NULL`** la mantiene.
- **Move discreto (PP)**: imposta `Data` **poi** `eCmd`. Per un **nuovo** target
  discreto, ri-triggera (passa da `AXIS_NULL` e torna a `AXIS_MOVE_ABS`, oppure
  emetti un altro comando). Per re-targeting **fluido/online** usa `AXIS_MOVE_CSP`
  (aggiorni `lrTargetPosition` mentre l'asse insegue).
- **Jog**: tieni `AXIS_JOG_POS`/`NEG`; rilascia mettendo `AXIS_NULL` o `AXIS_STOP`.

### 3.1 Esempio minimo per ogni comando

Tutto su `GVL_AXIS.…[1]` (asse 1). Imposta i `Data` **prima** di `eCmd`.

```pascal
// -- ENABLE / DISABLE / RESET -----------------------------------
GVL_AXIS.Ctrl[1].eCmd := AXIS_ENABLE;   // abilita  -> attendi Info[1].xEnabled
GVL_AXIS.Ctrl[1].eCmd := AXIS_DISABLE;  // disabilita (drive off)
GVL_AXIS.Ctrl[1].eCmd := AXIS_RESET;    // reset fault -> attendi State[1].xDone

// -- HOME -------------------------------------------------------
GVL_AXIS.Ctrl[1].HomeOffset := 0;       // offset [encoder counts], opzionale
GVL_AXIS.Ctrl[1].eCmd := AXIS_HOME;     // -> attendi State[1].xDone

// -- MOVE_ABS / MOVE_REL (PP, discreti) -------------------------
GVL_AXIS.Data[1].lrTargetPosition := 50.0;    // ABS: posizione; REL: distanza
GVL_AXIS.Data[1].lrVelocity       := 100.0;
GVL_AXIS.Data[1].lrAcceleration   := 1000.0;
GVL_AXIS.Data[1].lrDeceleration   := 1000.0;
GVL_AXIS.Ctrl[1].eCmd := AXIS_MOVE_ABS;       // o AXIS_MOVE_REL -> attendi State[1].xDone

// -- MOVE_VELOCITY (PV, continuo; il SEGNO = direzione) ---------
GVL_AXIS.Data[1].lrVelocity := -30.0;         // negativo = indietro
GVL_AXIS.Ctrl[1].eCmd := AXIS_MOVE_VELOCITY;  // a regime -> State[1].xDone (velocita' raggiunta)

// -- JOG (tenuto finche' resta il comando) ----------------------
GVL_AXIS.Data[1].lrVelocity := 20.0;          // magnitudo (senza segno)
GVL_AXIS.Ctrl[1].eCmd := AXIS_JOG_POS;        // o AXIS_JOG_NEG; rilascia con AXIS_NULL/AXIS_STOP

// -- MOVE_CSP (inseguimento online, jerk-limited) ---------------
GVL_AXIS.Data[1].lrTargetPosition := 120.0;   // ri-scrivibile OGNI ciclo
GVL_AXIS.Data[1].lrVelocity     := 200.0;
GVL_AXIS.Data[1].lrAcceleration := 2000.0;
GVL_AXIS.Data[1].lrJerk         := 20000.0;
GVL_AXIS.Ctrl[1].eCmd := AXIS_MOVE_CSP;       // -> State[1].xDone quando entro PositionWindow

// -- STOP / NULL ------------------------------------------------
GVL_AXIS.Ctrl[1].eCmd := AXIS_STOP;           // Halt controllato -> attendi State[1].xDone (fermo)
GVL_AXIS.Ctrl[1].eCmd := AXIS_NULL;           // nessun comando; mantiene lo stato (idle/errore)
```

Per il TouchProbe (parallelo al moto, non e' un `eCmd`) vedi §6.

---

## 4. Comandare via interfaccia `I_Axis` (metodi)

Alternativa alla struct, utile per coordinatori/librerie. I metodi sono **setter**:
impostano `Ctrl.eCmd`+`Data` e ritornano l'avanzamento (`E_PROGRESS`). L'esecuzione
avviene al normale ciclo del FB.

```pascal
VAR ax : I_Axis; END_VAR
ax := GVL_AXIS.ItfSmlAxis[1];

ax.Enable();
ax.Home();
ax.MoveAbsolute(lrPosition := 50.0, lrVelocity := 100.0);
ax.MoveVelocity(lrVelocity := -30.0);      // PV, segno = direzione
ax.Jog(xForward := TRUE, xReverse := FALSE, lrVelocity := 20.0);
ax.MoveFollow(lrPosition := 120.0, lrVelocity := 80.0);  // CSP online
ax.Stop();
ax.Disable();
ax.Reset();

// proprietà (lettura)
rPos := ax.Position;      // posizione attuale [unità]
bOn  := ax.Enabled;       // drive abilitato
```

Usa **o** i metodi **o** la struct sullo stesso asse, non entrambi nello stesso ciclo.

---

## 5. Leggere lo stato

### 5.1 `State` — esito del comando (tutti i campi)
| Campo | Tipo | Spiegazione dettagliata |
|---|---|---|
| `eState` | INT | **stato combinato** = stato funzionale (centinaia) + avanzamento (unità). Un solo numero, comodo per HMI/log. Scomposizione in §5.2 |
| `eProgress` | `E_PROGRESS` | avanzamento del comando corrente, già scomposto (è la parte "unità" di `eState`). Valori nella tabella sotto |
| `eCmdActive` | `E_AXIS_CTRL` | il comando **effettivamente in esecuzione** nel FB. Dice "cosa sta facendo l'asse" (es. `AXIS_HOME`), utile anche in caso di errore |
| `xDone` | BOOL | TRUE quando il comando corrente è **completato** (equivale a `eProgress = PROGRESS_DONE`). Per i comandi continui (JOG) non diventa TRUE |
| `xError` | BOOL | errore di moto **aggregato** (Power OR Home OR Move OR Vel OR Jog OR CSP OR Reset). **Non** include il TouchProbe |
| `DiagCode` | `SML_DiagCode` | codice della condizione diagnostica **live** (tabella completa in §12.2) |
| `DiagText` | STRING(80) | testo leggibile della condizione live (per HMI/log) |
| `FirstFaultCode` | `SML_DiagCode` | **primo** guasto latchato = root cause; resta finché non fai `AXIS_RESET` |
| `FirstFaultText` | STRING(80) | testo del primo guasto |

**Valori di `eProgress` (`E_PROGRESS`)**
| Valore | N | Significato |
|---|---|---|
| `PROGRESS_INVALID` | 0 | non inizializzato / a riposo, nessuna richiesta |
| `PROGRESS_INIT` | 1 | richiesta di start ricevuta |
| `PROGRESS_BUSY` | 2 | in esecuzione |
| `PROGRESS_PREPARE` | 3 | preparazione parametri / precondizioni |
| `PROGRESS_STARTUP` | 4 | fase di avvio |
| `PROGRESS_CHECK` | 5 | verifica esito / avanzamento indice |
| `PROGRESS_DONE` | 6 | completato con successo (= `xDone`) |
| `PROGRESS_ERROR` | 7 | errore, terminale fino al reset (= `xError`) |

> Lo **stato combinato** `eState` usa solo gli avanzamenti **0/2/6/7**; i valori
> intermedi 1/3/4/5 possono comparire in `eProgress` durante le transizioni interne.

### 5.2 Stato combinato `eState`
`eState = E_AXIS_STATE + E_PROGRESS`. Esempi: `306` = `AXIS_STATE_IDLE(300)` +
`PROGRESS_DONE(6)`; `502` = `AXIS_STATE_MOVING(500)` + `PROGRESS_BUSY(2)`.
Per scomporlo usa le funzioni:
```pascal
eBase := f_GetState(State.eState);      // 500  (parte funzionale)
eProg := f_GetProgress(State.eState);   // 2    (E_PROGRESS.PROGRESS_BUSY)
```
Stati funzionali: `NULL 0, INIT 100, DISABLED 200, IDLE 300, HOMING 400,
MOVING 500, VELOCITY 600, STOPPING 700, ERROR 900`.
Tabella dei valori concreti di `eState` (0/200/300/306/402/406/502/506/606/907…):
vedi [`API_Reference.it.md`](API_Reference.it.md#stato-combinato-estate-int).

### 5.3 `Info` — valori attuali (tutti i campi)
| Campo | Tipo | Spiegazione dettagliata |
|---|---|---|
| `lrActualPosition` | LREAL | posizione attuale in unità utente (raw drive / `Scale`) |
| `lrActualVelocity` | LREAL | velocità attuale [unità/s] |
| `xEnabled` | BOOL | drive in **OperationEnabled** (abilitato, può muovere). = `Status.OperationEnabled` |
| `xHomed` | BOOL | homing eseguito **almeno una volta** (latch; resta TRUE fino a spegnimento / nuovo home) |
| `xMoving` | BOOL | asse in movimento (velocità ≠ 0) |
| `xInPosition` | BOOL | entro `Ctrl.PositionWindow` dal target (percorso CSP/OTG) |
| `Status` | `ST_CiA402_Status` | decoder CiA402 completo — ogni bit in §5.4 |
| `xTouchProbeDoneR` | BOOL | latch sul fronte di **salita** acquisito (valore in `TouchProbeRisingValue`) |
| `xTouchProbeDoneF` | BOOL | latch sul fronte di **discesa** acquisito (valore in `TouchProbeFallingValue`) |
| `xTouchProbeBusy` | BOOL | TouchProbe armato e in attesa del trigger |
| `xTouchProbeError` | BOOL | errore TouchProbe (**ortogonale**: non setta `State.xError`) |
| `TouchProbeRisingValue` | DINT | posizione latchata sul fronte di salita [encoder counts] |
| `TouchProbeFallingValue` | DINT | posizione latchata sul fronte di discesa [encoder counts] |

### 5.4 `Info.Status` — decoder CiA402 completo (`ST_CiA402_Status`)
Bit e valori decodificati da `SML_Status` a partire dalla StatusWord (0x6041), dal
Mode-of-operation display (0x6061) e dagli oggetti di errore. Tutti in sola lettura.

**Macchina di stato CiA402** (stato del drive)
| Campo | Spiegazione |
|---|---|
| `NotReadyToSwitchOn` | drive non pronto (inizializzazione / autotest in corso) |
| `SwitchOnDisabled` | accensione inibita: stato sicuro, attende la sequenza di enable |
| `ReadyToSwitchOn` | pronto per lo Switch-On (StatusWord bit 0) |
| `SwitchedOn` | alimentato ma senza moto abilitato (bit 1) |
| `OperationEnabled` | **abilitato**: può muovere (bit 2). = `Info.xEnabled` |
| `QuickStopActive` | quick-stop in corso (bit 5 = 0) |
| `FaultReactionActive` | reazione a un fault in corso (decelerazione controllata) |
| `Fault` | drive in **fault**: richiede `AXIS_RESET` (bit 3) |

**Stato generale**
| Campo | Spiegazione |
|---|---|
| `Warning` | warning presente, il moto continua (bit 7) |
| `TargetReached` | target raggiunto — dipende dal modo (bit 10) |
| `Halted` | moto arrestato (dopo Halt/Stop) |
| `VoltageEnabled` | tensione di potenza presente (bit 4) |
| `InternalLimit` | limite interno attivo: coppia/velocità/posizione clampata (bit 11) |

**Modo operativo attivo** (da 0x6061; uno solo TRUE alla volta)
| Campo | Modo |
|---|---|
| `ProfilePositionMode` | PP — profilo di posizione (1) |
| `ProfileVelocityMode` | PV — profilo di velocità (3) |
| `VelocityMode` | modo velocità legacy (2) |
| `TorqueProfileMode` | profilo di coppia (4) |
| `HomingMode` | homing (6) |
| `SyncPositionMode` | CSP — sincrono di posizione (8) |
| `SyncVelocityMode` | CSV — sincrono di velocità (9) |

**Moto**
| Campo | Spiegazione |
|---|---|
| `Moving` | asse in movimento |
| `StandStill` | asse fermo (a regime, velocità nulla) |
| `Homed` | riferimento / zero valido |

**Errore / dettaglio** (valori, non solo bit)
| Campo | Tipo | Spiegazione |
|---|---|---|
| `Err` | BOOL | Error_Code ≠ 0 |
| `ErrId` | UINT | codice errore del drive (0x603F) |
| `ActTorque` | INT | coppia attuale (0x6077) [0.1% del nominale] |
| `FollowingError` | DINT | errore di inseguimento (0x60F4) [encoder counts] |

---

## 6. TouchProbe (latch di posizione)

È **parallelo** al moto (non è un `eCmd`). Comandi via `Ctrl`, leggi via `Info`.
```pascal
GVL_AXIS.Ctrl[1].xTouchProbeEnable := TRUE;   // arma l'hardware
GVL_AXIS.Ctrl[1].xTouchProbeRising := TRUE;   // arma fronte di salita
// ... al trigger:
// Info.xTouchProbeDoneR = TRUE
// Info.TouchProbeRisingValue = posizione latchata [encoder counts]
```
Un errore TouchProbe va nella **diagnostica** e in `Info.xTouchProbeError`, ma
**non** mette l'asse in stato di moto ERROR.

---

## 7. Configurazione (`Ctrl`)

| Campo | Default | Significato |
|---|---|---|
| `xEmergencyStop` | TRUE | **TRUE = marcia**, FALSE = quick-stop (CiA402 bit 2). Tienilo TRUE per funzionare |
| `xSimulation` | FALSE | specchia `Target → Actual` (test senza drive) |
| `Scale` | 4096.0 | counts per unità utente. **> 0** |
| `CycleTime` | 0.001 | periodo del task [s]. Deve combaciare (per l'OTG) |
| `PositionWindow` | 0.01 | deadband "in posizione" [unità] (CSP) |
| `HomeOffset` | 0 | offset homing [encoder counts → 0x607C] |
| `xTouchProbe*` | FALSE | controllo TouchProbe (vedi 6) |

Esempio di configurazione (una-tantum, tipicamente all'avvio):
```pascal
// Encoder 20 bit (1048576 counts/giro) su vite passo 10 mm -> counts/mm
GVL_AXIS.Ctrl[1].Scale          := 1048576.0 / 10.0;  // = 104857.6 counts/mm
GVL_AXIS.Ctrl[1].CycleTime      := 0.001;   // task a 1 ms (deve combaciare, per CSP/OTG)
GVL_AXIS.Ctrl[1].PositionWindow := 0.02;    // "in posizione" entro 0.02 mm (CSP)
GVL_AXIS.Ctrl[1].HomeOffset     := 0;       // lo zero coincide con lo zero macchina
GVL_AXIS.Ctrl[1].xEmergencyStop := TRUE;    // consenso marcia (TRUE = run)
```

`Data`: `lrTargetPosition` [unità], `lrVelocity` [unità/s, con segno per PV],
`lrAcceleration`, `lrDeceleration` [unità/s²], `lrJerk` [unità/s³, solo CSP].

---

## 8. Cosa fa ogni file

### Contratto dati (DUT)
| File | Ruolo |
|---|---|
| `ST_AXIS_CTRL` | comando (`eCmd`) + configurazione. App → layer |
| `ST_MOVE_DATA` | setpoint di moto in unità utente. App → layer |
| `ST_AXIS_STATE` | esito + stato combinato + diagnostica. Layer → app |
| `ST_AXIS_INFO` | valori attuali + Status CiA402 + TouchProbe. Layer → app |
| `ST_CiA402_Status` | tutti i bit di stato CiA402 (dentro `Info.Status`) |

### Enum (DUT)
| File | Ruolo |
|---|---|
| `E_AXIS_CTRL` | i comandi (`eCmd`) |
| `E_AXIS_STATE` | stati funzionali (×100, per lo stato combinato) |
| `E_PROGRESS` | avanzamento unificato (INVALID…DONE/ERROR) |
| `SML_DiagCode` | codici diagnostici categorizzati |

### Controllo
| File | Ruolo |
|---|---|
| `FB_AxisCtrl` | **il FB di controllo asse**: traduce `eCmd` in chiamate ai FB foglia + CSP/OTG; popola State/Info; implementa `I_Axis` |
| `FB_AxisCtrl_METHODS` | corpo dei metodi/proprietà di `I_Axis` (oggetti figli del FB) |
| `I_Axis` | interfaccia a metodi (Enable/Home/MoveAbsolute/…) |

### Esecuzione (FB foglia CiA402 — pilotati da FB_AxisCtrl)
| File | Ruolo (≈ PLCopen) |
|---|---|
| `SML_Power` | abilitazione drive (≈ MC_Power) |
| `SML_Reset` | reset fault (≈ MC_Reset) |
| `SML_Home` | homing (≈ MC_Home) |
| `SML_ProfilePosition` | move assoluto/relativo, modo PP (≈ MC_MoveAbsolute) |
| `SML_ProfileVelocity` | velocità, modo PV (≈ MC_MoveVelocity) |
| `SML_ProfileVelocity_Jog` | jog (≈ MC_Jog) |
| `SML_Stop` | Halt (≈ MC_Halt) |
| `SML_TouchProbe` | latch di posizione su fronte |
| `SML_Status` | decoder di tutti i bit CiA402 |
| `SML_Diagnostics` | aggrega errori → DiagCode/DiagText + primo guasto |
| `FB_S7RTT_OTG` | generatore di traiettoria jerk-limited (CSP) |

### Orchestrazione
| File | Ruolo |
|---|---|
| `GVL_App` | configurazione applicazione: `MAX_AXIS` (numero assi) |
| `GVL_SML_CONST` | costanti di libreria: `PROGRESS_SPAN`, `MAP_SIZE_*` |
| `GVL_AXIS` | array `[1..MAX_AXIS]` di Axis/Ctrl/Data/State/Info + `Control` (FB) + `ItfSmlAxis` |
| `MAIN` | esegue tutti gli assi ogni ciclo; chiama I/O bridge e MAPPING |

### I/O verso il drive (vedi `GUIDA_IO_Linking.it.md`)
| File | Ruolo |
|---|---|
| `OpenSML_Axis` | immagine PDO CiA402 dell'asse (ControlWord/StatusWord/…); `GVL_AXIS.Axis[n]` |
| `ST_DriveOut` / `ST_DriveIn` | metà output / input di OpenSML_Axis, per il bridge |
| `GVL_IO` | immagini `AT %Q*/%I*` + flag `IO_LINK_ENABLE` |
| `FB_IoLink_In` / `_Out` | loop di copia drive↔struct (FB di libreria, istanziati in MAIN) |

### MAPPING bus (esporre Ctrl/State su ADS/fieldbus — opzionale)
| File | Ruolo |
|---|---|
| `U_AXIS_CTRL` / `U_MOVE_DATA` / `U_AXIS_STATE` / `U_AXIS_INFO` | UNION struct/grezzo |
| `GVL_AXIS_MAP` | array UNION lato bus + flag `AXIS_MAP_ENABLE` |
| `FB_Mapping_In` / `FB_Mapping_Out` | copia bus↔interno (FB di libreria, gate da xEnable) |

### Utility / Test
| File | Ruolo |
|---|---|
| `f_GetProgress` / `f_GetState` | scompongono lo stato combinato |
| `PRG_LevelA_Test` / `PRG_LevelB_Test` / `PRG_MultiAxis_Test` | banchi in simulazione |

---

## 9. Sequenze tipiche

### 9.1 Abilita → homing → move (con attese di stato)
```pascal
CASE iSeq OF
  0: GVL_AXIS.Ctrl[1].eCmd := AXIS_ENABLE;
     IF GVL_AXIS.Info[1].xEnabled THEN iSeq := 10; END_IF
 10: GVL_AXIS.Ctrl[1].eCmd := AXIS_HOME;
     IF GVL_AXIS.State[1].xDone THEN iSeq := 20; END_IF
 20: GVL_AXIS.Data[1].lrTargetPosition := 50.0;
     GVL_AXIS.Data[1].lrVelocity       := 100.0;
     GVL_AXIS.Ctrl[1].eCmd := AXIS_MOVE_ABS;
     IF GVL_AXIS.State[1].xDone THEN iSeq := 30; END_IF
 30: GVL_AXIS.Ctrl[1].eCmd := AXIS_NULL;  // fermo, abilitato
END_CASE
```

### 9.2 Gestione errore
```pascal
IF GVL_AXIS.State[1].xError THEN
    // leggi GVL_AXIS.State[1].DiagText / FirstFaultText
    GVL_AXIS.Ctrl[1].eCmd := AXIS_RESET;      // pulisci il fault
    // poi ri-abilita: AXIS_ENABLE
END_IF
```

---

## 10. Integrazione nel progetto

1. **Import** dei file (ordine e passi in `IMPORT_CHECKLIST.it.md`). Ricorda di creare
   i METHOD/PROPERTY di `I_Axis` sotto `FB_AxisCtrl` (`FB_AxisCtrl_METHODS`).
2. **Numero assi**: imposta `GVL_App.MAX_AXIS`.
3. **Task**: metti **`MAIN`** in un task ciclico. (In simulazione usa un banco di
   test AL POSTO di MAIN.)
4. **I/O**: collega `GVL_AXIS.Axis[n]` ai PDO del drive — vedi `GUIDA_IO_Linking.it.md`
   (CoDeSys: link diretto; TwinCAT: bridge + `IO_LINK_ENABLE := TRUE`).
5. **Applicazione**: scrivi `GVL_AXIS.Ctrl[n]`/`Data[n]`, leggi `State[n]`/`Info[n]`.

---

## 11. Simulazione (senza drive)

Metti `Ctrl.xSimulation := TRUE` (specchia posizione). I banchi
`PRG_LevelB_Test` / `PRG_MultiAxis_Test` includono un mini-emulatore CiA402 e
provano la sequenza comandi: metti UN banco nel task (non MAIN) e verifica
`xTestPassed = TRUE`.

---

## 12. Diagnostica (`SML_DiagCode`)

Leggi `State.DiagCode`/`DiagText` (condizione **live**) e
`State.FirstFaultCode`/`FirstFaultText` (**primo** guasto latchato = root cause,
resta anche se la condizione live cambia; si azzera col reset).

### 12.1 Come sapere se — e dove — l'asse è andato in errore

Due livelli: "c'è un errore?" e "di quale comando / che tipo?".

| Domanda | Dove leggere |
|---|---|
| **C'è un errore di moto?** | `State[n].xError` = TRUE (o `eProgress = PROGRESS_ERROR`, o `eState = 907`) |
| **Di che tipo / quale FB?** | `State[n].DiagCode` (es. `DIAG_HOME_ERROR`) + `DiagText` per il testo |
| **Qual è la causa prima?** | `State[n].FirstFaultCode` / `FirstFaultText` (root cause latchata) |
| **Quale comando era attivo?** | `State[n].eCmdActive` (es. `AXIS_HOME`) |
| **Fault grezzo del drive?** | `Info[n].Status.Fault` / `Status.ErrId` (Error_Code 0x603F) |

> Il **TouchProbe** è a parte: un suo errore **non** setta `xError`; lo leggi in
> `Info[n].xTouchProbeError` (e `DiagCode = DIAG_TOUCHPROBE_ERROR`).

```pascal
IF GVL_AXIS.State[1].xError THEN
    CASE GVL_AXIS.State[1].DiagCode OF        // quale comando/FB ha fallito
        DIAG_HOME_ERROR:  ;                   // homing fallito
        DIAG_MOVE_ERROR:  ;                   // move PP fallito
        DIAG_DRIVE_FAULT: ;                   // fault del drive
    END_CASE
    sMsg  := GVL_AXIS.State[1].DiagText;       // testo condizione live (HMI)
    sRoot := GVL_AXIS.State[1].FirstFaultText; // testo causa prima
    GVL_AXIS.Ctrl[1].eCmd := AXIS_RESET;       // pulisce drive + FB foglia
END_IF
```

### 12.2 Tabella completa `SML_DiagCode`

| Codice | Val | Categoria | Significato / origine | Comando tipico |
|---|---|---|---|---|
| `DIAG_OK` | 0 | ok | nessun fault né warning | — |
| `DIAG_WARNING` | 1 | warning | StatusWord bit 7 (warning generico) | qualsiasi |
| `DIAG_INTERNAL_LIMIT` | 2 | warning | StatusWord bit 11 (limite interno attivo) | qualsiasi |
| `DIAG_DRIVE_FAULT` | 10 | drive | StatusWord bit 3 (Fault), senza codice | qualsiasi |
| `DIAG_DRIVE_ERRORCODE` | 11 | drive | Fault + Error_Code (0x603F) > 0 | qualsiasi |
| `DIAG_QUICK_STOP` | 12 | drive | QuickStop attivo (StatusWord bit 5 = 0) | qualsiasi |
| `DIAG_FOLLOWING_ERROR` | 13 | drive | StatusWord bit 13 (errore di inseguimento) | move/CSP |
| `DIAG_POWER_ERROR` | 20 | FB | errore sequenza abilitazione (`SML_Power`) | `AXIS_ENABLE` |
| `DIAG_HOME_ERROR` | 21 | FB | errore homing (`SML_Home`: stallo/timeout/abort) | `AXIS_HOME` |
| `DIAG_MOVE_ERROR` | 22 | FB | errore move PP (`SML_ProfilePosition`) | `AXIS_MOVE_ABS/REL` |
| `DIAG_VEL_ERROR` | 23 | FB | errore velocità (`SML_ProfileVelocity`) | `AXIS_MOVE_VELOCITY` |
| `DIAG_JOG_ERROR` | 24 | FB | errore jog (`SML_ProfileVelocity_Jog`) | `AXIS_JOG_POS/NEG` |
| `DIAG_TOUCHPROBE_ERROR` | 25 | FB | errore TouchProbe (**non** ferma il moto) | TouchProbe |
| `DIAG_OTG_ERROR` | 26 | FB | vincoli OTG non validi (limiti / `CycleTime`) | `AXIS_MOVE_CSP` |

Fasce: **0** ok · **1..9** warning (il moto continua) · **10..19** fault del drive ·
**20..29** fault di un FB foglia. Reset: **`AXIS_RESET`** (pulisce sia il drive sia gli FB foglia).

---

## 13. Errori tipici (gotchas)

- **Non parte / non si abilita** → `Ctrl.xEmergencyStop` deve essere **TRUE**
  (TRUE = marcia). Controlla `Info.Status.Fault`/`DiagText`.
- **Muove a posizione sbagliata** → `Ctrl.Scale` errato (counts/unità).
- **CSP non insegue / traiettoria strana** → `CycleTime` ≠ periodo reale del task.
- **Secondo move PP non parte** → è discreto: ri-triggera (passa da `AXIS_NULL`) o
  usa `AXIS_MOVE_CSP` per re-targeting online.
- **In posizione mai TRUE (CSP)** → `PositionWindow` troppo stretto.
- **TouchProbe non latcha** → PDO 0x60B8/9/A/B non mappati, o DC non attivo.

---

## 14. Glossario rapido

- **PP / PV / CSP**: Profile Position / Profile Velocity / Cyclic Synchronous Position (modo 8, con OTG).
- **OTG**: online trajectory generator (jerk-limited) = `FB_S7RTT_OTG`.
- **Unità utente**: le tue (mm, giri…); il layer converte in counts con `Scale`.
- **Stato combinato**: `eState = stato funzionale + avanzamento`.

---

## Appendice A — FB applicativa d'esempio (`FB_AxisCycleDemo`)

Esempio completo e importabile (`FB_AxisCycleDemo.txt`, **non** parte della
libreria) che mostra il pattern reale d'uso: una macchina a stati che comanda
un asse in un **ciclo continuo A↔B** con enable, homing una-tantum, soste,
stop e reset automatico su errore.

### A.1 Cosa fa
```
ENABLE → HOME (se non già azzerato) → A → sosta → B → sosta → A → … (fino a xStop)
errore in qualsiasi punto → RESET automatico → idle (serve nuovo xStart)
```

### A.2 Interfaccia
| Ingresso | Tipo | Uso |
|---|---|---|
| `nAxis` | UINT | indice asse in `GVL_AXIS` [1..MAX_AXIS] |
| `xStart` | BOOL | avvia il ciclo |
| `xStop` | BOOL | ferma (Halt) e chiude il ciclo |
| `lrPosA` / `lrPosB` | LREAL | le due posizioni [unità] |
| `lrVel` | LREAL | velocità dei move [unità/s] |
| `tDwell` | TIME | sosta a ogni estremo |

| Uscita | Tipo | Uso |
|---|---|---|
| `xBusy` / `xDone` / `xError` | BOOL | stato del ciclo |
| `iStep` | INT | passo corrente (per debug) |
| `sDiag` | STRING | testo diagnostico dell'asse |

### A.3 Le due tecniche dimostrate
1. **Comando → attesa esito**: ogni passo imposta `Ctrl.eCmd` (+ `Data`) e attende
   `GVL_AXIS.State[nAxis].xDone` prima di proseguire.
2. **Re-trigger dei move PP discreti**: tra `A` e `B` si passa per un passo con
   `AXIS_NULL` (la sosta), che crea il **fronte** necessario a far ripartire il
   move successivo. (Per re-targeting fluido useresti `AXIS_MOVE_CSP`.)
3. **Homing una-tantum**: salta l'homing se `Info.xHomed` è già TRUE.
4. **Gestione errore**: la supervisione in testa al FB intercetta `State.xError`
   e salta al passo di RESET.

### A.4 Come usarla
```pascal
PROGRAM PLC_APP
VAR
    cycleAx1 : FB_AxisCycleDemo;
    xAvvia, xFerma : BOOL;   // da HMI / pulsanti
END_VAR

cycleAx1(
    nAxis  := 1,
    xStart := xAvvia,
    xStop  := xFerma,
    lrPosA := 0.0,
    lrPosB := 250.0,
    lrVel  := 150.0,
    tDwell := T#1S);
// leggibili: cycleAx1.xBusy, cycleAx1.xError, cycleAx1.iStep, cycleAx1.sDiag
```
Mettila in un task **insieme a `MAIN`** (che esegue gli assi). Per due assi in
ciclo indipendente: due istanze con `nAxis := 1` e `nAxis := 2`.

### A.5 Provala in simulazione
Imposta `GVL_AXIS.Ctrl[1].xSimulation := TRUE` (o usa un banco con emulatore),
metti `MAIN` + `PLC_APP` nel task, forza `xAvvia := TRUE` e osserva `iStep`
avanzare 10→20→30→40→50→60→30… e `Info[1].lrActualPosition` oscillare tra A e B.

---

## Appendice B — Macchina completa a 2 assi (`PLC_APP`)

`PLC_APP.txt` è un esempio **realistico** e importabile (non parte della libreria):
misura/selezione pezzi su nastro. Asse 1 = nastro continuo (velocity + TouchProbe),
Asse 2 = posizionatore. Dimostra: due assi, TouchProbe, transizione velocity→PP
senza fermarsi, decisione su misura, moti simultanei, I/O macchina puliti.

### B.1 Ingressi / Uscite (integrazione)
| Ingresso (`PLC_APP.`) | Uso |
|---|---|
| `xStart` | avvia/mantiene il ciclo automatico |
| `xStop` | ferma (Halt) e va in idle |
| `xSafetyOk` | consenso sicurezza (TRUE = marcia; → `xEmergencyStop`) |
| `xPhotocell` | fotocellula misura, TRUE = coperta da un pezzo |
| `xExitPhotocell` | fotocellula di uscita, TRUE = pezzo scartato in transito |
| `xResume` | segnale esterno: riattiva il nastro dallo standby |
| `xHomeBelt` | ri-azzera (home) il nastro fermo in standby |
| `lrScale, lrCycleTime, lrBeltVel, lrBeltMove, lrThreshold, lrAx2Home, lrAx2Fwd, lrAx2Zero, lrAx2Vel, tWait, tIdle` | parametri (default d'esempio) |

| Uscita (`PLC_APP.`) | Uso |
|---|---|
| `xHomed` | homing completato |
| `xRunning` | ciclo attivo |
| `xStandby` | nastro fermo in standby (nessun pezzo) |
| `xError` / `sDiag` | errore asse + testo |
| `xOversize` | ultimo pezzo fuori misura |
| `xEjected` | impulso: scarto certificato espulso (fotocellula di uscita) |
| `nRejectCount` | contatore scarti espulsi |
| `lrLastMeasure` | ultima misura pezzo [mm] |
| `iStep` | passo corrente (debug) |

### B.2 Mappa dei passi (`iStep`)
```
 0 idle → 5 enable(2 assi) → [10 home nastro → 20 home asse2] (una volta)
   ↓
 30 nastro VELOCITY + arma TouchProbe
 40 attesa fotocellula COPERTA → nastro passa a PP relativo (lrBeltMove)
    │  (se nessun pezzo per tIdle → 100 STANDBY)
 50 attesa fotocellula LIBERATA → misura = |TP_falling − TP_rising|/Scale
      ├─ misura > soglia → nastro resta in VELOCITY, si torna a 30 SUBITO
      │                     (scarto espulso e contato in PARALLELO, pipelining)
      └─ misura ≤ soglia → 70 attesa nastro fermo → 75 attesa+asse2 avanti a +400
                            → 80 attesa asse2 → 85 attesa+(asse2→0 ∥ nastro velocity) → 30

 100 STANDBY: nastro decelera e ferma → 110 fermo
 110 attende xHomeBelt (→120 ri-azzera) o xResume (→30 riparte)
 120 home nastro sul posto → 110

 PARALLELO (sempre): fronte discesa xExitPhotocell → nRejectCount++, xEjected
200 STOP (Halt 2 assi) → idle     900 ERRORE (reset 2 assi) → idle
```

### B.3 Uso
```pascal
// nel task, INSIEME a MAIN:
PLC_APP.xSafetyOk      := xConsensoSicurezza;
PLC_APP.xPhotocell     := xFotocellulaMisura;
PLC_APP.xExitPhotocell := xFotocellulaUscita;
PLC_APP.xStart         := xAvviaMacchina;
PLC_APP.xStop          := xFermaMacchina;
PLC_APP.xResume        := xRipresaDaStandby;   // segnale esterno
PLC_APP.xHomeBelt      := xAzzeraNastro;       // in standby
// (opz.) PLC_APP.lrThreshold := 300.0;  PLC_APP.tIdle := T#5S;  ecc.
// leggi: PLC_APP.xRunning, PLC_APP.xStandby, PLC_APP.lrLastMeasure,
//        PLC_APP.xOversize, PLC_APP.nRejectCount, PLC_APP.iStep
```

### B.4 Requisiti sul drive (importante)
- **Fotocellula misura** cablata al **trigger TouchProbe** del drive dell'asse 1
  (per i latch rising/falling) **e** all'ingresso PLC `xPhotocell` (per sequenziare).
- **Fotocellula di uscita** (`xExitPhotocell`) a valle: certifica il passaggio del
  pezzo scartato (nessun attuatore: il pezzo cade dal nastro).
- **Homing asse 1**: metodo "set position" (sul posto).
- **Homing asse 2**: metodo con sensore in negativo + tacca encoder; il
  riferimento diventa `lrAx2Home` grazie a `HomeOffset` (impostato dall'app in
  counts). Sensore/indice li gestisce il metodo di homing del drive.
- **DC (Distributed Clocks)** attivi per il TouchProbe.
- Nastro rotativo: se l'encoder va in **modulo** (wrap), la misura
  `falling − rising` va gestita con l'aritmetica modulo (non coperto qui).

### B.5 Assunzioni (adattabili)
Corsa nastro PP = **relativa** dal trigger (`lrBeltMove`); asse2 "avanti" =
**assoluto** `lrAx2Fwd` (+400), "ritorno" = **assoluto** `lrAx2Zero` (0); scarto =
nastro che continua in velocity + certifica su `xExitPhotocell` (`nRejectCount`).
Se il tuo layout differisce, cambia i parametri o segnala e adatto la logica.
