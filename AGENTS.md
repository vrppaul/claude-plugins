# AGENTS.md

## Overview

Claude Code plugin marketplace for Pavel's personal plugins. No code, no build — a `marketplace.json` manifest plus per-plugin subdirs for inline wrappers.

| Shape | Example | Storage |
|---|---|---|
| External (git-subdir / url) | `claude-review` (planned) | referenced repo; nothing here |
| Inline MCP | `semantic-code` (planned) | `plugins/<name>/.mcp.json` + marketplace entry |
| Inline LSP | `ty-lsp` (shipped) | `lspServers` block in marketplace entry; subdir has docs only |

Authoritative plugin list: `.claude-plugin/marketplace.json`. User-facing: `README.md`.

## Structure

```
.claude-plugin/marketplace.json    # source of truth
plugins/<name>/
  .mcp.json                        # MCP plugins only
  README.md                        # per-plugin docs
pyproject.toml                     # unused stub
```

External plugins have no subdir.

## Commands

```bash
claude plugin validate <path>                 # schema-check
claude plugin marketplace add <path-or-url>   # register (path or git URL)
claude plugin marketplace remove <name>
claude plugin marketplace list
claude plugin marketplace update <name>       # pull latest
claude plugin install <plugin>@<marketplace>
claude plugin disable <plugin>@<marketplace>
claude plugin uninstall <plugin>@<marketplace>
claude plugin list
```

## Manifest conventions

Root skeleton — **put description under `metadata.description`**. Validator rejects top-level `$schema` and `description`; the official marketplace has both and also fails validation.

```json
{
  "name": "vrppaul-tools",
  "owner": { "name": "vrppaul", "email": "vrppaul@github.com" },
  "metadata": { "description": "…" },
  "plugins": [ … ]
}
```

**Inline LSP** (`ty-lsp`) — `lspServers` inline in the entry; subdir has docs only.

```json
{
  "name": "ty-lsp",
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

**Inline MCP** (`semantic-code`) — `.mcp.json` in the plugin subdir, omit `env: {}`.

```json
// plugins/<name>/.mcp.json
{ "<server-name>": { "command": "uvx", "args": ["…"] } }

// marketplace entry
{ "name": "<name>", "version": "0.1.0", "source": "./plugins/<name>", "category": "development" }
```

**External** (`claude-review`) — `ref` must match the target repo's default branch (`git -C <path> branch --show-current`), not an assumed `main`. Pin with `"sha"` for stability.

```json
{
  "name": "claude-review",
  "source": {
    "source": "git-subdir",
    "url": "https://github.com/vrppaul/claude-review.git",
    "path": "plugin",
    "ref": "master"
  },
  "category": "development"
}
```

Uncertain about a field? Read an entry in `~/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json`.

## Adding a new plugin

1. Inline: create `plugins/<name>/`. External: no subdir.
2. Add entry to `marketplace.json` (template above).
3. New inline entry starts at `version: "0.1.0"`. Later: `feat` → minor, `fix` → patch.
4. Update `README.md` plugin table.
5. `claude plugin validate .` — must pass.
6. Test locally (below) before pushing.
7. Commit: `feat(plugins): add <name>`.

## Testing workflow

1. `claude plugin marketplace add ~/projects/claude-plugins` — local path.
2. Install / disable, run verification, user restarts, confirm.
3. On failure: `claude plugin marketplace remove vrppaul-tools` + roll back configs, fix, revalidate.
4. Once local passes, push to github and swap:
   ```bash
   claude plugin marketplace remove vrppaul-tools
   claude plugin marketplace add https://github.com/vrppaul/claude-plugins.git
   ```

## Commit messages

Conventional Commits: `type(scope): description`. Types: `feat fix docs style refactor chore`. Version impact: `feat` → minor on affected plugin, `fix` → patch, others → none.

## Gotchas

- **`metadata.description`** — description at root fails validation.
- **Omit `env: {}`** in `.mcp.json`.
- **Two plugins claiming the same `.py` or MCP server name** — ambiguous dispatch. Disable outgoing *before* installing replacement.
- **Claude Code restart** required for every marketplace / plugin change. Flag before the change lands.
- **`~/.claude.json` is not under git.** Snapshot before editing.
- **First `uvx <tool>@latest`** downloads the tool; later sessions reuse the cache.
- **External plugin `ref`** — match the target repo's default branch. Don't assume `main`.
- **MCP cache keyed on absolute project path**, not launch command. Switching wiring preserves it.
- **`LSP workspaceSymbol` returns empty** for every backend in Claude Code 2.1.114 — upstream bug [claude-code#17149](https://github.com/anthropics/claude-code/issues/17149). Use `Grep`.
- **Migrating a plugin from old wiring** — remove the old raw `mcpServers` entry from `~/.claude.json` (or old marketplace) in the *same* session as the new install.

## Response Style

Lead with the answer; no preamble, recap, or trailing summary. One reason per claim — don't restate in different words. Structure (headers, bullets) only for ≥4 parallel items; otherwise prose. Keep code blocks, `file:line` cites, and concrete evidence. Drop hedge words when not uncertain, announcement verbs ("I found", "I ran"), and editorial commentary.

## Session Retro

At session close (ship/commit/handover or closure signal), propose: "Want a quick retro?"

If yes:
1. **Went well** — concrete behaviors to keep.
2. **Went poorly** — specific moments that cost time.
3. **Ask the user** — what they'd flag that you missed.
4. **Propose captures** — per lesson: rule, AGENTS.md edit, memory, or discard. User approves each.

## Boundaries

**Always:**
- `claude plugin validate .` before committing
- Test via local-path marketplace before pushing to github
- Snapshot `~/.claude/settings.json` and `~/.claude.json` to `~/.claude/backups/<ts>/` before editing either

**Ask first:**
- Schema-level changes to `marketplace.json` shape
- Plugins requiring auth, private repos, or user-specific credentials
- Renaming the marketplace `name` (cascades to every install)
- First-time `gh repo create` or `git push` for this repo

**Never:**
- Hand-edit `enabledPlugins` / `extraKnownMarketplaces` in `~/.claude/settings.json` — use the CLI
- Push to github before local verification passes
- Copy an empty `env: {}` into `.mcp.json`
- Install a plugin without disabling any existing one claiming the same file extension or MCP server name

## Documentation

- `README.md` — public: marketplace description, plugin table, install
- `CLAUDE.md` — thin pointer to this file; never duplicate content here
- `AGENTS.md` (this file) — everything else

**Maintenance:**

- Adding / removing / renaming a plugin: update `marketplace.json` and `README.md` plugin table in the same commit.
- New landmine surfaces in a session → add one bullet to `Gotchas`. Rule goes obsolete (e.g. upstream bug fixed) → remove or update it; don't leave stale guidance.
- No disposable docs (HANDOVER.md, STATUS.md, TODO.md). Durable info → `AGENTS.md`. Ephemeral → memory or chat.

**Writing style:**

- Terse. Only necessary info, concise without losing meaning.
- Prefer tables and code blocks over prose. No preamble, no trailing recap.
- One-line rules stay one line. Don't expand for symmetry.
