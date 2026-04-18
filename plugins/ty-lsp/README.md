# ty-lsp

Python LSP backed by [`ty`](https://github.com/astral-sh/ty), Astral's Rust-based type checker. Drop-in replacement for Pyright in Claude Code's LSP tool — 10–60× faster cold start, with the same surface for `hover`, `goToDefinition`, `findReferences`, and `documentSymbol`.

## Configuration

The LSP wiring lives in the marketplace manifest — there is no per-plugin config file. See the `lspServers.ty` block in `../../.claude-plugin/marketplace.json`. The server is launched as `uvx ty@latest server` and claims `.py` and `.pyi` files.

## Known limitations

- `workspaceSymbol` returns empty through Claude Code's LSP tool — upstream harness bug ([claude-code#17149](https://github.com/anthropics/claude-code/issues/17149)), not a ty issue. The harness hardcodes `query: ""` for every `workspaceSymbol` request regardless of cursor position, so this affects every LSP backend equally. ty answers correctly when queried via raw LSP. Workaround: use `Grep` for symbol search.
- No `incomingCalls` / `outgoingCalls` — fall back to `grep -rn '\.method_name('` for call-chain tracing.
- No `goToImplementation` — `goToDefinition` is usually what you want.

## First run

The initial `uvx ty@latest server` invocation downloads `ty` into the `uv` cache (one-time cost). Subsequent sessions reuse it.
