# GMBasicModules

Hosting pubblico delle **librerie native** scaricate on-demand da GMBasic (`gmb libs ensure`,
`gmb setup`). Il contratto è l'indice **`libs-manifest.json`**: descrive, per RID, i gruppi
disponibili con URL, sha256 e dimensione di ogni file. Non si scarica nulla che l'indice non
dichiari.

## Gruppi

| gruppo | cosa contiene | quando serve |
|---|---|---|
| **support** | terze parti: SDL2/SDL3, freetype/harfbuzz, crypto/ssl, curl, zstd/brotli/lz4/lzma, png/jpeg/tiff/webp, hdf5, pdf_oxide, tesseract+tessdata | sempre (dipendenze esterne) |
| **core** | nostre librerie C: runtime, gfx, screen, pkg, compress, grep, tray, ai_llm, ai_modelio, sql ext | sempre |
| **engine** | plumbing AI nostro: `libllmcodec_*` (17 codec), `libllmtoken`, `libllmengine_moe_stream` | sempre (`core` la linka) |
| **llmmotor** | motori di inferenza: MLX (+ metallib) e llama.cpp | solo per l'inferenza locale |
| **torch** (external) | `libtorch` da pytorch.org | solo per TORCH/TRAIN |

## Stato per sistema operativo

Una **release per gruppo** (`gmb-<gruppo>-<data>`), con **un asset per RID** dentro.

| OS (RID) | support | core | engine | llmmotor | torch |
|---|---|---|---|---|---|
| macOS Apple Silicon (`osx-arm64`) | ✅ | ✅ | ✅ | ✅ (MLX + llama.cpp) | ✅ |
| Linux arm64 (`linux-arm64`) | ✅ | ✅ | ✅ | ✅ (llama.cpp) | — |
| Linux x86-64 (`linux-x64`) | ✅ | ✅ | ✅ | ✅ (llama.cpp) | ✅ |
| macOS Intel (`osx-x64`) | ⚠️ incompleto | ❌ | ❌ | ❌ | — |
| Windows x64 (`win-x64`) | ✅ (bundle storico) | ❌ | ❌ | ❌ | ✅ |

- ⚠️ **`osx-x64`**: il bundle `support` contiene solo 11 file e `libcrypto`/`libssl` in versione
  **arm64** → non è linkabile per Intel. Va ricostruito (OpenSSL/zstd/brotli/… per x86_64) prima
  di poter produrre `core`/`engine`/`llmmotor` per quel RID.
- ❌ **`win-x64`**: il runtime C usa API POSIX (`dlopen`, `fork/exec`, `pthread`, `m`); il port
  Win32 è un lavoro a sé. Oggi su Windows c'è solo il bundle storico di supporto.

## Requisiti Linux

- **glibc ≥ 2.38** → **Ubuntu 24.04+** (i bundle `support` Linux sono compilati lì).
- pacchetti: `libsdl2-2.0-0 libsdl2-ttf-2.0-0 libssl3 zlib1g libzstd1 libbz2-1.0 liblzma5
  liblz4-1 libbrotli1 libcurl4 libgomp1 libglib2.0-0 libgraphite2-3`
- il `support` Linux **non è autocontenuto** (a differenza del macOS): dipende da quei pacchetti.

## Requisiti macOS

Nessuno oltre al sistema: le dylib del bundle `support` sono linkate con `@rpath` e risolte dalla
cache (`~/.gmb/libs/<rid>/`).

## Regole di pubblicazione

1. **Un tag per gruppo e per versione** (`gmb-<gruppo>-<YYYY.MM.DD>`) con **un asset per RID**.
2. Un RID nuovo o ricostruito si carica nello **stesso** tag (`gh release upload --clobber`):
   mai un tag nuovo per lo stesso gruppo (spezzerebbe i RID su due tag).
3. Indice rigenerato **dalle release reali** con `GMBasic/tools/libs_manifest_gh.sh build` e
   pubblicato con `push` (mai a mano).
4. I file vanno prodotti con `GMBasic/tools/libs_build_release.sh <gruppo>` (nativo) o
   `libs_build_container.sh <gruppo> --rid linux-*` (container), che verificano la copertura
   rispetto all'indice e passano il gate `dlopen` (`libs_verify_load.sh`).

## Come si consumano

```bash
gmb libs status          # cosa c'è in cache per il RID corrente
gmb libs ensure          # scarica i gruppi mancanti (indice: libs-manifest.json)
gmb libs check           # verifica sha256 di ogni file
```
