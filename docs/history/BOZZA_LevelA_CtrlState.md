# BOZZA — Livello A: E_PROGRESS + f_GetState/f_GetProgress + split Ctrl/State

**Stato:** proposta di design, **non ancora importata/compilata** in CoDeSys.
**Riferimento:** `SML/NOTA_Pattern_MotionLayer.md`, sezione "Livello A".
**Ambito:** solo scheletro. Non modifica gli FB di esecuzione (`SML_Power`, ecc.).

Tre pezzi:
1. `E_PROGRESS` — enum di avanzamento unificato (additivo, zero rischio)
2. `f_GetProgress` / `f_GetState` — decomposizione dello stato combinato (additivo)
3. `SML_AxisCtrl` / `SML_AxisState` — split del contratto dati (una via è breaking)

---

## 1. `E_PROGRESS` (DUT / ENUM)

```pascal
(* ============================================================
   E_PROGRESS — Avanzamento unificato per tutti gli FB SML
   Livello A — nuovo

   Sostituisce la semantica ad-hoc di iState:INT nei singoli FB con
   un vocabolario comune, leggibile da app/HMI/diagnostica.

   STATO COMBINATO:
     Uno stato macchina si codifica come  eStato_funzionale + E_PROGRESS
     con lo stato funzionale multiplo di PROGRESS_SPAN (=100).
     Es.: 300 (MOVE) + 6 (DONE) = 306  → "MOVE completato".
     f_GetState(306) = 300 ; f_GetProgress(306) = PROGRESS_DONE.
   ============================================================ *)
TYPE E_PROGRESS :
(
    PROGRESS_INVALID := 0,   // stato non inizializzato / non valido
    PROGRESS_INIT    := 1,   // richiesta di avvio ricevuta
    PROGRESS_BUSY    := 2,   // in elaborazione
    PROGRESS_PREPARE := 3,   // preparazione parametri/precondizioni
    PROGRESS_STARTUP := 4,   // esecuzione fase di startup
    PROGRESS_CHECK   := 5,   // verifica esito / avanzamento indice
    PROGRESS_DONE    := 6,   // completato con successo
    PROGRESS_ERROR   := 7    // errore (terminale finché non resettato)
) INT;
END_TYPE
```

> Costante di appoggio (aggiungere a un GVL, es. `GVL_SML_CONST`, oppure
> ridefinire inline nelle funzioni):
> ```pascal
> VAR_GLOBAL CONSTANT
>     PROGRESS_SPAN : DINT := 100;   // ampiezza banda progress nello stato combinato
> END_VAR
> ```

---

## 2a. `f_GetProgress` (FUNCTION)

```pascal
(* Estrae la parte di avanzamento (E_PROGRESS) da uno stato combinato. *)
FUNCTION f_GetProgress : E_PROGRESS
VAR_INPUT
    eCombined : DINT;   // stato combinato = stato_funzionale + progress
END_VAR
VAR CONSTANT
    PROGRESS_SPAN : DINT := 100;   // usare GVL condiviso se disponibile
END_VAR

f_GetProgress := TO_INT(eCombined MOD PROGRESS_SPAN);
```

## 2b. `f_GetState` (FUNCTION)

```pascal
(* Estrae la base di stato funzionale (multiplo di PROGRESS_SPAN)
   da uno stato combinato. In Livello B il ritorno diventa un enum
   di stato dedicato (es. E_SML_AXIS_STATE). *)
FUNCTION f_GetState : DINT
VAR_INPUT
    eCombined : DINT;
END_VAR
VAR CONSTANT
    PROGRESS_SPAN : DINT := 100;
END_VAR

f_GetState := (eCombined / PROGRESS_SPAN) * PROGRESS_SPAN;
```

> **Uso in un FB esistente (non invasivo):** mantenere `iState:INT` interno e,
> in coda al ciclo, esporre l'avanzamento standard:
> ```pascal
> // esempio dentro SML_ProfilePosition (pseudo)
> CASE iState OF
>     0:   eProgress := PROGRESS_INVALID;
>     10:  eProgress := PROGRESS_STARTUP;
>     20:  eProgress := PROGRESS_BUSY;
>     30:  eProgress := PROGRESS_DONE;
>     999: eProgress := PROGRESS_ERROR;
> END_CASE
> ```
> Nessuna riscrittura della macchina a stati: solo una proiezione.

---

## 3. Split del contratto dati Ctrl/State

