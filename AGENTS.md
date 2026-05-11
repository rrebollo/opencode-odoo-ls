# OpenCode Odoo LSP Setup Guide (Agent-Oriented)

Configure Odoo Language Server Protocol (LSP) support for OpenCode in a Doodba project. After setup, agents receive Odoo-aware diagnostics on file edits and can use the `lsp` tool for model/field queries.

This guide is for **code agents** to execute automatically. Each step includes a validation command with expected output.

---

## Prerequisites: Environment Detection

Detect the environment before configuring. All detected values become placeholders for subsequent steps.

```bash
# 0. Capture project root (needed for pip install path later)
PROJECT_ROOT=$(pwd)
echo "PROJECT_ROOT=${PROJECT_ROOT}"

# 1. Verify Doodba
grep -q 'Tecnativa/doodba-copier-template' .copier-answers.yml 2>/dev/null && echo "DOODBA_OK" || echo "NON_DOODBA"

# 2. Detect Odoo version
ODOO_VERSION=$(grep -oP 'odoo_version:\s*\K[\d.]+' .copier-answers.yml | cut -d. -f1)
echo "ODOO_VERSION=${ODOO_VERSION}"

# 3. Detect Python version from Docker image
PYTHON_VERSION=$(docker image inspect ghcr.io/tecnativa/doodba:${ODOO_VERSION}-onbuild 2>/dev/null | grep -oP 'PYTHON_VERSION=\K[^"]+' || echo "MISSING")
echo "PYTHON_VERSION=${PYTHON_VERSION}"

# 4. Detect tools
for tool in curl git docker pyenv jq gh unzip; do
  command -v "$tool" >/dev/null 2>&1 && echo "$tool: OK" || echo "$tool: MISSING"
done

# 5. Detect Odoo source dynamically
ODOO_SRC=$(find . -maxdepth 4 -name "setup.py" -path "*/odoo/*" ! -path "*/odoo/odoo/*" 2>/dev/null | head -1 | xargs -r dirname)
[ -z "$ODOO_SRC" ] && ODOO_SRC=$(find . -maxdepth 3 -name "setup.py" -exec grep -l "find_packages" {} \; 2>/dev/null | head -1 | xargs dirname)
echo "ODOO_SRC=${ODOO_SRC}"

# 6. Install matching Python (if missing)
pyenv install -s "${PYTHON_VERSION}"
```

**Expected:** All tools show `OK` or `MISSING` (only `gh` may be missing). `ODOO_SRC` points to the directory containing `setup.py`.

---

## Step 1: Install odoo_ls_server Binary and Typeshed

### 1a. Detect latest pre-release

The Odoo LSP project uses odd/even versioning: **odd minor = pre-release** (latest features), **even minor = stable**. Pre-release `1.3.x` is recommended for richer CSV support (go-to-references, xml_id validation, field validation).

```bash
# Primary: use gh CLI (more reliable)
if command -v gh >/dev/null 2>&1; then
  RELEASE=$(gh release list --repo odoo/odoo-ls --limit 5 --json tagName | jq -r '.[] | select(.tagName | startswith("1.3")) | .tagName' | head -1)
else
  # Fallback: curl (may fail due to API encoding issues)
  RELEASE=$(curl -s https://api.github.com/repos/odoo/odoo-ls/releases | grep -o '"tag_name": "1\.3[^"]*"' | head -1 | cut -d'"' -f4)
fi

echo "TARGET_RELEASE=${RELEASE}"
```

**Expected:** `TARGET_RELEASE` starts with `1.3` (e.g., `1.3.1`). If empty, retry with `curl` or check network connectivity.

### 1b. Download and install

**CRITICAL:** Tags are `1.3.1` (no `v` prefix). Extract to a subdirectory, not `/tmp` root (avoids `utime` permission errors).

