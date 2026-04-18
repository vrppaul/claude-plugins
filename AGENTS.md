# AGENTS.md

## Overview

This is a [Claude Code plugin marketplace](https://docs.anthropic.com/en/docs/claude-code/plugins). It has no code and no build — just a `marketplace.json` manifest plus per-plugin subdirs. Plugins hosted here:

| Name | Kind | Source |
|---|---|---|
| `claude-review` | full plugin (skills + UI) | external git-subdir → `vrppaul/claude-review` |
| `semantic-code` | MCP server | inline — wraps PyPI `semantic-code-mcp` via `uvx` |
| `ty-lsp` | LSP backend | inline — wraps `uvx ty@latest server` |

## Structure

```
.claude-plugin/
  marketplace.json          # manifest — source of truth for the plugin list
plugins/
  <plugin-name>/
    .mcp.json               # MCP plugins only — auto-discovered by Claude Code
    README.md               # per-plugin docs
pyproject.toml              # stub from `uv init --bare`, no deps, no build
```

The marketplace manifest is the only file that defines what plugins exist. Plugin subdirs are empty for LSP plugins (the `lspServers` block lives in `marketplace.json`) and contain only `.mcp.json` for MCP plugins.

## Commands

This repo has no build, tests, or linters. All tooling is the `claude plugin` CLI:

```bash
claude plugin validate <path>                        # schema-check a marketplace or plugin
claude plugin marketplace add <path-or-url>          # register a marketplace (local path or git URL)
claude plugin marketplace remove <name>              # unregister
claude plugin marketplace list                       # show registered marketplaces
claude plugin install <plugin>@<marketplace>         # install and enable a plugin
claude plugin disable <plugin>@<marketplace>         # disable without uninstalling
claude plugin uninstall <plugin>@<marketplace>       # remove entirely
claude plugin list                                   # show installed plugins
```

## Manifest conventions

The `marketplace.json` schema follows Anthropic's official marketplaces. Three plugin shapes:

### Inline LSP plugin (`ty-lsp` pattern)

`lspServers` block lives **inline in the marketplace.json entry**. The plugin subdir contains docs only.

```json
{
  "name": "ty-lsp",
  "description": "Python LSP via ty (Astral) — fast Rust-based type checker.",
  "version": "0.1.0",
  "source": "./plugins/ty-lsp",
  "category": "development",
  "strict": false,
  "lspServers": {
    "ty": {
      "command": "uvx",
      "args": ["ty@latest", "server"],
      "extensionToLanguage": {".py": "python", ".pyi": "python"}
    }
  }
}
```

Template source: `pyright-lsp` entry in `claude-plugins-official/.claude-plugin/marketplace.json` (around line 1119).

### Inline MCP plugin (`semantic-code` pattern)

The `.mcp.json` lives inside the plugin source dir and is auto-discovered. The marketplace.json entry just points `source` at the dir.

`plugins/semantic-code/.mcp.json`:
```json
{
  "semantic-code": {
    "command": "uvx",
    "args": ["--index", "pytorch-cpu=https://download.pytorch.org/whl/cpu", "semantic-code-mcp"]
  }
}
```

**Omit `env: {}` when there are no env vars** — the official `playwright` and `serena` `.mcp.json`s skip it. Do not copy empty `env` blocks forward from old raw `mcpServers` entries.

`marketplace.json` entry:
```json
{
  "name": "semantic-code",
  "description": "Semantic code search MCP — tree-sitter + sentence-transformers embeddings.",
  "version": "0.1.0",
  "source": "./plugins/semantic-code",
  "category": "development"
}
```

### External plugin (`claude-review` pattern)

For plugins whose source lives in another repo — either the root of that repo (`source: url`) or a subdirectory (`source: git-subdir`).

```json
{
  "name": "claude-review",
  "description": "Browser-based code review tool with inline comments and git diff UI.",
  "source": {
    "source": "git-subdir",
    "url": "https://github.com/vrppaul/claude-review.git",
    "path": "plugin",
    "ref": "main"
  },
  "category": "development"
}
```

Pin with an optional `"sha": "<commit>"` field when stability matters more than freshness.

## Adding a new plugin

1. Create `plugins/<name>/`. For MCP plugins, write `.mcp.json`. For LSP plugins, a `README.md` is enough — the config lives in `marketplace.json`.
2. Add an entry to `.claude-plugin/marketplace.json` using the matching template above.
3. Bump `version` on the plugin entry (semver).
4. Update the plugin table in `README.md`.
5. `claude plugin validate .`
6. Test locally — see *Testing workflow* below — before touching the github remote.
7. Commit with a Conventional Commits message: `feat(plugins): add <name>`.
8. Push. Users get the update via `claude plugin marketplace update vrppaul-tools`.

## Testing workflow

Never push to github before proving a change works locally. The `claude plugin marketplace add` command accepts either a filesystem path or a git URL — exploit that:

1. `claude plugin marketplace add ~/projects/claude-plugins` — register as a local-path marketplace.
2. Install / disable plugins as needed, run the verification steps, let the user restart and confirm.
3. If anything fails: `claude plugin marketplace remove vrppaul-tools` + roll back any config changes, fix the manifest, re-validate.
4. Once local verification passes, commit and push to github. Then swap the source:
   ```
   claude plugin marketplace remove vrppaul-tools
   claude plugin marketplace add https://github.com/vrppaul/claude-plugins.git
   ```

Before every config edit that touches `~/.claude/settings.json` or `~/.claude.json`, snapshot both files to `~/.claude/backups/<timestamp>/` so rollback is a single `cp` away.

## Commit messages

Conventional Commits: `type(scope): description`.

Types: `feat fix docs style refactor chore`. `feat(plugins): add <name>` for new plugins, `fix(manifest): …` for manifest bugs, `chore(docs): …` for README/AGENTS.md edits, `refactor(<plugin>): …` for per-plugin manifest changes.

Version impact: `feat` → minor bump on the affected plugin's `version` field. `fix` → patch bump. Others → no bump.

## Response style

Lead with the answer; no preamble, recap, or trailing summary. One reason per claim — don't restate in different words. Structure (headers, bullets) only for ≥ 4 parallel items; otherwise prose. Keep code blocks, `file:line` cites, and concrete evidence. Drop hedge words when not uncertain, announcement verbs ("I found", "I ran"), and editorial commentary.

## Session retro

When a session feels like it's wrapping up (after ship/commit/handover, or on a user closure signal), propose one: "Want a quick retro?"

If yes, four steps:

1. **Went well** — concrete behaviors to keep.
2. **Went poorly** — specific moments that cost time.
3. **Ask the user** — what they'd flag that you missed.
4. **Propose captures** — per lesson: new rule, AGENTS.md/CLAUDE.md edit, or discard. User approves each.

Session lessons decay fast; untracked retros waste the learning.

## Boundaries

**Always:**
- Validate the manifest with `claude plugin validate .` before committing
- Test via local-path marketplace before pushing to github
- Follow the official `claude-plugins-official` marketplace shapes — read one of its entries if uncertain about a field
- Snapshot `~/.claude/settings.json` / `~/.claude.json` before editing

**Ask first:**
- Schema-level changes to `marketplace.json` shape (beyond adding / editing plugin entries)
- Adding plugins that require auth, private repos, or user-specific credentials
- Renaming the marketplace (`name` field) — cascades to every user's settings.json and install references

**Never:**
- Manually edit `enabledPlugins` / `extraKnownMarketplaces` in `~/.claude/settings.json` — use the `claude plugin` CLI
- Push to github before local verification passes
- Copy an empty `env: {}` block into a `.mcp.json` — omit it
- Install a plugin without also disabling any existing plugin that claims the same file extension (LSPs) or MCP server name

## Documentation

- `README.md` — public-facing: what the marketplace is, plugin table, install instructions
- `CLAUDE.md` — Claude Code-specific rules only (thin pointer to this file)
- `AGENTS.md` (this file) — cross-agent context: structure, manifest conventions, workflow, boundaries
