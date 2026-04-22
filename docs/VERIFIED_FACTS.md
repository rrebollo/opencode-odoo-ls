# Verified Facts: OpenCode + Odoo LSP Integration

**Last updated:** 2026-04-22

This document separates **100% verified facts** from **speculation, assumptions, and failed experiments**.

---

## Verified Configuration (Phase 3 PoC Results)

### The Setup That Works

**opencode.json:**
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

**odools.toml:**
```toml
[Odoo]
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

[odoo]
addons_paths = [
  "${workspaceFolder}/odoo/auto/addons",
]
```

**Verification performed on:** `/home/roly/projects/binhex/OCA/oca-17/` (Doodba project)

**Verified behaviors:**
- ✅ `odoo_ls_server` binary (v1.2.1) starts automatically when OpenCode opens `.py` or `.xml` files
- ✅ `odools.toml` is discovered from project root
- ✅ `${workspaceFolder}` variables are expanded by the LSP server at initialization time
- ✅ Both the symlink farm (`odoo/auto/addons/`) and explicit paths resolve correctly
- ✅ Workspace symbols are indexed (verified with workspaceSymbol queries)
- ✅ LSP diagnostics are injected into agent tool output after edits
- ✅ The `lsp` tool (with `OPENCODE_EXPERIMENTAL_LSP_TOOL=true`) successfully calls LSP operations
- ✅ No duplicate diagnostics between pyright and odoo_ls_server

### Verified Requirements

- ✅ Binary on PATH: `odoo_ls_server` must be installed to `~/.local/bin/` or another PATH directory
- ✅ Typeshed stubs: The `~/.local/share/odoo-ls/typeshed/` directory must exist (downloaded from official releases)
- ✅ Project structure: Requires a Doodba or Odoo project with `odoo/custom/src/` or similar addon layout

---

## Speculation & Failed Experiments (Do Not Use)

### Experiments That Did NOT Work

1. **`--config-path` with hardcoded absolute paths**
   - Tested: `["odoo_ls_server", "--config-path", "/absolute/path/to/odools.toml"]`
   - Status: ❌ Non-portable, creates path fragility
   - Why: Hardcoded paths break when workspace is cloned elsewhere
   - Conclusion: Use auto-discovery instead

2. **LSP initialization options in opencode.json**
   - Tested: `"initialization": { "Odoo": { "selectedProfile": "default" } }`
   - Status: ❌ Does not affect odoo_ls_server behavior
   - Why: The server does not read `initializationOptions` at all
   - Source: Direct inspection of odoo_ls_server source code shows zero handling of initializationOptions
   - Conclusion: Remove from opencode.json

3. **Configuration files with explicit addon paths**
   - Tested: odools.toml with 17 explicit repo paths instead of symlink farm
   - Status: ⚠️ Works but adds maintenance burden
   - Why: When repos are cloned/removed, the list becomes stale
   - Conclusion: Use symlink farm (Option A) unless specific requirements demand granular control

4. **Environment variables in opencode.json LSP block**
   - Tested: `"env": { "OPENCODE_EXPERIMENTAL_LSP_TOOL": "true" }`
   - Status: ❌ Not a valid opencode.json key
   - Error: `Configuration is invalid - Unrecognized key: "env"`
   - Note: The `env` key is only valid inside individual LSP server blocks per OpenCode docs, but it's for server-side env vars, not client-side flags

---

## OpenCode LSP Behavior (Source-Verified)

From OpenCode source code analysis and official docs at https://opencode.ai/docs/lsp:

### How OpenCode Passes Config to LSP Servers

1. **Workspace folder discovery:**
   - OpenCode finds the Git repository root or current working directory
   - Sends it to the LSP server in the `initialize` request as `rootUri` and `workspaceFolders`

2. **Variable expansion:**
   - **In command arrays:** Variables like `${workspaceFolder}` are NOT expanded by OpenCode
     - Workaround: Use binaries on PATH or absolute paths
   - **In config files:** Variables like `${workspaceFolder}` ARE expanded by the LSP server itself
     - `odoo_ls_server` handles `${workspaceFolder}` at startup

3. **Initialization options:**
   - OpenCode sends `initialization` object from config to the LSP server's `initialize` request
   - **`odoo_ls_server` does NOT read these** — it only reads `odools.toml`
   - Only configuration mechanism for `odoo_ls_server` is:
     - Auto-discovery of `odools.toml` from workspace folder
     - Explicit `--config-path` CLI argument

---

## Odoo LSP Server Behavior (Source-Verified)

From odoo_ls_server source code inspection:

### Configuration Mechanisms

**Primary:** Auto-discovery of `odools.toml`
```rust
// server/src/core/config.rs
pub fn get_configuration(ws_folders: &HashMap<String, String>, cli_config_file: &Option<String>) {
    if let Some(path) = cli_config_file {
        // 1. Load from --config-path if provided
        let config_from_file = load_config_from_file(path.clone(), ws_folders)?;
        ws_confs.push(config_from_file);
    }
    
    // 2. Walk upward from each workspace folder looking for odools.toml
    let ws_confs_result: Result<Vec<_>, _> = ws_folders
        .iter()
        .map(|ws_f| load_config_from_workspace(ws_folders, ws_f.0, ws_f.1))
        .collect();
    
    ws_confs.extend(ws_confs_result?);
}
```

**Secondary:** `workspace/configuration` LSP request
- Server requests `Odoo.selectedProfile` from client settings
- Unused in current OpenCode integration

**Tertiary:** CLI arguments (for parse mode, not LSP mode)
- `--addons`, `--python`, etc. only work in parse mode (`-p`)
- Irrelevant to LSP server mode

### Variable Expansion

`${workspaceFolder}` and other variables are expanded by the server at runtime:
```toml
[Odoo]
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"  # Expanded at LSP init
```

---

## What Still Needs Investigation

1. **Type resolution without host Python deps**
   - Doodba runs Odoo in Docker; host has no Odoo dependencies
   - Hovering on fields returns type info inconsistently
   - Open: Does the server gracefully degrade? Is this expected?
   - Status: Not yet fully clarified

2. **JavaScript/OWL support**
   - Spec.md notes: "Do NOT add `.js`"
   - Reason: Unconfirmed whether `odoo_ls_server` supports it
   - Status: Assumed unsupported; not tested

3. **Non-Doodba Odoo environments**
   - Detection and configuration for plain Odoo, OCA multi-repo setups
   - Status: Designed but not tested in this session

---

## Summary: What to Tell Agents

When an agent asks about OpenCode + Odoo LSP setup:

**✅ Recommend:**
- Two-file setup: `opencode.json` + `odools.toml` (pure config)
- Simple `opencode.json` with no hardcoded paths or initialization options
- Symlink farm (`odoo/auto/addons`) for addon discovery
- `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` flag for agents using the lsp tool

**❌ Warn against:**
- Hardcoded absolute paths in `opencode.json` (breaks portability)
- Initialization options in opencode.json (ignored by the server)
- Manual addon path lists (use symlink farm)
- Running both pyright and odoo_ls_server (disable pyright)

**❓ Acknowledge as open:**
- Type resolution without host Python dependencies (may be limited in Doodba)
- JavaScript/OWL component support (untested)
