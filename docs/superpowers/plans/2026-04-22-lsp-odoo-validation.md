# Odoo LS Validation & Docker Python Integration

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Verify `odoo_ls_server` works correctly with OpenCode in headless/agent mode, fix the Python interpreter configuration for Docker projects, and document verified troubleshooting commands.

**Architecture:** Three independent investigation + fix tasks:
1. **Validation baseline** — confirm LSP indexing works outside OpenCode using `--parse` mode; collect evidence from actual logs
2. **Python on host setup** — create virtualenv with Odoo dependencies to fix import resolution when Python is isolated from Docker
3. **OpenCode integration fix** — add `initialization` field to `opencode.json` to ensure proper `workspace/configuration` responses; update AGENTS.md with troubleshooting commands

**Tech Stack:** `odoo_ls_server` v1.2.0, OpenCode LSP client, Python venv, `odools.toml`

---

## Context: Why These Tasks

**Problem discovered via investigation:**
- OpenCode's LSP client behaves differently than IDE plugins in one critical way: it does not pass `{ "selectedProfile": "default" }` via `workspace/configuration` responses (all IDE plugins do)
- The `python3` command on the host lacks Odoo dependencies, causing import resolution failures during indexing
- The 3-second timeout in OpenCode's diagnostic wait may be too short for large projects (oca-17 has 32 addon repos)

**Solution approach:**
- Fix #1 is the **highest ROI** — one-line change to `opencode.json` that aligns OpenCode with how VSCode, PyCharm, Neovim, and Zed all work
- Fix #2 (Python) ensures the LSP can resolve Odoo imports correctly
- Fix #3 is documentation and verification

---

## Task 1: Establish Baseline Evidence (LSP Indexing in Parse Mode)

**Files:**
- Read: `/home/roly/projects/binhex/OCA/oca-17/odools.toml`
- Read: `/home/roly/projects/binhex/OCA/oca-17/opencode.json`
- Run: one-shot `odoo_ls_server --parse` with oca-17 project
- Inspect: logs at `~/.local/bin/logs/`

**Why:** Confirm that the LSP itself can index the project (debugging LSP ≠ debugging OpenCode integration).

---

### Task 1.1: Verify current logs exist and check for previous errors

- [ ] **Step 1: Check if logs directory exists and what's in it**

```bash
ls -lah ~/.local/bin/logs/ || echo "No logs directory yet"
```

Expected: Either empty (first run) or contains `odoo_logs.*.log` files from previous sessions.

- [ ] **Step 2: If logs exist, check the most recent one for errors**

```bash
if [ -d ~/.local/bin/logs/ ]; then
  tail -100 $(ls -t ~/.local/bin/logs/odoo_logs.*.log 2>/dev/null | head -1)
else
  echo "No logs to inspect"
fi
```

Expected: Look for patterns like:
- `ERROR` or `panicked` — indicates crashes
- `import odoo` failures — indicates Python path issue
- `loadingStatusUpdate` with `stop` — indicates indexing completed
- Timing information — how long did indexing take?

---

### Task 1.2: Run one-shot `--parse` mode to test LSP on oca-17

- [ ] **Step 3: Create output directory for diagnostics**

```bash
mkdir -p /tmp/odoo-ls-validation
```

- [ ] **Step 4: Run parse mode with full oca-17 configuration**

```bash
cd /home/roly/projects/binhex/OCA/oca-17
odoo_ls_server --parse \
  -c odoo/custom/src/odoo \
  -a odoo/custom/src/account-reconcile \
  -a odoo/custom/src/bank-payment \
  -a odoo/custom/src/bank-statement-import \
  -a odoo/custom/src/binhex-calendar \
  -a odoo/custom/src/binhex-server-backend \
  -a odoo/custom/src/canal \
  -a odoo/custom/src/connector \
  -a odoo/custom/src/connector-interfaces \
  -a odoo/custom/src/crm \
  -a odoo/custom/src/e-learning \
  -a odoo/custom/src/event \
  -a odoo/custom/src/field-service \
  -a odoo/custom/src/hr \
  -a odoo/custom/src/hr-attendance \
  -a odoo/custom/src/hr-holidays \
  -a odoo/custom/src/partner-contact \
  -a odoo/custom/src/payroll \
  -a odoo/custom/src/private \
  -a odoo/custom/src/project \
  -a odoo/custom/src/queue \
  -a odoo/custom/src/reporting-engine \
  -a odoo/custom/src/sale-workflow \
  -a odoo/custom/src/server-backend \
  -a odoo/custom/src/server-env \
  -a odoo/custom/src/server-tools \
  -a odoo/custom/src/server-ux \
  -a odoo/custom/src/spreadsheet \
  -a odoo/custom/src/stock-logistics-transport \
  -a odoo/custom/src/stock-logistics-workflow \
  -a odoo/custom/src/storage \
  -a odoo/custom/src/timesheet \
  -a odoo/custom/src/web \
  -o /tmp/odoo-ls-validation/diagnostics.json \
  --log-level DEBUG
```

