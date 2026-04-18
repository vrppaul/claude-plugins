# ty-lsp

Python LSP backed by [`ty`](https://github.com/astral-sh/ty), Astral's Rust-based type checker. Drop-in replacement for Pyright in Claude Code's LSP tool — faster cold start (10–60×), working `workspaceSymbol`, and the same surface for `hover`, `goToDefinition`, `findReferences`, and `documentSymbol`.

## Configuration

The LSP wiring lives in the marketplace manifest — there is no per-plugin config file. See the `lspServers.ty` block in `../../.claude-plugin/marketplace.json`. The server is launched as `uvx ty@latest server` and claims `.py` and `.pyi` files.

## Known gaps vs Pyright

- No `incomingCalls` / `outgoingCalls` — fall back to `grep -rn '\.method_name('` for call-chain tracing.
- No `goToImplementation` — `goToDefinition` is usually what you want anyway.

## First run

The initial `uvx ty@latest server` invocation downloads `ty` into the `uv` cache (one-time cost). Subsequent sessions reuse it.
