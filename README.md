# GMBasicModules

Hosting pubblico dei moduli runtime scaricabili on-demand da **GMBasic** (comando `gmb setup`).

## Librerie native per-OS (`gmb-libs-2026.08.28`)

Bundle con tutte le librerie di supporto per ogni sistema operativo: grafica/audio (SDL), motore PDF, OCR (Tesseract), cifratura (SSL), ecc.

| OS | Bundle |
|---|---|
| macOS Apple Silicon | `gmb-libs-osx-arm64.zip` |
| macOS Intel | `gmb-libs-osx-x64.zip` |
| Linux x86-64 | `gmb-libs-linux-x64.zip` |
| Linux arm64 | `gmb-libs-linux-arm64.zip` |
| Windows x64 | `gmb-libs-win-x64.zip` |

Per installare: in GMBasic esegui `gmb setup LIBS` (oppure `gmb setup ALL` per includere anche Torch e Chromium).
