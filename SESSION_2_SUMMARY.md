# Session 2 Summary: Odoo LSP Configuration Restored and Validated

**Date:** 2026-04-22  
**Status:** Complete ✅  
**Result:** Odoo LSP fully operational in oca-17 with verified working config

---

## What Was Done

### 1. Restored LSP Configuration to oca-17
- **Removed:** Broken `.opencode/` stub (invalid JSON, wrong location)
- **Created:** `opencode.json` at project root (OpenCode LSP configuration)
- **Created:** `odools.toml` at project root (odoo_ls_server configuration)
- Both files verified as well-formed and functional

### 2. Validated Findings Against Live Integration
- Confirmed `odoo_ls_server` v1.2.1 starts when `.py` files are opened
- Verified `lsp documentSymbol` returns Odoo model definitions correctly
- Tested explicit `src/` subdirectory addon paths work reliably
- Confirmed pyright disabled, no duplicate diagnostics

### 3. Corrected All Contradictions in Documentation

**Critical Contradiction Fixed:**
- **AGENTS.md** now recommends explicit `odoo/custom/src/<repo>` paths instead of symlink farm
- **spec.md** flipped recommendation order: Option B (explicit paths) is now preferred
- **VERIFIED_FACTS.md** & **CLEANUP_LOG.md** already documented explicit paths

**Why:** The symlink farm (`odoo/auto/addons`) creates deduplication ambiguity in the LSP anti-deduplication algorithm (behavior is undocumented). Explicit paths are deterministic and avoid inode-based bugs.

### 4. Verified Zero Contradictions Remain

Comprehensive search confirmed:
- ✅ All symlink farm references in proper context (historical/warnings only)
- ✅ Binary name `odoo_ls_server` consistent across all docs
- ✅ Config file locations all at project root (not `.opencode/`)
- ✅ Pyright disabled everywhere
- ✅ No environment variable configuration recommended
- ✅ `--stdio` flag documented as non-existent

---

## Technical Findings

### Config File Locations (Verified via OpenCode Docs)
| File | Location | Why |
|------|----------|-----|
| `opencode.json` | Project root (Git repo root) | OpenCode traverses up to nearest Git root; doc explicit: "Add opencode.json in your project root" |
| `odools.toml` | Project root | odoo_ls_server walks upward from workspace folder looking for this file |
| `.opencode/` | Agents/commands/plugins only | NOT for opencode.json (schema docs are explicit) |

### Addon Path Configuration
| Config | Status | Reason |
|--------|--------|--------|
| `odoo/auto/addons` (symlink farm) | ❌ Not recommended | Creates deduplication ambiguity; documented as risky in spec.md |
| `odoo/custom/src/<repo>` (explicit paths) | ✅ Recommended | Deterministic; avoids inode-based LSP deduplication bugs |

### LSP Debug Commands (Undocumented in Public Docs)
```bash
opencode debug lsp diagnostics <file>      # Get diagnostics for a file
opencode debug lsp symbols <query>         # Search workspace symbols (requires file context)
opencode debug lsp document-symbols <uri>  # Get symbols from a document
```

### OPENCODE_EXPERIMENTAL_LSP_TOOL Flag
- **Type:** Client-side only
- **Purpose:** Enables `lsp` tool for agents
- **Set in:** Shell environment, NOT in LSP `env` block
- **Example:** `OPENCODE_EXPERIMENTAL_LSP_TOOL=true opencode run "..."`

---

## Files Modified

### In opencode-odoo-ls repository
1. **AGENTS.md** — Updated Step 4 with all 32 explicit `src/` paths; Gotchas section clarified
2. **docs/spec.md** — Swapped recommendation order (explicit paths now Option B: RECOMMENDED)
3. **docs/VERIFIED_FACTS.md** — Added config location requirements; added debug commands section
4. **CLEANUP_LOG.md** — Added Session 2 entry documenting config restoration

### In oca-17 project
1. **opencode.json** (created) — LSP configuration with pyright disabled
2. **odools.toml** (created) — Explicit addon paths for all 32 src/ subdirectories
3. **.opencode/** (deleted) — Removed broken stub directory

---

## Commits Created

```
53666dd fix: correct addon path recommendation from symlink farm to explicit src/ paths
bd7ff07 docs: session 2 — explicit src/ paths, config file locations verified, LSP operational
```

---

## How to Verify LSP Works

### Quick Test (no agent)
```bash
cd /home/roly/projects/binhex/OCA/oca-17
opencode debug lsp diagnostics odoo/custom/src/private/report_printed_flag/__init__.py
```

### Full Test (with agent)
```bash
cd /home/roly/projects/binhex/OCA/oca-17
OPENCODE_EXPERIMENTAL_LSP_TOOL=true opencode run \
  "Read odoo/custom/src/private/report_printed_flag/models/report_printed_mixin.py \
   and use the lsp documentSymbol tool to list all fields and methods. Report what you find."
```

Expected: LSP returns class definitions, field names, method names from Odoo models.

---

## Next Steps

1. **For future sessions:** Read `VERIFIED_FACTS.md` first to understand verified vs. speculative facts
2. **For extending this work:** Any new experiments should update `VERIFIED_FACTS.md`, not overwrite existing specs
3. **For Phase 4:** If plugin automation is needed, document findings in a separate `PHASE_4_RESULTS.md`
4. **For multi-environment support:** Explicit `src/` paths can be auto-generated from `repos.yaml` if extending to other Doodba instances

---

## Lessons Learned

1. **Config file locations matter:** OpenCode and odoo_ls_server have specific discovery patterns; must match documentation
2. **Symlink deduplication is risky:** LSP anti-deduplication behavior with symlinks is undocumented; explicit paths are safer
3. **Documentation must converge:** When Phase 3 testing contradicts Phase 1 assumptions, all referencing docs must be updated
4. **Experimental flags are client-side:** `OPENCODE_EXPERIMENTAL_LSP_TOOL` runs in OpenCode process, not passed to LSP server
5. **Archive documents correctly:** Old experiments should be archived in `docs/superpowers/archive/` and README should direct readers to current sources

---

## Archive Reference

This session built on Phase 3 PoC work documented in:
- `docs/superpowers/archive/2026-04-22-phase3-poc.md` (archived plan from Phase 3)
- See `docs/README.md` for navigation between verified, speculative, and archived content