```bash
mkdir -p ~/.local/bin ~/.local/share/odoo-ls /tmp/odoo-ls-extract

# Idempotency: skip binary download if version already matches
SKIP_BINARY=false
if command -v odoo_ls_server >/dev/null 2>&1; then
  INSTALLED=$(odoo_ls_server --version 2>&1 | grep -oE 'v?[0-9]+\.[0-9]+\.[0-9]+' | head -1 | sed 's/^v//')
  if [ "${INSTALLED}" = "${RELEASE}" ]; then
    echo "odoo_ls_server ${RELEASE} already installed — skipping download"
    SKIP_BINARY=true
  fi
fi

if [ "$SKIP_BINARY" = false ]; then
  # Download binary
  curl -L -o /tmp/odoo-ls.tar.gz \
    "https://github.com/odoo/odoo-ls/releases/download/${RELEASE}/odoo-linux-x86_64-${RELEASE}.tar.gz"

  tar -xzf /tmp/odoo-ls.tar.gz -C /tmp/odoo-ls-extract/
  cp /tmp/odoo-ls-extract/odoo_ls_server ~/.local/bin/
  chmod +x ~/.local/bin/odoo_ls_server
fi

# Idempotency: skip typeshed if already present
if [ -f ~/.local/share/odoo-ls/typeshed/stdlib/builtins.pyi ]; then
  echo "Typeshed already installed — skipping download"
else
  # Download typeshed from same release
  curl -L -o /tmp/typeshed.zip \
    "https://github.com/odoo/odoo-ls/releases/download/${RELEASE}/typeshed.zip"
  unzip -o /tmp/typeshed.zip -d ~/.local/share/odoo-ls/
fi

# Cleanup
rm -rf /tmp/odoo-ls-extract /tmp/odoo-ls.tar.gz /tmp/typeshed.zip

# Ensure PATH
grep -q '\.local/bin' ~/.bashrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
```

**Validation:**
```bash
odoo_ls_server --version
ls ~/.local/share/odoo-ls/typeshed/stdlib/builtins.pyi >/dev/null 2>&1 && echo "TYPESHED_OK"
```
**Expected:** Version shows `1.3.x`. `TYPESHED_OK`.

---

## Step 2: Create Shared Virtual Environment

### 2a. Create and install Odoo

The LSP server runs on your host (not in Docker) and needs a Python interpreter that matches the container's Python version exactly. Type resolution depends on version-specific bytecode and stdlib paths.

```bash
# Shared venv location: one venv per (Odoo version, Python version) pair.
# Multiple projects using the same versions reuse this venv safely —
# the LSP uses it for type resolution only, never for code execution.
VENV_DIR="$HOME/.local/share/odoo-ls/venvs/odoo${ODOO_VERSION}-py${PYTHON_VERSION}"

mkdir -p "${VENV_DIR}"

[ -d "${VENV_DIR}/bin" ] || \
  "$HOME/.pyenv/versions/${PYTHON_VERSION}/bin/python3" -m venv "${VENV_DIR}"

source "${VENV_DIR}/bin/activate"
pip install --upgrade pip

# Re-run on every setup (last-write wins): ensures the venv reflects
# the current project's Odoo source. Safe because installs are --no-deps.
pip install -e "${PROJECT_ROOT}/${ODOO_SRC}" --no-deps
```

### 2b. Import smoke test (CRITICAL)

Verify Odoo imports **before** deactivating the venv. Run **from outside the project directory** to avoid `odoo/` directory shadowing:

```bash
cd /tmp && "${VENV_DIR}/bin/python3" -c "import odoo; from odoo import models; print('IMPORT_OK')"
```

**Expected:** `IMPORT_OK`

**If it fails:** Read the error. Install the missing package, re-run.
```bash
# Example: No module named 'dateutil'
pip install python-dateutil
# Re-run smoke test
```

Repeat until `IMPORT_OK`. Common missing packages: `python-dateutil`, `pytz`, `werkzeug`, `lxml`, `psycopg2-binary`.

**Note:** In pre-release versions (1.3.x), missing dependencies may cause diagnostics or cross-reference failures even if basic `import odoo` succeeds. Install common packages aggressively if you encounter LSP errors.

**Validation:**
```bash
deactivate
```

---

## Step 3: Create or Update opencode.json

Merge LSP configuration at project root. Preserve existing settings.

```bash
if [ ! -f "opencode.json" ]; then
  cat > opencode.json << 'EOF'
{
  "$schema": "https://opencode.ai/config.json",
  "lsp": {
    "pyright": { "disabled": true },
    "odoo-ls": {
      "command": ["odoo_ls_server"],
      "extensions": [".py", ".xml", ".csv"],
      "initialization": { "selectedProfile": "default" }
    }
  }
}
EOF
else
  jq '.lsp.pyright = {"disabled": true} |
      .lsp["odoo-ls"] = {
        "command": ["odoo_ls_server"],
        "extensions": [".py", ".xml", ".csv"],
        "initialization": {"selectedProfile": "default"}
      }' opencode.json > opencode.json.tmp && mv opencode.json.tmp opencode.json
fi
```

