# xboxrecomp documentation index

Paths are relative to the **xboxrecomp repository root** (clone from https://github.com/sp00nznet/xboxrecomp). Read these in the local clone — do not rely on this index alone for API details.

## Start here

| Path | Topic |
|------|--------|
| `README.md` | Overview, pipeline diagram, runtime libs, FAQ |
| `docs/GETTING_STARTED.md` | End-to-end: extract XBE → run tools → CMake → debug |
| `tools/README.md` | All Python tools, flags, outputs |
| `CONTRIBUTING.md` | Dev setup, kernel imports, lifter work |

## Pipeline (`docs/pipeline/`)

| Path | Step |
|------|------|
| `01-xbe-parsing.md` | XBE format, extraction, entry point decode |
| `02-disassembly.md` | disasm tool, function detection |
| `03-function-id.md` | func_id classification |
| `04-lifting.md` | recomp x86→C |
| `05-runtime.md` | Runtime setup |
| `06-debugging.md` | ICALL trace, memory faults, iteration |

## Technical (`docs/technical/`)

| Path | Topic |
|------|--------|
| `register-model.md` | g_eax, g_esp, MEM32, void func(void) |
| `memory-layout.md` | CreateFileMapping, mirrors, 64MB layout |
| `indirect-calls.md` | RECOMP_ICALL, dispatch table, manual overrides |
| `kernel-replacement.md` | Kernel thunk → Win32 |
| `d3d-translation.md` | D3D8 → D3D11 |
| `d3d8ltcg-device-context.md` | LTCG device fields, push buffer |
| `seh-handling.md` | Structured exceptions in lifted code |
| `lessons-learned.md` | Pitfalls from real bring-ups |
| `gap-analysis.md` | vs xemu — what's missing |
| `candidate-games.md` | Good recomp targets |
| `xemu-debugging.md` | Using xemu as reference |

## Formats (`docs/formats/`)

| Path | Topic |
|------|--------|
| `xbe.md` | XBE header/sections reference |
| `kernel-exports.md` | 366 kernel functions |
| `disc-image.md` | XDVDFS extraction |

## Runtime (`docs/runtime/`)

| Path | Topic |
|------|--------|
| `README.md` | Four pillars: memory, kernel, D3D, input |

## Source module READMEs (`src/`)

| Path | Library |
|------|---------|
| `src/README.md` | Runtime overview |
| `src/kernel/README.md` | xbox_kernel |
| `src/d3d/README.md` | xbox_d3d8 |
| `src/audio/README.md` | xbox_dsound |
| `src/apu/README.md` | xbox_apu |
| `src/nv2a/README.md` | xbox_nv2a |
| `src/input/README.md` | xbox_input |

## Templates (`templates/`)

| Path | Purpose |
|------|---------|
| `templates/new-game/CMakeLists.txt` | Game project CMake |
| `templates/new-game/src/main.c` | Entry stub |
| `templates/new-game/src/recomp_manual.c` | Manual overrides |
| `templates/runtime/recomp_types.h` | Registers, ICALL, stack macros |
| `templates/runtime/xbox_memory.h` | Memory helpers |
| `templates/runtime/kernel_stubs.h` | Kernel stub patterns |

## Optional tools

| Path | Purpose |
|------|---------|
| `tools/ghidra_naming/README.md` | Headless Ghidra → merge names into functions.json |
| `tools/xmv` | XMV video demux (FMV) |

## Skill bundled references (`references/`)

| Path | Purpose |
|------|---------|
| `upstream-build-gaps.md` | `--gen-dir`, split stub output, `apu_xaudio2.h`, `xbox_host_char`, nested CMake |
| `tool-commands.md` | CLI quick reference |
| `pipeline-quickref.md` | Pipeline checklist |
| `debugging-quickref.md` | ICALL / crash patterns |
