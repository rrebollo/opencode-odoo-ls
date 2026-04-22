# opencode-odoo-ls

OpenCode integration for the official **Odoo Language Server Protocol** (Odoo LSP). The goal is to provide intelligent code assistance for Odoo development across Python models, XML templates, and JavaScript, working generically for any Odoo environment (Doodba, vanilla Odoo, OCA deployments, etc.).

## Project Status: Architecture Discovery Phase

This project is in **active discovery** of the optimal OpenCode integration strategy. The architecture is not yet decided — it could be pure LSP config, a plugin, a hybrid approach, or something else. This document guides the discovery process.

**Before proposing implementation:** research findings must resolve the open questions below. Decisions are made after discovery, not before.

## Official Odoo Language Server

**Repository:** https://github.com/odoo/odoo-ls (official Odoo maintainers)

| Property | Value |
|---|---|
| **Current version** | v1.2.1 (released Mar 19, 2026, pre-release quality) |
| **Binary name** | `odoo_ls_server` |
| **Tech stack** | Rust (85.8%), Python (13.8%), uses `lsp-server` crate from rust-analyzer |
| **Configuration** | `odools.toml` (TOML format in project root or specified via `--config-path`) |
| **Official editor support** | VSCode, Neovim, Zed, PyCharm, Helix |
| **Auto-detection** | Works without config if `odoo/` and `addons/` exist at workspace root |
| **Key dependency** | Python typeshed stdlib stubs (required alongside binary for type resolution) |

