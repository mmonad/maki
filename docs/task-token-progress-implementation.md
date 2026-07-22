# Task token progress implementation

## Changes

### Picker rendering

Update `ListPicker` so `is_spinning()` prepends the animated spinner to an item's existing detail. Build one owned display string and use its complete width for label truncation. A spinning item without detail keeps the current spinner-only rendering. Add an accessor that reports whether an open picker has a spinning item, and include that state in `App::is_animating` so idle tasks continue repainting the spinner.

### Task entries

Add an owned detail string to `TaskEntry`. Build all entries through one `App::task_entries` helper:

- Main chat: no finished state or detail.
- Finished subtask: completion detail and no spinner.
- Running subtask: spinner plus `<count> tokens` detail.

Compute count by converting `input`, `cache_read`, `cache_creation`, and `output` to `u64`, summing them, and clamping to `u32::MAX` before calling `format_tokens`.

### Refresh

Add `App::refresh_task_picker`, which calls `replace_items` only while the picker is open. Invoke it only in the creation branch of `resolve_or_create_chat`, after subtask token accumulation, after a `ToolDone` that matches a subtask, before the early return from matching `SubagentHistory`, after direct subtask cancellation, and after parent-error `finish_subagents`. Existing whole-run cancellation continues closing overlays and needs no refresh.

### Tests

Add unit tests for:

- Spinner and detail rendering together with correct width, preserving spinner-only behavior and animation scheduling.
- Main chat exclusion.
- Running zero-use display.
- Input, cache, cache-creation, and output inclusion.
- Saturated display calculation.
- Finished-task completion detail.
- Open-picker refresh after task creation, usage, normal completion, workflow completion, direct subtask cancellation, and parent error.
