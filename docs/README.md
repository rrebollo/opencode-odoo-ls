# Documentation Index

## Primary Documents (Use These)

### `spec.md`
**The complete technical specification.** Contains:
- Odoo LSP overview and architecture
- OpenCode integration options
- Doodba-specific configuration details  
- **Phase 3 PoC Results** — verified facts from actual testing
- odools.toml configuration options
- Known issues and gotchas

**Status:** ✅ Verified through Phase 3 PoC testing

### `VERIFIED_FACTS.md`
**Separates 100% verified facts from speculation.** Read this first if:
- You're skeptical of a claim in other docs
- You want to know exactly what was tested vs. what's assumed
- You're about to depend on something and want confidence

**Contents:**
- Verified configuration that works
- Failed experiments (with reasons why they failed)
- OpenCode LSP behavior (source-verified)
- Odoo LSP server behavior (source code analyzed)
- What still needs investigation

**Status:** ✅ Accurate; explicitly marks unknowns

## Reference Documents (Archived for Historical Context)

### `superpowers/archive/2026-04-22-phase3-poc.md`
**Phase 3 PoC plan (draft plan, not execution results).**

This is the **plan** that was written before Phase 3 testing. The **results** of that testing are documented in `spec.md` under "Phase 3 PoC Results" section.

**Status:** ⚠️ Archive only — refer to spec.md for actual results

## What Was Removed (and Why)

The following documents were removed because they contained **speculation, failed experiments, or incorrect conclusions:**

- ❌ `2026-04-22-opencode-lsp-investigation.md` — Investigation that led to incorrect claims about how OpenCode passes workspace folders to LSP
- ❌ `2026-04-22-phase4-explicit-addons.md` — Plan to use explicit addon paths from `addons.yaml`. Created maintenance burden with no clear benefit.
- ❌ `2026-04-22-phase5-explicit-addons-final.md` — Final attempt to use explicit addon paths with hardcoded absolute paths. Breaks portability.

These investigations are preserved in git history but should not be referenced as current truth.

## For Future Sessions

1. **Start with:** `spec.md` (complete overview)
2. **Verify claims with:** `VERIFIED_FACTS.md` (separates truth from assumption)
3. **When implementing:** Use `AGENTS.md` in project root (production setup instructions)
4. **When extending:** Document new findings in `VERIFIED_FACTS.md`, not spec.md

Any new experiments should clearly mark themselves as "unverified" and update these docs accordingly.
