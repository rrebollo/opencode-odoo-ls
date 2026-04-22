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

Review before proposing implementation approach:

1. **LSP Server Mode** — OpenCode discovers Odoo LSP as a local LSP via `opencode.json` config
   - Simplest if Odoo LSP already runs as standalone binary
   - Matches OpenCode's `/docs/lsp/` architecture
   - Zero plugin code required, pure config

2. **Plugin + LSP** — Plugin that launches/manages Odoo LSP + registers LSP client
   - Plugin hooks: `lsp.updated`, `installation.updated`
   - Custom tool exports for module indexing or dev workflows
   - See `/docs/plugins/` for plugin event types and examples

3. **Plugin-only (Embedded Logic)** — OpenCode plugin implementing Odoo-aware features without external LSP
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

**Runtime build artifacts** (generated, not source):
- `odoo/auto/odoo.conf` — generated config with single `addons_path = /opt/odoo/auto/addons`
- `odoo/auto/addons/` — symlink farm (936 symlinks) pointing to all addons across `custom/src/`
- These are inside the Docker container and irrelevant to LSP configuration

**Python environment:**
- Host: source code only, no Python dependencies installed
- Container: Docker volume-mounts `odoo/custom/src/` read-only; provides full Python runtime with Odoo dependencies
- **Implication for LSP:** `python_path` in `odools.toml` cannot reliably point to host Python (no Odoo deps). Open question: does LSP degrade gracefully without Python type resolution?

**Repos declaration:**
- `odoo/custom/src/repos.yaml` — lists all git repos to clone (created by `gitaggregate`)
- Can be used for future automation (e.g., listing which OCA repos are in scope)

**For `odools.toml` in a Doodba workspace:**
```toml
[Odoo]
# odoo_path points to the Odoo CE source tree (cloned under custom/src/)
odoo_path = "${workspaceFolder}/odoo/custom/src/odoo"

# addons_paths lists each repo-collection directory
# Each contains one or more addons as subdirectories
addons_paths = [
  "${workspaceFolder}/odoo/custom/src/account-reconcile",
  "${workspaceFolder}/odoo/custom/src/bank-payment",
  "${workspaceFolder}/odoo/custom/src/connector",
  # ... (one per repo group, NOT auto/addons)
  "$autoDetectAddons",  # re-enable auto-detection for any addons at workspace root
]

[python]
# Host Python has no Odoo deps; runtime is in Docker
# Omit or set to system Python; type resolution may degrade gracefully
```

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

**Phase 2: Architecture Proposal** *(propose 2-3 approaches with tradeoffs)*
- [ ] Search OpenCode ecosystem for similar LSP integrations (reference examples)
- [ ] Compare LSP Server Mode vs. Plugin+LSP vs. Plugin-only approaches
- [ ] Design generic detection + configuration generation mechanism (environment-agnostic interface, environment-specific implementations)
- [ ] Draft opencode.json example and odools.toml examples for Doodba
- [ ] Identify OpenCode plugin hooks needed (if plugin is needed)
- [ ] Write architecture decision document with tradeoffs

**Phase 3: Proof of Concept**
- [ ] Implement chosen approach (LSP config, plugin, or hybrid)
- [ ] Test Odoo LSP binary discovery and launch on Doodba workspace
- [ ] Verify odools.toml auto-generation or manual setup works
- [ ] Test code assistance (completions, go-to-definition, diagnostics) on real Odoo addon code
- [ ] Test on multiple environment types (Doodba, plain Odoo if possible)
- [ ] Document setup and usage for future sessions

## Key Questions for Agents

When exploring or proposing changes:

1. **Binary Distribution** — Does Odoo LSP need building or can we download pre-built binaries?
   - If binary, from where? (GitHub Releases, cargo binstall, etc.)
   - Which platforms must we support?

2. **Configuration Surface** — What Odoo LSP settings matter for OpenCode?
   - Root markers (.copier-answers.yml for Doodba, odoo-bin for plain Odoo)
   - Initialization options (Python interpreter, Odoo paths)
   - Feature toggles (syntax highlighting, completion)

3. **Plugin Scope** — If plugin is needed, what should it do?
   - Just launch LSP? (minimal)
   - Also provide custom tools? (e.g., "index Odoo addons from repos.yaml")
   - Hook session events? (e.g., log workspace type on startup)
   - Auto-generate or validate odools.toml?

4. **Public Precedent** — Are there well-maintained OpenCode plugins integrating other LSPs?
   - Look for repos with 10+ stars, active maintenance
   - Analyze their plugin.ts/plugin.js structure

5. **Python Path Resolution** — How does LSP work without host Python Odoo dependencies?
   - Test graceful degradation in Doodba (type hints may be limited)
   - May require separate type-checking container or skipping type resolution

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