**Validation:**
```bash
jq -e '.lsp["odoo-ls"].extensions | contains([".csv"])' opencode.json >/dev/null && echo "CONFIG_OK"
```
**Expected:** `CONFIG_OK`

---

## Step 4: Create or Update odools.toml

### 4a. Build addon paths dynamically

**For non-Doodba projects:** Adapt the paths below to match your layout. The LSP setup itself is the same for any Odoo environment.

```bash
# Find all directories containing Odoo addons (have at least one __manifest__.py)
# Excludes the core Odoo source directory
ADDON_DIRS=$(find "${ODOO_SRC}/.." -maxdepth 1 -type d ! -name "$(basename ${ODOO_SRC})" ! -name ".*" | while read dir; do
  if [ -n "$(find "$dir" -maxdepth 2 -name "__manifest__.py" -print -quit 2>/dev/null)" ]; then
    echo "$dir"
  fi
done | sort)
```

**Note on `private/`:** Doodba projects often have `odoo/custom/src/private/` for custom modules. It is not listed in `addons.yaml` but should be included if it contains modules. The filter above will include it automatically if it has `__manifest__.py` files.

### 4b. Determine absolute paths

```bash
PYTHON_PATH="${VENV_DIR}/bin/python3"
TYPESHED=$(realpath ~/.local/share/odoo-ls/typeshed/stdlib/)/
```

**CRITICAL:** `stdlib` MUST end with `/`.

### 4c. Write config

Preserve existing `addons_paths` if present:

```bash
if [ -f "odools.toml" ] && grep -q "addons_paths" odools.toml; then
  EXISTING_ADDONS=$(sed -n '/addons_paths = \[/,/\]/p' odools.toml)
else
  EXISTING_ADDONS="addons_paths = [\n$(echo "$ADDON_DIRS" | sed 's|^|  "\${workspaceFolder}/|' | sed 's|$|",|')\n]"
fi

cat > odools.toml << EOF
[[config]]
name = "default"
odoo_path = "\${workspaceFolder}/${ODOO_SRC}"
python_path = "${PYTHON_PATH}"
stdlib = "${TYPESHED}"
diag_missing_imports = "only_odoo"
refresh_mode = "adaptive"

${EXISTING_ADDONS}
EOF
```

**Example complete odools.toml (Doodba project):**

Use this as a reference to validate your generated config:

```toml
[[config]]
name = "default"
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"
python_path = "/home/roly/.local/share/odoo-ls/venvs/odoo17-py3.11/bin/python3"
stdlib = "/home/roly/.local/share/odoo-ls/typeshed/stdlib/"
diag_missing_imports = "only_odoo"
refresh_mode = "adaptive"

addons_paths = [
  "${workspaceFolder}/odoo/custom/src/account-reconcile",
  "${workspaceFolder}/odoo/custom/src/bank-payment",
  "${workspaceFolder}/odoo/custom/src/custom-modules",
  "${workspaceFolder}/odoo/custom/src/private",
  "${workspaceFolder}/odoo/custom/src/web",
]
```

**Field notes:**
- `odoo_path` uses `${workspaceFolder}` which is expanded by the LSP server at runtime
- `python_path` must be absolute — the TOML parser does not expand variables here
- `stdlib` must end with `/` (see Appendix A)

**Validation:**
```bash
python3 -c "import tomllib; d=tomllib.load(open('odools.toml','rb')); assert d['config'][0]['name']=='default'; assert 'python_path' in d['config'][0]; print('ODOOLS_OK')"
```
**Expected:** `ODOOLS_OK`

---

## Step 5: Verify LSP Configuration

### 5a. Parse mode test (standalone)

Tests that the server can index the project without OpenCode running.

**IMPORTANT (v1.3.2+):** The `--tracked-folders` parameter is now **mandatory** when using `--parse`. This specifies which folders contain your custom addon code.

```bash
odoo_ls_server --parse \
  -c "${ODOO_SRC}" \
  --tracked-folders "odoo/custom/src/private" \
  -a "odoo/custom/src/account-reconcile" \
  -a "odoo/custom/src/bank-payment" \
  --python "${PYTHON_PATH}" \
  --stdlib "${TYPESHED}" \
  -o /tmp/diagnostics.json \
  --log-level warn
```

**Validation:**
```bash
echo "EXIT_CODE=$?"
python3 -c "import json; data=json.load(open('/tmp/diagnostics.json')); print('DIAGNOSTICS_OK')"
```

