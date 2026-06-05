# Odoo LSP Setup for Doodba Projects

**Goal:** Configure `odoo_ls_server` for OpenCode in a Doodba project so agents receive Odoo-aware diagnostics on every file edit.

**Architecture:** The LSP server runs on the host (not in Docker) using a shared Python virtual environment per (Odoo version, Python version) pair. OpenCode auto-starts the server when opening `.py`/`.xml`/`.csv` files and injects diagnostics into tool results.

**Tech Stack:** `odoo_ls_server`, `pyenv`, `pip`, `opencode.json`, `odools.toml`

**Target:** Doodba projects using Odoo 16–18+ on Linux.

---

## Task 0: Detect Environment

**Purpose:** Gather project metadata. Keep this shell session alive — all detected variables carry forward.

- [ ] **Run environment detection script**

```bash
# Capture project root
PROJECT_ROOT=$(pwd)
echo "PROJECT_ROOT=${PROJECT_ROOT}"

# Verify Doodba
grep -q 'Tecnativa/doodba-copier-template' .copier-answers.yml 2>/dev/null && echo "DOODBA_OK" || echo "NON_DOODBA"

# Detect Odoo version
ODOO_VERSION_FULL=$(grep -oP 'odoo_version:\s*\K[\d.]+' .copier-answers.yml)
ODOO_VERSION=$(echo "${ODOO_VERSION_FULL}" | cut -d. -f1)
echo "ODOO_VERSION_FULL=${ODOO_VERSION_FULL}"
echo "ODOO_VERSION=${ODOO_VERSION}"

# Detect Python version from Docker image
PYTHON_VERSION=$(docker image inspect ghcr.io/tecnativa/doodba:${ODOO_VERSION_FULL}-onbuild 2>/dev/null | grep -oP 'PYTHON_VERSION=\K[^"]+' || echo "MISSING")
echo "PYTHON_VERSION=${PYTHON_VERSION}"
if [ "${PYTHON_VERSION}" = "MISSING" ]; then
  echo "ERROR: Confirm image exists: ghcr.io/tecnativa/doodba:${ODOO_VERSION_FULL}-onbuild"
  exit 1
fi

# Detect tools
for tool in curl git docker pyenv jq gh unzip; do
  command -v "$tool" >/dev/null 2>&1 && echo "$tool: OK" || echo "$tool: MISSING"
done

# Detect Odoo source dynamically
ODOO_SRC=$(find . -maxdepth 5 -name "setup.py" -path "*/odoo/*" ! -path "*/odoo/odoo/*" 2>/dev/null | head -1 | xargs -r dirname)
[ -z "$ODOO_SRC" ] && ODOO_SRC=$(find . -maxdepth 3 -name "setup.py" -exec grep -l "find_packages" {} \; 2>/dev/null | head -1 | xargs dirname)
echo "ODOO_SRC=${ODOO_SRC}"
if [ -z "${ODOO_SRC}" ]; then
  echo "ERROR: Expected a setup.py under */odoo/ at depth ≤5."
  exit 1
fi
```

- [ ] **Resolve pyenv Python version** (captures `PYTHON_VERSION` as 2-number minor, `PYENV_VERSION` for install path)

```bash
PYTHON_MINOR=$(echo "${PYTHON_VERSION}" | cut -d. -f1-2)
PYENV_VERSION=$(pyenv versions --bare | grep -E "^${PYTHON_MINOR}\.[0-9]+$" | tail -1)
if [ -z "${PYENV_VERSION}" ]; then
  PYENV_VERSION=$(pyenv install --list | grep -E "^\s+${PYTHON_MINOR}\.[0-9]+$" | tail -1 | tr -d ' ')
  pyenv install -s "${PYENV_VERSION}"
fi
PYTHON_VERSION="${PYTHON_MINOR}"
echo "PYTHON_VERSION=${PYTHON_VERSION}"
```

- [ ] **Validate all required variables**

