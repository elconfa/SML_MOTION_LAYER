# SML_MOTION_LAYER

**Motion layer per azionamenti CiA402 (SoftMotion Light) — CoDeSys 3.5 / TwinCAT 3**

Libreria di controllo assi per drive **CiA402** su EtherCAT che porta il pattern
architetturale di [PLC_MOTION_LAYER](https://github.com/haud-ba/PLC_MOTION_LAYER)
su **SML** (ramo di [OpenSML](https://github.com/feecat/opensml)): **SML sostituisce
le librerie `Tc2_*`/NC** come livello di esecuzione. Un unico FB per asse
(`FB_SmlAxisCtrl`), comandato via **struttura dati** o **interfaccia**, pilota
direttamente gli FB CiA402 di SML (Power/Home/ProfilePosition/ProfileVelocity/
Jog/Stop) + **CSP/OTG**, con **Status completo** e **TouchProbe**.

> I sorgenti sono **export ST testuali** (`.txt`) da importare in CoDeSys/TwinCAT.
> Vedi [`docs/IMPORT_CHECKLIST.md`](docs/IMPORT_CHECKLIST.md).

---

## Caratteristiche

- **Un solo entry-point per asse**: `FB_SmlAxisCtrl` (comando `eCmd` + contratto
  dati Ctrl/State/Info/Data) — superset delle vecchie facce (PP/PV/Jog **e** CSP+OTG).
- **Interfaccia** `I_SmlAxis` (Enable/Home/MoveAbsolute/MoveVelocity/Jog/MoveFollow/
  Stop + proprietà) per coordinatori di alto livello.
- **Avanzamento unificato** `E_PROGRESS` + **stato combinato** `E_AXIS_STATE`.
- **Multi-asse**: `ARRAY[1..MAX_AXIS]` orchestrato da `MAIN`.
- **Portabile**: stesso codice su CoDeSys e TwinCAT; cambia solo lo strato I/O.
- **TouchProbe** e **decoder Status CiA402** integrati.
- **Diagnostica** aggregata (`SML_DiagCode` + testo + primo guasto).
- **MAPPING** opzionale (UNION) per esporre Ctrl/State su bus/ADS.
- **Bridge I/O** pronto (`ST_DriveOut`/`ST_DriveIn` + `SML_IoLink_*`).
- **Simulazione** senza hardware (banchi di test con emulatore CiA402).

---

## Architettura

```
   APPLICAZIONE  (scrive Ctrl+Data, legge State+Info)
        │
        ▼
   MAIN  ──►  FOR n := 1..MAX_AXIS ──►  GVL_AXIS.Control[n]  (FB_SmlAxisCtrl)
        │                                      │  pilota gli FB foglia
        │                                      ▼
        │        SML_Power / Reset / Home / ProfilePosition / ProfileVelocity /
        │        Jog / Stop / Status / Diagnostics / TouchProbe / OTG(CSP)
        ▼
   GVL_AXIS.Axis[n]  (OpenSML_Axis = immagine PDO CiA402)  ──►  DRIVE EtherCAT
```

Comando di un asse (due modi equivalenti):
```pascal
// struttura
GVL_AXIS.Data[1].lrTargetPosition := 50.0;
GVL_AXIS.Ctrl[1].eCmd := AXIS_MOVE_ABS;
// oppure interfaccia
GVL_AXIS.ItfSmlAxis[1].MoveAbsolute(50.0, 100.0);
```

---

## Struttura del repository

| Cartella | Contenuto |
|---|---|
| `src/` | libreria (DUT, enum, interfaccia, FB, GVL, MAIN, MAPPING, bridge I/O) |
| `src/_legacy/` | facce superate (FB_SML, SML_AxisController, …), preservate ma non usate |
| `examples/` | `FB_AxisCycleDemo` (ciclo A↔B) e `PLC_APP` (macchina 2 assi reale) |
| `tests/` | banchi di simulazione (Livello A / B / multi-asse) con emulatore |
| `docs/` | **manuali e guide** (vedi sotto) |
| `docs/history/` | changelog v4→v9, session handoff, bozza Livello A |
| `docs/origin/` | analisi originale OpenSML da cui siamo partiti |
| `binaries/` | **spazio per i progetti compilati CoDeSys e TwinCAT** |

### Documentazione (`docs/`)
- **[`MANUALE_SML.md`](docs/MANUALE_SML.md)** — manuale d'uso stile PLCopen:
  comandi `eCmd`, struct, lettura stato, TouchProbe, cosa fa ogni file, sequenze,
  diagnostica, + **Appendice A** (`FB_AxisCycleDemo`) e **Appendice B** (`PLC_APP`).
- **[`GUIDA_IO_Linking.md`](docs/GUIDA_IO_Linking.md)** — collegamento dei PDO del
  drive: link diretto (CoDeSys) vs bridge (TwinCAT), tabella campo↔oggetto CiA402.
- **[`IMPORT_CHECKLIST.md`](docs/IMPORT_CHECKLIST.md)** — ordine e passi d'import,
  errori tipici, verifica.
- **[`NOTA_Pattern_MotionLayer.md`](docs/NOTA_Pattern_MotionLayer.md)** — il pattern
  architetturale estratto da PLC_MOTION_LAYER.

---

## Avvio rapido

1. Importa i file `src/` in un progetto CoDeSys/TwinCAT (ordine in
   [`IMPORT_CHECKLIST`](docs/IMPORT_CHECKLIST.md); crea i METHOD/PROPERTY di
   `I_SmlAxis` sotto `FB_SmlAxisCtrl` da `FB_SmlAxisCtrl_METHODS.txt`).
2. Imposta `GVL_SML_CONST.MAX_AXIS`.
3. Metti `MAIN` in un task ciclico.
4. Collega `GVL_AXIS.Axis[n]` ai PDO del drive — vedi [`GUIDA_IO_Linking`](docs/GUIDA_IO_Linking.md).
5. Comanda: `GVL_AXIS.Ctrl[n].eCmd := ...` e leggi `GVL_AXIS.State[n]`/`Info[n]`.

Prova senza hardware: `Ctrl[n].xSimulation := TRUE` o esegui un banco di `tests/`.

---

## Esempio macchina (`examples/PLC_APP.txt`)

Macchina di misura/selezione pezzi a **2 assi**:
- **Asse 1** = nastro a rotazione continua (velocity) + **TouchProbe**;
- **Asse 2** = posizionatore.

Ciclo: homing (una volta) → misura pezzo via TouchProbe tra fotocellula coperta e
liberata → se **fuori misura** scarto **non bloccante** (nastro prosegue, pezzo
espulso e contato in parallelo, **pipelining**: entra già il pezzo successivo) →
se **buono** asse 2 avanti a +400 mm poi ritorno a 0 mentre il nastro riparte →
**standby** con timeout (ri-azzerabile con home, ripresa con segnale esterno).
I/O macchina puliti (comandi in ingresso, stato in uscita). Dettaglio in
[`MANUALE_SML` Appendice B](docs/MANUALE_SML.md).

---

## Progetti compilati (`binaries/`)

Caricare qui i **progetti compilati**:
- CoDeSys: archivio progetto / libreria (`.library`, `.compiled-library`, `.projectarchive`);
- TwinCAT: soluzione / libreria (`.tsproj`, `.library`, `.tmc`).

Vedi [`binaries/README.md`](binaries/README.md).

---

## Evoluzione

Da libreria di facce (v4) a motion layer completo (v9): Livello A (`E_PROGRESS`),
Livello B (contratto + interfaccia + FB comando + multi-asse + MAPPING), superset
(Status + TouchProbe), bridge I/O, manuali ed esempi. Changelog in `docs/history/`.

## Crediti / origine

- Base: **OpenSML** (feecat/opensml) — SoftMotion Light CiA402.
- Pattern architetturale: **PLC_MOTION_LAYER** (haud-ba) — TwinCAT NC/PLCopen.
- OTG: `FB_S7RTT_OTG` (generatore di traiettoria jerk-limited).

## Licenza

**GPL-3.0** — vedi [`LICENSE`](LICENSE).

> Copyright (C) 2024-2026 Massimo Confalonieri
>
> Questo programma è software libero: puoi ridistribuirlo e/o modificarlo secondo
> i termini della GNU General Public License versione 3, pubblicata dalla Free
> Software Foundation.

SML deriva da **OpenSML** (feecat/opensml), rilasciato sotto **GPL-3.0**: per
copyleft l'intera opera è quindi GPL-3.0 (una licenza permissiva non sarebbe
compatibile). L'OTG (`FB_S7RTT_OTG`) e le librerie del produttore mantengono le
rispettive licenze.

### Git LFS
I binari in [`binaries/`](binaries/) (`.library`, `.compiled-library*`,
`.projectarchive`, `.tpzip`, `.tszip`, `.zip`) sono tracciati con **Git LFS**
(vedi [`.gitattributes`](.gitattributes)). Per lavorarci: installa git-lfs
(`brew install git-lfs` o equivalente) e `git lfs install` una volta; poi
`git add`/`push` dei binari usa automaticamente LFS.
