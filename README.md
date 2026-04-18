# vrppaul-tools

Claude Code plugin marketplace for Pavel's personal plugins.

## Plugins

| Plugin | Description | Source |
|---|---|---|
| `ty-lsp` | Python LSP via [`ty`](https://github.com/astral-sh/ty) — a fast Rust-based type checker and language server. | Inline |

More plugins (`claude-review`, `semantic-code`) will land here as they are migrated from their existing wirings.

## Install

```bash
claude plugin marketplace add https://github.com/vrppaul/claude-plugins.git
claude plugin install ty-lsp@vrppaul-tools
```

Restart Claude Code to activate the plugin.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
