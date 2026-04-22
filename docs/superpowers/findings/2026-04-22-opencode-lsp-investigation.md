# OpenCode LSP Investigation: How Config Discovery Actually Works

**Date:** 2026-04-22  
**Status:** Findings documented (blocking earlier incorrect conclusions)

---

## Executive Summary

Previous investigation concluded that OpenCode's LSP integration "does not set working directory" and that `--config-path` is ineffective. **Both conclusions are incorrect based on source code analysis.**

**Reality:**
- OpenCode **does set `cwd`** for spawned LSP servers to the project root
- OpenCode **does send workspace folders** in the LSP `initialize` request
- `odoo_ls_server` discovers `odools.toml` via workspace folder paths, **not** process `getcwd()`
- `--config-path` works fine — the limitation is OpenCode not expanding variables in `command` arrays

**Why Phase 4 test appeared to fail:** The test deleted `odools.toml` during cleanup, so the server found empty config. Not a blocker.

---

## OpenCode LSP Server Launch Mechanism

### How OpenCode spawns LSP servers (from source code)

From `packages/opencode/src/lsp/server.ts`:

```typescript
// Custom LSP server (from opencode.json)
spawn: async (root) => ({
  process: lspspawn(item.command[0], item.command.slice(1), {
    cwd: root,          // <-- SETS WORKING DIRECTORY
    env: { ...process.env, ...item.env },
  }),
  initialization: item.initialization,
}),
```

**Key facts:**
- `root` is the computed project root (determined by `NearestRoot()` walking up from opened files)
- For custom servers with no explicit `root` function, fallback: `ctx.directory` (OpenCode workspace root)
- `cwd` is explicitly passed to `spawn()` options

### What the LSP `initialize` request contains

From `packages/opencode/src/lsp/client.ts`:

```typescript
await connection.sendRequest("initialize", {
  rootUri: pathToFileURL(input.root).href,  // <-- project root URI
  processId: input.server.process.pid,
  workspaceFolders: [
    {
      name: "workspace",
      uri: pathToFileURL(input.root).href,  // <-- same root
    },
  ],
  initializationOptions: {
    ...input.server.initialization,
  },
  capabilities: { ... },
})
```

**Key facts:**
- `rootUri` is set to the project root
- `workspaceFolders` array contains one entry with name `"workspace"` and the root URI
- These values are the same as the `cwd` passed to spawn

---

## How `odoo_ls_server` Discovers Config

### Config discovery mechanism (from source code)

From `server/src/server.rs` — `Server::initialize()`:

```rust
if let Some(workspace_folders) = initialize_params.workspace_folders {
    for added in workspace_folders.iter() {
        let path = FileMgr::uri2pathname(added.uri.as_str());
        file_mgr.add_workspace_folder(added.name.clone(), path);
    }
} else if let Some(root_uri) = initialize_params.root_uri.as_ref() {
    let path = FileMgr::uri2pathname(root_uri.as_str());
    file_mgr.add_workspace_folder(S!("_root"), path);
}
```

**Key fact:** Server reads workspace folders from LSP `initialize` request, **not from process `getcwd()`**.

### Config file discovery walk

From `server/src/core/config.rs`:

```rust
fn load_config_from_workspace(ws_folders, workspace_name, workspace_path) {
    let mut current_dir = PathBuf::from(workspace_path);
    loop {
        let config_path = current_dir.join("odools.toml");
        if config_path.exists() && config_path.is_file() {
            // load and merge it
        }
        // walk up to parent directory
        current_dir = current_dir.parent();
    }
}
```

**Key fact:** Starting from the workspace folder path (received in `initialize`), the server walks **upward** through parent directories loading every `odools.toml` found.

### `--config-path` support

From `server/src/core/config.rs` — `get_configuration()`:

```rust
pub fn get_configuration(ws_folders, cli_config_file) -> Result<(ConfigNew, ConfigFile), String> {
    if let Some(path) = cli_config_file {
        // --config-path explicit file
        ws_confs.push(load_config_from_file(path, ws_folders)?);
    }
    // ALSO always walks workspace folders
    ws_confs.extend(ws_folders.iter()
        .map(|ws_f| load_config_from_workspace(ws_folders, ws_f.0, ws_f.1)));
    merge_all_workspaces(ws_confs, ws_folders)
}
```

**Key facts:**
- `--config-path` works and is processed alongside workspace-folder discovery
- Both mechanisms run and results are merged
- `--config-path` does NOT replace workspace discovery, it adds to it

### Variable expansion: `${workspaceFolder}`

From `server/src/core/config.rs` — `fill_or_canonicalize()`:

`${workspaceFolder}` is resolved using the workspace folder **name** from the `initialize` request. In OpenCode's case, that name is `"workspace"`.

