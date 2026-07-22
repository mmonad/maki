# Stale task state

## Problem

The task picker copies each chat's completion state when it opens. If a task finishes while the picker remains open, the chat becomes terminal but its picker entry keeps spinning. A parent turn can also complete while a tool remains in progress if its terminal tool event is missing, leaving false-running state elsewhere in the UI.

## Design

Treat chat state as authoritative. Each task entry stores its stable chat index. After an event creates a child chat or changes child lifecycle state, rebuild an open task picker from the current chats and restore selection by that index. If filtering excludes the previously selected chat, keep the picker's normal clamped selection. Do not refresh or open a closed picker.

Use existing lifecycle boundaries. Parent `ToolDone` is authoritative for standalone task completion. Preserve the current `SubagentHistory` handling for session-backed chats. Child `Done` stays turn-scoped and is not forwarded or treated as session completion. Correcting standalone error classification after `SubagentHistory` is a separate lifecycle issue outside this fix.

Before handling a successful parent `Done`, enforce the terminal UI invariant for agent-owned work. Mark unresolved child chats as errors and fail unresolved agent tools with a shared missing-completion diagnostic. Track active shell tool IDs explicitly in `ShellState`; add an ID on `ShellEvent::Start`, remove it on `ShellEvent::Done`, and retain tracking across unrelated agent errors and cancellation because those paths do not own shell execution. Shell tools remain owned by the independent shell lifecycle and are excluded by tracked identity rather than an ID prefix. Reconciliation retires affected live tool buffers before the session is saved and routing state is cleared. Normal task execution emits `ToolDone` before parent `Done`; the sweep only handles broken or missing completion paths.

Whole-run error and cancellation retain their existing terminal cleanup and messages. Synchronize the picker after error cleanup. Cancellation continues to close all overlays, including the picker.

The agent event channel preserves producer order, and a normal parent `Done` is emitted after tool processing completes. This change does not add a completed-run fence. Queued prompts and their existing `run_id` behavior remain unchanged.

This is a localized reconciliation step rather than a live picker-data redesign. It keeps `ListPicker` generic and avoids references into mutable chat state.

## Scope

- Give task entries stable chat-index identity.
- Synchronize an open task picker when a child chat is created and after `ToolDone`, `SubagentHistory`, parent success, and parent error.
- Reconcile unresolved child state immediately before successful parent completion while excluding shell tools tracked by `ShellState`.
- Preserve whole-run cancellation's existing behavior of closing the picker.
- Do not change agent protocol events, task execution, queue semantics, or persisted session formats.

## Acceptance criteria

- A standalone task whose `ToolDone` arrives while the picker is open stops spinning without reopening the picker.
- A workflow child whose `SubagentHistory` arrives while the picker is open stops spinning without reopening the picker.
- A newly created child appears in an already-open task picker.
- Parent completion cannot leave a child chat, agent-owned tool, or its live tool buffer shown as running.
- A missing tool completion is rendered as an error containing the shared diagnostic.
- Picker selection remains on the same chat when entries refresh if that chat still matches the current filter.
- Whole-run error updates an open picker after terminal cleanup.
- Whole-run cancellation closes the picker and leaves no task state running.
- A closed picker is not opened by lifecycle events.
