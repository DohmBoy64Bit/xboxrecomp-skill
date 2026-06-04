# xboxrecomp (Cursor Agent Skill)

Agent skill for **[sp00nznet/xboxrecomp](https://github.com/sp00nznet/xboxrecomp)** — static recompilation of Original Xbox games from `default.xbe` to native C + link-time runtime libraries.

This skill is a **playbook and reference layer** for the AI agent. It does **not** include the xboxrecomp repository, Python tools, or game binaries.

---

## What you get

| Included in the skill | Not included (you provide separately) |
|----------------------|----------------------------------------|
| `SKILL.md` — pipeline, CMake scaffold, ICALL debugging | Clone of [xboxrecomp](https://github.com/sp00nznet/xboxrecomp) |
| `references/` — tool commands, build gaps, doc index | `game_files/default.xbe` and disc assets (you must own the game) |
| Eval prompts under `evals/` (development only) | Python, CMake, MSVC (or Linux toolchain), `capstone` |

---

## Requirements

- **Cursor** (or compatible agent) with skills support
- **Git clone** of xboxrecomp on disk (e.g. next to your game project)
- **Python 3.10+** — `pip install capstone`
- **CMake 3.20+** and **VS 2022** C++ workload (Windows) or Linux deps per upstream `tools/linux/install_deps.sh`
- Tool to extract ISO → `default.xbe` ([extract-xiso](https://github.com/XboxDev/extract-xiso), [xdvdfs](https://github.com/antangelo/xdvdfs))

See `references/upstream-build-gaps.md` for known MSVC/subdirectory CMake issues in the current upstream tree.

---

## Installation

### Option 1: Skill folder

Copy or symlink this directory to a skills path Cursor reads:

| Scope | Path |
|-------|------|
| Personal (all projects) | `~/.cursor/skills/xboxrecomp/` |
| Agents (Codex-style) | `~/.agents/skills/xboxrecomp/` |
| Project (repo-only) | `.cursor/skills/xboxrecomp/` |

The folder must contain `SKILL.md` at its root.

### Option 2: Download `xboxrecomp.skill` (recommended for install)

**[GitHub Releases](https://github.com/DohmBoy64Bit/xboxrecomp-skill/releases)** attach a pre-built **`xboxrecomp.skill`** file.

| What it is | What it is for |
|------------|----------------|
| A **ZIP archive** (`.skill` extension) of this skill folder | **One-file install** in Cursor — import/install the skill without cloning this repo |
| Contains `SKILL.md` + `references/` | Gives the agent the xboxrecomp playbook and quickrefs |
| Does **not** contain the [xboxrecomp toolkit](https://github.com/sp00nznet/xboxrecomp) | You still clone upstream separately for Python tools and C runtime |

**Steps:**

1. Open [Releases](https://github.com/DohmBoy64Bit/xboxrecomp-skill/releases) and download **`xboxrecomp.skill`** from the latest tag.
2. Install it through Cursor’s skill import UI (or unpack into `~/.cursor/skills/xboxrecomp/` if you manage skills manually).
3. Clone [sp00nznet/xboxrecomp](https://github.com/sp00nznet/xboxrecomp) and follow **Clone the toolkit** below.

`evals/` is omitted from the package (development-only test prompts).

### Option 3: Build `.skill` yourself

From [Anthropic skill-creator](https://github.com/anthropics/skills) `package_skill.py`, package a clean tree (no `.git`):

```bash
python -m scripts.package_skill path/to/xboxrecomp path/to/output
```

Produces `xboxrecomp.skill`. The packager skips `evals/` by default.

---

## Clone the toolkit (required)

```bash
git clone https://github.com/sp00nznet/xboxrecomp.git
cd xboxrecomp
pip install capstone
cmake -S . -B build
cmake --build build --config Release
```

Place your extracted game under `game_files/default.xbe` (and assets) in that tree or in a separate game repo.

---

## How to use in Cursor

1. Open a workspace that contains (or can reference) your xboxrecomp clone.
2. Start an **Agent** chat.
3. **Attach** the `xboxrecomp` skill or ask a task that matches its description (pipeline, ICALL crash, new recomp project, D3D/kernel bring-up).
4. Give a concrete goal, for example:
   - *“Run the Python pipeline for `game_files/default.xbe` and list output folders.”*
   - *“ICALL FAIL VA=0x1C45BA68 — I have a MAP file.”*
   - *“Scaffold a RenderWare game from `templates/new-game/`.”*

The agent should read **your local clone** (`README.md`, `docs/`, `tools/`) and the skill’s `references/` — not invent APIs.

### Typical pipeline (run from xboxrecomp root)

```bash
# Parse (use script if -m tools.xbe_parser has no __main__.py)
py -3 tools/xbe_parser/xbe_parser.py game_files/default.xbe

py -3 -m tools.disasm game_files/default.xbe --text-only -v
py -3 -m tools.func_id game_files/default.xbe -v
py -3 -m tools.recomp game_files/default.xbe --all --split 1000 --gen-dir src/recomp/gen
```

Use `--gen-dir src/recomp/gen` when following `templates/new-game/` (default recomp output is `src/game/recomp/gen/`).

### Typical game project

1. Copy `templates/new-game/` from the clone.
2. Set `XBOXRECOMP_DIR` to the xboxrecomp path; link target `xboxrecomp`.
3. Copy lifted `gen/*.c` and `templates/runtime/*.h` into `src/recomp/`.
4. Iterate on crashes — prefer `src/recomp_manual.c` over editing generated `gen/*.c`.

---

## Skill layout

```
xboxrecomp/
├── README.md                 # This file
├── SKILL.md                  # Agent instructions (required)
├── evals/
│   └── evals.json            # Skill-creator test prompts (optional)
└── references/
    ├── docs-index.md         # Map to upstream docs/
    ├── tool-commands.md      # CLI quick reference
    ├── pipeline-quickref.md  # Step checklist
    ├── debugging-quickref.md # ICALL / crash patterns
    └── upstream-build-gaps.md # gen-dir, stubs, MSVC gaps
```

---

## Skill vs `.skill` vs upstream repo

| Artifact | Purpose |
|----------|---------|
| **This Git repo** | Source for `SKILL.md`, references, README; clone to contribute |
| **Release `xboxrecomp.skill`** | Pre-built ZIP for Cursor install — no clone of this repo required |
| **Skill folder** (after install or git clone) | Live copy the agent reads; edit and reinstall as needed |
| **[sp00nznet/xboxrecomp](https://github.com/sp00nznet/xboxrecomp)** | Toolkit source of truth — tools, runtime, templates, full docs (always separate) |

The skill’s `description` in `SKILL.md` frontmatter controls when the agent auto-selects this skill. The body loads when the skill is invoked.

---

## Development / evals

This skill was built with the **skill-creator** workflow. Test prompts live in `evals/evals.json`. Benchmark runs can be stored under `xboxrecomp-workspace/iteration-N/` (not shipped in `.skill`).

To improve the skill: edit `SKILL.md` or `references/`, re-run evals, repackage with `package_skill.py`.

---

## License and legal

- Skill text: follow your project policy; upstream xboxrecomp is **MIT**.
- You must **own** any game you extract and recompile. Do not distribute copyrighted XBEs or assets via the skill or your repos.

---

## Links

- **Releases (`.skill` download):** https://github.com/DohmBoy64Bit/xboxrecomp-skill/releases  
- Upstream toolkit: https://github.com/sp00nznet/xboxrecomp  
- Xbox Dev Wiki (kernel, XBE): https://xboxdevwiki.net/  
