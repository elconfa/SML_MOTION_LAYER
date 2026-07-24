# SML v8 — Changelog

**Base:** SML_v7
**Date:** 2026-07-22
**Scope:** **conclusione del Livello B** — layer **MAPPING/UNION** bus-agnostico
con compilazione condizionale. Ultimo elemento mancante del pattern
PLC_MOTION_LAYER.

Riferimenti: `CHANGELOG_v7.md` (multi-asse), `../NOTA_Pattern_MotionLayer.md`.

---

## Novità v8

### UNION (overlay tipizzato / grezzo)
- `U_AXIS_CTRL`, `U_MOVE_DATA`, `U_AXIS_STATE`, `U_AXIS_INFO` — ciascuna:
  `stData` (struct tipizzata) + `aRaw` (blocco byte a dimensione fissa
  `MAP_SIZE_*`). Consente di linkare l'elemento al fieldbus come struct o
  come blocco grezzo, con dimensione mappata stabile.

### Lato bus
- `GVL_AXIS_MAP.txt` — array `[1..MAX_AXIS]` di UNION: `Ctrl`/`Data` (bus->plc),
  `State`/`Info` (plc->bus). Da collegare al process image / esporre via ADS.

### Programmi di mapping (gate runtime)
- `PRG_Mapping_In.txt` — `GVL_AXIS_MAP.Ctrl/Data -> GVL_AXIS.Ctrl/Data`.
- `PRG_Mapping_Out.txt` — `GVL_AXIS.State/Info -> GVL_AXIS_MAP.State/Info`.
- Entrambi gate da `GVL_AXIS_MAP.AXIS_MAP_ENABLE` (BOOL): FALSE = bypass
  immediato (comportamento = v7). **Flag runtime al posto della pragma
  `{IF defined(AXIS_MAP)}`**: quest'ultima non e' supportata nella parte
  dichiarazione su CoDeSys. Check di dimensione a runtime
  (`SIZEOF(struct) <= MAP_SIZE_*`): se supera, il mapping si blocca.

### Costanti
- `GVL_SML_CONST` + `MAP_SIZE_CTRL/DATA/STATE/INFO` (byte, dimensione fissa
  dei blocchi bus; devono essere >= SIZEOF della struct).

### MAIN
- Chiama `PRG_Mapping_In()` prima del loop assi e `PRG_Mapping_Out()` dopo. Senza
  `AXIS_MAP` i due si auto-bypassano.

---

## Scelte di progetto (vs PLC_MOTION_LAYER)
- **Copia via assegnazione strutturata** (`GVL_AXIS.Ctrl[i] := ...stData`) invece
  di `memcpy`: portabile, nessuna dipendenza da libreria memcpy; corretta perche'
  i due lati hanno lo stesso tipo. La vista `.aRaw` resta disponibile per il
  linking grezzo sul fieldbus.
- **UNION a dimensione fissa** (`MAP_SIZE_*`): stabilita' del blocco mappato anche
  se la struct cambia (entro il limite), con Check di sicurezza — stessa filosofia
  del "sizeof a startup" di PLC_MOTION_LAYER.

## Livello B — COMPLETO
| Elemento pattern | Stato |
|---|---|
| Contratto Ctrl/State/Info/Data | ✅ v6 |
| Base esecuzione (FB foglia diretti) + CSP/OTG | ✅ v6 |
| FB comando eCmd + E_PROGRESS combinato | ✅ v6 |
| Interfaccia I_Axis | ✅ v6 |
| Orchestrazione ARRAY[1..MAX_AXIS] + MAIN | ✅ v7 |
| MAPPING/UNION + define | ✅ v8 |

## Note tecniche / rischi (CoDeSys)
- **MAPPING via flag runtime** `GVL_AXIS_MAP.AXIS_MAP_ENABLE` (BOOL), NON via
  pragma di compilazione condizionale: `{IF defined(...)}` non e' supportata
  nella parte dichiarazione su CoDeSys. Per esclusione a compile-time,
  opzionale, spostare la pragma nel CORPO di PRG_Mapping_In/Out.
- Bound array UNION da costante globale (`MAP_SIZE_*`): supportato.
- `SIZEOF` (UDINT) confrontato con `MAP_SIZE_*` (UDINT): coerente.
- `GVL_AXIS_MAP` presente sempre (anche senza AXIS_MAP): memoria inutilizzata,
  innocua.
- Restano i caveat precedenti (enum non-strict; scaling velocita'=posizione;
  creare i METHOD/PROPERTY di I_Axis all'import).

## Verifica
1. Build con `AXIS_MAP_ENABLE = FALSE`: compila; PRG_Mapping_In/Out bypassano;
   comportamento = v7.
2. Mettere `GVL_AXIS_MAP.AXIS_MAP_ENABLE := TRUE`, collegare `GVL_AXIS_MAP` al
   bus (o forzare in online), scrivere `GVL_AXIS_MAP.Ctrl[n].stData.eCmd` e
   verificare che compaia in `GVL_AXIS.Ctrl[n]` e che
   `GVL_AXIS_MAP.State[n].stData` rifletta lo stato.
3. Verificare il Check: portare temporaneamente un `MAP_SIZE_*` sotto la
   SIZEOF della struct e confermare che il mapping si blocca (nessuna copia).
