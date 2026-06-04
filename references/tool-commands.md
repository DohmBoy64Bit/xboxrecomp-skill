# xboxrecomp tool commands (quick reference)

Run from **xboxrecomp repo root**. Windows: `py -3`; Linux/macOS: `python3`.

## Prerequisites

```bash
pip install capstone
```

## 1. xbe_parser

```bash
py -3 -m tools.xbe_parser game_files/default.xbe
```

**If `-m tools.xbe_parser` fails** (`No module named tools.xbe_parser.__main__`), use the script directly (verified in clones without `__main__.py`):

```bash
py -3 tools/xbe_parser/xbe_parser.py game_files/default.xbe
py -3 tools/xbe_parser/xbe_parser.py game_files/default.xbe --json tools/xbe_parser/xbe_analysis.json
py -3 tools/xbe_parser/xbe_parser.py game_files/default.xbe --extract-sections game_files/sections/
```

**Stdout:** entry point, XDK version, sections (VA/size/raw offset), kernel import ordinals. No default output directory.

**Use for:** `xbox_memory_layout.h`, entry point in `main.c`.

---

## 2. disasm

```bash
py -3 -m tools.disasm game_files/default.xbe --text-only [-v]
py -3 -m tools.disasm game_files/default.xbe --text-only --seed-functions ...  # iterative / stripped XBEs
```

| Flag | Effect |
|------|--------|
| `--text-only` | Analyze .text only |
| `--output-dir DIR` | Default `tools/disasm/output/` |
| `-v` / `--verbose` | Progress |

**Outputs** (`tools/disasm/output/` by default): `functions.json`, `xrefs.json`, `strings.json`, `labels.json`, `summary.json`

---

## 3. func_id

```bash
py -3 -m tools.func_id game_files/default.xbe [-v]
```

| Flag | Effect |
|------|--------|
| `--output-dir DIR` | Default `tools/func_id/output/` |

**Outputs** (`tools/func_id/output/`): `identified_functions.json`, `crt_functions.json`, `rw_modules.json`, `summary.json`

**Categories:** CRT, RW (RenderWare), XDK, GAME, STUB

---

## 4. recomp

```bash
# Default gen dir: src/game/recomp/gen (under xboxrecomp root)
py -3 -m tools.recomp game_files/default.xbe --all --split 1000

# Match templates/new-game/ (glob src/recomp/gen/*.c):
py -3 -m tools.recomp game_files/default.xbe --all --split 1000 --gen-dir src/recomp/gen

py -3 -m tools.recomp game_files/default.xbe --func 0x00123456   # single function
```

| Flag | Effect |
|------|--------|
| `--all` | All detected functions |
| `--split N` | N functions per output .c file |
| `--gen-dir DIR` | Split output directory (default: `src/game/recomp/gen`) |
| `--output-dir DIR` | Non-split / header mode (`tools/recomp/output/`) |
| `--verbose` | Per-function progress |

**Split outputs** (`translator.py` `translate_batch_split`): `recomp_funcs.h`, `recomp_0000.c`, …, `recomp_dispatch.c`. **No** `recomp_stubs.c` — failed functions get inline stubs inside chunk files. See `references/upstream-build-gaps.md`.

---

## 5. xmv (optional)

```bash
py -3 -m tools.xmv game_files/Video/intro.xmv
```

Demux Xbox Media Video for FFmpeg conversion.

---

## 6. ghidra_naming (optional)

```bash
XBE=/path/to/default.xbe tools/ghidra_naming/run_ghidra.sh
py -3 tools/ghidra_naming/merge_names.py --apply
```

Merges recovered CRT/XDK names into `functions.json`. Set `GHIDRA_HOME` if needed.

---

## Build runtime libs

```bash
cmake -S . -B build
cmake --build build --config Release
```

Artifacts under `build/src/*/Release/` (or platform equivalent). Link target: `xboxrecomp` umbrella.
