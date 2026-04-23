# opencode-odoo-ls

Configure Odoo Language Server Protocol (LSP) support for OpenCode in a Doodba project.

## What This Does

Once configured, agents working in your Doodba project will receive Odoo-aware diagnostics on every file edit and can resolve model definitions, field types, and symbol locations via the LSP tool.

## Prerequisites

- OpenCode installed on the host
- Docker available, with the Doodba image for your Odoo version pulled locally (`ghcr.io/tecnativa/doodba:<VERSION>-onbuild`)
- A Doodba project to configure (oca-16, oca-17, etc.)

## How to Use

1. Open OpenCode in your Doodba project root:
   ```bash
   cd /path/to/your/doodba-project
   opencode
   ```

2. Paste this prompt in the chat:
   ```
   Follow the setup instructions at this URL to configure Odoo LSP 
   support for the current project:
   https://raw.githubusercontent.com/rrebollo/opencode-odoo-ls/refs/heads/master/AGENTS.md
   ```

The agent will read the guide and execute all setup steps.

## What the Agent Will Do

- Detect Odoo version and Python version from the project
- Install `odoo_ls_server` binary and typeshed stubs (if not present)
- Create `.venv-odoo<VERSION>/` at the project root
- Create `opencode.json` and `odools.toml` at the project root
- Verify the setup with a parse mode test

## Result

After the agent completes setup, your project will have:

```
<doodba-root>/
├── opencode.json        ← LSP client config for OpenCode
├── odools.toml          ← LSP server config (paths, addons)
└── .venv-odoo<VERSION>/ ← Python environment for type resolution
```

Agents can now:
- Get Odoo-aware Python diagnostics on every file edit
- Validate XML views, actions, reports
- Resolve model/field definitions via `lsp` tool
- See type hints for Odoo APIs
- Search across all addon repositories

## For More Information

See `AGENTS.md` for detailed step-by-step setup instructions, troubleshooting, and technical details.
