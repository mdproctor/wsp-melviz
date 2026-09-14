## D1: Fix location — server-side textEdit in completion.ts

**Choice:** Compute `textEdit` ranges in `completion.ts` where completion text is already built
**Alternatives:**
- Client-side fix in IntelliJ Kotlin plugin — fights the LSP abstraction, every client needs its own fix, LSP server remains non-conformant
**Rationale:** `completion.ts` already has `textBefore`, `position`, and `needsDash` — all context needed to compute the replacement range. Server-side fix benefits all clients (IntelliJ, VS Code, browser).
**Trade-offs:** Slightly more complex CompletionItem interface; tests need updating to verify ranges
**Sources:** packages/pages-lsp/src/completion.ts, packages/pages-lsp/src/server-node.ts, LSP spec §textDocument/completion
**Exploration:** quick
**Status:** captured
