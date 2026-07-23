# Progetti compilati

Caricare qui i **progetti/librerie compilate** dei due target.

## CoDeSys
- `SML_MOTION_LAYER.library` — libreria installabile, oppure
- `SML_MOTION_LAYER.compiled-library` — libreria compilata (protetta), oppure
- `*.projectarchive` — archivio progetto completo.

Suggerito: sottocartella `binaries/codesys/`.

## TwinCAT
- soluzione (`*.tsproj` + progetto), oppure
- `*.library` esportata, oppure
- `*.tmc` (module description).

Suggerito: sottocartella `binaries/twincat/`.

## Note
- I sorgenti ST testuali stanno in `../src/` (import) — qui vanno i **build**.
- I file binari grandi possono richiedere
  [Git LFS](https://git-lfs.com): `git lfs track "*.library" "*.compiled-library" "*.projectarchive"`.
- Indicare in ogni upload: **versione** dell'ambiente (es. CoDeSys 3.5 SPx /
  TwinCAT 3.1.40xx) e la **revisione** dei sorgenti da cui è stato compilato.
