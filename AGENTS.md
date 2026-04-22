# opencode-odoo-ls

OpenCode integration for Odoo ERP via the Odoo LSP (Language Server Protocol). The goal is to embed Odoo LSP capabilities into OpenCode, initially as an internal plugin, to provide intelligent code assistance for Odoo development (Python models, XML templates, JavaScript).

## Project Status: Discovery Phase

This project is in **active discovery**. The primary work is understanding:
- How Odoo LSP architecture maps to OpenCode plugin conventions
- Whether Odoo LSP is best integrated as an LSP server, a plugin, or a hybrid approach
- Precedents in OpenCode ecosystem and other tool integrations (public examples)
- Implementation strategy and dependencies

**Decisions are still being made about the final architecture.** Use the research findings to propose approaches before implementation.

## Reference Implementation: Odoo LSP

The Odoo LSP project (https://github.com/Desdaemon/odoo-lsp, 60 GH stars) is the authoritative reference:

- **Languages:** Rust (92%), TypeScript (4%), Python (2.9%)
- **Core:** Rust-based language server built with `tower-lsp`
- **Clients:** VSCode extension, Neovim config, Zed, Helix
- **Features:** Model/field completion, XML ID references, workspace symbols, syntax highlighting
- **Deployment:** Self-contained binary (~60 releases) + extension packages per editor

Key files for understanding architecture:
- `Cargo.toml` — Rust workspace structure (crates/)
- `crates/` — Modular Rust libraries
- `client/` — Extension implementations (VSCode, Helix, Zed)
- `justfile` — Build/dev tasks
- `pnpm-lock.yaml` — Node dependencies for tooling

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

Search for these patterns to find existing integrations (public repos with high signal):
- "odoo-lsp" + integration in Zed, Helix, Neovim configs (learn external client patterns)
- "tower-lsp" + OpenCode/Copilot/other editors (learn LSP server integration patterns)
- OpenCode ecosystem plugins at https://opencode.ai/docs/ecosystem (see "plugins" section)

Currently known:
- VSCode: Marketplace extension with auto-binary download
- Zed: Extension registry integration
- Neovim: Custom lspconfig configuration
- Helix: languages.toml configuration
- *No known OpenCode integration yet*

## Development Tasks Checklist

**Phase 1: Research** *(before proposing architecture)*
- [ ] Clone Odoo LSP, review `README.md`, `Cargo.toml`, binary build process
- [ ] Research public Odoo LSP integrations in other editors (Zed, Helix, Neovim)
- [ ] Search OpenCode ecosystem for similar LSP integrations (tower-lsp or language-specific)
- [ ] Document findings: deployment model, client protocol version, configuration surface
- [ ] Identify if Odoo LSP has pre-built binaries for macOS/Linux/Windows

**Phase 2: Architecture Proposal** *(propose 2-3 approaches with tradeoffs)*
- [ ] Compare LSP Server Mode vs. Plugin+LSP vs. Plugin-only approaches
- [ ] Draft configuration (opencode.json example)
- [ ] Identify OpenCode plugin hooks needed (lsp.updated, shell.env, etc.)
- [ ] Write design doc with decision rationale

**Phase 3: Proof of Concept**
- [ ] Implement chosen approach as internal plugin
- [ ] Verify Odoo LSP binary works via OpenCode LSP client
- [ ] Test with sample Odoo module code
- [ ] Document setup and usage for future sessions

## Key Questions for Agents

When exploring or proposing changes:

1. **Binary Distribution** — Does Odoo LSP need building or can we download pre-built binaries?
   - If binary, from where? (GitHub Releases, cargo binstall, etc.)
   - Which platforms must we support?

2. **Configuration Surface** — What Odoo LSP settings matter for OpenCode?
   - Root markers (.odoo_lsp, .odoo_lsp.json)
   - Initialization options (Python interpreter, Odoo paths)
   - Feature toggles (syntax highlighting, completion)

3. **Plugin Scope** — If plugin is needed, what should it do?
   - Just launch LSP? (minimal)
   - Also provide custom tools? (e.g., "index Odoo modules")
   - Hook session events? (e.g., log Odoo-specific context on compaction)

4. **Public Precedent** — Are there well-maintained OpenCode plugins integrating other LSPs?
   - Look for repos with 10+ stars, active maintenance
   - Analyze their plugin.ts/plugin.js structure

## Related References

- **OpenCode:** https://opencode.ai, https://github.com/anomalyco/opencode
- **OpenCode Docs:** https://opencode.ai/docs/ (plugins, LSP, config, ecosystem)
- **Odoo LSP:** https://github.com/Desdaemon/odoo-lsp
- **tower-lsp:** https://github.com/ebkendall/tower-lsp (underlying LSP framework)
- **OpenCode Ecosystem:** https://opencode.ai/docs/ecosystem

## Style & Conventions

- **Git commits:** Use conventional commits (feat:, fix:, docs:, chore:)
- **Code:** Follow rust-fmt for Rust, prettier for JS/TS
- **Research:** Always prefer executable sources (Cargo.toml, opencode.json) over prose
- **Decisions:** Document architectural choices in design docs before implementation

## Notes for Future Sessions

When resuming:
1. Check `git log` for recent decisions and implementation attempts
2. Verify Odoo LSP version stability (currently v0.6.2, active development)
3. Test OpenCode plugin discovery if changes are made
4. Keep this file updated as architecture decisions solidify
