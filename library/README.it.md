[English](README.md) | **Italiano**

# SML — Libreria (core riusabile)

Il componente **libreria** del Motion Layer: solo oggetti riusabili — **nessuna istanza, nessuna
configurazione di macchina, nessun PROGRAM**. Impacchettalo come *CoDeSys Library* / *TwinCAT Library*,
poi referenzialo da ogni applicazione PLC.

La libreria è **autosufficiente**: ogni oggetto qui dipende solo da altri oggetti di questa cartella più
la libreria IEC **Standard** (`TON`, e gli operatori `ABS/MIN/MAX/SQRT/SIZEOF/…`). Verificato: nessun
oggetto referenzia in codice `MAX_AXIS`, i GVL di istanza, o file legacy/applicativi.

---

## Manifest (imposta questi campi in *Project Information* creando la libreria)

| Campo | Valore |
|---|---|
| Title | `SML` |
| Subtitle | SoftMotion Light — Motion Layer CiA402 |
| Namespace | `SML` |
| Version | `0.9.0` (pre-1.0: giovane, testata in simulazione) |
| Company / Author | *(imposta il tuo)* |
| Licenza | GPL-3.0 (vedi [`../LICENSE`](../LICENSE)) |
| Placeholder | lascia il default (o `SML, * (SML)`) |

## Dipendenze

- **Standard** (CoDeSys) / **Tc2_Standard** (TwinCAT) — per l'FB IEC standard `TON` (`TOF`/`F_TRIG` sono
  usati solo nei file *applicativi*, non qui). Di norma già referenziata in un progetto standard.
- Tutto il resto è **built-in** del compilatore: conversioni di tipo (`LREAL_TO_DINT`, …), operatori
  matematici (`ABS/MIN/MAX/SQRT`), `SIZEOF`, `STRING`. Niente puntatori, niente `MEMCPY`/`MEMSET`, niente
  libreria di funzioni stringa.
- **Nessuna libreria di motion**: niente SoftMotion / `SM3_` / `SMC_`, niente `Tc2_MC2` / NC, niente
  `MC_*`, niente `AXIS_REF`. È la ragione tecnica per cui **non serve alcuna licenza asse/motion**.
- `FB_S7RTT_OTG` (generatore di traiettoria jerk-limited) è **incluso qui come sorgente** — non è una
  dipendenza esterna; usa solo matematica built-in.

---

## Oggetti esportati (`src/`) — 33 oggetti, in ordine di build

Crea gli oggetti **dal basso verso l'alto** (ogni riga dipende solo dalle precedenti):

1. **Costanti** — GVL
   - `GVL_SML_CONST` (`PROGRESS_SPAN`, `MAP_SIZE_*`)
2. **Enum** — DUT → Enumeration
   - `SML_DiagCode`, `E_PROGRESS`, `E_AXIS_CTRL`, `E_AXIS_STATE`
3. **Struct** — DUT → Structure
   - `OpenSML_Axis` (immagine PDO CiA402), `ST_CiA402_Status`,
     `ST_AXIS_CTRL`, `ST_MOVE_DATA`, `ST_AXIS_STATE`, `ST_AXIS_INFO`,
     `ST_DriveIn`, `ST_DriveOut` (metà del bridge I/O)
4. **Funzioni** — POU → Function
   - `f_GetProgress`, `f_GetState`
5. **Interfaccia** — POU → Interface (+ i suoi METHOD/PROPERTY figli, vedi sotto)
   - `I_Axis`
6. **FB foglia CiA402** — POU → Function Block
   - `SML_Power`, `SML_Reset`, `SML_Home`, `SML_ProfilePosition`, `SML_ProfileVelocity`,
     `SML_ProfileVelocity_Jog`, `SML_Stop`, `SML_Status`, `SML_Diagnostics`, `SML_TouchProbe`,
     `FB_S7RTT_OTG`
