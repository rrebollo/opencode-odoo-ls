# opencode-odoo-ls

Configure Odoo Language Server Protocol (LSP) support for OpenCode in a Doodba project. After completing setup, agents working in this project will receive Odoo-aware diagnostics on every file edit and can use the `lsp` tool to query model definitions, field types, and symbol locations.

---

## For Agents: How to Use This Guide

This guide is designed for agents to follow. It provides:

- **LITERAL instructions** — marked clearly (e.g., "LITERAL — use exactly as shown")
- **FLEXIBLE choices** — marked clearly (e.g., "FLEXIBLE — your choice") where you should decide based on project structure
- **Variable placeholders** — paths like `<ODOO_VERSION>`, `<PYTHON_VERSION>`, or `${workspaceFolder}` that YOU must substitute with actual values
- **Decision points** — places where the setup depends on your project's layout

**Approach:** Read through the entire guide first. Then execute the steps, adapting paths and choices to your specific project structure. If something doesn't match your layout, check the "Troubleshooting" section or refer to `docs/VERIFIED_FACTS.md` for more context.

---

## Prerequisites: Confirm Environment

Before configuring LSP, determine your project environment and gather required information.

### Verify Doodba Project

For Doodba-based Odoo projects:

```bash
grep -q 'Tecnativa/doodba-copier-template' .copier-answers.yml 2>/dev/null && echo "DOODBA OK"
```

If this command succeeds, you have a Doodba project (Docker-based, recommended target).

For non-Doodba projects (plain Odoo, OCA multi-repo): Adapt the paths in Step 3 and Step 4 (addon discovery) to match your layout. The LSP setup itself (Steps 1–5) is the same for any Odoo environment.

### Determine Your Odoo Version

The Odoo version determines which Python version and dependency versions you'll need.

**For Doodba projects:**
```bash
# Check the project's Odoo version
grep -E "ODOO_VERSION|odoo_version" .copier-answers.yml || grep -E "ODOO_VERSION" docker-compose.yml || cat .env | grep ODOO_VERSION
# Common values: 16, 17, 18 (major version only)
```

**For non-Doodba projects:**
Check your `odoo/__init__.py` or your project's dependency manifest.

Document this value as `<ODOO_VERSION>` for use in subsequent steps.

### Detect Python Version from Docker Image

Because `odoo_ls_server` runs on your host (not in Docker), it must use a Python interpreter that matches the container's Python version exactly. Type resolution depends on version-specific bytecode and stdlib paths.

**For Doodba projects:**

1. Get your Odoo version from `.copier-answers.yml`:
   ```bash
   grep "odoo_version" .copier-answers.yml
   # Output example: odoo_version: 17.0
   ```
   Document this as `<ODOO_VERSION>`.

2. The Doodba base image follows a standard pattern. Confirm it:
   ```bash
   cat odoo/Dockerfile
   # Should show: FROM ghcr.io/tecnativa/doodba:${ODOO_VERSION}-onbuild
   ```

3. Inspect the image's Python version:
   ```bash
   # Construct the full image name and inspect it
   docker image inspect ghcr.io/tecnativa/doodba:<ODOO_VERSION>-onbuild | grep PYTHON_VERSION
   # Output example: "PYTHON_VERSION=3.10.19"
   ```

