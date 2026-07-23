# SML v8 — Checklist di import in CoDeSys (primo Build)

**Scopo:** portare `SML_v8/*.txt` in un progetto CoDeSys con il primo Build il
piu' liscio possibile. I `.txt` sono **export ST testuali** (dichiarazione +
implementazione), da **incollare** in oggetti creati a mano — non sono file di
progetto importabili.

Convenzione: per DUT/GVL/interfacce il file e' tutto "dichiarazione". Per POU
(FB/FUNCTION/PROGRAM) il file contiene l'header + le VAR (dichiarazione) e il
corpo ST (implementazione): separarli nei due riquadri dell'editor.

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
- [ ] `GVL_SML_CONST`  → GVL

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
- [ ] `I_SmlAxis` + i suoi METHOD/PROPERTY figli

### 1f. FB foglia CiA402 (livello esecuzione) — base della libreria
Set **minimo** richiesto da `FB_SmlAxisCtrl` (v9 include Status + TouchProbe):
- [ ] `SML_Power`, `SML_Reset`, `SML_Home`, `SML_ProfilePosition`,
      `SML_ProfileVelocity`, `SML_ProfileVelocity_Jog`, `SML_Stop`,
      `SML_Diagnostics`, `SML_Status`, `SML_TouchProbe`, `FB_S7RTT_OTG`
Resto della libreria (non richiesto dalla catena Livello B, importare se serve):
- [ ] `SML_SyncPosition`, `SML_SyncVelocity`, `FB_SML`, `SML_AxisController`,
      `SML_TC3Link`

### 1g. FB di controllo (POU → Function Block) — vedi §2
- [ ] `FB_SmlAxisCtrl` (`IMPLEMENTS I_SmlAxis`) + i suoi METHOD/PROPERTY

### 1h. UNION (DUT → Union)
- [ ] `U_AXIS_CTRL`, `U_MOVE_DATA`, `U_AXIS_STATE`, `U_AXIS_INFO`
      (usano `MAP_SIZE_*` di `GVL_SML_CONST`)

### 1i. Bridge I/O hardware (opzionale — solo se usi il bridge, vedi GUIDA_IO_Linking)
- [ ] `ST_DriveOut`, `ST_DriveIn`  (DUT → Structure)
- [ ] `GVL_IO`  (GVL; contiene `DriveOut AT %Q*` / `DriveIn AT %I*` + `IO_LINK_ENABLE`)
- [ ] `SML_IoLink_In`, `SML_IoLink_Out`  (POU → Program)

### 1j. GVL e orchestrazione
- [ ] `GVL_AXIS`      (usa FB_SmlAxisCtrl, I_SmlAxis, ST_*, OpenSML_Axis, MAX_AXIS)
- [ ] `GVL_AXIS_MAP`  (usa le UNION)
- [ ] `MAPPING_in`, `MAPPING_out`  (POU → Program)
- [ ] `MAIN`         (POU → Program; chiama IoLink + MAPPING + ciclo assi)

### 1j. Banchi di prova (POU → Program) — sviluppo
- [ ] `SML_LevelA_Test`, `SML_LevelB_Test`, `SML_MultiAxis_Test`

---

## 2. I metodi di I_SmlAxis (punto critico)

`FB_SmlAxisCtrl` dichiara `IMPLEMENTS I_SmlAxis`: **non compila** finche' tutti i
metodi/proprieta' dell'interfaccia non esistono come oggetti figli del FB.

1. [ ] Creare l'INTERFACE `I_SmlAxis`. Per ogni voce in `I_SmlAxis.txt` aggiungere
       un **Method** figlio (Enable/Disable/Reset/Home/MoveAbsolute/MoveRelative/
       MoveVelocity/Jog/MoveFollow/Stop) e una **Property** figlia con **Get**
       (Position, Enabled). Incollare le firme (tipo di ritorno + VAR_INPUT).
2. [ ] Su `FB_SmlAxisCtrl`: tasto destro → **Implement interfaces…** (genera gli
       stub dei metodi/proprieta') **oppure** aggiungerli a mano come Method/
       Property figli.
3. [ ] Incollare i **corpi reali** da `FB_SmlAxisCtrl_METHODS.txt` in ciascun
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
      opzionale: racchiudere il CORPO di MAPPING_in/out tra
      `{IF defined (AXIS_MAP)} ... {END_IF}` nell'IMPLEMENTAZIONE.
- [ ] Librerie: nessuna dipendenza da `memcpy` (copia via assegnazione). Servono
      solo Standard + eventuale lib dell'OTG.

---

## 4. Configurazione task

- [ ] Aggiungere **UN SOLO** program-chiamante al task ciclico:
      - produzione/HW: `MAIN` (+ bridge I/O per `GVL_AXIS.Axis[]`);
      - simulazione: **uno** dei banchi (`SML_MultiAxis_Test` o `SML_LevelB_Test`).
      **NON** mettere MAIN e un banco insieme (doppia chiamata degli stessi FB).
- [ ] Impostare il tempo di ciclo del task e allineare `Ctrl.CycleTime`
      (o `GVL_AXIS.Ctrl[n].CycleTime`) allo **stesso** valore (correttezza OTG).

---

## 5. Errori tipici al primo Build e rimedi

| Sintomo | Causa | Rimedio |
|---|---|---|
| `FB_SmlAxisCtrl` non implementa I_SmlAxis | metodi/proprieta' mancanti | creare i Method/Property (§2) |
| `FB_S7RTT_OTG` non trovato | OTG non importato/licenza | importare il FB/libreria OTG |
| conversione INT→E_PROGRESS non consentita | enum in modalita' strict | vedi §3 (adattare) |
| `GVL_SML_CONST.MAX_AXIS` non valido come bound | costante non risolta | importare `GVL_SML_CONST` per primo |
| pragma `{IF defined}` non supportata in dichiarazione | conditional-compile in VAR | usare il flag `AXIS_MAP_ENABLE` (gia' applicato) o spostare la pragma nel corpo |
| `CONCAT`/`TON` non definiti | Standard non referenziata | aggiungere libreria Standard |

---

## 6. Verifica (dopo Build pulito)

1. [ ] **Build**: zero errori nuovi; `E_*`, `ST_*`, `U_*`, `I_SmlAxis`, `GVL_*`,
       `FB_SmlAxisCtrl`, `MAIN`, `MAPPING_*` risolti.
2. [ ] **Livello A** — task con `SML_LevelA_Test`: `xTestPassed → TRUE`
       (`eProgress` INVALID→BUSY→DONE→ERROR→INVALID).
3. [ ] **Livello B mono-asse** — `SML_LevelB_Test`: `xTestPassed → TRUE`;
       osservare `State.eState` (es. 306 = IDLE+DONE, 502 = MOVING+BUSY) e
       `eStProg = eProgress`.
4. [ ] **Multi-asse** — `SML_MultiAxis_Test`: `xTestPassed → TRUE`, `xIndependent`
       TRUE (asse 1 CSP e asse 2 JOG in parallelo).
5. [ ] **MAPPING** — mettere `GVL_AXIS_MAP.AXIS_MAP_ENABLE := TRUE`, collegare/
       forzare `GVL_AXIS_MAP.Ctrl[n].stData.eCmd` e verificare la propagazione a
       `GVL_AXIS.Ctrl[n]` e il ritorno in `GVL_AXIS_MAP.State[n].stData`.
6. [ ] **Check dimensioni**: abbassare temporaneamente un `MAP_SIZE_*` sotto la
       SIZEOF della struct → il mapping deve bloccarsi (nessuna copia).