Expected: Command runs to completion (may take 1–5 minutes for 32 repos). Exit code 0.

- [ ] **Step 5: Check parse mode output for errors**

```bash
# Count total diagnostics
jq 'length' /tmp/odoo-ls-validation/diagnostics.json

# Check for common error categories (show first 10)
jq -r '.[] | select(.code | startswith("OLS")) | .code' /tmp/odoo-ls-validation/diagnostics.json | sort | uniq -c | head -20
```

Expected output structure:
```json
[
  {
    "file": "/path/to/file.py",
    "line": 123,
    "column": 4,
    "severity": "error|warning|info",
    "code": "OLS03001",
    "message": "..."
  }
  ...
]
```

If many `ModuleNotFound` or `ImportError` errors exist, that confirms Python path issue.

- [ ] **Step 6: Inspect fresh logs from parse mode run**

```bash
tail -200 $(ls -t ~/.local/bin/logs/odoo_logs.*.log 2>/dev/null | head -1) | grep -E "(ERROR|WARN|loadingStatusUpdate|import|Traceback)" | tail -50
```

Expected: Diagnostics summary, no crashes. Look for patterns indicating what failed.

---

### Task 1.3: Analyze findings

- [ ] **Step 7: Summarize findings in a report comment**

Create a text file `/tmp/odoo-ls-validation/BASELINE_REPORT.txt` with:
```
=== Baseline LSP Validation Report ===
Date: $(date)
Project: /home/roly/projects/binhex/OCA/oca-17
LSP Version: $(odoo_ls_server --version 2>&1 || echo "unknown")

1. Parse mode completed: YES/NO
2. Total diagnostics found: $(jq 'length' /tmp/odoo-ls-validation/diagnostics.json)
3. Error types:
   - Import errors (ModuleNotFound, ImportError): [COUNT]
   - Type errors (OLS codes): [COUNT]
   - Other: [COUNT]
4. Hypothesis: [Python path issue? Indexing timing? Config error?]
5. Next action: [Fix Python path / Investigate timing / Check config]
```

- [ ] **Step 8: Decide on hypothesis**

Based on Step 7 output, record your hypothesis about what's broken:
- If many `ImportError: No module named 'odoo'` → **Python issue**
- If diagnostics are empty or sparse → **LSP isn't indexing**
- If no errors → **LSP works; OpenCode integration issue**

This drives Task 2.

---

## Task 2: Virtualenv with Odoo Dependencies (Python Fix)

**Files:**
- Create: `~/.venv/odoo17/` (virtualenv)
- Modify: `/home/roly/projects/binhex/OCA/oca-17/odools.toml` (add `python_path`)
- Run: parse mode again to compare

**Why:** The host Python doesn't have Odoo libraries. LSP can't resolve `from odoo import models`. Fix: install Odoo from the source tree already on disk as an editable package.

---

### Task 2.1: Create virtualenv for Odoo 17

- [ ] **Step 1: Create venv**

```bash
python3 -m venv ~/.venv/odoo17
source ~/.venv/odoo17/bin/activate
python --version
```

Expected: Python 3.x output (matching or close to system Python).

- [ ] **Step 2: Upgrade pip**

```bash
source ~/.venv/odoo17/bin/activate
pip install --upgrade pip setuptools wheel
```

Expected: `Successfully installed` messages, no errors.

---

### Task 2.2: Install Odoo as editable package

- [ ] **Step 3: Install Odoo from source (editable mode)**

```bash
source ~/.venv/odoo17/bin/activate
cd /home/roly/projects/binhex/OCA/oca-17/odoo/custom/src/odoo
pip install -e .
```

