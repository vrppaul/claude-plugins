# Contributing

PRs welcome. This repo is a manifest-only Claude Code plugin marketplace — see [AGENTS.md](AGENTS.md) for structure and conventions.

## Adding a plugin

1. Fork.
2. Add a plugin entry to `.claude-plugin/marketplace.json`. `AGENTS.md` covers the three shapes (external, inline MCP, inline LSP) and their templates.
3. For inline plugins, add `plugins/<name>/` with `.mcp.json` (MCP) or `README.md` (LSP).
4. Update the plugin table in `README.md`.
5. Validate: `claude plugin validate .`
6. Test locally:
   ```bash
   claude plugin marketplace add /path/to/your/fork
   claude plugin install <plugin>@vrppaul-tools
   # restart Claude Code, verify
   ```
7. Open a PR against `master`. Commit message follows Conventional Commits (`feat(plugins): add <name>`).

## Fixing a bug

Open an issue first with the failing command output — for manifest issues include the output of `claude plugin validate .`. PRs go through the same flow as adding a plugin.

## Branch policy

`master` is protected. Direct pushes are restricted to the maintainer; everyone else needs a PR. Approvals are not required but local validation must pass.

## License

[MIT](LICENSE).
