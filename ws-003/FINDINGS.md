# Investigation Results: Warp's Built-in Code Editor

## What Warp Claims

Warp advertises a **built-in native code editor** (not just terminal input) with:
- Syntax highlighting, tabbed file viewer, find & replace, file tree
- Vim keybindings
- Files auto-update when changed on disk (e.g., after branch switch)
- Designed "for quick, in-flow edits alongside Agent conversations"

## Is it implemented? (Codebase Evidence)

### 1. File watching / live updates — YES, implemented

- [`crates/watcher/`](https://github.com/warpdotdev/warp/tree/master/crates/watcher) uses `notify_debouncer_full` (Rust `notify` crate) for filesystem watching
- [`global_buffer_model.rs`](https://github.com/warpdotdev/warp/blob/master/app/src/code/global_buffer_model.rs) has `IncrementalAutoReload` feature flag — when a file changes on disk, a background diff is computed and the buffer is updated without losing cursor position
- `FileModelEvent::FileUpdated` triggers auto-reload logic with version tracking to avoid conflicts
- Docs confirm: "when a file changes on disk, every view updates together"

### 2. Vim keybindings — YES, implemented

- [`crates/vim/src/vim.rs`](https://github.com/warpdotdev/warp/blob/master/crates/vim/src/vim.rs) — full VimFSA (finite state automaton) with Normal, Insert, Visual, Replace modes
- [`app/src/code/editor/view.rs:354`](https://github.com/warpdotdev/warp/blob/master/app/src/code/editor/view.rs#L354) — gated by `FeatureFlag::VimCodeEditor`
- Supports motions, text objects, registers, dot-repeat, find char (`f`/`t`), etc.

### 3. Co-editing while LLM edits — PARTIALLY implemented, with conflict resolution

This is the nuanced one. It's **not "Google Docs-style" co-editing** but rather:
- The editor has **conflict detection** between local user edits and external changes (including from LLM/agent)
- [`global_buffer_model.rs:207-211`](https://github.com/warpdotdev/warp/blob/master/app/src/code/global_buffer_model.rs#L207-L211) — `RemoteBufferConflict` event fires when server-side changes conflict with local edits
- [`local_code_editor.rs:204`](https://github.com/warpdotdev/warp/blob/master/app/src/code/local_code_editor.rs#L204) — `ConflictResolutionBannerMouseStates` renders a banner offering "discard" or "overwrite" choices
- `SyncClock`-based version tracking for remote buffers
- PR [#10520](https://github.com/warpdotdev/warp/pull/10520) ("Migrate editor to use remote backed buffer") merged May 12, 2026 — wires conflict resolution banner to remote logic
- `BufferUpdatedPush` handler applies incremental edits from the server, but emits conflict if versions diverge

So: **if the LLM edits a file and there are no local unsaved changes**, the editor updates automatically. **If you have unsaved local edits**, a conflict banner appears letting you choose.

### 4. Remote SSH editing — NOT YET user-facing

- `ServerLocal` buffer variant and conflict resolution infrastructure exist in code
- Gated behind `FeatureFlag::SshRemoteServer`
- [Issue #6831](https://github.com/warpdotdev/Warp/issues/6831) (67 upvotes) requesting AI features over SSH

## GitHub Issues

| # | Title | Status |
|---|-------|--------|
| [#10208](https://github.com/warpdotdev/warp/issues/10208) | Code editor: auto-save + auto-refresh external file changes | Open — requesting VS Code-like auto-save; maintainers note auto-reload is "existing behavior" |
| [#10297](https://github.com/warpdotdev/warp/pull/10297) | specs: auto-save for code editor | Open PR (spec only) |
| [#10520](https://github.com/warpdotdev/warp/pull/10520) | Migrate editor to use remote backed buffer | Merged May 12, 2026 |
| [#6921](https://github.com/warpdotdev/warp/issues/6921) | File synchronization/update for splitter screen | Open — users want better auto-sync after Agent edits |
| [#6831](https://github.com/warpdotdev/Warp/issues/6831) | Support Warp AI features for remote server/SSH | Open, 67 upvotes |

## Summary

| Claim | Status |
|-------|--------|
| Editor updates when file changes | **True** — file watcher + incremental auto-reload implemented |
| Vim bindings | **True** — full vim FSA with modes, motions, text objects |
| Co-edit while LLM edits | **Partially true** — not simultaneous co-editing, but auto-reload + conflict resolution banner when versions diverge. Users still report friction (#6921, #10208) |

## Implementation: Built from Scratch in Rust

### Core editor — fully custom

- **[`crates/editor/`](https://github.com/warpdotdev/warp/tree/master/crates/editor) (`warp_editor`)** — entire text editing engine written from scratch by Warp
  - [`content/buffer.rs`](https://github.com/warpdotdev/warp/blob/master/crates/editor/src/content/buffer.rs) — buffer model backed by a custom **`SumTree`** (B-tree variant, [`crates/sum_tree/`](https://github.com/warpdotdev/warp/tree/master/crates/sum_tree)) — same data structure pattern Zed uses (branching factor 6, `Arc`-wrapped nodes). This is Warp's own implementation, not a dependency on Zed
  - [`content/text.rs`](https://github.com/warpdotdev/warp/blob/master/crates/editor/src/content/text.rs) — rich text representation with styled fragments, blocks, markdown
  - [`content/edit.rs`](https://github.com/warpdotdev/warp/blob/master/crates/editor/src/content/edit.rs), [`content/undo.rs`](https://github.com/warpdotdev/warp/blob/master/crates/editor/src/content/undo.rs), [`content/cursor.rs`](https://github.com/warpdotdev/warp/blob/master/crates/editor/src/content/cursor.rs), [`content/selection.rs`](https://github.com/warpdotdev/warp/blob/master/crates/editor/src/content/selection.rs) — all hand-rolled
  - [`render/`](https://github.com/warpdotdev/warp/tree/master/crates/editor/src/render) — custom layout and painting engine for the editor surface (paragraphs, tables, code blocks, images, mermaid diagrams, etc.)
  - No dependency on `ropey`, `xi-rope`, `crop`, or any off-the-shelf text rope library

- **[`crates/vim/`](https://github.com/warpdotdev/warp/tree/master/crates/vim)** — custom Vim FSA, zero external vim libraries. Pure Rust state machine for keystroke interpretation

- **[`crates/warpui/`](https://github.com/warpdotdev/warp/tree/master/crates/warpui)** — Warp's own UI framework. Not gpui, not egui, not any standard GUI toolkit. Custom rendering, windowing, element tree, layout

- **[`crates/lsp/`](https://github.com/warpdotdev/warp/tree/master/crates/lsp)** — custom LSP client implementation wrapping the `lsp-types` crate for protocol types

### External libraries used for specific subsystems

| Library | Purpose |
|---------|---------|
| `arborium` (tree-sitter fork) | Syntax highlighting / parsing via [`crates/syntax_tree/`](https://github.com/warpdotdev/warp/tree/master/crates/syntax_tree) and [`crates/languages/`](https://github.com/warpdotdev/warp/tree/master/crates/languages) — **not** used by the editor buffer itself |
| `syntect` | Fallback syntax highlighting |
| `notify` (via `notify_debouncer_full`) | File system watching ([`crates/watcher/`](https://github.com/warpdotdev/warp/tree/master/crates/watcher)) |
| `lsp-types` | LSP protocol type definitions |
| `imara-diff` | Diff computation for incremental auto-reload |
| `markdown_parser` (internal) | Markdown parsing for rich text |

### Architecture summary

The editor is **not built on any existing IDE framework** (not Monaco, not CodeMirror, not Zed's gpui, not Sublime's engine). It's a ground-up Rust implementation: custom SumTree text storage, custom rendering pipeline, custom vim engine, custom UI framework. The only "borrowed" pattern is the SumTree approach (similar to Zed/xi-editor conceptually), but it's Warp's own code. External dependencies are limited to syntax highlighting (tree-sitter), file watching (notify), and protocol types (lsp-types).

## Sources

- [Built-in code editor - Warp docs](https://docs.warp.dev/code/code-editor/)
- [Code Editor Vim Keybindings - Warp docs](https://docs.warp.dev/code/code-editor/code-editor-vim-keybindings)
- [Building a first-class code editor in Warp](https://www.warp.dev/blog/building-a-first-class-code-editor-in-warp)
- [GitHub Issue #10208](https://github.com/warpdotdev/warp/issues/10208)
- [GitHub Issue #6921](https://github.com/warpdotdev/warp/issues/6921)
- [GitHub Issue #6831](https://github.com/warpdotdev/Warp/issues/6831)
- [GitHub PR #10520](https://github.com/warpdotdev/warp/pull/10520)
