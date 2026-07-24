# Nota riutilizzabile — Pattern "Motion Layer" (Ctrl/State)

**Fonte:** `github.com/haud-ba/PLC_MOTION_LAYER` (TwinCAT 3, autore HAUD)
**Astratta il:** 2026-07-22
**Scopo:** distillare il pattern architetturale — indipendente dal codice TwinCAT —
per un livello di astrazione motion riusabile in progetti IEC 61131-3 / CoDeSys.

> Il **codice** di PLC_MOTION_LAYER non è portabile (dipende da NC Beckhoff +
> `Tc2_MC2`). Ciò che si porta è lo **scheletro architetturale** descritto qui.

---

## 1. Idea centrale

Separare nettamente **tre livelli**:

```
   Applicazione / HMI / bus di campo
            │  (solo strutture dati piatte)
            ▼
   ┌─────────────────────────────┐
   │ LIVELLO COMANDO  (Ctrl FB)  │  macchina a stati, traduce eCmd → chiamate
   │  - accetta Ctrl, scrive State│
   └─────────────────────────────┘
            │  (interfaccia I_*)
            ▼
   ┌─────────────────────────────┐
   │ LIVELLO ESECUZIONE (Base FB) │  incapsula i FB di motion "grezzi"
   │  ABSTRACT, IMPLEMENTS I_*     │
   └─────────────────────────────┘
            │
            ▼
   Motion primitivo (in TwinCAT: MC_*; in SML: drive CiA402 diretto)
```

L'applicazione **non chiama mai** i function block di motion: scrive solo una
**struttura di comando** e legge una **struttura di stato**.

---

## 2. I sei elementi del pattern

### 2.1 Contratto dati Ctrl/State/Info/Data
Per ogni asse (o funzione) quattro strutture piatte, senza logica:

| Struct | Verso | Contenuto |
|--------|-------|-----------|
| `Ctrl`  | app → layer | comando (`eCmd`), enable, richieste |
| `State` | layer → app | esito del comando, stato macchina |
| `Data`  | app → layer | setpoint e vincoli dinamici (pos, vel, acc, jerk) |
| `Info`  | layer → app | valori act/set, bit di stato dell'asse |

Regola d'oro: **strutture serializzabili** (nessun puntatore/interfaccia dentro),
così sono accessibili via simbolo ADS **o** copiabili su qualunque bus.

### 2.2 Base astratta + FB di controllo (ereditarietà)
- `FB_..Base` = `FUNCTION_BLOCK ABSTRACT IMPLEMENTS I_..` — incapsula i FB di
  motion primitivi (in TwinCAT: `MC_Power`, `MC_MoveAbsolute`…), startup, errori.
- `FB_..Ctrl EXTENDS FB_..Base` — aggiunge la **macchina a stati a comandi**
  guidata da un enum (`E_AXIS_CTRL`: `AXIS_INIT`, `AXIS_ENABLE`, `AXIS_MOVE…`).

### 2.3 Interfaccia per accesso trasversale
`I_McAxis` (metodi `Enable/Disable/Halt/Stop/Reset/Read-WriteParameter`,
proprietà `Position/AxisRef`). Serve a sottosistemi di alto livello (CNC, camming)
per manovrare gli assi **senza dipendere** dall'implementazione.

### 2.4 Macchina a stati universale `E_PROGRESS`
Un unico enum di avanzamento condiviso da tutti i sottosistemi:
```
INVALID → INIT → BUSY → PREPARE → STARTUP → CHECK → DONE
                                                   ↘ ERROR
```
Trucco chiave: lo **stato combinato** = `stato_funzionale + E_PROGRESS`
(es. `AXIS_INIT + PROGRESS_DONE`). Due utility lo decompongono:
`f_GetState()` e `f_GetProgress()`. Rende il codice di orchestrazione
identico per assi, CNC, camming, ecc.

