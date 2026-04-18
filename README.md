# vrppaul-tools

Claude Code plugin marketplace hosting code review, semantic search, and LSP tooling.

> **Status:** bootstrap. The manifest and plugin files have not been populated yet — see `HANDOVER.md`.

## Plugins

| Plugin | Description | Source |
|---|---|---|
| `claude-review` | Browser-based code review UI with inline comments and git-diff view. | External — [vrppaul/claude-review](https://github.com/vrppaul/claude-review) |
| `semantic-code` | MCP server for semantic code search backed by tree-sitter + embeddings. | External — [vrppaul/semantic-code-mcp](https://github.com/vrppaul/semantic-code-mcp) |
| `ty-lsp` | Python LSP via [`ty`](https://github.com/astral-sh/ty) — a fast Rust-based type checker and language server. | Inline |

## Install

```bash
claude plugin marketplace add https://github.com/vrppaul/claude-plugins.git
claude plugin install claude-review@vrppaul-tools
claude plugin install semantic-code@vrppaul-tools
claude plugin install ty-lsp@vrppaul-tools
```

Restart Claude Code to activate the plugins.

## Contributing

See `AGENTS.md` for structure, manifest conventions, plugin lifecycle, testing workflow, and commit conventions. See `CLAUDE.md` for Claude Code-specific rules.
