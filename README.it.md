[English](README.md) | **Italiano**

# SML_MOTION_LAYER

**Un motion layer senza licenze per drive CiA402 — CoDeSys 3.5 e TwinCAT 3, puro Structured Text IEC 61131-3.**

Controlla drive servo/passo EtherCAT che parlano il profilo **CiA402** (homing, profile position/velocity, jog,
posizione ciclica sincrona con generatore di traiettoria online jerk-limited) **interamente in Structured Text** —
un function block per asse, comandato tramite una struttura dati *oppure* un'interfaccia. Nessun runtime di
motion del produttore sotto.

![License: GPL-3.0](https://img.shields.io/badge/license-GPL--3.0-blue.svg)
![Platform](https://img.shields.io/badge/PLC-CoDeSys%203.5%20%7C%20TwinCAT%203-informational)
![Language](https://img.shields.io/badge/IEC%2061131--3-Structured%20Text-orange)
![Status](https://img.shields.io/badge/status-early%20%2F%20simulation--tested-yellow)

> **I sorgenti sono export testuali Structured Text (`.txt`)** da importare in un progetto CoDeSys/TwinCAT.
> Vedi [`docs/IMPORT_CHECKLIST.it.md`](docs/IMPORT_CHECKLIST.it.md).

---

## 💡 Nessuna licenza asse/motion richiesta

È il punto centrale. SML implementa **da sé** la macchina a stati **CiA402** e i **profili di moto**, in puro ST
IEC 61131-3, comandando il drive **direttamente sull'immagine di processo EtherCAT** (PDO). Di conseguenza:

- **CoDeSys** — **non** serve **SoftMotion** (l'add-on di motion a pagamento, spesso per-asse).
- **TwinCAT** — **non** serve **TF5000 TC3 NC PTP** (la libreria di motion PLCopen `MC_*`).

Ti tieni solo ciò che hai già: il **runtime PLC** standard e un **master EtherCAT**. Il motion — la parte che
normalmente costa una licenza — sta in questo codice open-source.

> Ambito onesto: sul tuo sistema restano comunque una licenza base di runtime PLC e il fieldbus, come sempre. Il
> punto è che **nessuna opzione *motion*** è necessaria. E, come per qualunque codice di moto, **sei responsabile
> della validazione sul tuo hardware e della sicurezza macchina** — vedi [Maturità e sicurezza](#maturità-e-sicurezza).

---

## Perché ti può interessare

- **Motion senza licenze** sia su CoDeSys sia su TwinCAT (vedi sopra).
- **Un solo punto d'ingresso per asse** — `FB_AxisCtrl`: un enum comando (`eCmd`) più un contratto dati pulito
  `Ctrl / Data / State / Info`. È un **superset** delle classiche FB di profilo: PP/PV/Jog **e** CSP+OTG, **decodifica
  di stato** e **touch-probe** tutto in un unico blocco.
- **Due modi per comandarlo**, usa quello che preferisci: una **struttura dati** (ottima per codice HMI/ricette) o
  un'**interfaccia** `I_Axis` (ottima per coordinatori di alto livello).
- **Portabile** — lo *stesso* ST gira su CoDeSys e TwinCAT; cambia solo il mapping I/O verso il drive.
- **Modello di stato unificato** — un enum di avanzamento `E_PROGRESS` e uno stato combinato `E_AXIS_STATE`: leggere
  "dov'è quest'asse" è un campo solo, non una caccia tra i flag.
- **Multi-asse nativo** — un `ARRAY[1..MAX_AXIS]` orchestrato da un solo `MAIN`; scali cambiando una costante.
- **Layer MAPPING bus-agnostico** (basato su UNION) per esporre Ctrl/State degli assi su fieldbus o ADS con
  un'immagine grezza a dimensione fissa — opzionale, disattivo di default.
- **Simulazione senza hardware** — i banchi di test pilotano un emulatore CiA402: porti su la logica prima di cablare.
- **Split libreria + applicazione** — un `library/` riusabile (namespace `SML`) e un template `application/` con una
  vera **macchina d'esempio a 2 assi**.

---

## Architettura

```
   LA TUA APPLICAZIONE   (scrive Ctrl + Data, legge State + Info)
        │
        ▼
   MAIN  ──►  FOR n := 1..MAX_AXIS ──►  GVL_AXIS.Control[n]  (FB_AxisCtrl)
                                             │  pilota le FB foglia CiA402
                                             ▼
        Power / Reset / Home / ProfilePosition / ProfileVelocity /
        Jog / Stop / Status / Diagnostics / TouchProbe / CSP+OTG
                                             │
                                             ▼
   GVL_AXIS.Axis[n]  (OpenSML_Axis = immagine PDO CiA402)  ──►  DRIVE EtherCAT
```

---

## Uso in 30 secondi

Comando dell'asse 1 tramite **struttura dati**:

```pascal
// Abilita, poi homing (dai i comandi man mano che l'asse è pronto)
GVL_AXIS.Ctrl[1].eCmd := AXIS_ENABLE;
GVL_AXIS.Ctrl[1].eCmd := AXIS_HOME;

// Movimento assoluto: vai a 250.0 unità a 100.0 unità/s
GVL_AXIS.Data[1].lrTargetPosition := 250.0;
GVL_AXIS.Data[1].lrVelocity       := 100.0;
GVL_AXIS.Ctrl[1].eCmd             := AXIS_MOVE_ABS;

// Leggi dov'è
IF GVL_AXIS.State[1].eProgress = PROGRESS_DONE THEN
    // movimento finito — GVL_AXIS.Info[1].lrActualPosition è al target
END_IF
IF GVL_AXIS.State[1].xError THEN
    GVL_AXIS.Ctrl[1].eCmd := AXIS_RESET;
END_IF
```

Oppure lo stesso movimento tramite l'**interfaccia** `I_Axis`:

```pascal
GVL_AXIS.ItfSmlAxis[1].Enable();
GVL_AXIS.ItfSmlAxis[1].Home();
GVL_AXIS.ItfSmlAxis[1].MoveAbsolute(lrPosition := 250.0, lrVelocity := 100.0);
```

Velocità continua e movimento inseguitore jerk-limited (CSP + OTG):

```pascal
GVL_AXIS.Data[1].lrVelocity := 80.0;                  // con segno
GVL_AXIS.Ctrl[1].eCmd       := AXIS_MOVE_VELOCITY;

GVL_AXIS.ItfSmlAxis[1].MoveFollow(...);               // CSP con generatore di traiettoria online
```

Comandi (`eCmd`): `AXIS_ENABLE`, `AXIS_DISABLE`, `AXIS_RESET`, `AXIS_HOME`, `AXIS_MOVE_ABS`, `AXIS_MOVE_REL`,
`AXIS_MOVE_VELOCITY`, `AXIS_JOG_POS`, `AXIS_JOG_NEG`, `AXIS_MOVE_CSP`, `AXIS_STOP`.

Riferimento completo comandi/API: [`docs/MANUALE_SML.it.md`](docs/MANUALE_SML.it.md).

---

## Struttura del repository (due livelli)

| Cartella | Contenuto |
|---|---|
| **`library/`** | core riusabile (enum, DUT, `I_Axis`, `FB_AxisCtrl`, FB foglia CiA402, funzioni, OTG) — [README](library/README.md) |
| **`application/`** | template macchina che referenzia la libreria — [README](application/README.md) |
| `application/src/` | `GVL_App` (`MAX_AXIS`), `GVL_AXIS`, `MAIN`, MAPPING, bridge I/O opzionale |
| `application/examples/` | `FB_AxisCycleDemo`, `PLC_APP` (la macchina a 2 assi) |
| `application/tests/` | banchi di simulazione (emulatore CiA402) |
| `legacy/` | facce superate, conservate per tracciabilità |
| `docs/` | manuale, guida collegamento I/O, checklist di import, la nota di design estratta |
| `binaries/` | progetti compilati CoDeSys (`codesys/`) e TwinCAT (`twincat/`) (Git LFS) |

---

## Avvio rapido

1. Crea la **libreria** da `library/src/` (namespace `SML`) — vedi [`library/README.md`](library/README.md) — oppure
   importa tutto in un progetto unico seguendo [`docs/IMPORT_CHECKLIST.it.md`](docs/IMPORT_CHECKLIST.it.md). Crea i
   metodi/proprietà di `I_Axis` sotto `FB_AxisCtrl` da `FB_AxisCtrl_METHODS.txt`.
2. Nell'**applicazione** referenzia la libreria e imposta `GVL_App.MAX_AXIS`.
3. Metti `MAIN` in un task ciclico.
4. Mappa `GVL_AXIS.Axis[n]` sui PDO del drive — vedi [`docs/GUIDA_IO_Linking.it.md`](docs/GUIDA_IO_Linking.it.md).
5. Comanda con `GVL_AXIS.Ctrl[n].eCmd := ...` e leggi `GVL_AXIS.State[n]` / `Info[n]`.

**Provalo senza drive:** metti `Ctrl[n].xSimulation := TRUE`, o esegui uno dei banchi in `application/tests/`.

---

## Esempio reale — `application/examples/PLC_APP`

Una macchina di misura e selezione a 2 assi:

- **Asse 1** = nastro a rotazione continua (modo velocità) con **touch-probe**;
- **Asse 2** = posizionatore.

Ciclo: homing una volta → misura di ogni pezzo via touch-probe tra "fotocellula coperta" e "fotocellula liberata" →
se il pezzo è **fuori tolleranza**, **scarto non bloccante** (il nastro prosegue, il pezzo viene espulso e contato da
un tracker parallelo, e **il pezzo successivo sta già entrando** — pipelining) → se **buono**, l'asse 2 va a +400 poi
torna a 0 mentre il nastro riparte → **standby** con timeout (ri-azzeri con un comando, riprendi con un segnale
esterno). I/O macchina puliti in ingresso e in uscita. Dettagli in [`docs/MANUALE_SML.it.md`](docs/MANUALE_SML.it.md)
(Appendice B).

---

## Maturità e sicurezza

Sii consapevole di cos'è:

- **Progetto giovane, testato in simulazione.** Il core compila correttamente in CoDeSys 3.5 e gira contro un
  emulatore CiA402 nei banchi di test. La recente riorganizzazione a due livelli e la rinomina degli oggetti sono
  **pronte all'import** ma vanno **ri-verificate con un Build** dopo l'import (i sorgenti sono export `.txt`, quindi
  non c'è ancora un binario di cui fidarsi).
- **Il motion control è rilevante per la sicurezza.** Questo codice muove hardware reale. **Valida ogni funzione sul
  tuo drive**, mantieni un percorso hardware di **STO / arresto d'emergenza** indipendente da questa logica, e rispetta
  le norme di sicurezza macchina che ti riguardano. La licenza esclude ogni garanzia (GPL-3.0, "as is").

Feedback, issue e segnalazioni dal campo sono benvenuti — è proprio la fase in cui servono di più.

---

## Origine e crediti — e la parte onesta su come è stato fatto

Questo progetto poggia apertamente su due spalle, ed è stato costruito con l'aiuto di un assistente AI. Nulla di
tutto ciò è nascosto:

- **[OpenSML](https://github.com/feecat/opensml)** (feecat) — le FB di esecuzione CiA402 "SoftMotion Light" che questo
  layer pilota. SML è un **derivato di OpenSML**, ed è per questo che il repo è **GPL-3.0** (vedi [Licenza](#licenza)).
- **[PLC_MOTION_LAYER](https://github.com/haud-ba/PLC_MOTION_LAYER)** (haud-ba) — il **pattern architetturale**
  (contratto Ctrl/State/Info/Data, stato di avanzamento combinato, FB per-asse, orchestrazione ad array, mapping a
  UNION) è stato estratto da questo progetto TwinCAT NC/PLCopen e ri-applicato sopra SML.
- **Sviluppo assistito da AI.** L'estrazione del design, il porting su SML, i layer multi-asse e MAPPING, la macchina
  d'esempio e questa documentazione sono stati prodotti in collaborazione con **Claude di Anthropic**. Una persona ha
  rivisto e diretto il lavoro; l'AI ha scritto e rifattorizzato codice e documentazione sotto quella direzione. Il
  valore offerto è l'*integrazione* — un motion layer coerente e senza licenze per due runtime — non la pretesa di una
  novità artigianale.

Quindi: ciò che è nuovo qui è mettere le FB di esecuzione di OpenSML dietro il contratto di PLC_MOTION_LAYER, sia su
CoDeSys sia su TwinCAT, senza licenza di motion. L'OTG (`FB_S7RTT_OTG`) e le eventuali librerie del produttore
mantengono le proprie licenze.

---

## Licenza

**GPL-3.0** — vedi [`LICENSE`](LICENSE). Poiché SML deriva da OpenSML (GPL-3.0), per copyleft l'intera opera è GPL-3.0
(una licenza permissiva non sarebbe compatibile, e una compiled-library chiusa ridistribuita a terzi violerebbe la GPL).

> Copyright (C) 2024-2026 Massimo Confalonieri. Questo è software libero: puoi ridistribuirlo e/o modificarlo secondo i
> termini della GNU General Public License versione 3, pubblicata dalla Free Software Foundation.
