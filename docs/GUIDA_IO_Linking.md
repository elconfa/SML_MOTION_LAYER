# SML — Guida al collegamento I/O degli assi fisici (CoDeSys e TwinCAT)

**Versione:** SML_v9 · **Data:** 2026-07-23

Come agganciare `GVL_AXIS.Axis[n]` (la struct `OpenSML_Axis`, cioè l'immagine
PDO CiA402) ai PDO reali del drive EtherCAT — su **CoDeSys** e su **TwinCAT**.

---

## 1. Concetto chiave

`GVL_AXIS.Axis[n]` è una **struct software** (senza `AT %I*/%Q*`): così il codice
resta **portabile** e **testabile in simulazione**. Il codice della libreria
(`MAIN`, `FB_SmlAxisCtrl`, `GVL_AXIS`) è **identico** su CoDeSys e TwinCAT.
Cambia **solo il sottile strato di aggancio I/O**.

I campi di `OpenSML_Axis` sono **già ordinati output-poi-input**:
- primi 10 campi = **PLC → Drive** (RxPDO)
- restanti 12 campi = **Drive → PLC** (TxPDO)

Questo ordine è ciò che rende puliti sia il link diretto sia il bridge.

Due strategie di aggancio:
- **(A) Link diretto** — ogni oggetto PDO è legato al membro della struct. Nessun
  codice extra. Idiomatico su **CoDeSys**.
- **(B) Bridge** — i PDO sono mappati su due immagini `AT %Q*`/`AT %I*`
  (`ST_DriveOut`/`ST_DriveIn`) e un FB copia immagine↔struct ogni ciclo.
  Idiomatico su **TwinCAT** (è ciò che faceva `SML_TC3Link`).

Passo comune a entrambe: nel drive, assicurarsi che gli oggetti siano
nell'**assegnazione PDO** (RxPDO 0x160x, TxPDO 0x1A0x). Se un oggetto non è nel
PDO, non compare per il link.

---

## 2. Tabella di riferimento (campo ↔ oggetto CiA402)

| Campo `Axis[n].` | Oggetto | Dir. | PDO |
|---|---|---|---|
| ControlWord | 0x6040 | →Drive | RxPDO |
| Modes_of_operation | 0x6060 | →Drive | RxPDO |
| Target_Position | 0x607A | →Drive | RxPDO |
| Profile_Velocity | 0x6081 | →Drive | RxPDO |
| Target_Velocity | 0x60FF | →Drive | RxPDO |
| Target_Torque | 0x6071 | →Drive | RxPDO |
| Profile_Acceleration | 0x6083 | →Drive | RxPDO |
| Profile_Deceleration | 0x6084 | →Drive | RxPDO |
| Home_Offset | 0x607C | →Drive | RxPDO |
| TouchProbe_ControlWord | 0x60B8 | →Drive | RxPDO |
| StatusWord | 0x6041 | ←Drive | TxPDO |
| Modes_of_operation_display | 0x6061 | ←Drive | TxPDO |
| Torque_Actual_Value | 0x6077 | ←Drive | TxPDO |
| Position_Actual_Value | 0x6064 | ←Drive | TxPDO |
| Velocity_Actual_Value | 0x606C | ←Drive | TxPDO |
| Current_Actual_Value | 0x6078 | ←Drive | TxPDO |
| Following_Error_Actual_Value | 0x60F4 | ←Drive | TxPDO |
| Error_Code | 0x603F | ←Drive | TxPDO |
| Digital_Inputs | 0x60FD | ←Drive | TxPDO |
| TouchProbe_StatusWord | 0x60B9 | ←Drive | TxPDO |
| TouchProbe_Rising_Value | 0x60BA | ←Drive | TxPDO |
| TouchProbe_Falling_Value | 0x60BB | ←Drive | TxPDO |

---

## 3. CoDeSys — link diretto nella scheda I/O Mapping (consigliato)

Nessun codice bridge, nessun `AT`: si lega ogni canale PDO al membro della struct.

1. Albero **Devices** → EtherCAT Master → drive → tab **`<Slave> I/O Mapping`**
   (o *EtherCAT I/O Mapping*).
2. Per ogni riga (oggetto PDO), colonna **Variable** → doppio clic → **Browse** →
   seleziona il membro, es. `GVL_AXIS.Axis[1].ControlWord`,
   `GVL_AXIS.Axis[1].StatusWord`, … (indice **1-based**: `[1]`, `[2]`, …).
3. Ripeti per l'asse 2 → `GVL_AXIS.Axis[2].…`, e così via.
4. Sul modulo, **"Always update variables" = Enabled 2 (always in bus cycle
   task)**, così i campi si aggiornano anche se non usati esplicitamente nel codice.
5. Se un oggetto PDO manca: abilita **Expert settings** sul drive e aggiungilo
   alla RxPDO/TxPDO.

> In alternativa si possono usare indirizzi `%IW/%QW` con `AT`, ma il binding
> nella scheda I/O Mapping alla struct è l'idioma CoDeSys corretto e portabile.