7. **FB di controllo** — POU → Function Block (`IMPLEMENTS I_Axis`)
   - `FB_AxisCtrl` (+ i corpi di metodi/proprietà da `FB_AxisCtrl_METHODS.txt`)
8. **UNION** (per il layer MAPPING opzionale) — DUT → Union
   - `U_AXIS_CTRL`, `U_MOVE_DATA`, `U_AXIS_STATE`, `U_AXIS_INFO`

> I `.txt` sono **export ST testuali**: crea ogni oggetto a mano (Add Object → il tipo indicato) e incolla
> dichiarazione/implementazione. Per i POU, separa header+VAR (dichiarazione) e corpo ST (implementazione)
> nei due riquadri dell'editor.

---

## I metodi di `I_Axis` (l'unico passo delicato)

`FB_AxisCtrl` dichiara `IMPLEMENTS I_Axis`, quindi non compila finché i metodi/proprietà dell'interfaccia
non esistono come **oggetti figli**:

1. Crea l'INTERFACE `I_Axis`; per ogni voce in `I_Axis.txt` aggiungi un **Method** figlio
   (Enable/Disable/Reset/Home/MoveAbsolute/MoveRelative/MoveVelocity/Jog/MoveFollow/Stop) e una
   **Property** figlia con **Get** (Position, Enabled).
2. Su `FB_AxisCtrl`: tasto destro → **Implement interfaces…** per generare gli stub (o aggiungili a mano).
3. Incolla i **corpi reali** da `FB_AxisCtrl_METHODS.txt` in ciascun metodo/Get.

---

## Creare la libreria in CoDeSys

1. *File → New Project → Empty project* (o *Library*). Imposta *Project → Project Information*:
   Title/Namespace/Version/Company come nel manifest.
2. Aggiungi gli oggetti di `src/` nell'ordine di build sopra (crea i figli di `I_Axis` sotto `FB_AxisCtrl`).
3. Verifica che la libreria **Standard** sia referenziata (Library Manager) — di solito già presente.
4. *Build* → zero errori.
5. *File → Save Project as Library* (o *Save as Compiled Library* per una `.compiled-library`
   distribuibile), poi *Install* nel Library Repository.

## Creare la libreria in TwinCAT

1. *PLC → Library Project* (o un progetto PLC standard da salvare come libreria). Imposta i metadati
   (Title/Version/Company) nelle proprietà del progetto.
2. Aggiungi gli oggetti di `src/` nello stesso ordine di build; referenzia **Tc2_Standard**.
3. *Build* → *Save as library* → compare nel *Library Repository*.

---

## Referenziarla da un'applicazione

1. Nel progetto applicativo: *Library Manager → Add library → SML*.
2. Poiché la libreria usa il namespace `SML`, prefissa le referenze dove il compilatore lo richiede:
   `SML.FB_AxisCtrl`, `SML.I_Axis`, `SML.AXIS_MOVE_ABS`, `SML.E_PROGRESS.PROGRESS_DONE`, …
3. Crea gli oggetti **lato applicazione** (NON sono nella libreria — vedi
   [`../application/README.md`](../application/README.md) e
   [`../docs/IMPORT_CHECKLIST.it.md`](../docs/IMPORT_CHECKLIST.it.md)):
   `GVL_App` (`MAX_AXIS`), `GVL_AXIS` (gli array `[1..MAX_AXIS]` + `Control : SML.FB_AxisCtrl` +
   `ItfSmlAxis : SML.I_Axis`), `MAIN`, e opzionalmente `GVL_IO` + `PRG_IoLink_In/_Out` e
   `GVL_AXIS_MAP` + `PRG_Mapping_In/_Out`.

---

## Versioning

`0.9.0` riflette uno stato giovane, testato in simulazione (vedi "Maturità e sicurezza" nel README root).
Incrementa il minor per aggiunte, il patch per correzioni; riserva `1.0.0` alla prima release validata su
hardware. Mantieni stabile il namespace `SML` così le applicazioni non devono ri-prefissare.
