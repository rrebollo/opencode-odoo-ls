# Phase 4: Test Explicit Addon Paths from addons.yaml

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Empirically test the explicit addon paths strategy derived from `addons.yaml` and decide whether to recommend it in AGENTS.md (replacing `auto/addons`) or document it as an advanced alternative.

**Architecture:** Instead of using the `auto/addons` symlink farm (which creates overlap with `odoo_path`), build the `addons_paths` array directly from entries in `addons.yaml`, excluding `odoo/` and including implicit `private/`.

**Tech Stack:** `odoo_ls_server` v1.2.1, `odools.toml`, Doodba project at `/home/roly/projects/binhex/OCA/oca-17/`.

---

## Background: The Overlap Problem

In Doodba projects:
- `addons.yaml` lists which OCA/external repos are active (non-commented entries)
- `auto/addons/` is a symlink farm built at container startup, containing all addons from active repos
- `odoo_path` points to `odoo/custom/src/odoo/` (Odoo CE core source)

**The issue:** When Odoo CE is listed in repos (via `repos.yaml` → `gitaggregate`), its addons (`sale`, `account`, `base`, etc.) appear in `auto/addons/` as symlinks. Meanwhile, `odoo_path` scans the same source directory directly. Result: duplicate indexing of core addons.

**The solution:** Build `addons_paths` from **only** the active OCA repos + `private/` from `addons.yaml`, and let `odoo_path` handle the core. Zero overlap.

---

## File Structure

| File | Action | Location |
|------|--------|----------|
| `addons.yaml` | Parse (read-only) | `/home/roly/projects/binhex/OCA/oca-17/odoo/custom/src/addons.yaml` |
| `odools.toml` | Create/modify | `/home/roly/projects/binhex/OCA/oca-17/odools.toml` |
| `opencode.json` | Verify exists | `/home/roly/projects/binhex/OCA/oca-17/opencode.json` |

---

## Task 1: Prepare explicit addons_paths config

**Source:** Active (non-commented) entries from `addons.yaml` + implicit `private/`

From investigation, active entries are:
```
e-learning, field-service, hr, hr-attendance, hr-holidays,
payroll, project, queue, sale-workflow, server-env,
server-tools, spreadsheet, stock-logistics-transport,
storage, timesheet, web
```

Plus implicit `private/` (exists with 3 addons: `report_printed_flag*`)

- [ ] **Step 1: Verify `addons.yaml` content is still:**
  ```yaml
  # (commented entries omitted)
  e-learning: "*"
  field-service: "*"
  hr: "*"
  hr-attendance: "*"
  hr-holidays: "*"
  payroll: "*"
  project: "*"
  queue: "*"
  sale-workflow: "*"
  server-env: "*"
  server-tools: "*"
  spreadsheet: "*"
  stock-logistics-transport: "*"
  storage: "*"
  timesheet: "*"
  web: "*"
  ```

- [ ] **Step 2: Verify `private/` directory exists and has content:**
  ```bash
  ls /home/roly/projects/binhex/OCA/oca-17/odoo/custom/src/private/
  ```
  Expected: 3 addon directories present.

---

## Task 2: Write new odools.toml with explicit paths

- [ ] **Step 1: Create `/home/roly/projects/binhex/OCA/oca-17/odools.toml`**

```toml
[Odoo]
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

[odoo]
addons_paths = [
  "${workspaceFolder}/odoo/custom/src/e-learning",
  "${workspaceFolder}/odoo/custom/src/field-service",
  "${workspaceFolder}/odoo/custom/src/hr",
  "${workspaceFolder}/odoo/custom/src/hr-attendance",
  "${workspaceFolder}/odoo/custom/src/hr-holidays",
  "${workspaceFolder}/odoo/custom/src/payroll",
  "${workspaceFolder}/odoo/custom/src/project",
  "${workspaceFolder}/odoo/custom/src/queue",
  "${workspaceFolder}/odoo/custom/src/sale-workflow",
  "${workspaceFolder}/odoo/custom/src/server-env",
  "${workspaceFolder}/odoo/custom/src/server-tools",
  "${workspaceFolder}/odoo/custom/src/spreadsheet",
  "${workspaceFolder}/odoo/custom/src/stock-logistics-transport",
  "${workspaceFolder}/odoo/custom/src/storage",
  "${workspaceFolder}/odoo/custom/src/timesheet",
  "${workspaceFolder}/odoo/custom/src/web",
  "${workspaceFolder}/odoo/custom/src/private",
]
```

Note: **No `auto/addons` entry**, and **no `odoo/` directory** (covered by `odoo_path`).

