# Pipeline quick reference

## Artifacts per step

| Step | Command | Key outputs |
|------|---------|-------------|
| Extract | extract-xiso / xdvdfs | `game_files/default.xbe`, assets |
| Parse | `xbe_parser` (see script fallback in tool-commands.md) | stdout; optional `--json` |
| Disasm | `tools.disasm --text-only -v` | `tools/disasm/output/functions.json`, `xrefs.json`, … |
| Identify | `tools.func_id -v` | `tools/func_id/output/identified_functions.json`, … |
| Lift | `tools.recomp --all --split 1000 --gen-dir src/recomp/gen` | `recomp_*.c`, `recomp_funcs.h`, `recomp_dispatch.c` (no separate stubs file) |
| Integrate | `templates/new-game/` + link `xboxrecomp` | `my_game.exe`; see `upstream-build-gaps.md` if MSVC fails |

## Typical scale

- Functions: ~10k–25k per game
- Lift time: ~5–15 min large .text
- Disasm: ~30–60 s for ~3 MB .text

## Register / call model (lifted C)

- Globals: `g_eax`, `g_ecx`, `g_edx`, `g_esp`, `g_ebx`, `g_esi`, `g_edi`
- Memory: `MEM32(xbox_va)` in 64 MB mapped space
- Direct call: `sub_00XXXXXX();`
- Indirect: `RECOMP_ICALL(va)` → manual lookup → dispatch → kernel

## Boot phases (debug order)

1. CRT / entry
2. Hardware init (D3D, sound, input)
3. Asset load (file I/O)
4. Menus / render loop
5. Gameplay

Stub early blockers; fix ICALLs and kernel ordinals as stderr reveals them.

## Regeneration warning

Re-running `tools.recomp` overwrites `gen/*.c`. Keep fixes in `recomp_manual.c` or a patch log.
