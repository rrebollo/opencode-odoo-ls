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

- [ ] **Detect latest 1.5.x pre-release**

```bash
if command -v gh >/dev/null 2>&1; then
  RELEASE=$(gh release list --repo odoo/odoo-ls --limit 5 --json tagName | jq -r '.[] | select(.tagName | startswith("1.5")) | .tagName' | head -1)
else
  RELEASE=$(curl -s https://api.github.com/repos/odoo/odoo-ls/releases | grep -o '"tag_name": "1\.5[^"]*"' | head -1 | cut -d'"' -f4)
fi
echo "TARGET_RELEASE=${RELEASE}"

# Fetch config schema for this release (used for config validation)
mkdir -p ~/.local/share/odoo-ls
curl -sL "https://github.com/odoo/odoo-ls/releases/download/${RELEASE}/config_schema.json" \
  -o ~/.local/share/odoo-ls/config_schema.json
echo "CONFIG_SCHEMA=${HOME}/.local/share/odoo-ls/config_schema.json"
```

**Expected:** `TARGET_RELEASE` starts with `1.5`. `CONFIG_SCHEMA` points to a valid JSON file. If empty, check network.

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

if [ -f ~/.local/share/odoo-ls/stdlib/builtins.pyi ]; then
  echo "Typeshed already installed — skipping download"
else
  curl -L -o /tmp/typeshed.zip \
    "https://github.com/odoo/odoo-ls/releases/download/${RELEASE}/typeshed.zip"
  unzip -o /tmp/typeshed.zip -d ~/.local/share/odoo-ls/
fi

rm -rf /tmp/odoo-ls-extract /tmp/odoo-ls.tar.gz /tmp/typeshed.zip ~/.local/share/odoo-ls/typeshed 2>/dev/null
grep -q '\.local/bin' ~/.bashrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
```

- [ ] **Verify installation**

```bash
odoo_ls_server --version
ls ~/.local/share/odoo-ls/stdlib/builtins.pyi >/dev/null 2>&1 && echo "TYPESHED_OK"
```

**Expected:** Version shows `1.5.x`. `TYPESHED_OK`.

---

## Task 1.5: Install TypeScript 6 (tsserver)

**Required for JavaScript/OWL/template features in odoo-ls 1.5+**

- [ ] **Install TypeScript 6 globally**

```bash
if ! command -v tsserver >/dev/null 2>&1; then
  npm install -g typescript@6
fi
node -e "console.log('TypeScript', JSON.parse(require('fs').readFileSync(process.argv[1],'utf8')).version)" "$(npm root -g)/typescript/package.json"
```

**Expected:** Output shows `Version 6.x.x` (not 7.x)

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

**Common build failures (use binary wheels as fallback):**
- `gevent` C extension fails → `pip install gevent` (gets a binary wheel ≥23.x)
- `psycopg2` needs `pg_config` → `pip install psycopg2-binary` (same API for dev setups)
- Other packages with C extensions → prefer binary wheels via `pip install <package> --only-binary :all:`

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
      "extensions": [".py", ".xml", ".csv", ".js"],
      "initialization": { "selectedProfile": "default" }
    }
  }
}
EOF
else
  jq '.lsp.pyright = {"disabled": true} |
      .lsp["odoo-ls"] = {
        "command": ["odoo_ls_server"],
        "extensions": [".py", ".xml", ".csv", ".js"],
        "initialization": {"selectedProfile": "default"}
      }' opencode.json > opencode.json.tmp && mv opencode.json.tmp opencode.json
fi
```

- [ ] **Verify**

```bash
jq -e '.lsp["odoo-ls"].extensions | contains([".js"])' opencode.json >/dev/null && echo "CONFIG_OK"
```

**Expected:** `CONFIG_OK`

---

## Task 4: Create or Update odools.toml

- [ ] **Build addon paths**

```bash
# Find category-level directories (e.g. odoo/custom/src/oca-16/account) containing addons.
# This avoids scanning hundreds of individual addon subdirectories and keeps --parse fast.
ADDON_DIRS=$(find "$(realpath ${ODOO_SRC}/..)" -maxdepth 2 -mindepth 2 -type d ! -name "$(basename ${ODOO_SRC})" ! -name ".*" | while read dir; do
  if [ -n "$(find "$dir" -name "__manifest__.py" -print -quit 2>/dev/null)" ]; then
    echo "$dir"
  fi
done | sort -u)
```

- [ ] **Set path variables**

```bash
PYTHON_PATH="${VENV_DIR}/bin/python3"

if [ ! -d ~/.local/share/odoo-ls/stdlib/ ]; then
  echo "ERROR: typeshed not found. Run Task 1 first."
  exit 1
fi
TYPESHED=$(realpath ~/.local/share/odoo-ls/stdlib/)/
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
# refresh_mode = "adaptive"  # Not in config_schema.json as of 1.5.x; may be ignored by server

# JavaScript/OWL support (odoo-ls 1.5+)
# disable_javascript = false   # (default) enable JS/OWL features
ts_check = true                 # enable TypeScript diagnostics in JS files
tsserver_command = "tsserver"   # uses globally installed TypeScript 6

${EXISTING_ADDONS}
EOF
```