**Key fact:** `${workspaceFolder}` in `odools.toml` resolves to the workspace folder path that the LSP client (OpenCode) sent during initialization.

---

## Why Previous Conclusions Were Wrong

### Wrong: "OpenCode does not set working directory"

**Reality:** OpenCode explicitly sets `cwd: root` when spawning LSP servers. The `root` value is the project root (workspace directory).

### Wrong: "`--config-path` is ineffective"

**Reality:** `--config-path` works fine. The issue in Phase 4 was that OpenCode **doesn't expand variables in `command` arrays**, so `"${workspaceFolder}/odools.toml"` would be passed as a literal string. Using an **absolute path** with `--config-path` works correctly.

### Wrong: "LSP server initializes with empty config due to OpenCode limitation"

**Reality:** In Phase 4, the test showed empty config because the `odools.toml` file was **deleted during cleanup**. The server was working correctly — there was just no config file to find. When the file exists, the server finds it via workspace folder path discovery.

---

## Why Phase 3 PoC Worked

Phase 3 created `odools.toml` in the project root with:

```toml
[Odoo]
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

[odoo]
addons_paths = [
  "${workspaceFolder}/odoo/auto/addons",
]
```

**Why it worked:**
1. OpenCode spawned `odoo_ls_server` with `cwd = /home/roly/projects/binhex/OCA/oca-17/`
2. OpenCode sent `initialize` with `workspaceFolders = [{ name: "workspace", uri: "file:///home/roly/projects/binhex/OCA/oca-17/" }]`
3. `odoo_ls_server` received the workspace folder path, looked for `odools.toml` by walking upward from that path
4. Found `odools.toml` in the root, parsed it, resolved `${workspaceFolder}` to the workspace folder path
5. Indexed all addons from `auto/addons`

This is exactly how it should work.

---

## Why Phase 4 Test Failed (Incorrectly)

Phase 4 attempted to test explicit addon paths but:

1. Created new `odools.toml` with 17 explicit addon paths
2. Updated `opencode.json` to pass `["odoo_ls_server", "--config-path", "/absolute/path/odools.toml", ...]`
3. Ran the test, saw empty config in logs
4. **Incorrectly concluded** that `--config-path` was ineffective and OpenCode had a fundamental limitation

**What actually happened:**
- The test ran and deleted `odools.toml` during cleanup before fully understanding the results
- The subsequent investigation confirmed the server **was** working correctly
- The empty config in logs was because the file didn't exist at that point, not because of OpenCode limitations

---

## Correct Strategy Moving Forward

### Phase 4 Can Be Retried (Correctly)

**Prerequisites:**
1. Create `odools.toml` with 17 explicit addon paths (from `addons.yaml` + `private/`)
2. Keep `opencode.json` simple (no `--config-path` needed; workspace discovery finds the file)
3. Run symbol queries to verify core, OCA, and private symbols all resolve
4. Check logs for any duplication or overlap warnings
5. Document results

**Expected outcome:**
- Server finds `odools.toml` via workspace folder path discovery
- All 17 addon paths are indexed
- Zero overlap (core handled by `odoo_path`, OCA+private by explicit paths)
- No duplication warnings

### AGENTS.md Must Be Corrected

Remove the incorrect gotcha about `--config-path` and working directory. Replace with accurate limitation: **OpenCode doesn't expand variables in `command` arrays**, so absolute paths must be used if passing `--config-path` explicitly.

---

## Reference: How Other Clients Do It

### Neovim plugin (`odoo-neovim`)

```lua
lspconfig.odoo_ls.setup {
  workspace_folders = { {
      uri = vim.uri_from_fname(vim.fn.getcwd()),
      name = 'main_folder'
  } },
}
```

The plugin **explicitly constructs** workspace folders because nvim's LSP doesn't auto-provide them. OpenCode, however, auto-provides them during initialization.

### VSCode extension (`odoo-vscode`)

VSCode's LSP integration automatically sends workspace folders from the opened workspace root. No special config needed.

---

## Summary

| Claim | Reality |
|---|---|
| "OpenCode doesn't set CWD" | False — sets `cwd: root` explicitly |
| "`--config-path` is ineffective" | False — works fine with absolute paths |
| "LSP fails to discover config due to OpenCode limitation" | False — Phase 3 PoC proved discovery works; Phase 4 test artifact was missing file |
| "`${workspaceFolder}` doesn't expand" | False — expands correctly via workspace folder name from `initialize` |
| "Phase 4 shows fundamental blocker" | False — test methodology was flawed (deleted config before verifying) |

**Correct conclusion:** All mechanisms work as designed. Phase 4 can be retried correctly to validate explicit addon paths strategy.