Expected: Installation completes. Look for warning about missing `lxml`, `psycopg2-binary`, etc. — those are OK for now (we only need import resolution, not runtime).

- [ ] **Step 4: Verify `odoo` module is importable**

```bash
source ~/.venv/odoo17/bin/activate
python -c "import odoo; print(odoo.__file__)"
```

Expected output: Path to odoo module in the venv, e.g.:
```
/home/roly/.venv/odoo17/lib/python3.x/site-packages/odoo/__init__.py
```

- [ ] **Step 5: Test that LSP's `sys.path` discovery works with venv**

```bash
source ~/.venv/odoo17/bin/activate
python -c "import sys; import json; print(json.dumps(sys.path))" | jq .
```

Expected: `sys.path` contains paths under `~/.venv/odoo17/` and the odoo source directory.

---

### Task 2.3: Update `odools.toml` with new Python path

- [ ] **Step 6: Read current `odools.toml`**

```bash
cat /home/roly/projects/binhex/OCA/oca-17/odools.toml | head -30
```

- [ ] **Step 7: Add `python_path` to the `[Odoo]` section**

Edit `/home/roly/projects/binhex/OCA/oca-17/odools.toml`. Add this line in the `[Odoo]` section (near the top, before `[[config]]`):

```toml
python_path = "${userHome}/.venv/odoo17/bin/python3"
```

Or use the explicit absolute path:
```toml
python_path = "/home/roly/.venv/odoo17/bin/python3"
```

**IMPORTANT:** Check the exact location of your home directory first:
```bash
echo $HOME
```

Then use that path. If using `${userHome}`, verify it's expanded correctly by the LSP server.

- [ ] **Step 8: Verify the edit was applied**

```bash
grep python_path /home/roly/projects/binhex/OCA/oca-17/odools.toml
```

Expected: Output shows the line you just added.

---

### Task 2.4: Re-run parse mode to compare

- [ ] **Step 9: Run parse mode again with the venv Python**

```bash
cd /home/roly/projects/binhex/OCA/oca-17
odoo_ls_server --parse \
  -c odoo/custom/src/odoo \
  -a odoo/custom/src/account-reconcile \
  -a odoo/custom/src/bank-payment \
  -a odoo/custom/src/bank-statement-import \
  -a odoo/custom/src/binhex-calendar \
  -a odoo/custom/src/binhex-server-backend \
  -a odoo/custom/src/canal \
  -a odoo/custom/src/connector \
  -a odoo/custom/src/connector-interfaces \
  -a odoo/custom/src/crm \
  -a odoo/custom/src/e-learning \
  -a odoo/custom/src/event \
  -a odoo/custom/src/field-service \
  -a odoo/custom/src/hr \
  -a odoo/custom/src/hr-attendance \
  -a odoo/custom/src/hr-holidays \
  -a odoo/custom/src/partner-contact \
  -a odoo/custom/src/payroll \
  -a odoo/custom/src/private \
  -a odoo/custom/src/project \
  -a odoo/custom/src/queue \
  -a odoo/custom/src/reporting-engine \
  -a odoo/custom/src/sale-workflow \
  -a odoo/custom/src/server-backend \
  -a odoo/custom/src/server-env \
  -a odoo/custom/src/server-tools \
  -a odoo/custom/src/server-ux \
  -a odoo/custom/src/spreadsheet \
  -a odoo/custom/src/stock-logistics-transport \
  -a odoo/custom/src/stock-logistics-workflow \
  -a odoo/custom/src/storage \
  -a odoo/custom/src/timesheet \
  -a odoo/custom/src/web \
  --python /home/roly/.venv/odoo17/bin/python3 \
  -o /tmp/odoo-ls-validation/diagnostics-after-venv.json \
  --log-level DEBUG
```

Note: `--python` flag only works in `--parse` mode. It tells the server which Python to use for `sys.path` discovery.

- [ ] **Step 10: Compare before vs. after diagnostics**

```bash
echo "=== Before (system Python) ==="
jq 'map(select(.code == "OLS03001")) | length' /tmp/odoo-ls-validation/diagnostics.json

echo "=== After (venv Python) ==="
jq 'map(select(.code == "OLS03001")) | length' /tmp/odoo-ls-validation/diagnostics-after-venv.json

echo "=== Total errors before ==="
jq 'length' /tmp/odoo-ls-validation/diagnostics.json

echo "=== Total errors after ==="
jq 'length' /tmp/odoo-ls-validation/diagnostics-after-venv.json
```