```bash
_SETUP_OK=true
for _var in PROJECT_ROOT ODOO_VERSION_FULL ODOO_VERSION PYTHON_VERSION ODOO_SRC; do
  eval _val=\$$_var
  if [ -z "${_val}" ]; then
    echo "ERROR: ${_var} is empty"
    _SETUP_OK=false
  fi
done
if [ "${_SETUP_OK}" = "false" ]; then
  echo "Environment detection incomplete — fix errors above before continuing"
  exit 1
fi
echo "Environment detection OK"
echo "  Odoo:   ${ODOO_VERSION_FULL}"
echo "  Python: ${PYTHON_VERSION}"
echo "  Source: ${ODOO_SRC}"
```

**Expected:** All tools show `OK` or `MISSING` (only `gh` may be missing). `ODOO_SRC` points to the directory containing `setup.py`. Final output shows `Environment detection OK`.

---

## Task 1: Install odoo_ls_server and Typeshed

- [ ] **Detect latest 1.3.x pre-release**

```bash
if command -v gh >/dev/null 2>&1; then
  RELEASE=$(gh release list --repo odoo/odoo-ls --limit 5 --json tagName | jq -r '.[] | select(.tagName | startswith("1.3")) | .tagName' | head -1)
else
  RELEASE=$(curl -s https://api.github.com/repos/odoo/odoo-ls/releases | grep -o '"tag_name": "1\.3[^"]*"' | head -1 | cut -d'"' -f4)
fi
echo "TARGET_RELEASE=${RELEASE}"
```

**Expected:** `TARGET_RELEASE` starts with `1.3`. If empty, check network.

- [ ] **Download binary and typeshed**

```bash
mkdir -p ~/.local/bin ~/.local/share/odoo-ls /tmp/odoo-ls-extract

SKIP_BINARY=false
if command -v odoo_ls_server >/dev/null 2>&1; then
  INSTALLED=$(odoo_ls_server --version 2>&1 | grep -oE 'v?[0-9]+\.[0-9]+\.[0-9]+' | head -1 | sed 's/^v//')
  if [ "${INSTALLED}" = "${RELEASE}" ]; then
    echo "odoo_ls_server ${RELEASE} already installed — skipping download"
    SKIP_BINARY=true
  fi
fi

if [ "$SKIP_BINARY" = false ]; then
  curl -L -o /tmp/odoo-ls.tar.gz \
    "https://github.com/odoo/odoo-ls/releases/download/${RELEASE}/odoo-linux-x86_64-${RELEASE}.tar.gz"
  tar -xzf /tmp/odoo-ls.tar.gz -C /tmp/odoo-ls-extract/
  cp /tmp/odoo-ls-extract/odoo_ls_server ~/.local/bin/
  chmod +x ~/.local/bin/odoo_ls_server
fi

if [ -f ~/.local/share/odoo-ls/typeshed/stdlib/builtins.pyi ]; then
  echo "Typeshed already installed — skipping download"
else
  curl -L -o /tmp/typeshed.zip \
    "https://github.com/odoo/odoo-ls/releases/download/${RELEASE}/typeshed.zip"
  unzip -o /tmp/typeshed.zip -d ~/.local/share/odoo-ls/
fi

rm -rf /tmp/odoo-ls-extract /tmp/odoo-ls.tar.gz /tmp/typeshed.zip
grep -q '\.local/bin' ~/.bashrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
```

- [ ] **Verify installation**

```bash
odoo_ls_server --version
ls ~/.local/share/odoo-ls/typeshed/stdlib/builtins.pyi >/dev/null 2>&1 && echo "TYPESHED_OK"
```

**Expected:** Version shows `1.3.x`. `TYPESHED_OK`.

---

## Task 2: Create Shared Virtual Environment

- [ ] **Create and install Odoo**

