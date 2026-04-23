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

## Testing Evidence

- Parse mode baseline: `/tmp/odoo-ls-validation/diagnostics.json`
- Baseline report: `/tmp/odoo-ls-validation/BASELINE_REPORT.txt`
- After-venv diagnostics: `/tmp/odoo-ls-validation/diagnostics-after-venv.json`
- LSP logs: `~/.local/bin/logs/odoo_logs.*.log`

---

## Next Steps

1. Test OpenCode with the fixed `opencode.json` on a real Python file
2. Monitor logs to confirm `$Odoo/loadingStatusUpdate: "stop"` appears
3. Verify diagnostics are shown in OpenCode agent output