4. If the image is not available locally, check the Doodba wiki (https://github.com/Tecnativa/doodba-copier-template/wiki):
   - Look up your `<ODOO_VERSION>` in the Doodba documentation
   - Note the Python version listed there

Document this value as `<PYTHON_VERSION>` (e.g., `3.10.19`, `3.11.8`).

### Install Matching Python on Host

Use `pyenv` to install the exact Python version detected above. This ensures type resolution works correctly.

**Install pyenv (if not already installed):**
```bash
curl https://pyenv.run | bash
# Then add to your shell profile (~/.bashrc, ~/.zshrc, etc.):
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init --path)"
eval "$(pyenv init -)"
# Reload shell: exec $SHELL
```

**Install matching Python version:**
```bash
pyenv install <PYTHON_VERSION>
# Verify
pyenv versions | grep <PYTHON_VERSION>
```

**Confirm it's available:**
```bash
~/.pyenv/versions/<PYTHON_VERSION>/bin/python3 --version
# Should output: Python <PYTHON_VERSION>
```

### Verify Required System Tools

Ensure these tools are available on your system:

```bash
# Essential
which curl      || echo "Install: curl"
which git       || echo "Install: git"
which docker    || echo "Install: docker (or docker desktop)"

# Recommended (for convenience)
which jq        || echo "Install: jq (for parsing JSON)"
which unzip     || echo "Install: unzip"
which python3   || echo "Install: python3 (system default, separate from pyenv)"
```

---

## Step 1: Install odoo_ls_server Binary and Typeshed

The LSP server (`odoo_ls_server`) and typeshed stubs must be installed on your host.

### Install the LSP Server Binary

Download and install the latest release of `odoo_ls_server` to `~/.local/bin/`:

```bash
# Create directories
mkdir -p ~/.local/bin ~/.local/share/odoo-ls

# Find the latest release tag
RELEASE=$(curl -s https://api.github.com/repos/odoo/odoo-ls/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
echo "Installing odoo_ls_server version: $RELEASE"

# Download the Linux x86_64 binary
curl -L -o /tmp/odoo-ls.tar.gz \
  "https://github.com/odoo/odoo-ls/releases/download/${RELEASE}/odoo-linux-x86_64-${RELEASE}.tar.gz"

# Extract and install
tar -xzf /tmp/odoo-ls.tar.gz -C /tmp/
cp /tmp/odoo_ls_server ~/.local/bin/
chmod +x ~/.local/bin/odoo_ls_server

# Clean up
rm /tmp/odoo_ls_server /tmp/odoo-ls.tar.gz
```

**Add ~/.local/bin to your PATH (if not already there):**
```bash
grep -q '\.local/bin' ~/.bashrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
```

**Verify installation:**
```bash
odoo_ls_server --help
# Should print usage information without errors
```

### Install Typeshed Stubs (Required for Type Resolution)

Typeshed provides Python standard library type definitions. Without it, the LSP server cannot resolve types.

```bash
# Download typeshed.zip from the same release
curl -L -o /tmp/typeshed.zip \
  "https://github.com/odoo/odoo-ls/releases/download/${RELEASE}/typeshed.zip"

# Extract to the shared location (use -o to overwrite if already present)
unzip -o /tmp/typeshed.zip -d ~/.local/share/odoo-ls/

# Verify the structure (you should see a typeshed/ subdirectory)
ls -la ~/.local/share/odoo-ls/typeshed/stdlib/ | head -10
# Should show .pyi files like builtins.pyi, sys.pyi, etc.

# Clean up
rm /tmp/typeshed.zip
```

---

## Step 2: Create Host Virtual Environment with Odoo

The LSP server needs a Python environment where Odoo can be imported for type resolution.

### Create Virtual Environment

Use the Python version detected in Prerequisites:

```bash
~/.pyenv/versions/<PYTHON_VERSION>/bin/python3 -m venv ~/.venv/odoo<ODOO_VERSION>
```

### Activate and Install Prerequisites

```bash
source ~/.venv/odoo<ODOO_VERSION>/bin/activate
pip install --upgrade pip
```

### Install Odoo from Project Source

Assuming your Odoo source is at `odoo/custom/src/odoo/` (Doodba structure):

```bash
# Replace the path with your actual Odoo source location
pip install -e /path/to/your/project/odoo/custom/src/odoo
```

**Note:** The `-e` flag installs in "editable" mode, which allows setuptools to locate Odoo modules from the source directory.

### Verify Odoo Import Works

```bash
~/.venv/odoo<ODOO_VERSION>/bin/python3 -c "import odoo; print(odoo.__file__)"
# Should print something like: /path/to/odoo/custom/src/odoo/__init__.py
```

If this fails, check:
- The path to Odoo source is correct
- Setuptools is installed
- The virtual environment was activated

### Deactivate (Optional)

You don't need the venv activated for normal work — the LSP server will use it via the path in `odools.toml`.

```bash
deactivate
```

---

## Step 3: Create opencode.json at Project Root

This file configures OpenCode to launch and manage the LSP server.

### LITERAL — Use Exactly As Shown

Create the file `/path/to/your/project/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "lsp": {
    "pyright": { "disabled": true },
    "odoo-ls": {
      "command": ["odoo_ls_server"],
      "extensions": [".py", ".xml"],
      "initialization": {
        "selectedProfile": "default"
      }
    }
  }
}
```

### Explanation of Each Field

- **`"pyright": { "disabled": true }`** — Disables OpenCode's built-in Python LSP to avoid duplicate diagnostics. The `odoo_ls_server` provides its own Python diagnostics.
- **`"command": ["odoo_ls_server"]`** — The LSP server binary to run. It must be on your `PATH` (installed in Step 1 at `~/.local/bin/`).
- **`"extensions": [".py", ".xml"]`** — File types that trigger the LSP server:
  - `.py` for Python model/model mixin definitions
  - `.xml` for Odoo view, action, and report definitions
  - Do NOT add `.js` — `odoo_ls_server` does not support JavaScript/OWL components yet.
- **`"initialization": { "selectedProfile": "default" }`** — Matches the profile name in `odools.toml` (see Step 4).

### Important Notes

- `opencode.json` must be at your Git project root, NOT inside `.opencode/` directory.
- The `.opencode/` directory is for agents, commands, and plugins only.

---

## Step 4: Create odools.toml at Project Root

This file configures the LSP server itself with paths to Odoo, Python, and addon directories.

### Understanding the Structure: Array-of-Tables Syntax

The `odools.toml` file uses TOML's `[[config]]` array-of-tables syntax, NOT bare `[Odoo]` or `[odoo]` sections. The server requires this specific format.

### LITERAL — Use Exactly As Shown (Base Template)

Create the file `/path/to/your/project/odools.toml`:

```toml
[[config]]
name = "default"
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"
python_path = "/home/<username>/.venv/odoo<ODOO_VERSION>/bin/python3"
stdlib = "/home/<username>/.local/share/odoo-ls/typeshed/stdlib/"
diag_missing_imports = "only_odoo"
refresh_mode = "adaptive"

addons_paths = [
  # ADD YOUR ADDON DIRECTORIES HERE
]
```

Replace placeholders:
- `<username>` — Your system username (e.g., `roly`)
- `<ODOO_VERSION>` — Major version (e.g., `17`)
- Add addon directories (see next section)

### Understanding Each Field

- **`name = "default"`** — Profile identifier. Must match `selectedProfile` in `opencode.json` above.
- **`odoo_path`** — Path to the Odoo source directory (the one containing `odoo/__init__.py`). Uses `${workspaceFolder}` which is expanded by the LSP server at initialization to the Git repository root.
- **`python_path`** — Absolute path to the Python executable in your venv from Step 2. This Python must have Odoo installed. **Cannot use variables** — must be absolute path.
- **`stdlib`** — Absolute path to typeshed stdlib from Step 1. **CRITICAL: Must have trailing `/`** — Without it, the server fails to find `builtins.pyi`. See https://github.com/odoo/odoo-ls/issues/425#issuecomment-3311402591
- **`diag_missing_imports = "only_odoo"`** — For Docker projects: suppresses `ModuleNotFound` errors for non-Odoo packages (lxml, werkzeug, psycopg2, etc.) that the host Python doesn't have. These are only available in the Docker container.
- **`refresh_mode = "adaptive"`** — Correct for OpenCode, which never sends `didSave` notifications, only `didChange`. This tells the server not to wait for explicit save events.

### Configure addons_paths: Two Approaches

Choose **one** approach for populating the `addons_paths` array:

#### Option A: Explicit Repository Paths (RECOMMENDED)

List each addon repository directory explicitly. This avoids symlink deduplication ambiguity.

**Steps:**
1. Check your Doodba `addons.yaml` for the canonical addon paths:
   ```bash
   cat odoo/custom/src/addons.yaml
   # Lists the main addon repositories (excludes private/ which is auto-detected)
   ```

2. List all directories under `odoo/custom/src/`:
   ```bash
   ls -d odoo/custom/src/*/
   # Output example:
   # odoo/custom/src/account-reconcile/
   # odoo/custom/src/bank-payment/
   # odoo/custom/src/custom-module/
   # odoo/custom/src/odoo/  (skip this one - it's the core Odoo package)
   # odoo/custom/src/private/  (include if it exists and has modules)
   # odoo/custom/src/web/
   ```

3. Add all paths EXCEPT `odoo/custom/src/odoo/` to `addons_paths`:
   ```toml
   addons_paths = [
     "${workspaceFolder}/odoo/custom/src/account-reconcile",
     "${workspaceFolder}/odoo/custom/src/bank-payment",
     "${workspaceFolder}/odoo/custom/src/custom-module",
     "${workspaceFolder}/odoo/custom/src/private",  # Include if it has Odoo modules
     "${workspaceFolder}/odoo/custom/src/web",
     # ... (continue for all addon repositories)
   ]
   ```

**Important notes:**
- Include `private/` if it contains Odoo modules (it won't be listed in `addons.yaml`)
- Do NOT include `odoo/custom/src/odoo/` — that's configured separately via `odoo_path`
- The LSP auto-detects modules inside these directories and their subdirectories

**Why explicit paths?**
- Avoids deduplication ambiguity with `odoo/auto/addons/` symlink farm
- LSP anti-deduplication behavior with symlinks is undocumented
- Deterministic and clear
- Easy to maintain if repos are cloned/removed

#### Option B: Symlink Farm

If your project auto-generates `odoo/auto/addons/` via `gitaggregate`:

```toml
addons_paths = [
  "${workspaceFolder}/odoo/auto/addons",
]
```

**Trade-offs:**
- ✅ Simpler (1 path instead of 30+)
- ❌ Creates inode aliasing ambiguity (LSP deduplication behavior is undocumented)
- ❌ May trigger unpredictable deduplication bugs

**Recommendation:** Use Option A unless you have specific reasons for the symlink farm.

### Example Complete odools.toml (Doodba Project)

```toml
[[config]]
name = "default"
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"
python_path = "/home/roly/.venv/odoo17/bin/python3"
stdlib = "/home/roly/.local/share/odoo-ls/typeshed/stdlib/"
diag_missing_imports = "only_odoo"
refresh_mode = "adaptive"

addons_paths = [
  "${workspaceFolder}/odoo/custom/src/account-reconcile",
  "${workspaceFolder}/odoo/custom/src/bank-payment",
  "${workspaceFolder}/odoo/custom/src/custom-modules",
  "${workspaceFolder}/odoo/custom/src/web",
]
```

---

## Step 5: Verify LSP Configuration

Test that the LSP server is correctly configured before using it with agents.

### Test in Parse Mode (Standalone, No OpenCode Needed)

Parse mode loads the project, generates diagnostics, writes them to a JSON file, then exits. It's useful for testing without needing the LSP server to stay running.

**Important:** `odools.toml` is NOT read in parse mode — all configuration is passed via CLI flags. This is confirmed by the maintainer in issue #425. When testing with `--parse`, always pass all flags explicitly.

```bash
cd /path/to/your/project

# Test with a known addon directory
odoo_ls_server --parse \
  -c "odoo/custom/src/odoo" \
  -a "odoo/custom/src/account-reconcile" \
  -a "odoo/custom/src/bank-payment" \
  --python "/home/<username>/.venv/odoo<ODOO_VERSION>/bin/python3" \
  --stdlib "/home/<username>/.local/share/odoo-ls/typeshed/stdlib/" \
  -o /tmp/diagnostics.json \
  --log-level debug
```

Replace:
- `<username>` — Your system username
- `<ODOO_VERSION>` — Major version (e.g., `17`)
- `-c <path>` — Odoo source path
- `-a <path>` — Each addon directory (add one `-a` per directory, or use just 1-2 for testing)
- `--python <path>` — Python executable from Step 2
- `--stdlib <path>` — Typeshed path from Step 1 (must have trailing `/`)
- `-o <file>` — Output file for diagnostics JSON
- `--log-level debug` — Verbose logging (use `info` or `warn` for less output)

**What success looks like:**
```
$ odoo_ls_server --parse ... (command above)
[INFO] Loading project...
[INFO] Indexing <N> modules...
[INFO] Complete. Written diagnostics to /tmp/diagnostics.json
Exit code: 0
```

**Check the output:**
```bash
cat /tmp/diagnostics.json | head -50
# Should contain JSON with diagnostics entries, or empty array [] if no issues found
```

**If parse mode fails:**
- Check error messages (usually related to Python or stdlib paths)
- Verify all `-c`, `-a`, `--python`, `--stdlib` paths exist and are correct
- See Troubleshooting section below for common errors

### Test with OpenCode LSP Server Mode

Once parse mode works, test the LSP server in OpenCode:

```bash
# Test diagnostics for a Python file
opencode debug lsp diagnostics odoo/custom/src/account-reconcile/models/account_move.py
# (Adapt the path to a real file in your project)

# Should print diagnostics, or "No diagnostics" if file is clean
```

### Monitor Indexing Status

The LSP server loads and indexes your project in the background. Check the logs to see when indexing completes:

```bash
# Find the logs directory
opencode debug paths
# Then monitor the LSP logs
tail -f <logs_directory>/opencode-lsp-*.log | grep -E "loadingStatusUpdate|ERROR|WARN"
# When you see: loadingStatusUpdate: "stop"
# Indexing is complete and diagnostics are ready
```

### Test with Agents (Optional)

Once LSP is working, test it with agents using the `lsp` tool:

```bash
OPENCODE_EXPERIMENTAL_LSP_TOOL=true opencode run \
  "Use the lsp tool to query workspaceSymbol for 'AccountMove' (or any model in your project). \
   Report the file path and line number where this model is defined."
```

Expected: The agent returns a file path to a Python model definition in your addon directories.

---

## Important Notes and Limitations

**OpenCode `command` arrays do NOT expand variables** — When using `--config-path`, you must use absolute paths or binaries on `PATH`. Variables like `${workspaceFolder}` are NOT expanded by OpenCode. Variable expansion happens in certain contexts (like `instructions` paths), but not in `lsp.*.command` arrays. If you want to use `--config-path`, pass an absolute path like `["odoo_ls_server", "--config-path", "/absolute/path/odools.toml"]`.

**Symlink farm deduplication issue** — The `odoo/auto/addons/` symlink farm resolves to the same inodes as repository directories. The LSP anti-deduplication algorithm behavior with symlink aliasing is undocumented. To avoid unpredictable deduplication, use explicit `odoo/custom/src/<repo>` paths in `addons_paths` instead of the symlink farm (Option A in Step 4).

**Setting `addons_paths` disables auto-detection** — Any value (including `[]`) disables Odoo LSP's auto-detection. If you need auto-detection alongside explicit paths, add the literal string `"$autoDetectAddons"` to the array.

**`lsp` tool requires an experimental flag** — `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` must be set for agents to call lsp tool operations (hover, workspaceSymbol, goToDefinition, etc.). Without it, agents silently fall back to grep/read and never query the LSP.

---

## IDE Plugin References

The following IDE plugins have been investigated and verified to work with Odoo LSP by sending `workspace/configuration` correctly:

- **VSCode**: https://github.com/odoo/odoo-vscode
- **PyCharm**: https://github.com/odoo/odoo-pycharm
- **Neovim**: https://github.com/odoo/odoo-neovim
- **Zed**: https://github.com/odoo/odoo-zed

All use the same pattern: the profile name in `odools.toml` must match the `selectedProfile` sent by the client.

---

## Additional Resources

- **Odoo LSP Server Repository:** https://github.com/odoo/odoo-ls
- **Odoo LSP Configuration Wiki:** https://github.com/odoo/odoo-ls/wiki/3.-Configuration-files
- **Doodba Project Template:** https://github.com/Tecnativa/doodba-copier-template
- **OpenCode LSP Documentation:** https://opencode.ai/docs/lsp
- **This Project's Verified Facts:** See `docs/VERIFIED_FACTS.md` in this repository
