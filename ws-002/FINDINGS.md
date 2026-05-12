# Multi-Tab File Sync — Implementation Findings

## Feature

When the same file is opened in multiple tabs or panes, Warp keeps them in sync automatically. Edits in one view reflect in all others, and disk changes (e.g. branch switches) update every view together.

## Core Mechanism: Shared Buffer Model

All editor views of the same file reference a single `ModelHandle<Buffer>` managed by the `GlobalBufferModel` singleton. This means edits flow through one buffer and are visible to all views immediately.

## Key Files

| File | Role |
|------|------|
| `app/src/code/global_buffer_model.rs` | Central singleton. `HashMap<FileId, InternalBufferState>` + `BiMap<FileLocation, FileId>` maps paths to shared buffers. |
| `crates/warp_files/src/lib.rs` | `FileModel` — file I/O, change detection, emits `FileUpdated` events. |
| `crates/watcher/src/lib.rs` | `BulkFilesystemWatcher` — OS-level file monitoring via `notify` crate, 200ms debounce. |
| `app/src/code/view.rs` | `CodeView::construct_editor_for_location` — creates or reuses shared buffer when opening a file. |
| `app/src/code/local_code_editor.rs` | `LocalCodeEditorView` — subscribes to buffer `ContentChanged` events, syncs with LSP. |
| `app/src/code/buffer_location.rs` | `FileLocation` + `SyncClock` — version tracking and conflict detection. |

## How Sync Works

1. **Same file opened twice**: `GlobalBufferModel::open` checks the `location_to_id` BiMap. If buffer exists, returns the existing `BufferState`. Both tabs share the same `ModelHandle<Buffer>`.

2. **Edit in one tab**: `Buffer` model updates -> `ContentChanged` event -> all subscribed views update.

3. **File changes on disk** (branch switch, external edit): `BulkFilesystemWatcher` detects change -> `FileModel` emits `FileUpdated` -> `GlobalBufferModel` computes incremental diff via background task -> applies diff to shared buffer -> all views update.

4. **Conflict detection**: If the buffer has unsaved user edits when a disk change arrives, version mismatch defers the reload to avoid overwriting user work.

## Test Coverage

- **`app/src/code/buffer_location_tests.rs`** — 759 lines covering:
  - Sync clock management
  - Client edits with version matching
  - Server pushes and conflict detection
  - Echo loop prevention
  - Batched edits
  - Sequential operations and conflict resolution

- Tests focus on the `SyncClock`/`BufferLocation` layer (remote buffer sync). The `GlobalBufferModel` multi-tab sharing path has less direct test coverage — it relies on the shared `ModelHandle<Buffer>` reference pattern being correct by construction.
