# GMBasicModules

Prebuilt native libraries for **GMBasic**, published as GitHub releases and downloaded on demand.

Nothing here is vendored in the compiler repository: the compiler, the C# app and every compiled
BASIC program fetch exactly the file they need, verify its SHA-256 and cache it locally.

Two machine-readable indexes describe what exists:

| index | shape | where |
|---|---|---|
| `libs-manifest.json` | structured: per RID → per file (`group`, `sha256`, `size`, `url`) | repo root (published on `main`) and in the consumer checkout under `shared/settings/` |
| `index-<rid>.txt` | flat TSV, one line per file: `name⇥sha256⇥size⇥url[⇥scope]` | rolling release **`libs-index`** |

The flat index starts with a header line stating the required versions, e.g.

```
# requires runtime-v1 llmmotor-v1 dev-v1
```

---

## Model at a glance

* **One release per group and per version** — `runtime-v1`, `dev-v1`, `llmmotor-v1`, `support-v1`.
* **One asset per library and per RID** inside that release: `<stem>-<rid><ext>`.
  A download is therefore a single HTTP request, and a single library can be updated alone.
* **Version tags are immutable.** An asset is never overwritten with different bytes; a change
  ships as `-v2` and the old tag stays available forever.
* **The cache is the only install location**: `~/.gmbasic/libs/<rid>/`.

## Groups

| Group | Contents | Needed for |
|---|---|---|
| **`runtime`** | boot loader (`libgmb_boot`), our C libraries (`libgmb_gfx`, `libgmb_screen`, `libgmb_pkg`, …), AI plumbing (`libllmcodec_*`, `libllmtoken`), SQL extensions, **and all third-party runtime dependencies** (SDL2, OpenSSL, curl, zstd/brotli/lz4/lzma, png/jpeg/tiff/webp, hdf5, tesseract, MoltenVK, …) | running any program that touches graphics, network, compression, AI or SQL |
| **`dev`** | third-party **headers** (`include/`) and our **static libraries** (`lib/*.a`) | compiling/linking without Homebrew or system dev packages |
| **`llmmotor`** | inference engines: MLX (+ `mlx.metallib`) and llama.cpp / ggml | local LLM inference only |
| **`support`** | third-party libraries alone | RIDs where `runtime` is not published yet |
| **`torch`** | `libtorch` — **external**, hosted by pytorch.org | `TORCH` / `TRAIN` |

`runtime` is deliberately a *single* closed group: `core` needs `engine`, and our runtimes link
third-party libraries, so splitting them produced installs that were complete on paper but did not
run. `dev` and `llmmotor` stay separate because they are large and optional.

## Asset naming

**Flat files** — the RID sits immediately before the extension:

```
libgmb_gfx-osx-arm64.dylib         libgmb_gfx-linux-arm64.so
libzstd.1.5.7-osx-arm64.dylib      libllama-linux-x64.so.0
mlx-osx-arm64.metallib             SDL2-win-x64.dll
```

**Tree files (`dev`)** — the relative path is encoded in the asset name with `__`, and the RID
still comes before the extension:

```
include__SDL2__SDL-osx-arm64.h      → include/SDL2/SDL.h
include__openssl__ssl-osx-arm64.h   → include/openssl/ssl.h
lib__libgmb_runtime-osx-arm64.a     → lib/libgmb_runtime.a
```

The manifest reverses the rule, so consumers always see real paths.

## Platform status

The table lists what is **published** (assets in the releases above). "Buildable" means the
source side is ready and can be cross-compiled locally, even if the bundle is not published yet.

| RID | `runtime` | `dev` | `llmmotor` | `support` | `ffi` | `torch` |
|---|---|---|---|---|---|---|
| `osx-arm64` (macOS Apple Silicon) | ✅ 94 files | ✅ 431 files | ✅ 5 files | ✅ (inside `runtime`) | ✅ 7 files | ✅ external |
| `linux-arm64` | — | — | — | ✅ 40 files | — | — |
| `linux-x64` | — | — | — | ✅ 20 files | — | ✅ external |
| `osx-x64` (macOS Intel) | ✅ 26 files | — | — | ⚠️ legacy 11 files | — | — |
| `win-x64` | ✅ 26 files | — | — | ✅ 11 files | — | ✅ external |
| `win-arm64` | ✅ 26 files | — | — | — | ✅ 6 files | — |

