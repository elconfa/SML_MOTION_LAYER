[English](IMPORT_CHECKLIST.md) | **Italiano**

# SML v9 — Checklist di import in CoDeSys (primo Build)

**Scopo:** portare `SML_v9/*.txt` in un progetto CoDeSys con il primo Build il
piu' liscio possibile. I `.txt` sono **export ST testuali** (dichiarazione +
implementazione), da **incollare** in oggetti creati a mano — non sono file di
progetto importabili.

Convenzione: per DUT/GVL/interfacce il file e' tutto "dichiarazione". Per POU
(FB/FUNCTION/PROGRAM) il file contiene l'header + le VAR (dichiarazione) e il
corpo ST (implementazione): separarli nei due riquadri dell'editor.

---

## Due progetti: libreria + applicazione

Puoi importare tutto in un **progetto unico** (l'ordine in §1 funziona così com'è), oppure separare in una
**libreria** riusabile + un'**applicazione** per macchina (consigliato, vedi [`../library/README.it.md`](../library/README.it.md)).

**Progetto libreria** (riusabile, nessuna istanza) — §1a…1h tranne `GVL_App`:
`GVL_SML_CONST`, tutti gli enum (`SML_DiagCode`, `E_PROGRESS`, `E_AXIS_CTRL`, `E_AXIS_STATE`), tutte le struct
(`OpenSML_Axis`, `ST_CiA402_Status`, `ST_AXIS_CTRL`, `ST_MOVE_DATA`, `ST_AXIS_STATE`, `ST_AXIS_INFO`,
`ST_DriveIn`, `ST_DriveOut`), funzioni (`f_GetProgress`, `f_GetState`), `I_Axis`, gli FB foglia
(`SML_Power/Reset/Home/ProfilePosition/ProfileVelocity/ProfileVelocity_Jog/Stop/Status/Diagnostics/TouchProbe`,
`FB_S7RTT_OTG`), `FB_AxisCtrl` (+ metodi) e le UNION (`U_AXIS_CTRL/MOVE_DATA/AXIS_STATE/AXIS_INFO`).

**Progetto applicazione** (creato per macchina; NON nella libreria):
`GVL_App` (`MAX_AXIS`), `GVL_AXIS` (array di istanze + `Control : FB_AxisCtrl` + `ItfSmlAxis`), `MAIN`,
e opzionalmente `GVL_IO` + `PRG_IoLink_In/_Out`, `GVL_AXIS_MAP` + `PRG_Mapping_In/_Out`. Anche gli esempi
(`PLC_APP`, `FB_AxisCycleDemo`) e i banchi di test (`PRG_*_Test`) stanno qui.

> Con la libreria impacchettata sotto namespace `SML`, prefissa le referenze nell'applicazione
> (`SML.FB_AxisCtrl`, `SML.AXIS_MOVE_ABS`, …). L'ordine §1 sotto è per la via a progetto unico.

---

## 0. Pre-flight

- [ ] **Versione CoDeSys** 3.5 (o TwinCAT 3, essendo CoDeSys-based).
- [ ] **FB_S7RTT_OTG** disponibile: e' incluso come `.txt`, ma dipende dal
      generatore di traiettoria (Ruckig/Struckig light). Verificare che compili
      standalone.
- [ ] **Libreria Standard** referenziata (TON/TOF, CONCAT, SIZEOF, MEMSET…).
- [ ] Decidere il modo del primo Build: **simulazione** (nessun HW, usa i banchi
      di test) — consigliato per il primo Build.
- [ ] MAPPING **disattivo** al primo Build: `GVL_AXIS_MAP.AXIS_MAP_ENABLE = FALSE`
      (flag runtime; il MAPPING si auto-bypassa → comportamento come v7).
      NB: niente pragma `{IF defined(...)}` nella dichiarazione (non portabile
      su CoDeSys).
- [ ] Nota: `SML_TC3Link` e' **TwinCAT-specifico** (bridge all'I/O tree TwinCAT).
      In simulazione non serve; su CoDeSys puro l'I/O si mappa diversamente.

---

## 1. Ordine di import (le dipendenze si risolvono dal basso verso l'alto)

Creare gli oggetti in QUEST'ORDINE (Add Object → tipo indicato):

### 1a. Costanti — **per prime** (usate come bound degli array)
- [ ] `GVL_SML_CONST`  → GVL  (libreria: `PROGRESS_SPAN`, `MAP_SIZE_*`)
- [ ] `GVL_App`        → GVL  (applicazione: `MAX_AXIS` — bound degli array)

