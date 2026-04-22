# Cleanup Log: 2026-04-22

**Status:** All speculation and failed experiments removed. Project now contains only verified facts.

## What Was Cleaned

### Removed from oca-17 Project
- ❌ `odools-absolute.toml` — test file with hardcoded paths (no longer needed)
- ❌ `.opencode/` directory — test artifacts from failed LSP init attempts

### Simplified Configuration Files

**Before (oca-17/odools.toml):**
```toml
[Odoo]
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

[odoo]
addons_paths = [
  "${workspaceFolder}/odoo/custom/src/e-learning",
  "${workspaceFolder}/odoo/custom/src/field-service",
  ... (17 explicit paths)
  "${workspaceFolder}/odoo/custom/src/private",
]
```

**After (oca-17/odools.toml):**
```toml
[Odoo]
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

[odoo]
addons_paths = [
  "${workspaceFolder}/odoo/auto/addons",
]
```

**Before (oca-17/opencode.json):**
```json
{
  "lsp": {
    "pyright": { "disabled": true },
    "odoo-ls": {
      "command": ["odoo_ls_server", "--config-path", "/home/roly/projects/binhex/OCA/oca-17/odools-absolute.toml"],
      "extensions": [".py", ".xml"],
      "initialization": {
        "Odoo": {
          "selectedProfile": "default"
        }
      }
    }
  }
}
```

**After (oca-17/opencode.json):**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "lsp": {
    "pyright": { "disabled": true },
    "odoo-ls": {
      "command": ["odoo_ls_server"],
      "extensions": [".py", ".xml"]
    }
  }
}
```

### Updated Documentation

- ✅ `spec.md` — Removed references to "Phase 4 verified behavior" (was speculation)
- ✅ Created `VERIFIED_FACTS.md` — Separates 100% verified from speculative content
- ✅ `AGENTS.md` — Already correct; no changes needed

## What Stayed (Because It's Verified)

- ✅ Minimum viable config structure (two files)
- ✅ Variable expansion mechanism (works as documented)
- ✅ Pyright coexistence strategy (disable pyright)
- ✅ LSP diagnostics injection (verified working)
- ✅ Experimental flag requirement (`OPENCODE_EXPERIMENTAL_LSP_TOOL=true`)
- ✅ Binary installation instructions

## Why These Changes Matter

1. **Portability** — Symlink farm instead of hardcoded absolute paths
2. **Simplicity** — 17 paths → 1 path (auto-generated)
3. **Maintainability** — No manual addon list to update
4. **Accuracy** — Removed speculation that could confuse future agents

---

## Session 2 Cleanup: 2026-04-22 (Phase 3.5 — Config Restoration)

**Status:** Phase 3 PoC verified. Config files written to oca-17 root. LSP operational with explicit `src/` addon paths.

### What Changed in oca-17

**Deleted:**
- ❌ `.opencode/` directory entirely (broken stub `opencode.json` with invalid JSON and wrong location)

**Created (at project root):**
- ✅ `opencode.json` (verified LSP config, pyright disabled, odoo_ls_server registered)
- ✅ `odools.toml` (32 explicit `src/` subdirectories as `addons_paths`, no symlink farm)

### Why Explicit `src/` Paths?

Previous verified config used `odoo/auto/addons` (symlink farm). This session:
1. Confirmed the symlink farm works but creates deduplication ambiguity (see spec.md — Addon Path Collision)
2. Switched to explicit 32 `odoo/custom/src/<repo>` entries for clarity and to avoid inode-based deduplication bugs
3. Verified LSP indexing works with explicit paths via `lsp documentSymbol` queries

### Newly Verified Facts

- ✅ **Config file locations:** `opencode.json` and `odools.toml` must be at Git project root (per OpenCode docs)
- ✅ **`.opencode/` subdirectory:** For agents/commands/plugins only — NOT for `opencode.json` (OpenCode docs explicit)
- ✅ **LSP debug commands:** `opencode debug lsp diagnostics|symbols|document-symbols` exist and work (undocumented in public docs)
- ✅ **`OPENCODE_EXPERIMENTAL_LSP_TOOL` is client-side only:** Must be set in the shell that runs OpenCode, not passed to LSP server via `env` block
- ✅ **Explicit paths avoid ambiguity:** LSP `documentSymbol` successfully indexes all modules when explicit `src/` paths are used
- ❌ **`{env:VAR}` interpolation:** Does NOT exist in OpenCode LSP config `env` blocks (values are plain strings)
- ❌ **`--stdio` flag:** Does NOT exist in `odoo_ls_server` (server defaults to stdio transport)

### Documentation Updates

- ✅ `VERIFIED_FACTS.md` — Updated odools.toml to show explicit `src/` paths, added config location requirements, added debug commands section
- ✅ `AGENTS.md` — Already correct, no changes needed

## Next Steps for Future Sessions

When resuming work on this project:

1. Read `VERIFIED_FACTS.md` first (separates truth from speculation)
2. Use `opencode.json` + `odools.toml` at project root as the standard (not `.opencode/`)
3. Use explicit `src/` paths in `odools.toml` to avoid deduplication ambiguity
4. Use `opencode debug lsp diagnostics|symbols|document-symbols` for testing without agent overhead
5. Any new experiments should update `VERIFIED_FACTS.md`, not spec.md
6. If Phase 4 plugin work begins, document findings in a separate `PHASE_4_RESULTS.md`