Expected: Fewer errors in the second run (especially fewer `ModuleNotFound` errors).

- [ ] **Step 11: Document improvement**

Update `/tmp/odoo-ls-validation/BASELINE_REPORT.txt` with:
```
Python Fix Results:
Before venv: [COUNT] errors
After venv: [COUNT] errors
Improvement: [COUNT] errors fixed (percentage)

Conclusion: [Venv fixed the issue / Venv helped but issues remain / No improvement]
```

---

## Task 3: OpenCode Integration Fix (initialization field)

**Files:**
- Modify: `/home/roly/projects/binhex/OCA/oca-17/opencode.json`
- Modify: `/home/roly/projects/opencode/opencode-odoo-ls/AGENTS.md`

**Why:** All IDE plugins (VSCode, PyCharm, Neovim, Zed) respond to `workspace/configuration` requests with `{ "selectedProfile": "default" }`. OpenCode's generic LSP client responds with whatever is in the `initialization` field of the server config. Currently, we don't have one — OpenCode responds with `{}` (empty object). The fix: add the field.

---

### Task 3.1: Fix opencode.json with initialization field

- [ ] **Step 1: Read current opencode.json in oca-17**

```bash
cat /home/roly/projects/binhex/OCA/oca-17/opencode.json
```

Expected:
```json
{
  "$schema": "https://opencode.ai/config.json",
  "lsp": {
    "pyright": {
      "disabled": true
    },
    "odoo-ls": {
      "command": ["odoo_ls_server"],
      "extensions": [".py", ".xml"]
    }
  }
}
```

- [ ] **Step 2: Add `initialization` field to `odoo-ls` config**