Notes:

* **`osx-x64`** — the runtime is published and **self-contained**: third-party libraries
  (OpenSSL, SDL2, freetype, zstd/brotli/lz4/lzma, pcre2, curl) are linked statically into our
  dylibs, so no separate assets are needed. The old `support` set for this RID is **legacy** and
  must not be used: its `libcrypto`/`libssl` are **arm64** and only the runtime bundle is
  linkable/runnable on Intel. `runtime` supersedes it.
* **Windows** — the runtime bundle is 26 files: our libraries (`libgmb_ai_llm`, `libgmb_boot`,
  `libgmb_compress`, `libgmb_gfx`, `libgmb_grep`, `libgmb_pkg`, `libgmb_screen`, `libgmb_tray`),
  the tokenizer (`libllmtoken`) and the 17 `libllmcodec_*` plugin codecs. Third-party
  dependencies (SDL2, OpenSSL, curl, zstd/brotli/lz4/lzma, freetype, pcre2) are linked
  **statically** into those DLLs, so they are not separate assets. `win-arm64` also publishes the
  six FFI libraries; `win-x64` also carries the older `support` set. Verified with
  `tools/c/pe_deps.py --check`: no non-system import is missing from the bundle.

### Cross-compiling (Windows and macOS Intel)

The Windows port of the C runtime is done and builds from macOS with **Zig** (no MSVC, no admin,
no VM). Two scripts in the monorepo:

```bash
# 1. third-party libraries for every target (zlib, sqlite3, zstd, lz4, brotli, lzma, pcre2,
#    openssl, freetype, SDL2, SDL2_ttf, curl) → ~/gmb-deps/<triple>/{include,lib}
tools/c/build-deps.sh --all-targets

# 2. the C runtime itself (CMake toolchain generated on the fly)
tools/c/build-windows.sh                        # win-arm64
tools/c/build-windows.sh --rid win-x64 --triple x86_64-windows-gnu
```

`build-windows.sh` produces `libgmb_runtime.a` plus the DLLs listed above, for `PE32+ Aarch64`
and `x86-64`. The bundles published here come from exactly those outputs:

```bash
tools/sh/libs_publish_files_gh.sh runtime win-arm64 --source <build>/bundle/win-arm64 --tag runtime-v1
tools/sh/libs_publish_files_gh.sh runtime win-x64   --source <build>/bundle/win-x64   --tag runtime-v1
python3 tools/c/pe_deps.py --check <build>/bundle/win-arm64    # 0 dipendenze mancanti
```

The binaries import only Windows API sets (`api-ms-win-crt-*`): on Windows 10/11 no Visual C++
redistributable is needed.

## System requirements

**macOS** — nothing beyond the OS. Every dylib is linked with `@rpath` and resolved from the cache.

**Linux** — `glibc ≥ 2.38`, i.e. **Ubuntu 24.04 or newer** (the Linux bundles are built there).
The `support` set is *not* self-contained: it relies on the distribution packages

```
libsdl2-2.0-0 libsdl2-ttf-2.0-0 libssl3 zlib1g libzstd1 libbz2-1.0 liblzma5
liblz4-1 libbrotli1 libcurl4 libgomp1 libglib2.0-0 libgraphite2-3
```

---

## Consuming the libraries

### From the compiler / app

```bash
gmb install --runtime-only     # what a program needs to RUN
gmb install --dev              # headers + static libs, to BUILD without Homebrew
gmb install --components       # torch / chromium / obscura / ocr (external downloads)
gmb install --all --verify     # everything, with SHA-256 verification
```

```bash
gmb libs status                # what is in the cache for the current RID
gmb libs check                 # verify every cached file against the manifest
```

Cache layout (a single root, no alternatives):