**Expected:** `EXIT_CODE=0`, `DIAGNOSTICS_OK`

**Note:** In 1.3.x, using `--config-path` in parse mode may show `Invalid key (workspaceFolder)` errors. These are **non-blocking** — validate by exit code and JSON output, not by log absence. In CLI `--parse` mode, variables like `${workspaceFolder}` are NOT expanded—use the `--tracked-folders` parameter instead.

### 5b. OpenCode integration test

OpenCode auto-starts the LSP server when opening files of configured extensions. Once indexing completes, OpenCode **automatically injects diagnostics into tool results after every file edit** — no explicit command required.

Monitor indexing progress to know when diagnostics are ready:

```bash
# Find the logs directory
opencode debug paths
# Then monitor the LSP logs
tail -f <logs_directory>/opencode-lsp-*.log | grep -E "loadingStatusUpdate|ERROR|WARN"
# When you see: loadingStatusUpdate: "stop"
# Indexing is complete and diagnostics are ready
```

Verify diagnostics work for each extension:

```bash
# Find real files to test
PY_FILE=$(find odoo/custom/src -name "*.py" -path "*/models/*" | head -1)
XML_FILE=$(find odoo/custom/src -name "*.xml" | head -1)
CSV_FILE=$(find odoo/custom/src -name "*.csv" | head -1)

# Test diagnostics for each extension
opencode debug lsp diagnostics "$PY_FILE"
opencode debug lsp diagnostics "$XML_FILE"
opencode debug lsp diagnostics "$CSV_FILE"
```

**Expected:** Each command returns a large JSON object with diagnostics. The output includes diagnostics for all indexed files, not just the specified one.

### 5c. Active queries test (optional)

For cross-reference capabilities (`workspaceSymbol`, `goToDefinition`, `hover`), set the experimental flag:

```bash
OPENCODE_EXPERIMENTAL_LSP_TOOL=true opencode run \
  "Use the lsp tool to query documentSymbol for the file at ${PY_FILE}"
```

**Note:** The first call may fail with `No LSP server available` (cold-start race condition). Retry after a few seconds. Complex cross-references (`goToDefinition`, `hover` on model classes) may timeout on large projects.

---

## Step 6: Post-Setup Feedback

After successful setup, the agent **MUST** ask the human operator before creating any PR.

```
Setup completed successfully. Friction points detected:
- [List any issues encountered: missing deps, path detection ambiguity, etc.]

Proposed improvements to the setup guide (generic, version-agnostic):
- [List improvements]

Source instructions: https://github.com/rrebollo/opencode-odoo-ls
Do you want me to create a PR with these improvements? (yes/no)
```

**Rules:** No PR without explicit authorization. Frame all changes generically.

---

## Appendix A: Field Reference

| File | Field | Value | Notes |
|------|-------|-------|-------|
| `opencode.json` | `lsp.odoo-ls.extensions` | `[ ".py", ".xml", ".csv" ]` | `.csv` requires 1.3.x+ |
| `opencode.json` | `lsp.odoo-ls.initialization.selectedProfile` | `"default"` | Must match `odools.toml` profile |
| `odools.toml` | `python_path` | `~/.local/share/odoo-ls/venvs/odoo<VER>-py<PYVER>/bin/python3` | Absolute path; shared across projects with same version pair |
| `odools.toml` | `stdlib` | `.../typeshed/stdlib/` | **Must end with `/`** |
| `odools.toml` | `refresh_mode` | `"adaptive"` | OpenCode only sends `didChange` |
| `odools.toml` | `diag_missing_imports` | `"only_odoo"` | Suppresses Docker-only deps |

## Appendix B: Troubleshooting Quick Reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| `import odoo` fails | Missing deps or local `odoo/` shadowing | `pip install <missing>`; test from `/tmp` |
| `psycopg2` build failure | Missing `pg_config` | `pip install psycopg2-binary` |
| Parse mode exit code ≠ 0 | Wrong paths or broken import | Re-run smoke test; verify all CLI flags |
| `Invalid key (workspaceFolder)` in parse mode | 1.3.x `--config-path` limitation | Non-blocking — check exit code + JSON |
| `No LSP server available` on first call | Cold-start race condition | Wait 5s, retry |
| `goToDefinition`/`hover` timeout | Complex model cross-references | Simpler queries work; full indexing may need time |
