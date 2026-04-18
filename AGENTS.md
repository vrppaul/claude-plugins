# AGENTS.md

## Overview

This repo exists for one reason: to be the [Claude Code plugin marketplace](https://docs.anthropic.com/en/docs/claude-code/plugins) hosting Pavel's personal plugins. No code here, no build — just a `marketplace.json` manifest plus per-plugin subdirs for the thin-wrapper cases.

Plugins come in three shapes, all supported:

| Shape | Example | How it's stored |
|---|---|---|
| External (git-subdir / url) | `claude-review` (planned — lives in `vrppaul/claude-review`) | nothing in this repo; pulled from the referenced repo at install time |
| Inline MCP | `semantic-code` (planned — wraps PyPI `semantic-code-mcp` via `uvx`) | `plugins/<name>/.mcp.json` + marketplace entry |
| Inline LSP | `ty-lsp` (shipped — wraps `uvx ty@latest server`) | marketplace entry holds the `lspServers` block; plugin subdir has docs only |

The authoritative plugin list is `.claude-plugin/marketplace.json`. `README.md` is the user-facing view.

## Structure

```
.claude-plugin/
  marketplace.json          # manifest — source of truth for the plugin list
plugins/
  <plugin-name>/
    .mcp.json               # MCP plugins only — auto-discovered by Claude Code
    README.md               # per-plugin docs (optional but expected)
pyproject.toml              # stub from `uv init --bare`; unused, kept for future tooling
```

External plugins have no subdir here at all.

## Commands

All tooling is the `claude plugin` CLI:

```bash
claude plugin validate <path>                        # schema-check a marketplace or plugin
claude plugin marketplace add <path-or-url>          # register a marketplace (local path or git URL)
claude plugin marketplace remove <name>              # unregister
claude plugin marketplace list                       # show registered marketplaces
claude plugin marketplace update <name>              # pull latest for a registered marketplace
claude plugin install <plugin>@<marketplace>         # install and enable a plugin
claude plugin disable <plugin>@<marketplace>         # disable without uninstalling
claude plugin uninstall <plugin>@<marketplace>       # remove entirely
claude plugin list                                   # show installed plugins
```

A Claude Code **restart is required** for any marketplace or plugin change to take effect. Flag this to the user before the change lands.

## Manifest conventions

The `marketplace.json` schema follows Anthropic's official marketplaces, with one non-obvious rule:

> **Marketplace description goes under `metadata.description`, not at the root.** The validator rejects top-level `$schema` and `description` — keys the official marketplace itself includes (it also fails validation, but the runtime tolerates them). Use the shape below to get a clean validate.

Top-level skeleton:

```json
{
  "name": "vrppaul-tools",
  "owner": { "name": "vrppaul", "email": "vrppaul@github.com" },
  "metadata": { "description": "…" },
  "plugins": [ … ]
}
```

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

Reference the `pyright-lsp` entry in `~/.claude/plugins/marketplaces/claude-plugins-official/.claude-plugin/marketplace.json` when in doubt about a field.

### Inline MCP plugin (`semantic-code` pattern)

The `.mcp.json` lives inside the plugin source dir and is auto-discovered. The marketplace.json entry just points `source` at the dir.

`plugins/<name>/.mcp.json`:
```json
{
  "<server-name>": {
    "command": "uvx",
    "args": ["…"]
  }
}
```

**Omit `env: {}` when there are no env vars** — official `playwright` and `serena` `.mcp.json`s skip it. Do not copy empty `env` blocks forward from raw `mcpServers` entries in `~/.claude.json`.

`marketplace.json` entry:
```json
{
  "name": "semantic-code",
  "description": "…",
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
  "description": "…",
  "source": {
    "source": "git-subdir",
    "url": "https://github.com/vrppaul/claude-review.git",
    "path": "plugin",
    "ref": "master"
  },
  "category": "development"
}
```

`ref` must match the default branch of the target repo — verify with `git -C <repo-path> branch --show-current`. Pin with an optional `"sha": "<commit>"` when stability matters more than freshness.

## Adding a new plugin

1. Create `plugins/<name>/` if inline. For MCP plugins, write `.mcp.json`. For LSP plugins, a `README.md` is enough — the config lives in `marketplace.json`. External plugins have no subdir here.
2. Add an entry to `.claude-plugin/marketplace.json` using the matching template above.
3. Set `version: "0.1.0"` on a brand-new inline entry. On subsequent edits, bump: `feat` → minor, `fix` → patch.
4. Update the plugin table in `README.md`.
5. `claude plugin validate .` — must pass.
6. Test locally (see *Testing workflow*) before touching the github remote.
7. Commit: `feat(plugins): add <name>`.
8. Push. Users get the update via `claude plugin marketplace update vrppaul-tools`.

## Testing workflow

Never push to github before proving a change works locally. `claude plugin marketplace add` accepts either a filesystem path or a git URL — exploit that:

1. `claude plugin marketplace add ~/projects/claude-plugins` — register as a local-path marketplace.
2. Install / disable plugins as needed, run verification, let the user restart and confirm.
3. If anything fails: `claude plugin marketplace remove vrppaul-tools` + roll back any config changes, fix the manifest, re-validate.
4. Once local verification passes, commit and push to github. Then swap the source:
   ```bash
   claude plugin marketplace remove vrppaul-tools
   claude plugin marketplace add https://github.com/vrppaul/claude-plugins.git
   ```

## Commit messages

Conventional Commits: `type(scope): description`.

Types: `feat fix docs style refactor chore`. `feat(plugins): add <name>` for new plugins, `fix(manifest): …` for manifest bugs, `chore(docs): …` for README/AGENTS.md edits, `refactor(<plugin>): …` for per-plugin manifest changes.

Version impact: `feat` → minor bump on the affected plugin's `version`. `fix` → patch bump. Others → no bump.

## Gotchas

- **Validator rejects top-level `$schema` / `description`.** Put description under `metadata.description`. Official marketplace violates this too — don't copy its skeleton blindly.
- **`env: {}` in `.mcp.json`.** Omit. Empty env blocks break nothing but diverge from the official shape.
- **Two plugins claiming the same `.py` (or same MCP server name).** Dispatch is ambiguous. Disable the outgoing plugin *before* installing the replacement.
- **`~/.claude.json` is not under git.** The only rollback path for it is a manual snapshot (see Boundaries:Always).
- **First `uvx <tool>@latest server` invocation downloads the tool.** One-time cost, later sessions reuse the `uv` cache.
- **External plugin `ref` must match the target repo's default branch.** `claude-review`'s is `master`, not `main`; don't assume.
- **MCP cache is keyed on absolute project path**, not on how the server is launched. Switching from a raw `mcpServers` entry to an inline MCP plugin does not invalidate the existing cache.
- **`LSP workspaceSymbol` returns empty for every LSP backend in Claude Code 2.1.114.** Upstream harness bug — [claude-code#17149](https://github.com/anthropics/claude-code/issues/17149). Use `Grep` for symbol search.
- **When migrating a plugin from an old wiring:** remove the old raw `mcpServers` entry from `~/.claude.json` (or remove the old single-purpose marketplace) in the *same* session as the new install. Leaving both wired causes name conflicts at startup.

## Response style

Lead with the answer; no preamble, recap, or trailing summary. One reason per claim — don't restate in different words. Structure (headers, bullets) only for ≥ 4 parallel items; otherwise prose. Keep code blocks, `file:line` cites, and concrete evidence. Drop hedge words when not uncertain, announcement verbs ("I found", "I ran"), and editorial commentary.

## Session retro

When a session feels like it's wrapping up (after ship/commit/handover, or on a user closure signal), propose one: "Want a quick retro?"

If yes, four steps:

1. **Went well** — concrete behaviors to keep.
2. **Went poorly** — specific moments that cost time.
3. **Ask the user** — what they'd flag that you missed.
4. **Propose captures** — per lesson: new rule, AGENTS.md edit, memory, or discard. User approves each.

Session lessons decay fast; untracked retros waste the learning.

## Boundaries

**Always:**
- Validate the manifest with `claude plugin validate .` before committing
- Test via local-path marketplace before pushing to github
- Snapshot `~/.claude/settings.json` and `~/.claude.json` to `~/.claude/backups/<timestamp>/` before editing either
- Read one of the official `claude-plugins-official` entries if uncertain about a field

**Ask first:**
- Schema-level changes to `marketplace.json` shape (beyond adding / editing plugin entries)
- Adding plugins that require auth, private repos, or user-specific credentials
- Renaming the marketplace (`name` field) — cascades to every user's settings.json and install references
- First-time `gh repo create` or `git push` for this repo (Phase B publish)

**Never:**
- Manually edit `enabledPlugins` / `extraKnownMarketplaces` in `~/.claude/settings.json` — use the `claude plugin` CLI
- Push to github before local verification passes
- Copy an empty `env: {}` block into a `.mcp.json`
- Install a plugin without disabling any existing plugin that claims the same file extension (LSPs) or MCP server name

## Documentation

- `README.md` — public-facing: what the marketplace is, plugin table, install instructions
- `CLAUDE.md` — thin pointer to this file
- `AGENTS.md` (this file) — everything else: structure, manifest conventions, workflow, gotchas, boundaries
