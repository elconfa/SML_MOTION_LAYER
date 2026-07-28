# Applicazione (template macchina)

Livello **applicativo**: istanzia e cabla la libreria `SML` per una macchina
concreta. Referenzia la libreria (`../library/`) e contiene la configurazione,
le istanze, l'orchestratore, il bridge I/O e gli esempi.

> **Namespace**: se la libreria è pubblicata con namespace `SML`, aggiungi il
> prefisso `SML.` alle referenze di libreria dove il compilatore lo richiede
> (es. `SML.FB_AxisCtrl`, `SML.AXIS_MOVE_ABS`). I sorgenti in `src/` sono senza
> prefisso (uso come progetto unico o namespace non forzato).

## `starter/` — il minimo per iniziare (consigliato)
Tre oggetti pronti da incollare in un progetto nuovo che **referenzia la libreria**
(`GVL_App`, `GVL_AXIS`, `MAIN`) già con il prefisso `SML.`, più una guida rapida in
4 passi. È il punto di partenza più semplice. Vedi [`starter/README.md`](starter/README.md).

## `src/` — template completo (configurazione + orchestrazione)

| File | Ruolo |
|---|---|
| `GVL_App` | **configurazione macchina**: `MAX_AXIS` (cambia solo qui per scalare) |
| `GVL_AXIS` | array `[1..MAX_AXIS]` di Axis/Ctrl/Data/State/Info + `Control` (FB) + `ItfSmlAxis` |
| `MAIN` | esegue tutti gli assi ogni ciclo; **istanzia** gli FB bridge di libreria (I/O + MAPPING) |
| `GVL_IO` | immagini `DriveIn/DriveOut` per il bridge I/O (opzionale; gli FB `SML.FB_IoLink_In/_Out` sono in libreria) |
| `GVL_AXIS_MAP` | array UNION lato bus per esporre Ctrl/State su bus/ADS (opzionale; gli FB `SML.FB_Mapping_In/_Out` sono in libreria) |

## `examples/`
- `FB_AxisCycleDemo` — ciclo A↔B di un asse (pattern comando→attesa).
- `PLC_APP` — macchina reale a 2 assi (nastro+TouchProbe / posizionatore) con
  standby e pipelining. Vedi `../docs/MANUALE_SML.md` (Appendici A/B).

## `tests/`
Banchi di simulazione con emulatore CiA402 (Livello A / B / multi-asse).
Eseguire UN banco AL POSTO di `MAIN` nel task (non entrambi).

## Come usarlo

1. Referenzia la libreria `SML` (Library Manager).
2. Imposta `GVL_App.MAX_AXIS`.
3. Metti `MAIN` in un task ciclico.
4. Collega `GVL_AXIS.Axis[n]` ai PDO del drive — vedi `../docs/GUIDA_IO_Linking.md`.
5. Comanda: `GVL_AXIS.Ctrl[n].eCmd := ...`, leggi `GVL_AXIS.State[n]`/`Info[n]`.

Riferimento comandi e API: `../docs/MANUALE_SML.md`.