### 1b. Enum (DUT → Enumeration)
- [ ] `SML_DiagCode` (se non gia' presente dalla base)
- [ ] `E_PROGRESS`
- [ ] `E_AXIS_CTRL`
- [ ] `E_AXIS_STATE`

### 1c. Struct (DUT → Structure)
- [ ] `OpenSML_Axis`, `OpenSML_Control` (base, se non presenti)
- [ ] `ST_CiA402_Status`  (usato da ST_AXIS_INFO)
- [ ] `ST_AXIS_CTRL`
- [ ] `ST_MOVE_DATA`
- [ ] `ST_AXIS_STATE`  (usa `SML_DiagCode`, `E_PROGRESS`, `E_AXIS_CTRL`)
- [ ] `ST_AXIS_INFO`   (usa `ST_CiA402_Status`)

### 1d. Funzioni (POU → Function)
- [ ] `f_GetProgress`  (usa `E_PROGRESS`, `GVL_SML_CONST`)
- [ ] `f_GetState`

### 1e. Interfaccia (POU → Interface) — vedi §2
- [ ] `I_Axis` + i suoi METHOD/PROPERTY figli

### 1f. FB foglia CiA402 (livello esecuzione) — base della libreria
Set **minimo** richiesto da `FB_AxisCtrl` (v9 include Status + TouchProbe):
- [ ] `SML_Power`, `SML_Reset`, `SML_Home`, `SML_ProfilePosition`,
      `SML_ProfileVelocity`, `SML_ProfileVelocity_Jog`, `SML_Stop`,
      `SML_Diagnostics`, `SML_Status`, `SML_TouchProbe`, `FB_S7RTT_OTG`
Oggetti **legacy** (in `legacy/`, non istanziati — importare solo se ti servono
esplicitamente, es. il bridge TwinCAT `SML_TC3Link`):
- [ ] `SML_SyncPosition`, `SML_SyncVelocity`, `FB_SML`, `SML_AxisController`,
      `SML_TC3Link`

### 1g. FB di controllo (POU → Function Block) — vedi §2
- [ ] `FB_AxisCtrl` (`IMPLEMENTS I_Axis`) + i suoi METHOD/PROPERTY

### 1h. UNION (DUT → Union)
- [ ] `U_AXIS_CTRL`, `U_MOVE_DATA`, `U_AXIS_STATE`, `U_AXIS_INFO`
      (usano `MAP_SIZE_*` di `GVL_SML_CONST`)

### 1i. Bridge I/O hardware (opzionale — solo se usi il bridge, vedi GUIDA_IO_Linking)
- [ ] `ST_DriveOut`, `ST_DriveIn`  (DUT → Structure)
- [ ] `GVL_IO`  (GVL; `DriveOut`/`DriveIn` array semplici + `IO_LINK_ENABLE`; niente `AT` → evita C0128, mappa nel configuratore)
- [ ] `PRG_IoLink_In`, `PRG_IoLink_Out`  (POU → Program)

### 1j. GVL e orchestrazione
- [ ] `GVL_AXIS`      (usa FB_AxisCtrl, I_Axis, ST_*, OpenSML_Axis, MAX_AXIS)
- [ ] `GVL_AXIS_MAP`  (usa le UNION)
- [ ] `PRG_Mapping_In`, `PRG_Mapping_Out`  (POU → Program)
- [ ] `MAIN`         (POU → Program; chiama IoLink + MAPPING + ciclo assi)

### 1j. Banchi di prova (POU → Program) — sviluppo
- [ ] `PRG_LevelA_Test`, `PRG_LevelB_Test`, `PRG_MultiAxis_Test`

---

## 2. I metodi di I_Axis (punto critico)

`FB_AxisCtrl` dichiara `IMPLEMENTS I_Axis`: **non compila** finche' tutti i
metodi/proprieta' dell'interfaccia non esistono come oggetti figli del FB.

1. [ ] Creare l'INTERFACE `I_Axis`. Per ogni voce in `I_Axis.txt` aggiungere
       un **Method** figlio (Enable/Disable/Reset/Home/MoveAbsolute/MoveRelative/
       MoveVelocity/Jog/MoveFollow/Stop) e una **Property** figlia con **Get**
       (Position, Enabled). Incollare le firme (tipo di ritorno + VAR_INPUT).
2. [ ] Su `FB_AxisCtrl`: tasto destro → **Implement interfaces…** (genera gli
       stub dei metodi/proprieta') **oppure** aggiungerli a mano come Method/
       Property figli.
3. [ ] Incollare i **corpi reali** da `FB_AxisCtrl_METHODS.txt` in ciascun
       metodo/Get corrispondente.

---

## 3. Impostazioni di progetto

- [ ] **Enum non-strict**: gli enum sono senza `{attribute 'strict'}`. Il codice
      usa `TO_INT(enum)` (stato combinato) e assegna INT→enum in `f_GetProgress`:
      con enum non-strict compila. Se il progetto forza enum strict (attributo o
      opzione), adattare `f_GetProgress` (mappatura esplicita) e i `TO_INT`.
- [ ] **Flag MAPPING** `GVL_AXIS_MAP.AXIS_MAP_ENABLE`: **FALSE** al primo Build.
      Per attivare il mapping poi: metterlo TRUE (design-time o online). Nessuna
      pragma di compilazione condizionale (`{IF defined(...)}` non e' supportata
      nella parte dichiarazione su CoDeSys). Per esclusione a compile-time,
      opzionale: racchiudere il CORPO di PRG_Mapping_In/Out tra
      `{IF defined (AXIS_MAP)} ... {END_IF}` nell'IMPLEMENTAZIONE.
- [ ] Librerie: nessuna dipendenza da `memcpy` (copia via assegnazione). Servono
      solo Standard + eventuale lib dell'OTG.

---

## 4. Configurazione task

- [ ] Aggiungere **UN SOLO** program-chiamante al task ciclico:
      - produzione/HW: `MAIN` (+ bridge I/O per `GVL_AXIS.Axis[]`);
      - simulazione: **uno** dei banchi (`PRG_MultiAxis_Test` o `PRG_LevelB_Test`).
      **NON** mettere MAIN e un banco insieme (doppia chiamata degli stessi FB).
- [ ] Impostare il tempo di ciclo del task e allineare `Ctrl.CycleTime`
      (o `GVL_AXIS.Ctrl[n].CycleTime`) allo **stesso** valore (correttezza OTG).

---

## 5. Errori tipici al primo Build e rimedi

| Sintomo | Causa | Rimedio |
|---|---|---|
| `FB_AxisCtrl` non implementa I_Axis | metodi/proprieta' mancanti | creare i Method/Property (§2) |
| `FB_S7RTT_OTG` non trovato | OTG non importato/licenza | importare il FB/libreria OTG |
| conversione INT→E_PROGRESS non consentita | enum in modalita' strict | vedi §3 (adattare) |
| `GVL_App.MAX_AXIS` non valido come bound | costante non risolta | importare `GVL_App` (e `GVL_SML_CONST`) per primi |
| pragma `{IF defined}` non supportata in dichiarazione | conditional-compile in VAR | usare il flag `AXIS_MAP_ENABLE` (gia' applicato) o spostare la pragma nel corpo |
| `CONCAT`/`TON` non definiti | Standard non referenziata | aggiungere libreria Standard |

---

## 6. Verifica (dopo Build pulito)

1. [ ] **Build**: zero errori nuovi; `E_*`, `ST_*`, `U_*`, `I_Axis`, `GVL_*`,
       `FB_AxisCtrl`, `MAIN`, `PRG_Mapping_*` risolti.
2. [ ] **Livello A** — task con `PRG_LevelA_Test`: `xTestPassed → TRUE`
       (`eProgress` INVALID→BUSY→DONE→ERROR→INVALID).
3. [ ] **Livello B mono-asse** — `PRG_LevelB_Test`: `xTestPassed → TRUE`;
       osservare `State.eState` (es. 306 = IDLE+DONE, 502 = MOVING+BUSY) e
       `eStProg = eProgress`.
4. [ ] **Multi-asse** — `PRG_MultiAxis_Test`: `xTestPassed → TRUE`, `xIndependent`
       TRUE (asse 1 CSP e asse 2 JOG in parallelo).
5. [ ] **MAPPING** — mettere `GVL_AXIS_MAP.AXIS_MAP_ENABLE := TRUE`, collegare/
       forzare `GVL_AXIS_MAP.Ctrl[n].stData.eCmd` e verificare la propagazione a
       `GVL_AXIS.Ctrl[n]` e il ritorno in `GVL_AXIS_MAP.State[n].stData`.
6. [ ] **Check dimensioni**: abbassare temporaneamente un `MAP_SIZE_*` sotto la
       SIZEOF della struct → il mapping deve bloccarsi (nessuna copia).