### 3b. CoDeSys — variante con bridge (opzionale, come TwinCAT)
Se preferisci un unico punto di mappatura anche su CoDeSys, usa i due DUT
`ST_DriveOut`/`ST_DriveIn` come al §4: dichiari le due immagini, le mappi nella
scheda I/O e usi lo stesso codice bridge del §5.

---

## 4. TwinCAT — bridge con immagine `AT %I*/%Q*` (idiomatico)

TwinCAT non ama linkare una struct a direzione mista. Approccio pulito
(quello di `SML_TC3Link`): **due immagini separate** + copia ciclica.

1. Dichiara le due metà come immagini mappabili (i DUT sono in `ST_DriveOut.txt`
   / `ST_DriveIn.txt`, stesso ordine/tipi della struct):
   ```pascal
   VAR_GLOBAL
       DriveOut AT %Q* : ARRAY[1..GVL_SML_CONST.MAX_AXIS] OF ST_DriveOut; // RxPDO
       DriveIn  AT %I* : ARRAY[1..GVL_SML_CONST.MAX_AXIS] OF ST_DriveIn;   // TxPDO
   END_VAR
   ```
2. Nel drive EtherCAT: **Change Link** su ogni oggetto PDO → collega a
   `DriveOut[1].ControlWord`, `DriveIn[1].StatusWord`, … (blocchi contigui →
   link rapido; oppure "one-to-one" se i tipi combaciano).
3. Copia ciclica (vedi §5), con l'ordine corretto rispetto a `MAIN`.

> Alternativa TwinCAT senza bridge: **Change Link** diretto di ogni PDO a
> `GVL_AXIS.Axis[1].<membro>`. Funziona, ma sono ~22 link × N assi a mano.

---

## 5. Il bridge pronto (file già nel progetto)

Oggetti forniti (import da `.txt`):
- **`ST_DriveOut`** = primi 10 campi di `OpenSML_Axis` (output), stesso ordine/tipi.
- **`ST_DriveIn`**  = restanti 12 campi (input), stesso ordine/tipi.
- **`GVL_IO`** — `DriveOut AT %Q*` / `DriveIn AT %I*` (`ARRAY[1..MAX_AXIS]`) +
  flag `IO_LINK_ENABLE : BOOL`.
- **`SML_IoLink_In`** — PROGRAM: `DriveIn[n] -> Axis[n]` (input).
- **`SML_IoLink_Out`** — PROGRAM: `Axis[n] -> DriveOut[n]` (output).

**Timing corretto (zero ritardo):** `MAIN` chiama già `SML_IoLink_In()` **prima**
del ciclo assi e `SML_IoLink_Out()` **dopo** (accanto a MAPPING_in/out). Non c'è
il ritardo di 1 ciclo del bridge singolo. Entrambi si auto-bypassano se
`GVL_IO.IO_LINK_ENABLE = FALSE`.

**Attivazione:** metti `GVL_IO.IO_LINK_ENABLE := TRUE` e collega nel configuratore
`GVL_IO.DriveOut[n]` ai RxPDO e `GVL_IO.DriveIn[n]` ai TxPDO del drive.

> **CoDeSys con link diretto (§3):** NON usare il bridge — lascia
> `IO_LINK_ENABLE = FALSE` e mappa `GVL_AXIS.Axis[n]` direttamente. I due
> approcci sono mutuamente esclusivi (il flag evita conflitti).

### Nota: copia campo-a-campo vs blocco
Poiché `ST_DriveOut`/`ST_DriveIn` hanno gli **stessi tipi e lo stesso ordine**
delle rispettive metà di `OpenSML_Axis`, i due programmi copiano **campo-a-campo**:
la scelta più sicura (indipendente dal padding). Evita `memcpy` di blocco tra la
struct mista e le due metà: il padding potrebbe non coincidere.

---

## 6. Verifiche e accortezze (entrambe le piattaforme)

- **Distributed Clocks (DC)**: attivali sul drive se usi CSP (`AXIS_MOVE_CSP`) o
  TouchProbe (latch sincronizzato).
- **Modi supportati**: CSP (modo 8) per il percorso OTG; PP(1)/PV(3)/HM(6) per gli
  altri comandi.
- **`Ctrl.Scale`** = counts/unità dell'encoder; **`Ctrl.CycleTime`** = periodo
  reale del task (per l'OTG).
- **PDO TouchProbe** (0x60B8/0x60B9/0x60BA/0x60BB): spesso non nel PDO di default →
  aggiungili se usi il TouchProbe.
- **"Always update variables"** (CoDeSys) o mapping completo (TwinCAT): assicura
  che tutti i campi si aggiornino.

---

## 7. Riassunto

| | CoDeSys | TwinCAT |
|---|---|---|
| Strategia idiomatica | Link diretto (scheda I/O Mapping) | Bridge (`ST_DriveOut`/`ST_DriveIn` + copia) |
| Codice extra | nessuno | programma bridge (`SML_IoLink`) |
| `AT %I*/%Q*` | no (binding alla struct) | sì (sulle due immagini) |
| DUT bridge | opzionali (variante §3b) | necessari |
| Codice libreria (MAIN/FB/GVL) | identico | identico |
