---
name: xboxrecomp
description: Guide static recompilation of Original Xbox (OG Xbox) games using the sp00nznet/xboxrecomp toolkit — XBE extraction, Python pipeline (xbe_parser, disasm, func_id, recomp), MSVC/CMake runtime linking, and iterative ICALL/kernel/D3D debugging. Use whenever the user mentions xboxrecomp, static recomp, OG Xbox native port, lifting XBE to C, RECOMP_ICALL, xbox_kernel, NV2A/D3D8 translation, Burnout 3 recomp, or building a native Windows exe from default.xbe — even if they only say "recompile my Xbox game" or "fix my recomp crash."
compatibility: Requires Python 3.10+ (capstone), CMake 3.20+, MSVC 2022 or GCC/Clang (Linux OpenGL path). User must own game discs; no copyrighted XBEs in repos.
---

# Xbox Static Recompilation (xboxrecomp)

Help users run the **[xboxrecomp](https://github.com/sp00nznet/xboxrecomp)** pipeline: retail `default.xbe` → generated C → native `.exe` linked against `xbox_kernel`, `xbox_d3d8`, `xbox_dsound`, `xbox_apu`, `xbox_nv2a`, `xbox_input`.

**Upstream source of truth:** If a local clone exists (e.g. `xboxrecomp/` next to the game project), read its `README.md`, `docs/`, and `tools/README.md` before advising. Do not invent tool flags or APIs — verify in that tree.

**Game-specific work** lives in a separate game repo (often with a `code/` folder for original/reference game sources). The toolkit repo is generic; per-game overrides go in `recomp_manual.c`, asset loaders, and notes — not in upstream xboxrecomp unless contributing back.

---

## Mental model

| Layer | What it is |
|-------|------------|
| **Toolchain** (`tools/`) | Python: parse XBE → disasm → classify → lift x86→C |
| **Generated code** (`src/.../recomp/gen/`) | Mechanical C; `void func(void)` + global register model |
| **Runtime** (`src/` libs) | Kernel, D3D8→D3D11/OpenGL, audio, NV2A, input — link-time, not emulation |
| **Game project** | `main.c`, memory layout, manual overrides, CMake linking `xboxrecomp` |

Static recomp ≠ emulator: recompiled functions run as native code. Crashes during bring-up are **expected**; progress is iterative stub/fix cycles.

---

## Prerequisites (verify before pipeline)

- **Windows 10/11** (D3D11) or **Linux** (`tools/linux/install_deps.sh`, OpenGL backend)
- `pip install capstone` (Python 3.10+; on Windows use `py -3`)
- **CMake 3.20+**, **VS 2022** C++ workload (or GCC/Clang on Linux)
- Owned disc image → extract `default.xbe` + assets ([extract-xiso](https://github.com/XboxDev/extract-xiso) or [xdvdfs](https://github.com/antangelo/xdvdfs))

Optional: **xemu** + GDB for reference behavior; **Ghidra** via `tools/ghidra_naming/` for CRT/XDK symbol names (never required).

---

## Standard pipeline (run in order)

All commands assume repo root and `game_files/default.xbe` exists. **Verify each `-m tools.*` entry point in the local clone** (`tools/<name>/__main__.py`); if missing, use the script path documented in `references/tool-commands.md`.

```bash
# 0. Build runtime libraries once
cmake -S . -B build
cmake --build build --config Release

# 1. Parse XBE — record entry point, sections, kernel imports for xbox_memory_layout.h
py -3 -m tools.xbe_parser game_files/default.xbe
# Fallback when tools/xbe_parser/__main__.py is absent (common in current tree):
# py -3 tools/xbe_parser/xbe_parser.py game_files/default.xbe

# 2. Disassemble — functions.json, xrefs.json
py -3 -m tools.disasm game_files/default.xbe --text-only -v

# 3. Classify CRT / RW / XDK / GAME (needs disasm output)
py -3 -m tools.func_id game_files/default.xbe -v

# 4. Lift to C (minutes for large games; needs disasm + func_id)
py -3 -m tools.recomp game_files/default.xbe --all --split 1000 --gen-dir src/recomp/gen
```

**`--gen-dir`:** Required when using `templates/new-game/` — that template globs `src/recomp/gen/*.c`, but the recomp default is `src/game/recomp/gen/` (`tools/recomp/__main__.py` `--gen-dir` help text). Run from the xboxrecomp root with `--gen-dir src/recomp/gen`, then copy into your game repo if needed.

### Expected artifacts (default paths)

| Step | Output location | Key files |
|------|-----------------|-----------|
| Parse | stdout (no default folder) | Entry point, sections, kernel ordinals; optional `--json` |
| Disasm | `tools/disasm/output/` | `functions.json`, `xrefs.json`, `strings.json`, `summary.json` |
| func_id | `tools/func_id/output/` | `identified_functions.json`, `crt_functions.json`, `rw_modules.json`, `summary.json` |
| recomp | `--gen-dir` or default `src/game/recomp/gen/` | `recomp_0000.c`…, `recomp_dispatch.c`, `recomp_funcs.h`; **`recomp_stubs.c` is not emitted** in split mode (failed lifts are inline stubs in chunk `.c` files per `tools/recomp/translator.py`) |

Optional: `py -3 -m tools.disasm ... --seed-functions` for stripped XBEs (`--seed-functions` in README).

Then **game project**: copy `templates/new-game/`, set `XBOXRECOMP_DIR`, glob `src/recomp/gen/*.c`, link `xboxrecomp`, implement `main.c` (load XBE, `xbox_MemoryLayoutInit`, kernel + D3D init, `recomp_lookup(entry_point)()`).

Full walkthrough: `docs/GETTING_STARTED.md`. Tool flags: `tools/README.md`, `references/pipeline-quickref.md`.

---

## New game project scaffold

1. `mkdir my_game && cd my_game && git init`
2. Copy **`templates/new-game/`** wholesale: `CMakeLists.txt`, `src/main.c`, **`src/recomp_manual.c`**, `.gitignore`
3. Set **`XBOXRECOMP_DIR`** → `add_subdirectory(${XBOXRECOMP_DIR} ...)`; link **`xboxrecomp`** (umbrella for all six runtime libs)
4. Copy **`templates/runtime/`** into `src/recomp/`: `recomp_types.h` (registers + `RECOMP_ICALL`), `xbox_memory.h`, `kernel_stubs.h`
5. Run recomp with **`--gen-dir src/recomp/gen`** so output matches the template glob (see `references/upstream-build-gaps.md`)
6. Template also links Win32: `d3d11`, `dxgi`, `dxguid`, `xinput`, `winmm`; MSVC `/bigobj`, `/LARGEADDRESSAWARE`; add **`/MAP`** for ICALL debugging (not in template by default)
7. Configure **entry point** and **section VAs** from xbe_parser output
8. After `add_subdirectory(${XBOXRECOMP_DIR})`, if includes fail, add `target_include_directories(${PROJECT_NAME} PRIVATE ${XBOXRECOMP_DIR}/src)` — kernel uses `${CMAKE_SOURCE_DIR}/src`, which points at the **game** root when nested (`src/kernel/CMakeLists.txt`)

**RenderWare games:** RW is lifted game code, not a separate link library. Run `func_id` for RW classification; expect many overrides in **`recomp_manual.c`** (vtable/ICALL). Optional game-only `rw_bridge.c` per template comment.

Integration callbacks (from README / `include/xbox/`):

```c
typedef void (*recomp_func_t)(void);
recomp_func_t recomp_lookup(uint32_t xbox_va);
recomp_func_t recomp_lookup_manual(uint32_t xbox_va);
```

---

## Iterative debugging (default mode after first run)

Read `references/debugging-quickref.md` and, for depth, `docs/pipeline/06-debugging.md` + `docs/technical/indirect-calls.md`.

**Loop:** run → read stderr → classify failure → fix → rebuild.

| Symptom | Likely cause | Fix direction |
|---------|--------------|---------------|
| ICALL unknown VA | Missing dispatch / garbage vtable | See ICALL workflow below |
| `0xFD...` access | GPU MMIO | NV2A init / VEH |
| `0xFE...` access | APU MMIO | APU init |
| `[KERNEL] Unimplemented ordinal` | Missing kernel thunk | Implement in `src/kernel/`, register in `kernel_thunks.c` |
| Stack overflow | Bad ESP / recursion | Entry stack setup, check prologue/epilogue |
| Infinite loop | Waiting on hardware | Stub wait or fake state |

**Always prefer `src/recomp_manual.c`** (from `templates/new-game/`) over editing `gen/*.c` — regeneration wipes gen patches. Register overrides in **`recomp_lookup_manual()`** so `RECOMP_ICALL` hits tier 1. Use `#if 0` around gen functions when replacing.

### ICALL FAIL workflow (MAP + garbage VA)

When stderr shows `ICALL FAIL: VA=0x........ caller=0x00007FF6........`:

1. **Caller → function:** Build with `/MAP` (`MapFile=true`). Search the `.map` for the symbol containing **`caller`** (native return address inside the recompiled function) → name like **`sub_001B4170`** (`docs/pipeline/06-debugging.md`).
2. **Trace context:** Inspect **`g_icall_trace[0..15]`** — failing VA is usually newest; prior entries show what ran before (`docs/pipeline/06-debugging.md`).
3. **Classify the VA:**
   - **Garbage** (e.g. **`0x1C45BA68`** — documented in `docs/technical/indirect-calls.md` as corrupted vtable, not a missing lift): confirm **`RECOMP_ICALL`** range guard in `templates/runtime/recomp_types.h` drops `[0x00400000, 0xFE000000)`; add **per-function vtable guard** in the MAP-identified `sub_*`; trace object init — do not add dispatch entries for garbage.
   - **Valid code VA, not in table:** extend dispatch or add **`recomp_manual.c`** stub/override.
4. **Kernel range `0xFE000000+`:** tier-3 `recomp_lookup_kernel`, not manual game stubs.

**ICALL trace:** ring buffer `g_icall_trace[]` + stderr; template `recomp_manual.c` can dump the ring — add `caller=` logging if missing for MAP workflow.

---

## When answering user questions

1. **Identify phase:** extraction, pipeline, CMake/build, first boot, ICALL, kernel, D3D/rendering, audio, input.
2. **Cite upstream docs** with path under the xboxrecomp clone (e.g. `docs/technical/memory-layout.md`).
3. **Cite game code** when the user has a `code/` or game repo — compare addresses, symbols, and behavior to lifted C; do not guess game logic.
4. **Runtime gaps:** check `docs/technical/gap-analysis.md` vs xemu before claiming something is implemented.
5. **Legal:** user must own the game; no distributing XBEs or assets.

---

## Runtime libraries (link-time)

| Library | Role |
|---------|------|
| `xbox_kernel` | 147+ kernel imports → Win32 |
| `xbox_d3d8` | D3D8 FFP, combiners, NV2A VS, unswizzle → D3D11/OpenGL |
| `xbox_dsound` | DirectSound compat |
| `xbox_apu` | MCPX APU (from xemu) |
| `xbox_nv2a` | GPU MMIO, push buffer, PGRAPH |
| `xbox_input` | XInput mapping |

Per-module READMEs: `src/kernel/README.md`, `src/d3d/README.md`, etc.

---

## Known upstream build gaps (MSVC / nested CMake)

Before blaming your game project, read **`references/upstream-build-gaps.md`** (verified against the clone). Summary:

| Issue | Where | Symptom |
|-------|--------|---------|
| Missing `apu_xaudio2.h` | `src/apu/apu_core.c:24` | `xbox_apu` compile fails |
| `xbox_host_char` not defined | `src/kernel/kernel_path.c:107` (Win32) | `xbox_kernel` compile fails |
| Wrong `CMAKE_SOURCE_DIR` includes | `src/kernel/CMakeLists.txt:29` | `platform/xbox_winnt.h` not found when xboxrecomp is a subdirectory |
| Doc vs tool: `recomp_stubs.c` | `translator.py` split output | Harmless if CMake does not list missing file |

Building **from the xboxrecomp root** (`cmake -S . -B build`) is the simplest path to validate the toolkit; game repos often need the include-dir workaround above.

---

## Contributing upstream

See `CONTRIBUTING.md`: new games, kernel ordinals (366 exports, ~147 implemented), lifter edge cases, format docs. Kernel wiki: [Xbox Dev Wiki Kernel](https://xboxdevwiki.net/Kernel).

---

## Bundled references (read as needed)

| File | Use when |
|------|----------|
| `references/docs-index.md` | Full map of upstream `docs/` |
| `references/tool-commands.md` | CLI flags for all pipeline tools |
| `references/pipeline-quickref.md` | Step checklist + outputs |
| `references/debugging-quickref.md` | Crash patterns + ICALL workflow |
| `references/upstream-build-gaps.md` | `--gen-dir`, stub output, MSVC/subdir build failures |

---

## Related projects (not this toolkit)

- **Xbox 360:** XenonRecomp (PowerPC) — wrong ISA for OG Xbox
- **Emulators:** xemu, Cxbx-Reloaded — use for reference/debug, not replacement for static recomp goal
- **Reference targets in README:** Burnout 3 (playable), Dashboard, Wreckless, Blood Wake
