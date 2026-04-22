# opencode-odoo-ls

Configure Odoo Language Server Protocol (LSP) support for OpenCode in a Doodba project. After completing setup, agents working in this project will receive Odoo-aware diagnostics on every file edit and can use the `lsp` tool to query model definitions, field types, and symbol locations.

---

## Step 1: Confirm you are in a Doodba project

Before doing anything, verify:

```bash
grep -q 'Tecnativa/doodba-copier-template' .copier-answers.yml 2>/dev/null && echo "DOODBA OK" || echo "NOT DOODBA"
ls odoo/auto/addons/ | wc -l   # must be > 0 (gitaggregate has run)
ls odoo/custom/src/odoo/odoo-bin  # must exist
```

If `odoo/auto/addons/` is empty: gitaggregate has not run. Stop — the symlink farm must exist before configuring LSP.

---

## Step 2: Install odoo_ls_server

```bash
which odoo_ls_server && echo "already installed, skip to Step 3"
```

If not installed, fetch the latest release from https://github.com/odoo/odoo-ls/releases and install the Linux x86_64 binary (asset named `odoo-linux-x86_64-*.tar.gz`). Also download `typeshed.zip` from the same release — it is required for type resolution.

```bash
# Find latest release tag
RELEASE=$(curl -s https://api.github.com/repos/odoo/odoo-ls/releases/latest | grep '"tag_name"' | cut -d'"' -f4)

# Install binary
mkdir -p ~/.local/bin ~/.local/share/odoo-ls
curl -L -o /tmp/odoo-ls.tar.gz \
  "https://github.com/odoo/odoo-ls/releases/download/${RELEASE}/odoo-linux-x86_64-${RELEASE}.tar.gz"
tar -xzf /tmp/odoo-ls.tar.gz -C /tmp/
cp /tmp/odoo_ls_server ~/.local/bin/
chmod +x ~/.local/bin/odoo_ls_server

# Install typeshed stubs (required — LSP will not work without them)
curl -L -o /tmp/typeshed.zip \
  "https://github.com/odoo/odoo-ls/releases/download/${RELEASE}/typeshed.zip"
unzip -o /tmp/typeshed.zip -d ~/.local/share/odoo-ls/

# Ensure ~/.local/bin is on PATH
grep -q '\.local/bin' ~/.bashrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
```

Verify: `odoo_ls_server --help` prints usage without errors.

---

## Step 3: Create opencode.json at project root

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

- Do NOT add `.js` — `odoo_ls_server` does not support JavaScript.
- Do NOT skip `"pyright": { "disabled": true }` — running both produces duplicate diagnostics (see Gotchas).

---

## Step 4: Create odools.toml at project root

```toml
[Odoo]
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

[odoo]
addons_paths = [
  "${workspaceFolder}/odoo/auto/addons",
]
```

`${workspaceFolder}` is resolved by the LSP server at startup, not by OpenCode.

---

## Step 5: Verify

```bash
OPENCODE_EXPERIMENTAL_LSP_TOOL=true opencode run \
  "Use the lsp tool with workspaceSymbol to search for any class name you know exists in this project. Report the raw result including file path and line number."
```

Expected: result includes a file path under `odoo/custom/src/` or `odoo/auto/addons/`. If the lsp tool reports "no LSP server available": restart OpenCode so it picks up the new `opencode.json`.

---

## Gotchas

**`odoo_ls_server` ≠ `odoo-lsp`** — The community fork (github.com/Desdaemon/odoo-lsp) has a different binary name. The official server (github.com/odoo/odoo-ls) is `odoo_ls_server`. Don't confuse them.

**Disable pyright or get noise** — `odoo_ls_server` is a full Python LSP with its own type resolution. Running alongside OpenCode's built-in pyright gives duplicate completions and conflicting diagnostics. `"pyright": { "disabled": true }` applies only to this project's `opencode.json` — other projects are unaffected.

**`odoo/auto/addons/` is on the host, not Docker-only** — After gitaggregate, this directory contains symlinks that resolve correctly on the host filesystem. Using it as `addons_paths` is correct and intentional.

**Setting `addons_paths` disables auto-detection** — Any value (including `[]`) disables Odoo LSP's auto-detection. If you need auto-detection alongside explicit paths, add the literal string `"$autoDetectAddons"` to the array.

**`--addons` and `--python` CLI flags are irrelevant** — They only work in parse mode, not in LSP server mode. Config is entirely via `odools.toml`.

**`lsp` tool requires an experimental flag** — `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` must be set for agents to call lsp tool operations (hover, workspaceSymbol, goToDefinition, etc.). Without it, agents silently fall back to grep/read and never query the LSP.
