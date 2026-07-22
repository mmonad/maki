# Task token progress

## Problem

The task picker shows whether a subtask is running or finished, but gives no indication that a running subtask is making progress. This can make active work appear stuck.

## Design

Show each running subtask's cumulative token usage in the task picker's detail column, alongside its spinner. Extend `ListPicker` rendering so spinning items can retain and render their detail instead of replacing it with the spinner. Use the existing per-chat `TokenUsage`, widening every input, cache, and output component to `u64` before summing and clamping for the UI's existing compact token formatter, with a `tokens` label.

The main chat keeps its current presentation. Finished subtasks keep the completion mark because activity is useful while waiting, while completion state is more useful afterward.

Token usage is a per-turn activity indicator, not a completion percentage or continuous streaming counter. Compact rounding means some small updates may not change the displayed value. A single task-entry builder is the authoritative projection of chat state. When the picker is open, usage updates, task creation, normal completion, workflow history completion, direct subtask cancellation, and parent failure replace its items in place so search and selection are preserved where possible. Whole-run cancellation keeps its existing behavior of closing overlays.

## Scope

This change only affects task-picker presentation and refresh behavior. It does not alter agent events, persistence, cancellation, timeout policy, or task lifecycle states.

## Acceptance criteria

- A running subtask displays a spinner and its cumulative total input and output token count.
- Cached and cache-creation input tokens are included.
- The count uses the existing compact token format and cannot wrap during display calculation.
- An open task picker updates when new usage arrives or the task finishes.
- Finished subtasks display the existing completion mark instead of token activity.
- The main chat does not display a token-activity detail.
- Tests cover running, zero-use, cached-input, finished, main-chat, task creation, usage refresh, normal and workflow completion, cancellation, and parent-error refresh cases.