### 2.5 Orchestrazione multi-asse in un unico PRG
`MAIN` per `nAxis := 1 TO MAX_AXIS`:
1. **cabla** i puntatori/riferimenti delle struct all'FB `Control[nAxis]`
   (`ADR(...)` / `REF=`);
2. esegue una **sequenza di INIT** asse-per-asse gated da `E_PROGRESS`;
3. chiama ciclicamente `Control[nAxis]()`.
`MAX_AXIS` è una costante globale → scala senza toccare la logica.

### 2.6 Layer di MAPPING opzionale (bus-agnostico)
- `MAPPING_in` / `MAPPING_out`: `memcpy` GVL ↔ strutture bus, con **UNION**
  (`U_AXIS_CTRL`…) per reinterpretare gli stessi byte come struct o blocco grezzo.
- Attivazione via **compilazione condizionale** (`{IF defined (AXIS_MAP)}`):
  se il define manca, il mapping è bypassato → paghi solo ciò che usi.
- Verifica della **dimensione dati** a startup (`Check()`).

### 2.7 (bonus) Sottosistema messaggi/log
`E_MessageType` (Verbose/Info/Error), coda con timestamp, scrittura su file,
livelli di log per-asse. Osservabilità senza sporcare la logica di motion.

---

## 3. Vantaggi / costi

**Pro:** disaccoppiamento app↔motion; multi-asse scalabile; riuso via
ereditarietà+interfacce; bus-agnostico; osservabilità integrata.

**Contro:** uso intensivo di `POINTER`/`memcpy` (fragile rispetto al layout di
memoria — `Check()` sulla dimensione mitiga solo in parte); forte accoppiamento
al posizionamento delle GVL; curva d'ingresso ripida.

---

## 4. Applicazione a SML (CoDeSys / CiA402) — mappatura

SML **ha già** il livello di esecuzione: i suoi FB (`SML_Power`, `SML_Home`,
`SML_ProfilePosition`, `SML_ProfileVelocity(_Jog)`, `SML_SyncPosition/Velocity`,
`SML_Stop`, `SML_TouchProbe`) sono l'equivalente dei wrapper `MC_*`.
`SML_TC3Link` è già un embrione del MAPPING (struct ↔ immagine PDO).

| Elemento pattern | Stato in SML | Azione per adottarlo |
|------------------|--------------|----------------------|
| Livello esecuzione | ✅ presente (`SML_*` FB) | nessuna |
| Contratto Ctrl/State/Info/Data | ⚠️ fuso in `OpenSML_Control` | separare comando/stato |
| Base astratta + interfaccia `I_Axis` | ❌ assente | opzionale (livello completo) |
| FB comando con `eCmd` | ⚠️ `FB_SML` usa xExecute+arbiter | introdurre `FB_AxisCtrl` con enum |
| `E_PROGRESS` unificato | ❌ ogni FB usa `iState:INT` | uniformare + `f_GetState/f_GetProgress` |
| Orchestrazione `ARRAY[1..MAX_AXIS]` | ❌ single-axis per istanza | PRG `MAIN` con array |
| MAPPING/UNION/define | ⚠️ `SML_TC3Link` parziale | generalizzare (opzionale) |
| Log/diagnostica | ✅ `SML_Diagnostics`+`DiagCode` | già allineato |

**Conclusione:** l'incompatibilità (NC/`Tc2_MC2` vs CiA402 diretto) sta **tutta
sotto la linea di astrazione**, quindi non blocca il pattern. L'adozione è
**additiva**, non una riscrittura.

### Due livelli di adozione
- **Livello A — leggero (basso rischio):** enum `E_PROGRESS` uniforme +
  `f_GetState/f_GetProgress` + split esplicito Ctrl/State su `OpenSML_Control`.
  Nessuna modifica all'esecuzione.
- **Livello B — completo (alto valore multi-asse):** `FB_AxisCtrl`
  (comando `eCmd` + Ctrl/State/Info/Data), interfaccia `I_Axis`,
  orchestrazione `ARRAY[1..MAX_AXIS]` in `MAIN`, MAPPING opzionale.
