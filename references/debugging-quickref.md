# Debugging quick reference

## ICALL failure

```
ICALL FAIL: VA=0x1C45BA68 caller=0x00007FF6A1234567
```

### MAP lookup (find who called)

1. Build Release with **`.map`** (`/MAP` or `MapFile=true`).
2. Take **`caller=`** from stderr (native address inside your `.exe`).
3. Search the map for the symbol whose range contains `caller` → **`sub_XXXXXXXX`** (Xbox VA in the name).
4. Open that `sub_*` in `src/recomp/gen/recomp_*.c` and find `RECOMP_ICALL(...)` sites.

Source: `docs/pipeline/06-debugging.md`, `docs/technical/lessons-learned.md`.

### Classify the VA

| VA pattern | Meaning | First fix |
|------------|---------|-----------|
| `0x1C45BA68` and similar high/non-code | **Garbage vtable** (documented example in `docs/technical/indirect-calls.md`) | Range guard in `recomp_types.h`; per-function guard in MAP-identified `sub_*`; fix init — **not** dispatch table |
| Valid code range, missing symbol | Missing lift/dispatch | `recomp_manual.c` or dispatch |
| `0xFE000000+` | Kernel thunk | `recomp_lookup_kernel` / implement ordinal |

### Fix order for garbage ICALLs

1. Confirm centralized **`RECOMP_ICALL`** early-out for `[0x00400000, 0xFE000000)` (`templates/runtime/recomp_types.h`).
2. Guard the specific caller `sub_*` (skip ICALL when pointer invalid).
3. Trace **object construction / init order** and memory layout.

### Trace ring buffer

Inspect **`g_icall_trace[0..15]`** on crash — newest entry is usually the bad VA; prior entries are context.

### Manual override (`src/recomp_manual.c`)

```c
void sub_001A3F50(void) {
    g_eax = 1;
    g_esp += 4;
    return;
}
```

Wire through **`recomp_lookup_manual()`** in the same file (from `templates/new-game/src/recomp_manual.c`). Do not patch `gen/*.c` if you will re-run `tools.recomp`.

---

## Memory fault patterns

| Address range | Likely issue |
|---------------|--------------|
| NULL / low | Uninit pointer |
| `0x00010000`–`0x00780000` | Xbox RAM — mapping or bad global |
| `0x00780000`–`0x00F80000` | Stack — ESP corruption |
| `0xFD000000+` | NV2A MMIO — need GPU init/VEH |
| `0xFE000000+` | APU MMIO |
| `0xCDCDCDCD` / `0xDDDDDDDD` | MSVC heap debug fill |

## Kernel

```
[KERNEL] Unimplemented ordinal XXX
```

→ `docs/formats/kernel-exports.md`, implement in `src/kernel/`, register in `kernel_thunks.c`.

## Build tips

- `/w` on generated sources (noise)
- `/bigobj`, `/LARGEADDRESSAWARE` for large gen files
- `/MAP` for native caller → recompiled symbol

## Reference debugging

- **xemu** GDB stub: compare memory and execution at same VA
- **gap-analysis.md**: don't assume full hardware coverage

## 80/20 rule

~80% of lifted functions run without edits; time goes to ICALLs, kernel gaps, D3D state, and **`recomp_manual.c`** overrides.