Deriva 1:1 dai due blocchi già commentati in `OpenSML_Control`
("Command inputs" → Ctrl, "Status outputs" → State). I limiti/scaling sono
input applicativi: nel pattern completo andrebbero in una struct `Data`;
in Livello A li tengo dentro `Ctrl` per minimizzare i tipi.

### 3a. `SML_AxisCtrl` (app → layer)

```pascal
TYPE SML_AxisCtrl :
STRUCT
    // ── comando ──────────────────────────────────────────────
    xAxisEnable      : BOOL;            // abilita hardware drive
    xEmergencyStop   : BOOL := TRUE;    // TRUE=run / FALSE=quick-stop (ControlWord bit 2)
    xMoveEnable      : BOOL;            // consenti moto (jog/follow)
    xHomeEnable      : BOOL;            // avvia homing
    xSimulation      : BOOL;            // specchia Target → Actual
    JogPos           : BOOL;            // jog +
    JogNeg           : BOOL;            // jog -
    xFollowEnable    : BOOL;            // usa lrFollowPosition invece del jog
    lrFollowPosition : LREAL;           // target follow [unità]; ri-targettabile online
    xDiagReset       : BOOL;            // pulse: ack del first-fault latchato

    // ── parametri / limiti (Data) ────────────────────────────
    JogVelocity      : LREAL := 10.0;   // [unità/s]
    lrFollowVelocity : LREAL := 20.0;   // [unità/s]
    MaxVelocity      : LREAL := 100.0;  // [unità/s]
    MaxAcceleration  : LREAL := 1000.0; // [unità/s²]
    MaxJerk          : LREAL := 10000.0;// [unità/s³]
    Scale            : LREAL := 4096.0; // [counts/unità] — > 0
    CycleTime        : LREAL := 0.001;  // [s] — deve combaciare col task
    PositionWindow   : LREAL := 0.01;   // deadband in-posizione [unità]
    HomeOffset       : DINT  := 0;      // [counts] → 0x607C
END_STRUCT
END_TYPE
```

### 3b. `SML_AxisState` (layer → app)

```pascal
TYPE SML_AxisState :
STRUCT
    // ── avanzamento unificato (NUOVO) ────────────────────────
    eProgress        : E_PROGRESS;      // avanzamento standard
    eState           : DINT;            // stato combinato (funzionale + progress)

    // ── stato ────────────────────────────────────────────────
    lrActualPosition : LREAL;           // posizione attuale [unità]
    xEnabled         : BOOL;            // OperationEnabled
    xHomeDone        : BOOL;            // homing completato
    xInPosition      : BOOL;            // entro PositionWindow
    xBusy            : BOOL;            // macchina attiva
    xError           : BOOL;            // errore latchato

    // ── diagnostica ──────────────────────────────────────────
    DiagCode         : SML_DiagCode;
    DiagText         : STRING(80);
    xActiveFault     : BOOL;
    xActiveWarning   : BOOL;
    FirstFaultCode   : SML_DiagCode;
    FirstFaultText   : STRING(80);
    DriveErrorCode   : UINT;            // 0x603F
    FollowingError   : DINT;            // 0x60F4
    FaultCount       : UINT;
END_STRUCT
END_TYPE
```

---

## 4. Due vie di adozione dello split

**Via 1 — NON-breaking (rischio nullo).**
Lasciare `OpenSML_Control` intatto. Usare `SML_AxisCtrl`/`SML_AxisState` solo
in un futuro strato di orchestrazione (Livello B). Adotti subito solo enum +
funzioni (che sono già additivi). Nessuna riga degli FB cambia.

**Via 2 — split reale su OpenSML_Control (una-tantum, meccanico).**
Ridefinire:
```pascal
TYPE OpenSML_Control :
STRUCT
    Ctrl  : SML_AxisCtrl;    // input applicativi
    State : SML_AxisState;   // output del layer
END_STRUCT
END_TYPE
```
Conseguenza: i percorsi campo negli FB cambiano
(`Control.xAxisEnable` → `Control.Ctrl.xAxisEnable`,
`Control.lrActualPosition` → `Control.State.lrActualPosition`).
È una rinomina meccanica, ma tocca `SML_AxisController` e ogni FB che riceve
`Control`. Da fare in un unico commit, con `E_PROGRESS` popolato in coda ciclo.

**Raccomandazione:** partire dalla Via 1 (enum + funzioni, valore immediato,
zero rischio); passare alla Via 2 solo insieme all'introduzione del layer
Livello B, così la rinomina si paga una volta sola.
