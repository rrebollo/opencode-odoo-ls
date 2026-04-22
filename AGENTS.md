# opencode-odoo-ls

OpenCode integration for the official **Odoo Language Server Protocol** (Odoo LSP). Provides intelligent code assistance for Odoo development (Python models, XML templates, JavaScript) across multiple Odoo environments.

**Full spec and research:** see `docs/spec.md`  
**Current status:** Phase 1–2 complete; Phase 3 (PoC) ready to begin  
**Next step:** See "Landmines" section below, then execute Phase 3

---

## Landmines: Non-Obvious Gotchas

### Binary name is `odoo_ls_server`, NOT `odoo-lsp`

The community fork (https://github.com/Desdaemon/odoo-lsp) has a different binary name. The official repository (https://github.com/odoo/odoo-ls) uses `odoo_ls_server`. Don't confuse the two.

### CRITICAL: Disable pyright or get duplicate completions

`odoo_ls_server` is a **full Python LSP** that implements its own type resolution. Running it alongside OpenCode's built-in pyright causes:
- Duplicate completions (both return `self.env`, model fields, etc.)
- Conflicting diagnostics (pyright flags valid Odoo patterns as errors)
- User experience degrades from signal to noise

**Official guidance** (from VSCode extension): disable the generic Python LSP.

**In project-local `opencode.json`:**
```json
{
  "lsp": {
    "pyright": { "disabled": true },
    "odoo-ls": {
      "command": ["odoo_ls_server"],
      "extensions": [".py", ".xml", ".js"]
    }
  }
}
```

### The symlink farm is on the HOST, not Docker-only

After running `gitaggregate` in Doodba, `odoo/auto/addons/` becomes a symlink farm (936 symlinks) that **resolves correctly on the host filesystem**. Example: `odoo/auto/addons/hr_timesheet_begin_end` → `../../custom/src/timesheet/hr_timesheet_begin_end`

This is counter-intuitive — most people assume build artifacts are Docker-only. The symlinks work on the host, making `${workspaceFolder}/odoo/auto/addons` a valid LSP `addons_paths` entry.

### Setting `addons_paths` disables auto-detection

In `odools.toml`, setting `addons_paths` to **any value** (including `[]`) disables Odoo LSP's auto-detection of addon directories. To restore auto-detection alongside explicit paths, include the literal string `"$autoDetectAddons"`:

```toml
addons_paths = [
  "${workspaceFolder}/odoo/auto/addons",
  "$autoDetectAddons"
]
```

### CLI args `--addons` and `--python` are parse-mode only

These look relevant but **are not used in LSP server mode**. Don't waste effort configuring them. Real config is via `odools.toml` file discovery.

---

## Minimum Viable Config (Copy-Paste Ready)

Two files, placed at project root alongside `odoo/` and `.copier-answers.yml`:

### `opencode.json`

```json
{
  "$schema": "https://opencode.ai/config.json",
  "lsp": {
    "pyright": { "disabled": true },
    "odoo-ls": {
      "command": ["odoo_ls_server"],
      "extensions": [".py", ".xml", ".js"]
    }
  }
}
```

### `odools.toml`

```toml
[Odoo]
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

[odoo]
addons_paths = [
  "${workspaceFolder}/odoo/auto/addons",
]
```

**Notes:**
- Both are project-local; don't affect other workspaces
- `odoo_ls_server` must be on `PATH`
- Path variables are resolved by the LSP server at startup, not by OpenCode

---

## Blocked Before Phase 3 PoC

- [ ] Does `odoo_ls_server` need `--stdio` flag, or is it automatic?
- [ ] Does it handle `.js` files (Odoo OWL components)?
- [ ] Does disabling pyright at workspace scope affect non-Odoo `.py` files in the same project?

---

## Conventions

- **Research before code:** Always prefer reading source code (Cargo.toml, args.rs, wiki) over prose
- **Executable over documentation:** Verify facts by inspection, not assumption
- **Conventional commits:** Use feat:, fix:, docs:, chore: prefixes
