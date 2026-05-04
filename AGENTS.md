# jsc2js — AGENTS.md

## Repository purpose
Converts V8 JSC bytecode back to approximate JavaScript. Two components:
- **Root-level scripts + GitHub Actions**: Build patched `d8` binaries (with `loadjsc()` builtin) for multiple V8 versions
- **View8/**: Standalone decompiler that converts bytecode text dumps to JS

## Quick usage
```bash
# 1. Disassemble bytecode with patched d8
./d8 -e "loadjsc('path/to/xxx.jsc')" > xxx.txt

# 2. Decompile to JS
cd View8
python view8.py --disassembled ../xxx.txt ../xxx.js
```

## Python dependencies
- `parse` library (used in `Parser/sfi_file_parser.py:from parse import parse` — not a stdlib module)
- No `requirements.txt` exists; install manually: `pip install parse`

## Encoding gotchas (Windows)
- View8 code uses `str.removeprefix()` (Python 3.9+ only)
- CI sets `PYTHONUTF8=1` and `PYTHONIOENCODING=UTF-8` to avoid GBK decode errors on Windows
- Output files force UTF-8 with `newline='\n'`

## Build infrastructure (root scripts)
| File | Role |
|------|------|
| `determine_versions.py` | Fetches V8 versions from Node.js/Electron releases, computes unprocessed batch |
| `apply_patch.py` | Cross-platform 3-way patch applier with auto-conflict resolution |
| `build_versions_batch.py` | Orchestrates parallel d8 builds per V8 version slot |
| `partition_versions.py` | Splits version list into CI matrix slots |

Patch files (`patch.diff`, `patch_v2.diff`, `patch_v3.diff`, etc.) correspond to different V8 version branches. `compile.yml` uses `patch_v3.diff`, `patch_1_v3.diff`, `patch_old_v3.diff`.

## Decompiler architecture (View8/)
```
view8.py              # CLI entry point
Parser/
  sfi_file_parser.py  # Parses d8 textual output into SharedFunctionInfo objects
  shared_function_info.py  # Per-function bytecode + const pool + exception table
Translate/
  translate.py        # Bytecode → intermediate representation
  translate_table.py  # Opcode operand definitions
  jump_blocks.py      # Control flow analysis
Simplify/
  simplify.py         # IR simplification passes
  global_scope_replace.py  # Variable name resolution
  function_context_stack.py  # Scope tracking
```

## CI workflows
- `main.yml`: Scheduled batch build (cron 2am) — determines versions, builds across 6 slots × 2 OSes, creates GitHub Releases
- `compile.yml`: Manual single-version build via `workflow_dispatch`
- `sync.yml`, `cleanup.yml`, `update_*`: Support workflows