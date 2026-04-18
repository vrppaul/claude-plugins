# Handover — bootstrap to populated marketplace

> **This file is disposable.** It exists only to hand the bootstrap commit off to the next agent. Once Phase A passes end-to-end, delete this file and commit `chore: remove bootstrap handover`. The persistent rules live in `AGENTS.md` and `CLAUDE.md`; this file duplicates some of that content so you can execute without juggling tabs.

## Goal

Replace the three separate Claude Code extension wirings with a single `vrppaul-tools` marketplace hosting all three:

| What | Current state | Target |
|---|---|---|
| `claude-review` | plugin in its own single-purpose `claude-review-marketplace` (repo `vrppaul/claude-review`) | plugin under `vrppaul-tools` via external git-subdir |
| `semantic-code` | raw `mcpServers` entry in `~/.claude.json:963-972` | inline MCP plugin under `vrppaul-tools` |
| Python LSP | `pyright-lsp@claude-plugins-official` enabled | disabled; swapped for `ty-lsp@vrppaul-tools` (inline LSP) |

`ty` is Astral's Rust-based type checker — 10–60× faster cold start than Pyright, same `hover` / `goToDefinition` / `findReferences` / `documentSymbol`. **Known gaps:** no `incomingCalls` / `outgoingCalls`, no `goToImplementation`. `workspaceSymbol` is also broken for every LSP backend in Claude Code 2.1.114 ([claude-code#17149](https://github.com/anthropics/claude-code/issues/17149) — harness sends `query: ""` regardless of cursor). That was the real cause of the "pyright silent-empty" regression that originally motivated this swap; it's not a ty advantage. Grep fallback is fine for call chains and symbol search.

## Phase A — local dry-run

Work in this repo (`~/projects/claude-plugins/`). No github push, no public commit, until Phase A verification passes.

| # | Step | Notes |
|---|---|---|
| A1 | Write `.claude-plugin/marketplace.json` with three plugin entries | templates below |
| A2 | Write `plugins/ty-lsp/README.md` | docs only; `lspServers` lives in `marketplace.json` |
| A3 | Write `plugins/semantic-code/.mcp.json` | copy `command` / `args` from `~/.claude.json:963-972`; **omit** `env: {}` |
| A4 | Write `plugins/semantic-code/README.md` | one-paragraph description |
| A5 | Update `README.md` plugin table — real rows, not bootstrap placeholder | |
| A6 | `git add -A && git commit -m "feat(plugins): populate initial plugin set"` | |
| A7 | `claude plugin validate .` (from repo root) | must pass before continuing |
| A8 | Snapshot the two live config files | `mkdir -p ~/.claude/backups/$(date +%Y%m%d-%H%M%S) && cp ~/.claude/settings.json ~/.claude.json ~/.claude/backups/$(date +%Y%m%d-%H%M%S)/` |
| A9 | `claude plugin marketplace add ~/projects/claude-plugins` | registers as local-path marketplace |
| A10 | `claude plugin marketplace list` | confirm `vrppaul-tools` present, sourced from the local path |
| A11 | `claude plugin disable pyright-lsp@claude-plugins-official` | do this **before** installing ty-lsp — avoids two plugins claiming `.py` |
| A12 | `claude plugin install ty-lsp@vrppaul-tools` | |
| A13 | `claude plugin install semantic-code@vrppaul-tools` | |
| A14 | `claude plugin install claude-review@vrppaul-tools` | |
| A15 | Manually remove the `mcpServers.semantic-code` block at `~/.claude.json:963-972` | the plugin takes over; leave the surrounding JSON valid |
| A16 | `claude plugin marketplace remove claude-review-marketplace` | no longer needed |
| A17 | **Ask user to restart Claude Code**, then run the verification checks below in the fresh session |

If any verification check fails, follow *Rollback* below and investigate before retrying.

## Phase B — github publish

Only after Phase A verification has passed in a restarted session.

| # | Step |
|---|---|
| B1 | Confirm with user before running: `gh repo create vrppaul/claude-plugins --public --source ~/projects/claude-plugins --push` |
| B2 | `claude plugin marketplace remove vrppaul-tools` |
| B3 | `claude plugin marketplace add https://github.com/vrppaul/claude-plugins.git` |
| B4 | **Ask user to restart**; re-run verification |

