# Known upstream build gaps (verify in your clone)

Checked against `tools/recomp/__main__.py`, `tools/recomp/translator.py`, `src/kernel/kernel_path.c`, `src/apu/apu_core.c`. Upstream may fix these — re-verify before blaming your game project.

---

## 1. `--gen-dir` vs `templates/new-game/`

**Default** (`tools/recomp/__main__.py` lines 87–89, 177–179):

```text
--gen-dir   (optional)
default:    <xboxrecomp>/src/game/recomp/gen
```

**Template** (`templates/new-game/CMakeLists.txt` line 45):

```cmake
file(GLOB RECOMP_GEN_SOURCES "src/recomp/gen/*.c")
```

**Fix:** Pass an explicit output dir when lifting so CMake globs match:

```bash
py -3 -m tools.recomp game_files/default.xbe --all --split 1000 --gen-dir src/recomp/gen
```

Use a path **relative to the xboxrecomp repo root** if you run recomp there, then copy `gen/` into your game repo. Or use `--gen-dir` with an absolute path into your game tree.

**Naming:** Split mode writes `recomp_0000.c`, `recomp_0001.c`, … (prefix `recomp`), not `gen_0000.c`. The glob `src/recomp/gen/*.c` still matches.

---

## 2. No separate `recomp_stubs.c` in split mode

**Docs** (`docs/GETTING_STARTED.md`, `tools/README.md`) list `recomp_stubs.c`.

**Actual split output** (`translator.py` `translate_batch_split`, lines 652–758):

| File | Generated? |
|------|----------------|
| `recomp_funcs.h` | Yes |
| `recomp_NNNN.c` | Yes (failed lifts become inline `/* FAILED */` stubs inside chunks) |
| `recomp_dispatch.c` | Yes |
| `recomp_stubs.c` | **No** — not created by current split path |

Do not add `recomp_stubs.c` to CMake unless you maintain it yourself. Remove it from game `CMakeLists.txt` if copied from older docs.

---

## 3. Missing `apu_xaudio2.h`

`src/apu/apu_core.c` line 24:

```c
#include "apu_xaudio2.h"
```

That header is **not** in `src/apu/` in the current tree (Windows build expects XAudio2 backend). **Symptom:** MSVC error C1083 on `xbox_apu`.

**Workarounds until upstream adds the file:**

- Build only the libs you need (if your CMake graph allows omitting `xbox_apu`).
- Stub or add the missing header/backend in a fork.
- Track xboxrecomp issues/PRs for APU Windows output.

---

## 4. `xbox_host_char` undefined on Win32

`src/kernel/kernel_path.c` line 107 (Win32 section):

```c
BOOL xbox_translate_path(const char* xbox_path, xbox_host_char* host_path_buf, DWORD buf_size)
```

Uses `WCHAR` / wide APIs in the body but **`xbox_host_char` is never typedef'd** in `kernel.h` or `platform/xbox_winnt.h` (Linux section uses `char` buffers at line 237).

**Symptom:** MSVC C2065 / unknown type `xbox_host_char` when compiling `xbox_kernel`.

**Workaround (game fork or local patch):** In `kernel.h` after includes:

```c
#if defined(_WIN32)
typedef WCHAR xbox_host_char;
#else
typedef char xbox_host_char;
#endif
```

---

## 5. `add_subdirectory(xboxrecomp)` and include paths

`src/kernel/CMakeLists.txt` line 29:

```cmake
target_include_directories(xbox_kernel PUBLIC ${CMAKE_SOURCE_DIR}/src)
```

When xboxrecomp is a **subdirectory of your game**, `CMAKE_SOURCE_DIR` is the **game** root, not the toolkit — `#include "platform/xbox_winnt.h"` fails.

**Workarounds:**

- Build xboxrecomp standalone first from its own root (`cmake -S . -B build`).
- In the **game** `CMakeLists.txt`, after `add_subdirectory`:

```cmake
target_include_directories(${PROJECT_NAME} PRIVATE ${XBOXRECOMP_DIR}/src)
```

- Or patch upstream to use `${CMAKE_CURRENT_LIST_DIR}/..` / generator expressions tied to the xboxrecomp tree.

---

## 6. Bring-up extras (not upstream bugs)

| Item | Note |
|------|------|
| `/MAP` | Not in `templates/new-game/CMakeLists.txt`; add for ICALL debugging (`docs/pipeline/06-debugging.md`) |
| Entry point | Template `main.c` may use `xbe_entry_point()` from labels; GETTING_STARTED uses `recomp_lookup(entry)` — match your pipeline |
