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

## Next Steps for Future Sessions

When resuming work on this project:

1. Read `VERIFIED_FACTS.md` first (separates truth from speculation)
2. Use the simplified config from `AGENTS.md` as the standard
3. Any new experiments should update `VERIFIED_FACTS.md`, not spec.md
4. If Phase 4 plugin work begins, document findings in a separate `PHASE_4_RESULTS.md`