```bash
VENV_DIR="$HOME/.local/share/odoo-ls/venvs/odoo${ODOO_VERSION}-py${PYTHON_VERSION}"
mkdir -p "${VENV_DIR}"

[ -d "${VENV_DIR}/bin" ] || \
  "$HOME/.pyenv/versions/${PYENV_VERSION}/bin/python3" -m venv "${VENV_DIR}"

source "${VENV_DIR}/bin/activate"
pip install --upgrade pip

pip install -e "${PROJECT_ROOT}/${ODOO_SRC}" --no-deps

if [ -f "${PROJECT_ROOT}/${ODOO_SRC}/requirements.txt" ]; then
  grep -v '^\s*#' "${PROJECT_ROOT}/${ODOO_SRC}/requirements.txt" \
    | sed 's/\s*#.*//' \
    | grep -v '^\s*$' > /tmp/odoo-reqs-clean.txt
  pip install -r /tmp/odoo-reqs-clean.txt
fi
```

- [ ] **Run smoke test** (run from outside project directory to avoid `odoo/` shadowing)

```bash
cd /tmp && "${VENV_DIR}/bin/python3" -c "import odoo; from odoo import models; print('IMPORT_OK')"
```

**Expected:** `IMPORT_OK`

**If it fails:** Read the error, then:
```bash
source "${VENV_DIR}/bin/activate"
pip install <missing-package>
# Re-run smoke test
```

Repeat until `IMPORT_OK`. Install only what the error requests — blind bulk installs may cause version conflicts.

- [ ] **Deactivate venv**

```bash
echo "IMPORT_OK"
deactivate
```

---

## Task 3: Create or Update opencode.json

- [ ] **Merge LSP configuration at project root**

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

- [ ] **Verify**

```bash
jq -e '.lsp["odoo-ls"].extensions | contains([".csv"])' opencode.json >/dev/null && echo "CONFIG_OK"
```

**Expected:** `CONFIG_OK`

---

## Task 4: Create or Update odools.toml

- [ ] **Build addon paths**

```bash
ADDON_DIRS=$(find "$(realpath ${ODOO_SRC}/..)" -maxdepth 1 -type d ! -name "$(basename ${ODOO_SRC})" ! -name ".*" | while read dir; do
  if [ -n "$(find "$dir" -maxdepth 2 -name "__manifest__.py" -print -quit 2>/dev/null)" ]; then
    echo "$dir"
  fi
done | sort)
```

- [ ] **Set path variables**

```bash
PYTHON_PATH="${VENV_DIR}/bin/python3"

if [ ! -d ~/.local/share/odoo-ls/typeshed/stdlib/ ]; then
  echo "ERROR: typeshed not found. Run Task 1 first."
  exit 1
fi
TYPESHED=$(realpath ~/.local/share/odoo-ls/typeshed/stdlib/)/
```

- [ ] **Write config**

```bash
if [ -f "odools.toml" ] && grep -q "addons_paths" odools.toml; then
  EXISTING_ADDONS=$(sed -n '/addons_paths = \[/,/\]/p' odools.toml)
else
  EXISTING_ADDONS="addons_paths = [\n$(echo "$ADDON_DIRS" | sed "s|^${PROJECT_ROOT}/|  \"\${workspaceFolder}/|" | sed 's|$|",|')\n]"
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

- [ ] **Verify**

```bash
python3 -c "import tomllib; d=tomllib.load(open('odools.toml','rb')); assert d['config'][0]['name']=='default'; assert 'python_path' in d['config'][0]; print('ODOOLS_OK')"
```

**Expected:** `ODOOLS_OK`

---

## Task 5: Standalone Parse Test

- [ ] **Run parse mode**

```bash
TRACKED_FLAGS=()
while IFS= read -r dir; do
  TRACKED_FLAGS+=(--tracked-folders "${dir}")
done <<< "$ADDON_DIRS"

odoo_ls_server --parse \
  -c "${ODOO_SRC}" \
  "${TRACKED_FLAGS[@]}" \
  --python "${PYTHON_PATH}" \
  --stdlib "${TYPESHED}" \
  -o /tmp/diagnostics.json \
  --log-level warn