**Configuration details** (from [wiki/3.-Configuration-files](https://github.com/odoo/odoo-ls/wiki/3.-Configuration-files)):
- `addons_paths` — array of directory paths (no glob support; each must exist as a directory containing addon subdirectories)
- Path variables supported: `${workspaceFolder}`, `${userHome}`, `${version}`, `${base}`, `${detectVersion}`, `"$autoDetectAddons"` (literal string to re-enable auto-detection when `addons_paths` is set)
- Setting `addons_paths` to any value (including empty array) disables auto-detection unless `"$autoDetectAddons"` is included
- `odoo_path` — optional, points to Odoo core source; auto-detected if not set
- `profiles` — multiple named configs for different project contexts (via LSP `settings` key `Odoo.selectedProfile`)

**LSP CLI arguments** (from [server/src/args.rs](https://github.com/odoo/odoo-ls/blob/release/server/src/args.rs)):
- In server (LSP) mode, only these matter: `--config-path`, `--stdlib`, `--stubs`, `--use-tcp`, `--log-level`, `--logs-directory`, `--clientProcessId`
- Other args (`--addons`, `--community-path`, `--python`, `--tracked-folders`) are parse mode only; irrelevant to OpenCode integration
- Real config is via `odools.toml` file discovery, not CLI args
- Neovim integration (reference client) passes zero CLI args; all configuration via `odools.toml`

**Stability note:** Pre-release quality. Odoo LSP active development continues; breaking changes possible. Monitor [releases](https://github.com/odoo/odoo-ls/releases) and [changelog](https://github.com/odoo/odoo-ls/blob/release/CHANGELOG.md).

## OpenCode Integration Models

**Recommended starting approach: LSP Server Mode (pure config, no plugin required).**

Three models exist:

1. **LSP Server Mode (Pure Config)** — OpenCode discovers Odoo LSP via project-local `opencode.json` (RECOMMENDED)
   - Simplest: two files at workspace root (`opencode.json` + `odools.toml`)
   - Matches OpenCode's `/docs/lsp/` architecture
   - Zero plugin code required
   - Activates only in Odoo projects (config is project-local)
   - Starts with this in Phase 3 PoC; assess if plugin is needed

2. **Plugin + LSP** — Plugin that auto-detects Doodba + generates `opencode.json` + `odools.toml`
   - Plugin hooks: workspace detection (`session.created` or equivalent)
   - Generates config files for users (removes manual setup)
   - Useful if the project wants to support multiple environments (plain Odoo, OCA, etc.)
   - Only pursue if pure config PoC reveals need for automation

3. **Plugin-only (Embedded Logic)** — Hypothetical plugin with embedded Odoo LSP logic
   - Requires porting subset of Odoo LSP logic to TypeScript/JS
   - Highest effort, lowest external dependency
   - Skip unless LSP binary distribution is not viable

OpenCode plugin examples: https://github.com/anomalyco/opencode/tree/dev/plugins

## Known Odoo LSP Integration Examples

**Official editor integrations (from Odoo maintainers):**
- VSCode: `odoo/odoo-vscode` — Marketplace extension with auto-binary download
- Neovim: `odoo/odoo-neovim` — via lspconfig; passes zero CLI args, all config via `odools.toml`
- Zed: `odoo/odoo-zed` — Extension registry integration
- PyCharm: `odoo/odoo-pycharm` — Built-in LSP client integration
- Helix: Via `languages.toml` configuration

**Reference implementation:** The Neovim integration is the simplest and most instructive:
- Binary on PATH, no special downloads required
- Single `odools.toml` file in project root (or via `--config-path`)
- Profile selection via LSP `settings` key `Odoo.selectedProfile`
- Workspace detection fully automatic once `odools.toml` exists

*No known OpenCode integration yet* — this project will be the first.

## LSP Coexistence: Odoo LSP vs. Python LSP

**Critical discovery:** `odoo_ls_server` is **a full Python Language Server**, not a thin Odoo-specific overlay.

**Facts** (from official Odoo sources and language server architecture):
- Odoo LSP implements its own **complete Python type resolution** via bundled typeshed stdlib stubs (required alongside binary)
- It provides Python completions, hover, go-to-definition, and diagnostics independently
- It does **not delegate to pyright or pylsp** — it is a fully standalone server with Odoo-specific semantics layered on top

**OpenCode context:**
- OpenCode ships **pyright as a built-in LSP** that auto-starts for any `.py` file
- OpenCode supports **multiple LSPs for the same file type** — both can run simultaneously
- Results from both servers are merged (completions concatenated, diagnostics unioned)

**The coexistence problem:**
- Both servers provide Python completions → duplicate results (both return `self.env`, model fields, etc.)
- Both servers publish diagnostics → conflicting or redundant error messages (pyright may flag valid Odoo patterns as errors)
- User experience degrades with noise, not signal

**Official Odoo maintainer guidance** (VSCode extension precedent):
- The `odoo/odoo-vscode` extension **actively prompts users to disable the Python language server** with the message: *"Disable Python Addon Language server for a better experience"*
- The extension provides a one-click action to set `python.languageServer = 'None'` at workspace scope
- No coexistence pattern is documented — the official position is: **use Odoo LSP alone in Odoo workspaces, disable generic Python LSP**

**Resolution for OpenCode:**
In project-local `opencode.json`, disable pyright and register odoo-ls:
```json
{
  "lsp": {
    "pyright": { "disabled": true },
    "odoo-ls": {
      "command": ["odoo_ls_server"],
      "extensions": [".py", ".xml", ".js"]
    }
  }
}
```

This ensures:
- Only Odoo LSP runs for Python/XML/JS files in this workspace (no duplication)
- Pyright still runs for non-Odoo projects (config is project-local)
- Configuration is declarative, not a hidden plugin behavior

## Doodba Integration: Project Fingerprint

**Doodba** (https://github.com/Tecnativa/doodba-copier-template) is a Docker-based Odoo development environment widely used in the OCA community. It is the first target for OpenCode integration.

**Detection markers:**
- `.copier-answers.yml` file with `_src_path: gh:Tecnativa/doodba-copier-template`
- Directory structure: `odoo/custom/src/` containing git-cloned repos and Odoo source
- `.odoo_version` file (human-readable, e.g., "17.0")

**Source layout** (verified on disk at `/home/roly/projects/binhex/OCA/oca-17/odoo/custom/src/`):
- `odoo/` — Odoo CE source tree (cloned via `gitaggregate`), contains:
  - `odoo/addons/` — 578 bundled addons (sale, account, crm, l10n_*, etc.)
  - `odoo/odoo/addons/` — core test modules (base + ~30 test addons)
  - `odoo-bin` — executable
- Sibling directories: `account-reconcile/`, `bank-payment/`, `connector/`, `crm/`, `hr/`, `project/`, `queue/`, `sale-workflow/`, `server-tools/`, `storage/`, `timesheet/`, etc. — each a flat collection of related addons (e.g., `timesheet/hr_timesheet_begin_end/`, `timesheet/sale_timesheet_line_exclude/`)
- Each repo group may contain 1–30+ addons nested at depth 1 (not flat; each addon in its own subdirectory)

**Build artifacts** (generated, available on host and in Docker):
- `odoo/auto/odoo.conf` — generated config with `addons_path = /opt/odoo/auto/addons` (Docker path)
- `odoo/auto/addons/` — **symlink farm on the host filesystem** (936 symlinks) pointing to all addons across `custom/src/`
  - Example: `odoo/auto/addons/hr_timesheet_begin_end` → `../../custom/src/timesheet/hr_timesheet_begin_end`
  - **Available on host**: the symlinks themselves resolve correctly on the host system (not just in Docker)
- **Implication for LSP**: `${workspaceFolder}/odoo/auto/addons` is a viable alternative to listing individual repo-collection directories

**Python environment:**
- Host: source code only, no Python dependencies installed
- Container: Docker volume-mounts `odoo/custom/src/` read-only; provides full Python runtime with Odoo dependencies
- **Implication for LSP:** `python_path` in `odools.toml` cannot reliably point to host Python (no Odoo deps). Open question: does LSP degrade gracefully without Python type resolution?

**Repos declaration:**
- `odoo/custom/src/repos.yaml` — lists all git repos to clone (created by `gitaggregate`)
- Can be used for future automation (e.g., listing which OCA repos are in scope)

**Recommended `odools.toml` in a Doodba workspace (Option A — Symlink Farm):**

The simplest configuration leverages the pre-built symlink farm:

```toml
[Odoo]
# odoo_path points to the Odoo CE source tree (cloned under custom/src/)
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

# Single path: use the generated symlink farm (all 936 addons in one flat dir)
addons_paths = [
  "${workspaceFolder}/odoo/auto/addons",
]

[python]
# Host Python has no Odoo deps; runtime is in Docker
# Omit or set to system Python; type resolution may degrade gracefully
```

**Alternative `odools.toml` (Option B — Explicit Repo Collections):**

If you prefer to list repo collections explicitly (finer control, diagnostic filtering):

```toml
[Odoo]
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

# List each repo-collection directory
# Each contains one or more addons as subdirectories
addons_paths = [
  "${workspaceFolder}/odoo/custom/src/account-reconcile",
  "${workspaceFolder}/odoo/custom/src/bank-payment",
  "${workspaceFolder}/odoo/custom/src/connector",
  # ... (one per repo group)
  "$autoDetectAddons",  # re-enable auto-detection for any addons at workspace root
]

[python]
# Host Python has no Odoo deps; runtime is in Docker
# Omit or set to system Python; type resolution may degrade gracefully
```

**Pros/Cons:**
- **Option A (Symlink Farm):** Simpler (1 path), auto-generated, all addons visible. Cons: LSP may treat 936 symlinks as one flattened namespace.
- **Option B (Repo Collections):** More control, maintains repo grouping structure, better for diagnostic scoping. Cons: requires listing each collection (but can be auto-generated from `repos.yaml`).

## Minimum Viable Configuration (Doodba)

The simplest working integration for a Doodba workspace requires **two files at the project root**, alongside `odoo/` and `.copier-answers.yml`:

**File 1: `opencode.json`** (disables pyright, registers odoo-ls)
```json
{
  "$schema": "https://opencode.ai/config.json",
  "lsp": {
    "pyright": { "disabled": true },
    "odoo-ls": {
      "command": ["odoo_ls_server"],
      "extensions": [".py", ".xml", ".js"]
    }
  }
}
```

**File 2: `odools.toml`** (configures odoo-ls server, using Option A)
```toml
[Odoo]
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

[odoo]
addons_paths = [
  "${workspaceFolder}/odoo/auto/addons",
]
```

**Notes:**
- Both files are project-local; they only affect this workspace, not global configuration
- `pyright` is disabled only in this project; it still runs in other projects
- `odoo_ls_server` binary must be on `PATH` (or the `command` must be an absolute path)
- `odools.toml` path variables are resolved by `odoo_ls_server` at startup; OpenCode does not process them

This is the **Phase 3 PoC target** — test these two files on a real Doodba project and verify:
1. OpenCode recognizes `odoo-ls` as a valid LSP server
2. `odoo_ls_server` starts successfully with `odools.toml` discovery
3. Pyright is suppressed (no duplicate completions for Odoo code)
4. Code assistance works: completions on Odoo models, go-to-definition for fields, etc.

## Non-Doodba Odoo Environments

The detection and configuration system must be environment-agnostic. Key differences:

**Plain Odoo (typical dev setup):**
- No `.copier-answers.yml`; detect by presence of `odoo/` directory + `odoo-bin` executable
- Custom addons in a single flat directory (not repo groups): `/opt/odoo/addons/`, `/opt/odoo/custom/`, etc.
- `odoo_path = /opt/odoo` or `/opt/odoo/odoo` (depending on install method)
- `addons_paths = ["/opt/odoo/addons", "/opt/odoo/custom"]` (explicit multiple paths, no `$autoDetectAddons` needed)
- Python: likely present on host (pip-installed Odoo development environment)

**OCA multi-repo multi-version (future support):**
- Detect by `repos.yaml` + version-specific directories (e.g., `src/17/`, `src/16/`)
- Support multiple profiles in `odools.toml`, one per version
- Complex detection: identify which version(s) the current file belongs to

## Development Tasks Checklist

**Phase 1: Research** *(before proposing architecture)*
- [x] Clone Odoo LSP, review `README.md`, `Cargo.toml`, binary build process
- [x] Research public Odoo LSP integrations in other editors (Zed, Helix, Neovim)
- [x] Verify official Odoo LSP repo (not community fork)
- [x] Document facts: deployment model, configuration surface, CLI args
- [x] Inspect Doodba source layout on disk
- [x] Identify detection markers and path derivation rules for Doodba
- [x] Document differences between Doodba, plain Odoo, and other environments

**Phase 2: Architecture Decision** *(decide: pure config PoC or plugin-first)*
- [x] Research OpenCode LSP architecture and pyright coexistence
- [x] Identify official Odoo maintainer guidance (VSCode extension: disable Python LS)
- [x] Draft pure config approach (two files: opencode.json + odools.toml)
- [ ] Identify remaining questions before PoC (binary on PATH, --stdio flag, .js handling, etc.)
- [ ] Document Phase 3 PoC acceptance criteria

**Phase 3: Proof of Concept (Pure Config)**
- [ ] Create test workspace or use existing Doodba project at `/home/roly/projects/binhex/OCA/oca-17/`
- [ ] Write `opencode.json` with pyright disabled + odoo-ls registered
- [ ] Write `odools.toml` using Option A (symlink farm) or Option B (repo collections)
- [ ] Ensure `odoo_ls_server` binary is on PATH (verify with `which odoo_ls_server`)
- [ ] Test in OpenCode: open a Python addon file and verify:
  - Pyright is suppressed (no diagnostic overlap)
  - odoo-ls starts without errors
  - Completions work (e.g., `self.env` suggestions)
  - Go-to-definition works for model fields
  - XML files are handled (if .xml registered with odoo-ls)
- [ ] Document findings and PoC results

**Phase 4: Plugin Design** *(only if PoC reveals gaps)*
- [ ] Assess what manual steps the PoC requires (is auto-detection needed?)
- [ ] If yes: design plugin for `.copier-answers.yml` detection + config generation
- [ ] If no: pure config approach is sufficient; document for future sessions

## Key Questions for Agents

When exploring or proposing changes:

1. **Binary Distribution** — Does Odoo LSP need building or can we download pre-built binaries?
   - If binary, from where? (GitHub Releases, cargo binstall, etc.)
   - Which platforms must we support?

2. **LSP Server Launch** — Technical questions for Phase 3 PoC:
   - Does `odoo_ls_server` accept `--stdio` flag for stdio-based LSP communication, or is it automatic?
   - Does it require any environment variables or initialization options?
   - Does it handle `.js` files (Odoo OWL components) or only `.py` and `.xml`?

3. **Workspace Scoping** — Does disabling pyright affect non-Odoo Python files in the same workspace?
   - Example: a project with both `odoo/custom/src/` (Odoo addons) and `scripts/` (utility Python)
   - Does pyright disable at the workspace level or per-file?

4. **Plugin Scope** — If plugin is needed (after Phase 3 PoC), what should it do?
   - Auto-detect Doodba markers (.copier-answers.yml, repos.yaml) and generate config files?
   - Provide custom tools? (e.g., "index Odoo addons from repos.yaml")
   - Hook session events? (e.g., log workspace type on startup)
   - Auto-generate or validate odools.toml?

5. **Python Path Resolution** — How does LSP work without host Python Odoo dependencies?
   - Test graceful degradation in Doodba (type hints may be limited)
   - Does the LSP still provide Odoo model completions without Python type resolution?
   - May require separate type-checking container or explicit Python path

## Related References

- **OpenCode:** https://opencode.ai, https://github.com/anomalyco/opencode
- **OpenCode Docs:** https://opencode.ai/docs/ (plugins, LSP, config, ecosystem)
- **Odoo LSP (official):** https://github.com/odoo/odoo-ls
- **Odoo LSP Wiki:** https://github.com/odoo/odoo-ls/wiki/3.-Configuration-files
- **tower-lsp:** https://github.com/ebkendall/tower-lsp (underlying LSP framework)
- **Doodba:** https://github.com/Tecnativa/doodba-copier-template
- **OpenCode Ecosystem:** https://opencode.ai/docs/ecosystem

## Style & Conventions

- **Git commits:** Use conventional commits (feat:, fix:, docs:, chore:)
- **Code:** Follow rust-fmt for Rust, prettier for JS/TS
- **Research:** Always prefer executable sources (Cargo.toml, opencode.json) over prose
- **Decisions:** Document architectural choices in design docs before implementation

## Notes for Future Sessions

When resuming:
1. Check `git log` for recent decisions and implementation attempts
2. Verify Odoo LSP version stability (currently v1.2.1, active development; pre-release)
3. Test OpenCode plugin discovery if changes are made
4. Keep this file updated as architecture decisions solidify