```
~/.gmbasic/libs/<rid>/                    flat library files (runtime, dev, support)
~/.gmbasic/libs/<component>/<rid>/        torch (ml/), chromium, obscura, ocr
~/.gmbasic/libs/.manifest.json            cached copy of libs-manifest.json
```

In a source checkout, our own build artifacts live in `build/libs/<rid>/` — never in the repo.

### From a compiled BASIC program

A compiled program is distributed **without a compiler**, so the build copies next to the
executable everything it needs:

```
<exe dir>/app.settings.json     runtime sections only (paths, consoleConfiguration, engine),
                                merged from the shared settings with the program's own overrides
<exe dir>/Resources/            fonts and sprites
<exe dir>/<exe>.libs.txt        the exact libraries it depends on: name⇥sha256⇥size⇥url
```

`<exe>.libs.txt` pins the version: the URLs point at an immutable tag, so the program keeps
resolving the same bytes even after a newer version is published.

Load-time dependencies cannot be downloaded by the program itself (`dyld`/`ld.so` fails before
`main()`), so a small launcher does it first:

```bash
gmb_libs_fetch --exec ./myprogram     # fetches missing libraries next to the exe, then execs it
```

The binaries keep only `@loader_path` (`$ORIGIN` on Linux) as an rpath, so the libraries placed
next to the executable are found on any machine, with or without a cache.

---

## Publishing (maintainers)

Bundles are produced and published by the scripts in the GMBasic monorepo
(`tools/sh/`, from `GMBasicMonoRepo`):

| script | purpose |
|---|---|
| `libs_build_release.sh <group>` | build + stage + coverage check + `dlopen` gate + publish |
| `libs_build_container.sh <group> --rid linux-*` | same pipeline inside Ubuntu (Linux bundles) |
| `libs_bundle_selfcheck.sh <dir> <rid>` | fails if any dependency is missing from the bundle |
| `libs_publish_files_gh.sh <group> <rid> --source DIR --tag <group>-vN` | uploads the files as per-file assets |
| `libs_manifest_gh.sh build --rid <rid> --groups …` / `push` | regenerates `libs-manifest.json` from the **real** releases |
| `libs_index_publish.sh --publish` | regenerates and publishes the flat `index-<rid>.txt` files |

Rules:

1. **Version tags are immutable.** The publisher compares SHA-256 against what is already
   published: identical → skip, different → **refuse** and tell you to publish `-vN+1`.
   `--clobber` is used only on the rolling `libs-index` release.
2. The manifest is regenerated **from the releases** (asset names + digests) and never edited by
   hand. A group that cannot be indexed is a **hard error**: the manifest is left untouched rather
   than written incomplete.
3. Each bundle must pass `libs_bundle_selfcheck.sh`: no dependency may point outside the bundle.

### A new version, step by step

```bash
# 1. build and publish the group for one RID
tools/sh/libs_build_release.sh runtime --rid osx-arm64

# 2. refresh the structured index (all files of the group for that RID)
tools/sh/libs_manifest_gh.sh build --rid osx-arm64 --groups runtime,llmmotor,dev
tools/sh/libs_manifest_gh.sh push

# 3. refresh the flat index used by the C loader and by compiled programs
tools/sh/libs_index_publish.sh --publish
```

## Licensing

The third-party libraries bundled here keep their own licenses (OpenSSL, SDL, zlib, brotli, lzma,
zstd, lz4, curl, libpng, libjpeg-turbo, libtiff, libwebp, harfbuzz, freetype, hdf5, tesseract,
MoltenVK, …). `libtorch` is downloaded from pytorch.org and is not redistributed here.
Our own libraries (`libgmb_*`, `libllm*`) are distributed under the licenses of the GMBasic
project; see the `LICENSE` / `NOTICE` files in the compiler repository.

## Note on the Windows codec plugins

The 17 `libllmcodec_*` assets for the Windows RIDs are **PE32+ DLLs** (verified with `file` and
by parsing their import table), even though their extension is `.so`: the plugin registry in
LLMEnv loads them through `dlopen`/`LoadLibrary` and historically used the `.so` suffix on every
platform. The loader now also accepts `.dll`, and publishes both RIDs' plugins either way.