echo "EXIT_CODE=$?"
```

- [ ] **Verify output**

```bash
python3 -c "import json; data=json.load(open('/tmp/diagnostics.json')); print('DIAGNOSTICS_OK')"
```

**Expected:** `EXIT_CODE=0`, `DIAGNOSTICS_OK`

**Note:** `Invalid key (workspaceFolder)` warnings in logs are non-blocking — validate by exit code and JSON output, not log absence.

---

## Task 6: OpenCode Integration Test

- [ ] **Monitor indexing progress** (in a separate terminal)

```bash
opencode debug paths
tail -f <logs_directory>/opencode-lsp-*.log | grep -E "loadingStatusUpdate|ERROR|WARN"
# When you see: loadingStatusUpdate: "stop" — indexing is complete
```

- [ ] **Find test files**

```bash
if [ -n "${ADDON_DIRS:-}" ]; then
  PY_FILE=$(find ${ADDON_DIRS} -name "*.py" -path "*/models/*" 2>/dev/null | head -1)
  XML_FILE=$(find ${ADDON_DIRS} -name "*.xml" 2>/dev/null | head -1)
  CSV_FILE=$(find ${ADDON_DIRS} -name "*.csv" 2>/dev/null | head -1)
else
  # Fallback for Doodba layout
  PY_FILE=$(find odoo/custom/src -name "*.py" -path "*/models/*" | head -1)
  XML_FILE=$(find odoo/custom/src -name "*.xml" | head -1)
  CSV_FILE=$(find odoo/custom/src -name "*.csv" | head -1)
fi
```

- [ ] **Test diagnostics for each extension**

```bash
opencode debug lsp diagnostics "$PY_FILE"
opencode debug lsp diagnostics "$XML_FILE"
opencode debug lsp diagnostics "$CSV_FILE"
```

**Expected:** Each command returns a JSON object with diagnostics. The output includes diagnostics for all indexed files, not just the specified one.

- [ ] **(Optional) Test cross-references**

```bash
OPENCODE_EXPERIMENTAL_LSP_TOOL=true opencode run \
  "Use the lsp tool to query documentSymbol for the file at ${PY_FILE}"
```

**Note:** First call may fail with `No LSP server available` (cold-start race condition). Retry after a few seconds.

---

## Task 7: Post-Setup Report

- [ ] **Summarise findings** in this format:

```text
Setup completed successfully. Friction points detected:
- [List any issues encountered: missing deps, path detection ambiguity, etc.]

Proposed improvements to the setup guide (generic, version-agnostic):
- [List improvements]

Source instructions: https://github.com/rrebollo/opencode-odoo-ls
Do you want me to raise a GitHub issue with these findings? (yes/no)
```

- [ ] **If yes, create GitHub issue** (never without explicit user confirmation)

```bash
gh issue create \
  --repo rrebollo/opencode-odoo-ls \
  --title "Setup friction: [brief summary]" \
  --body "[friction points and proposed improvements]"
```

**Rules:** No issue without explicit user authorization. Frame findings generically and version-agnostically.

---

## Appendix A: Field Reference

| File | Field | Value | Notes |
|------|-------|-------|-------|
| `opencode.json` | `lsp.odoo-ls.extensions` | `[ ".py", ".xml", ".csv" ]` | `.csv` requires `odoo_ls_server` 1.3.x+ |
| `opencode.json` | `lsp.odoo-ls.initialization.selectedProfile` | `"default"` | Must match `odools.toml` profile name |
| `odools.toml` | `python_path` | `~/.local/share/odoo-ls/venvs/odoo<VER>-py<PYVER>/bin/python3` | Absolute path; shared across projects with same version pair. `<VER>` is major only (e.g. `17`), `<PYVER>` is 2-number minor (e.g. `3.11`) |
| `odools.toml` | `stdlib` | `.../typeshed/stdlib/` | **Must end with `/`** |
| `odools.toml` | `refresh_mode` | `"adaptive"` | OpenCode only sends `didChange` |
| `odools.toml` | `diag_missing_imports` | `"only_odoo"` | Suppresses warnings for Docker-only deps |
