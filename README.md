# vrppaul-tools

Claude Code plugin marketplace for Pavel's personal plugins.

## Plugins

| Plugin | Description | Source |
|---|---|---|
| `ty-lsp` | Python LSP via [`ty`](https://github.com/astral-sh/ty) — a fast Rust-based type checker and language server. | Inline |

More plugins (`claude-review`, `semantic-code`) will land here as they are migrated from their existing wirings.

## Install

> This marketplace is not yet published to github. Until it is, clone this repo and register it as a local path:
>
> ```bash
> git clone git@github.com:vrppaul/claude-plugins.git ~/projects/claude-plugins
> claude plugin marketplace add ~/projects/claude-plugins
> claude plugin install ty-lsp@vrppaul-tools
> ```

Once published, the URL form will work too:

```bash
claude plugin marketplace add https://github.com/vrppaul/claude-plugins.git
claude plugin install ty-lsp@vrppaul-tools
```

Restart Claude Code to activate the plugin.

## Contributing

See `AGENTS.md` for structure, manifest conventions, plugin lifecycle, testing workflow, and commit conventions.