Edit `/home/roly/projects/binhex/OCA/oca-17/opencode.json` to:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "lsp": {
    "pyright": {
      "disabled": true
    },
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

The only change: add the `"initialization"` object with `"selectedProfile": "default"`. This tells the server which profile in `odools.toml` to use (the default profile named `"default"`).

- [ ] **Step 3: Verify the JSON is valid**

```bash
jq . /home/roly/projects/binhex/OCA/oca-17/opencode.json
```

Expected: Formatted JSON output with no errors.

- [ ] **Step 4: Test with OpenCode**

Start OpenCode in the oca-17 project:

```bash
cd /home/roly/projects/binhex/OCA/oca-17
opencode run "Read odoo/custom/src/account/models/account.py and report any LSP diagnostics you see"
```

Expected:
- OpenCode starts with LSP support
- File is read
- LSP diagnostics appear (or don't, depending on whether there are errors in the file — but the important thing is that the LSP client initialized without errors)

Check logs for any LSP initialization failures:
```bash
tail -50 ~/.local/share/opencode/log/session-*.log | grep -i "odoo\|lsp\|initialize"
```

---

### Task 3.2: Update AGENTS.md with troubleshooting section

- [ ] **Step 5: Read current AGENTS.md**

```bash
cat /home/roly/projects/opencode/opencode-odoo-ls/AGENTS.md | head -150
```

- [ ] **Step 6: Add Troubleshooting section before the final instructions**

At the end of AGENTS.md (before "Instruction Priority"), add:

```markdown
---

## Troubleshooting LSP Integration

### Verify LSP Indexing Status

The Odoo Language Server sends a custom notification when indexing completes:

```bash
# Monitor logs in real-time
tail -f ~/.local/bin/logs/odoo_logs.*.log | grep -E "loadingStatusUpdate|ERROR|WARN"
```

When you see `$Odoo/loadingStatusUpdate: "stop"`, indexing is complete and diagnostics are ready.

### One-Shot Parse Mode (No IDE Required)

Test the LSP without OpenCode:

```bash
cd /your/project/path
odoo_ls_server --parse \
  -c odoo/custom/src/odoo \
  -a odoo/custom/src/addon1 \
  -a odoo/custom/src/addon2 \
  -o /tmp/diagnostics.json \
  --log-level DEBUG
```

- `--parse` mode loads the project, generates diagnostics, writes JSON to `-o`, then exits
- Use `--python /path/to/python3` to test with a specific Python interpreter (only works in `--parse` mode)

### Debugging Python Import Errors

If you see many `ModuleNotFound: No module named 'odoo'` errors:

1. Ensure `python_path` in `odools.toml` points to a Python that has Odoo installed
2. Test the Python directly:
   ```bash
   /path/to/python3 -c "import odoo; print(odoo.__file__)"
   ```
3. If using Docker (Doodba), create a virtualenv on the host and install Odoo editable:
   ```bash
   python3 -m venv ~/.venv/odoo17
   source ~/.venv/odoo17/bin/activate
   pip install -e /path/to/odoo/source
   ```
4. Update `odools.toml`:
   ```toml
   python_path = "/home/user/.venv/odoo17/bin/python3"
   ```

### Known Limitations

- **XML support is in alpha** — field validation on Odoo views may produce false positives or crash on go-to-definition
- **3-second timeout in OpenCode** — if indexing a large project (30+ addon repos) takes longer than 3 seconds after `touchFile`, OpenCode may not capture diagnostics. Logs are still written to disk.
- **Python in Docker** — `odoo_ls_server` runs on the host and expects a Python interpreter on the host. Docker containers are not currently supported as a Python source.
- **Non-standard `odools.toml` locations** — if you use a non-default `odools.toml` path, pass `--config-path` via opencode.json: `"command": ["odoo_ls_server", "--config-path", "/path/to/odools.toml"]`

### Verify Integration with LSP Tool (Experimental)

If you have `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` enabled, you can query the LSP directly:

```bash
OPENCODE_EXPERIMENTAL_LSP_TOOL=true opencode run \
  "Use the lsp tool to query workspaceSymbol for 'AccountMove' in the current workspace"
```

This tests whether the LSP is indexing Python model definitions.

---
```

- [ ] **Step 7: Verify AGENTS.md still parses correctly**

```bash
head -200 /home/roly/projects/opencode/opencode-odoo-ls/AGENTS.md | tail -50
tail -50 /home/roly/projects/opencode/opencode-odoo-ls/AGENTS.md
```

Expected: Markdown is readable, code blocks are properly delimited.

---

## Task 4: Final Verification & Documentation

**Files:**
- Commit: changes to `opencode.json`, `odools.toml`, and AGENTS.md
- Create: summary document for the project

**Why:** Lock in the fixes and create a reference for future troubleshooting.

---

### Task 4.1: Verify all fixes are in place

- [ ] **Step 1: Check `opencode.json` in oca-17**

```bash
jq '.lsp."odoo-ls".initialization' /home/roly/projects/binhex/OCA/oca-17/opencode.json
```

Expected: `{ "selectedProfile": "default" }`

- [ ] **Step 2: Check `odools.toml` in oca-17**

```bash
grep -A 2 'python_path' /home/roly/projects/binhex/OCA/oca-17/odools.toml
```

Expected: Line showing the venv Python path.

- [ ] **Step 3: Verify Troubleshooting section in AGENTS.md**

```bash
grep -A 5 "Troubleshooting LSP" /home/roly/projects/opencode/opencode-odoo-ls/AGENTS.md
```

Expected: Section exists and contains parse mode instructions.

---

### Task 4.2: Create a final validation report

- [ ] **Step 4: Combine all findings into a summary**

Create `/home/roly/projects/opencode/opencode-odoo-ls/docs/LSP_VALIDATION_RESULTS.md`:

```markdown
# LSP Validation Results — April 22, 2026

## Summary

Three fixes applied to enable robust Odoo LSP support in OpenCode:

### Fix #1: OpenCode Integration (opencode.json)
- **Issue:** OpenCode's generic LSP client wasn't sending `selectedProfile` in `workspace/configuration` responses
- **All other IDE plugins (VSCode, PyCharm, Neovim, Zed)** send `{ "selectedProfile": "default" }`
- **Fix:** Add `"initialization": { "selectedProfile": "default" }` to `opencode.json` `odoo-ls` config
- **Impact:** High — enables proper profile selection from `odools.toml`
- **Status:** ✅ Applied

### Fix #2: Python on Host (virtualenv)
- **Issue:** Host Python (`/usr/bin/python3` or pyenv) lacks Odoo dependencies; LSP fails to resolve `import odoo`
- **Fix:** Create `~/.venv/odoo17` with Odoo installed editable; update `odools.toml` with `python_path = "/home/user/.venv/odoo17/bin/python3"`
- **Impact:** Medium — reduces false-positive import errors
- **Status:** ✅ Applied (if Task 2 completed)

### Fix #3: Documentation
- **Issue:** No verified troubleshooting commands in AGENTS.md
- **Fix:** Add Troubleshooting section with parse mode instructions, Python testing, and known limitations
- **Impact:** Low (documentation only) — enables future debugging
- **Status:** ✅ Applied

---

## Baseline Testing Results

Project tested: `/home/roly/projects/binhex/OCA/oca-17` (Doodba v9.4.0, Odoo 17.0)

### Before Fixes
- Parse mode diagnostics: [see BASELINE_REPORT.txt for count]
- Import errors: [COUNT]
- LSP initialization time: [TIME]

### After Fixes
- Parse mode diagnostics: [COUNT]
- Import errors: [COUNT] (should be significantly lower)
- LSP initialization time: [TIME]

### Improvement
- Errors fixed: [DELTA]
- Percentage improvement: [%]

---

## Configuration Reference

### opencode.json (in project root)
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

### odools.toml additions
In the `[Odoo]` section:
```toml
python_path = "/home/user/.venv/odoo17/bin/python3"
```

In the `[[config]]` section (each profile):
```toml
[[config]]
name = "default"
# ... existing settings ...
```

---

## Troubleshooting Quick Reference

**LSP not initializing?**
```bash
tail -50 ~/.local/bin/logs/odoo_logs.*.log | grep -i error
```

**Python import errors?**
```bash
/path/to/python3 -c "import odoo; print(odoo.__file__)"
```

**Check if indexing completed:**
```bash
grep loadingStatusUpdate ~/.local/bin/logs/odoo_logs.*.log | tail -1
```

**Test LSP without IDE:**
```bash
odoo_ls_server --parse -c odoo/custom/src/odoo -a odoo/custom/src/addon1 -o /tmp/diag.json
```

---

## Known Limitations

1. **XML support is alpha** — field validation and go-to-definition may crash or produce false positives
2. **3-second timeout in OpenCode** — large projects may exceed the wait time for diagnostics after file edit
3. **Python in Docker not supported** — LSP expects a host Python interpreter; cannot exec into containers
4. **Plugins investigated:** VSCode, PyCharm, Neovim, Zed all confirmed to send `selectedProfile` in `workspace/configuration`

---

## Commits

- [ ] Task 1: Commit baseline evidence logs and diagnostics
- [ ] Task 2: Commit virtualenv creation and odools.toml update
- [ ] Task 3: Commit opencode.json fix and AGENTS.md update
- [ ] Task 4: Commit LSP_VALIDATION_RESULTS.md

---

## Next Steps

1. Test OpenCode with the fixed `opencode.json` on a real Python file (e.g., `odoo/custom/src/account/models/account.py`)
2. Monitor logs to confirm `$Odoo/loadingStatusUpdate: "stop"` appears after file read
3. Verify diagnostics are shown in OpenCode agent output
4. Consider raising timeout in OpenCode (if possible) for large projects

```

- [ ] **Step 5: Commit all changes**

```bash
cd /home/roly/projects/opencode/opencode-odoo-ls
git add -A
git commit -m "fix(lsp): add opencode initialization field and python path fix for doodba projects

- Add 'initialization' field with selectedProfile to opencode.json for proper LSP configuration
  alignment with VSCode, PyCharm, Neovim, and Zed plugins
- Document virtualenv setup for Python on host (required for Docker-based Odoo projects)
- Add Troubleshooting section to AGENTS.md with verified commands
- Add LSP_VALIDATION_RESULTS.md with baseline evidence and known limitations

This ensures OpenCode's LSP client responds correctly to server configuration requests
and resolves Python import errors in Doodba projects where Odoo is in Docker but source
is on host."
```

---

## Summary of Deliverables

- ✅ **opencode.json fix** — one-line change for proper LSP initialization
- ✅ **odools.toml** — python_path configured for host virtualenv
- ✅ **AGENTS.md** — Troubleshooting section with verified commands
- ✅ **LSP_VALIDATION_RESULTS.md** — Evidence report and configuration reference
- ✅ **Git commits** — Changes tracked with clear messages

---