## Verification

Run after A17 and again after B4:

1. `claude plugin marketplace list` → `vrppaul-tools` present; `claude-review-marketplace` absent.
2. `claude plugin list` → `claude-review`, `semantic-code`, `ty-lsp` enabled under `vrppaul-tools`; `pyright-lsp` disabled.
3. `LSP` tool with `operation: documentSymbol` on any `.py` file → returns a symbol list.
4. `LSP hover` on a typed variable → returns an inferred type (confirms `ty` is answering).
5. ~~`LSP workspaceSymbol`~~ — skip. Broken for every LSP backend in Claude Code 2.1.114 ([claude-code#17149](https://github.com/anthropics/claude-code/issues/17149)). Re-enable once the harness bug is fixed upstream.
6. `mcp__semantic-code__index_status` for the current project → returns stats from `~/.cache/semantic-code-mcp/<hash>/`. Same cache as before the switch.
7. `mcp__semantic-code__search_code` with a simple query → returns results.
8. `/claude-review:review-ui` skill → opens the UI.

## Rollback

If Phase A verification fails:

1. Restore both config files from the backup: `cp ~/.claude/backups/<timestamp>/settings.json ~/.claude/ && cp ~/.claude/backups/<timestamp>/.claude.json ~/`.
2. `claude plugin marketplace remove vrppaul-tools` (if added).
3. If `claude-review-marketplace` was removed in A16, re-add it: `claude plugin marketplace add https://github.com/vrppaul/claude-review.git`.
4. Ask the user to restart; confirm the old wiring is back.

Pre-B1, nothing has been pushed to github — no public state to undo.

## Manifest templates

Drop these into `.claude-plugin/marketplace.json` under the `plugins` array. Copy `"$schema"`, `"name"`, `"description"`, `"owner"` from the current file skeleton (or lift the shape from `claude-plugins-official/.claude-plugin/marketplace.json`).

```json
{
  "name": "ty-lsp",
  "description": "Python LSP via ty (Astral) — fast Rust-based type checker and language server.",
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

```json
{
  "name": "semantic-code",
  "description": "Semantic code search MCP — tree-sitter chunking + sentence-transformers embeddings.",
  "version": "0.1.0",
  "source": "./plugins/semantic-code",
  "category": "development"
}
```

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

`plugins/semantic-code/.mcp.json`:

```json
{
  "semantic-code": {
    "command": "uvx",
    "args": ["--index", "pytorch-cpu=https://download.pytorch.org/whl/cpu", "semantic-code-mcp"]
  }
}
```

## Gotchas

- **No `incomingCalls` in ty LSP.** Fall back to grep for call chains (`grep -rn '\.method_name('`). Mention this tradeoff when the user reaches for LSP call hierarchy.
- **First `uvx ty@latest server` invocation downloads `ty`.** Cold start will pay that cost once; subsequent sessions reuse the `uv` cache.
- **`~/.claude.json` is not under git.** The A8 backup is the only rollback path for it — do not skip.
- **`env: {}` in `.mcp.json`.** Omit it. The old raw `mcpServers` entry at `~/.claude.json:971` has `"env": {}`; do not copy that forward. Official `.mcp.json` files in `claude-plugins-official` don't include empty `env` blocks.
- **Semantic-code cache survives the switch.** The cache at `~/.cache/semantic-code-mcp/<project-hash>/` is keyed on the absolute project path, not on how the MCP server is launched. Same `uvx` command + same project path = same cache. No re-index needed.
- **Ordering matters at A11.** Disable pyright-lsp before installing ty-lsp. If both are enabled for `.py` simultaneously, Claude Code's dispatch is ambiguous.
- **Follow-up cleanup (separate session).** After B4 succeeds, `vrppaul/claude-review`'s `.claude-plugin/marketplace.json` is dead weight — it advertises a marketplace no-one reads anymore. Delete it in a follow-up commit there, but leave `./plugin` alone since it's referenced via git-subdir.

## Closing the handover

Last step in Phase B:

```bash
git rm HANDOVER.md
git commit -m "chore: remove bootstrap handover"
git push
```

The persistent documentation in `AGENTS.md` and `CLAUDE.md` carries everything needed for long-term maintenance.
