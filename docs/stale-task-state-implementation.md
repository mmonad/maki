# Stale task state implementation

## Task picker synchronization

Add `chat_index` to `TaskEntry`. Extract entry construction from `open_tasks` into a helper that maps every chat to its current completion state.

Add `ListPicker::select_item_by`, accepting a predicate and returning whether a matching item is visible. Search the filtered item indices, set the selected filtered-row position, and call `ensure_visible`. This preserves search text and filtering while avoiding confusion between filtered-row and underlying-item indices.

Add `App::sync_task_picker`. Return immediately when the picker is closed. Copy the selected entry's `chat_index` into an owned `Option<usize>` before mutating the picker, replace entries from current chats, then restore that identity through `select_item_by` when it remains visible. Otherwise retain `ListPicker`'s normal clamped selection.

Call synchronization after `resolve_or_create_chat` adds a chat, after matching `SubagentHistory`, after `ToolDone`, and after parent success or error cleanup. Whole-run cancellation still closes overlays and does not reopen the picker.

On parent error, mark unresolved children terminal, remove only their routing entries before save so they cannot restore as active, fail in-progress tools, synchronize the picker, save completed child metadata plus authoritative shared history and outputs, then clear the remaining routing state. Correct `sync_ephemeral_state` to build subagent metadata from each surviving `(tool_id, chat_index)` mapping in chat-index order instead of zipping chats with unordered map iteration.

## Terminal reconciliation

Define one shared missing-completion diagnostic in `app/mod.rs`.

Extend message and chat failure helpers to accept an exclusion set of tool IDs. Apply shell exclusions only to the main chat, where shell tools are rendered; child chats fail every unresolved tool. The helper completes each selected tool with an error `ToolDoneEvent` and therefore retires its live buffer. Keep the existing fail-all method as a wrapper for error handling. Tool IDs already identify tool messages throughout the UI, so this preserves the existing uniqueness contract.

Extend `ShellState` with an active-ID set. Reserve the ID synchronously when the shell command is accepted, before returning `Action::ShellCommand`; `ShellEvent::Start` only renders it. Remove the ID on `ShellEvent::Done` after routing the terminal event. Expose a borrowed set for reconciliation. Do not clear it from agent error or cancellation paths.

Handle successful parent `Done` in this exact order: identify and mark unresolved child chats; remove only their routing entries so they cannot restore as active; fail unresolved main-chat tools except active shell IDs and fail all unresolved child-chat tools; synchronize the picker; save completed child metadata plus authoritative shared history and outputs; clear remaining routing state and answers; transition status and fire hooks. Synthetic display errors are not added to model history or persisted tool outputs. Restored tool uses remain terminal because history loading defaults calls without results to completed. Do not reuse a helper that clears all routing before the save.

## Tests

Add app tests proving:

- An open picker refreshes from spinning to complete after standalone `ToolDone`.
- `SubagentHistory` refreshes an open workflow entry.
- A child first observed while the picker is open is inserted.
- Identity selection survives replacement under filtering when still visible.
- Parent `Done` marks unresolved child chats and agent tools as errors, retires live buffers, includes the shared diagnostic, omits unresolved children from saved metadata, preserves completed children, and does not add synthetic failures to shared history or tool outputs.
- A reserved shell ID survives parent `Done` even when `ShellEvent::Start` has not arrived, then Start and Done route normally and ownership is removed only after Done.
- Parent error refreshes an open picker after cleanup and omits unresolved children from saved metadata while preserving completed children with the correct IDs, names, and models.
- Parent cancellation closes the picker, terminalizes child chats and agent tools, and retires live buffers.
- Lifecycle events do not open a closed picker.

Add focused `ListPicker` and message-panel tests for identity selection and exclusion-aware failure where that behavior is clearer below the app layer. Add a persistence test with interleaved completed and unresolved children that asserts surviving IDs, names, and models remain correctly associated and deterministically ordered.

## Validation

Run `cargo fmt --all -- --check`, `cargo clippy --all --tests -- -D warnings`, `cargo nextest run --workspace`, and `cargo test --workspace --doc`.
