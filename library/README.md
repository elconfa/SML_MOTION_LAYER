# SML — Libreria (core riusabile)

Componente **libreria** del Motion Layer: solo oggetti riusabili, **senza
istanze né configurazione di macchina**. Da impacchettare come *CoDeSys Library*
/ *TwinCAT Library*.

## Manifest (proprietà consigliate del progetto libreria)

| Proprietà | Valore consigliato |
|---|---|
| Title | `SML` (SoftMotion Light — Motion Layer CiA402) |
| Namespace | `SML` |
| Version | `0.9.0` (allineata alla v9 dei sorgenti) |
| Company / Author | *(da impostare)* |
| Licenza | GPL-3.0 (vedi `../LICENSE`) |

> **Namespace**: dopo aver creato la libreria con namespace `SML`, dall'applicazione
> le referenze possono richiedere il prefisso — es. `SML.FB_SmlAxisCtrl`,
> `SML.AXIS_MOVE_ABS`, `SML.E_PROGRESS.PROGRESS_DONE`. Vedi i sorgenti applicativi
> in `../application/` (senza prefisso): aggiungi `SML.` dove il compilatore
> segnala simboli non risolti/ambigui.

## Dipendenze

- **Standard** (TON/TOF, CONCAT, SIZEOF, MEMSET…).
- **FB_S7RTT_OTG** (generatore di traiettoria jerk-limited): incluso in `src/`.
  Se è di terzi (Ruckig/Struckig-light), valutare di **esternalizzarlo** come
  libreria referenziata (licenza + manutenzione).

## Oggetti esportati (`src/`)

| Categoria | Oggetti |
|---|---|
| Enum | `E_PROGRESS`, `E_AXIS_CTRL`, `E_AXIS_STATE`, `SML_DiagCode` |
| Contratto dati (DUT) | `ST_AXIS_CTRL`, `ST_MOVE_DATA`, `ST_AXIS_STATE`, `ST_AXIS_INFO`, `ST_CiA402_Status` |
| Interfaccia | `I_SmlAxis` |
| FB di controllo | `FB_SmlAxisCtrl` (+ `FB_SmlAxisCtrl_METHODS`) |
| FB foglia CiA402 | `SML_Power`, `SML_Reset`, `SML_Home`, `SML_ProfilePosition`, `SML_ProfileVelocity`, `SML_ProfileVelocity_Jog`, `SML_Stop`, `SML_Status`, `SML_Diagnostics`, `SML_TouchProbe` |
| I/O (tipi + utility) | `OpenSML_Axis`, `ST_DriveIn`, `ST_DriveOut`, `SML_TC3Link` |
| MAPPING (UNION) | `U_AXIS_CTRL`, `U_MOVE_DATA`, `U_AXIS_STATE`, `U_AXIS_INFO` |
| Funzioni | `f_GetProgress`, `f_GetState` |
| Costanti libreria | `GVL_SML_CONST` (`PROGRESS_SPAN`, `MAP_SIZE_*`) |
| OTG | `FB_S7RTT_OTG` |

**Nessun oggetto qui referenzia `MAX_AXIS` o `GVL_AXIS`** (verificato): il core è
mono-asse e disaccoppiato dalla macchina. La config (numero assi) e le istanze
stanno in `../application/`.

## Come creare la libreria in CoDeSys

1. *New Project → Library*. Imposta Title/Namespace/Version/Company come sopra.
2. Importa/incolla gli oggetti di `src/` (crea i METHOD/PROPERTY di `I_SmlAxis`
   sotto `FB_SmlAxisCtrl` da `FB_SmlAxisCtrl_METHODS.txt`).
3. Aggiungi le dipendenze (Standard; OTG se esternalizzato).
4. *Save Project as Library* / *Install* nel Library Repository.
5. Nell'applicazione: *Library Manager → Add* e referenzia `SML`.

(TwinCAT: analogo tramite *Library Repository*.)