- [ ] **Verify**

```bash
python3 -c "import tomllib; d=tomllib.load(open('odools.toml','rb')); assert d['config'][0]['name']=='default'; assert 'python_path' in d['config'][0]; print('ODOOLS_OK')"
```

**Expected:** `ODOOLS_OK`

---

## Task 4.5: Config Schema Reference

**Purpose:** The `config_schema.json` file (fetched in Task 1) defines all valid `odools.toml` options. Agents should use it to validate configurations during setup or when troubleshooting.

- [ ] **Validate odools.toml against schema**

```bash
# Quick validation: check that all keys in odools.toml are defined in the schema
python3 -c "
import tomllib, json, sys

with open('odools.toml', 'rb') as f:
    config = tomllib.load(f)

with open('${HOME}/.local/share/odoo-ls/config_schema.json') as f:
    schema = json.load(f)

valid_keys = set(schema['config']['items']['properties'].keys())
for profile in config.get('config', []):
    unknown = set(profile.keys()) - valid_keys
    if unknown:
        print(f'WARNING: Unknown keys in profile \"{profile.get(\"name\", \"?\")}\": {unknown}')
        sys.exit(1)

print('SCHEMA_VALIDATION_OK')
"
```

**Expected:** `SCHEMA_VALIDATION_OK` or a list of unknown keys.

**Key options (odoo-ls 1.5+):**

| Option | Type | Default | Description |
|---|---|---|---|
| `tsserver_command` | string | `"tsserver"` | Path/command for TypeScript tsserver (JS/OWL support) |
| `disable_javascript` | boolean | `false` | Set `true` to disable JS/OWL features |
| `ts_check` | boolean | `false` | Enable TypeScript diagnostics in JS files |
| `disable_semantic_tokens_python` | boolean | `false` | Disable Python semantic tokens |
| `disable_semantic_tokens_javascript` | boolean | `false` | Disable JavaScript semantic tokens |
| `disable_semantic_tokens_xml` | boolean | `false` | Disable XML semantic tokens |

**Note:** For the full list of valid options, refer to the schema at `~/.local/share/odoo-ls/config_schema.json` or fetch it dynamically from the latest release.

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
| `opencode.json` | `lsp.odoo-ls.extensions` | `[ ".py", ".xml", ".csv", ".js" ]` | `.csv` and `.js` require `odoo_ls_server` 1.5.x+ |
| `opencode.json` | `lsp.odoo-ls.initialization.selectedProfile` | `"default"` | Must match `odools.toml` profile name |
| `odools.toml` | `python_path` | `~/.local/share/odoo-ls/venvs/odoo<VER>-py<PYVER>/bin/python3` | Absolute path; shared across projects with same version pair. `<VER>` is major only (e.g. `17`), `<PYVER>` is 2-number minor (e.g. `3.11`) |
| `odools.toml` | `stdlib` | `.../stdlib/` | **Must end with `/`** |
| `odools.toml` | `# refresh_mode` | `"adaptive"` | **Not in config_schema.json** — may be ignored. Commented out by default. |
| `odools.toml` | `diag_missing_imports` | `"only_odoo"` | Suppresses warnings for Docker-only deps |
| `odools.toml` | `disable_javascript` | `false` | Enable JS/OWL/template features (1.5+). `false` = enabled |
| `odools.toml` | `ts_check` | `true` | Enable TypeScript diagnostics in JS files (1.5+) |
| `odools.toml` | `tsserver_command` | `"tsserver"` | Path to globally installed TypeScript 6 tsserver (1.5+) |

### Schema-only options (available but not used by this guide)

These options are defined in `config_schema.json` and can be added to `odools.toml` if needed:

| Field | Type | Default | Description |
|---|---|---|---|
| `extends` | string | — | Name of another profile to inherit from |
| `$version` | string | — | Odoo version override |
| `$base` | string | — | Base path |
| `addons_paths` | array[string] | — | Addon directory paths (auto-detected by this guide) |
| `addons_merge` | `"merge"` \| `"override"` | — | How to merge addon paths from parent profiles |
| `additional_stubs` | array[string] | — | Extra stub directories (e.g. lxml) |
| `additional_stubs_merge` | `"merge"` \| `"override"` | — | How to merge stubs from parent profiles |
| `additional_languages` | array[string] | — | Extra language codes for data files |
| `file_cache` | boolean | `true` | Enable file-level caching |
| `ac_filter_model_names` | boolean | `true` | Filter model names in autocompletion |
| `auto_refresh_delay` | integer | `1000` | Auto-refresh delay in ms (1000–15000) |
| `diagnostic_settings` | object | — | Per-diagnostic severity overrides |
| `diagnostic_filters` | array[object] | — | Path/code/type-based diagnostic filters |
| `no_typeshed_stubs` | boolean | `false` | Disable typeshed stubs |
| `disable_semantic_tokens_python` | boolean | `false` | Disable Python semantic tokens |
| `disable_semantic_tokens_javascript` | boolean | `false` | Disable JavaScript semantic tokens |
| `disable_semantic_tokens_xml` | boolean | `false` | Disable XML semantic tokens |
