# vrppaul-tools

Claude Code plugin marketplace. See [AGENTS.md](AGENTS.md) for structure, manifest conventions, plugin lifecycle, testing workflow, and commit conventions.

## Claude Code specifics

- The `claude plugin` CLI owns `enabledPlugins` and `extraKnownMarketplaces` in `~/.claude/settings.json`. Never edit those keys by hand — use `claude plugin install` / `disable` / `marketplace add` / `marketplace remove`.
- A Claude Code restart is required for marketplace or plugin changes to take effect. Flag this to the user before the change lands.
- `~/.claude/settings.json` and `~/.claude.json` are not under git. Before editing either, snapshot both to `~/.claude/backups/<timestamp>/`.
- When the user asks to install or swap a plugin, run the local-path marketplace flow first (`claude plugin marketplace add <path>`) to prove the manifest before pushing to github — public marketplace mistakes are visible and harder to undo.