- [ ] **Step 2: Verify TOML syntax:**
  ```bash
  python3 -c "
  import tomllib
  with open('/home/roly/projects/binhex/OCA/oca-17/odools.toml','rb') as f:
      print(tomllib.load(f))
  "
  ```
  Expected: dict printed cleanly, no errors.

---

## Task 3: Verify config with OpenCode

- [ ] **Step 1: Core symbol query (via odoo_path)**

```bash
cd /home/roly/projects/binhex/OCA/oca-17
OPENCODE_EXPERIMENTAL_LSP_TOOL=true opencode run \
  "Use the lsp tool with workspaceSymbol to search for 'SaleOrder'. \
   This is a core Odoo model from sale module. Report the raw result \
   including file path."
```

Expected: `SaleOrder` found in `odoo/custom/src/odoo/addons/sale/models/`.

- [ ] **Step 2: OCA symbol query (via addons_paths)**

```bash
cd /home/roly/projects/binhex/OCA/oca-17
OPENCODE_EXPERIMENTAL_LSP_TOOL=true opencode run \
  "Use the lsp tool with workspaceSymbol to search for 'HrAttendance'. \
   This is an OCA model from hr-attendance. Report the raw result \
   including file path."
```

Expected: `HrAttendance` found in `odoo/custom/src/hr-attendance/hr_attendance/models/`.

- [ ] **Step 3: Private symbol query (via addons_paths)**

```bash
cd /home/roly/projects/binhex/OCA/oca-17
OPENCODE_EXPERIMENTAL_LSP_TOOL=true opencode run \
  "Use the lsp tool with workspaceSymbol to search for 'ReportPrintedFlag'. \
   This is a private model from the private directory. Report the raw result."
```

Expected: `ReportPrintedFlag` found in `odoo/custom/src/private/report_printed_flag/models/`.

---

## Task 4: Review LSP server logs

- [ ] **Step 1: Locate latest LSP log**

```bash
ls -lt ~/.local/bin/logs/odoo_logs.*.log | head -1
```

- [ ] **Step 2: Check for indexing confirmation**

Search the log for lines mentioning paths and addons:
```bash
grep -i "addon\|path\|index" ~/.local/bin/logs/odoo_logs.*.log | tail -50
```

Expected:
- Lines referencing all 17 paths from `addons_paths`
- No warning or error about duplicate addons
- Confirmation that core modules are indexed (from `odoo_path`)

- [ ] **Step 3: Check for overlap/duplication errors**

```bash
grep -i "duplicate\|collision\|overlap" ~/.local/bin/logs/odoo_logs.*.log
```

Expected: No matches (clean).

---

## Task 5: Document findings

- [ ] **Step 1: Summarize results**

Create a brief summary noting:
- All 3 symbol queries successful (core, OCA, private)
- LSP logs show no duplication or overlap warnings
- All expected paths indexed without errors

- [ ] **Step 2: Determine recommendation**

**If all tests pass:**
- Decision: **Recommend explicit paths in AGENTS.md** (replacing `auto/addons`)
- Rationale: Zero overlap, transparent, matches Doodba's intent
- Trade-off: More verbose, requires sync with `addons.yaml` (can be automated)

**If any test fails:**
- Decision: **Keep `auto/addons` as default in AGENTS.md**, document this strategy as alternative
- Rationale: Current approach is simpler and proven working
- Next step: Investigate specific failure and document in spec.md

---

## Task 6: Update documentation (if successful)

- [ ] **If explicit paths strategy passes all tests:**

1. Update `AGENTS.md` Step 4 to use explicit paths instead of `auto/addons`
2. Add note in AGENTS.md Gotchas about the `odoo_path` + `auto/addons` overlap that explicit paths solves
3. Update `docs/spec.md` Addon Path Collision section with test results and recommendation
4. Commit with message: `docs: Phase 4 results — explicit addon paths from addons.yaml proven working`

- [ ] **If tests reveal issues:**

1. Document specific failures in `docs/spec.md`
2. Keep AGENTS.md unchanged (keeps `auto/addons` as default)
3. Commit with message: `docs: Phase 4 findings — explicit addon paths has [specific issue], keeping auto/addons default`

---

## Success Criteria

- ✅ Core symbols resolve via `odoo_path`
- ✅ OCA symbols resolve via explicit `addons_paths` entries
- ✅ Private symbols resolve via explicit `addons_paths` (private/)
- ✅ LSP server logs show no duplication warnings
- ✅ All paths are indexed without errors
- ✅ Clear decision documented about recommendation for AGENTS.md

---

## Notes for Future Sessions

When resuming:
1. Check `git log` to see if Phase 4 tests were already executed
2. If yes: review test results in `docs/spec.md` and follow up with AGENTS.md updates if needed
3. If no: execute Task 1–6 above in order
4. See AGENTS.md for current configuration and expected setup
