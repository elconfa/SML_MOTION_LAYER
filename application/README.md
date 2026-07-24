# Applicazione (template macchina)

Livello **applicativo**: istanzia e cabla la libreria `SML` per una macchina
concreta. Referenzia la libreria (`../library/`) e contiene la configurazione,
le istanze, l'orchestratore, il bridge I/O e gli esempi.

> **Namespace**: se la libreria è pubblicata con namespace `SML`, aggiungi il
> prefisso `SML.` alle referenze di libreria dove il compilatore lo richiede
> (es. `SML.FB_SmlAxisCtrl`, `SML.AXIS_MOVE_ABS`). I sorgenti qui sono senza
> prefisso (uso come progetto unico o namespace non forzato).

## `src/` — configurazione + orchestrazione

| File | Ruolo |
|---|---|
| `GVL_App` | **configurazione macchina**: `MAX_AXIS` (cambia solo qui per scalare) |
| `GVL_AXIS` | array `[1..MAX_AXIS]` di Axis/Ctrl/Data/State/Info + `Control` (FB) + `ItfSmlAxis` |
| `MAIN` | esegue tutti gli assi ogni ciclo; chiama I/O bridge + MAPPING |
| `GVL_IO`, `SML_IoLink_In/_Out` | bridge I/O verso i PDO del drive (opzionale) |
| `GVL_AXIS_MAP`, `MAPPING_in/_out` | esporre Ctrl/State su bus/ADS (opzionale) |

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
