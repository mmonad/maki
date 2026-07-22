# Task token progress

## Problem

A batch tool call shows its input tokens, output tokens, and cost at the right edge of its header. A task tool call shows its subagent model but not the subagent's cumulative usage, which makes active work harder to distinguish from a stalled task.

## Design

Show each task subagent's cumulative usage on its `task` tool header using the same right-aligned format as other tool usage: `<input>↑ <output>↓ $<cost>`. Keep the model annotation beside the task name and place usage in the right-side usage area, matching the top-level batch header in the reference UI. This applies both to standalone tasks and task children rendered inside a batch.

Update standalone task usage after every internal model turn. Batched task usage updates after each internal model turn through its live callback. Input includes uncached, cache-read, and cache-creation tokens. Output is the cumulative output count. Cost uses the subagent session model's pricing and fast-mode multiplier. If auto-compaction uses a separately priced model, its tokens retain the existing aggregate-cost approximation and are priced as the session model. Models with zero pricing omit cost, matching the existing formatter.

Usage is a per-turn activity indicator, not a completion percentage or continuous streaming counter. A long first model response or tool execution can still show no usage until that turn completes. Task usage is live presentation state and is not reconstructed when restoring a session. At narrow full-terminal widths, batch child headers omit right-side usage when it does not fit. A narrower split viewport may clip terminal-width-aligned batch usage because Lua buffers do not receive their eventual viewport width.

## Scope

This changes task usage propagation and parent tool-header presentation. It does not change the task picker, agent cancellation, timeout policy, or task lifecycle states.

## Acceptance criteria

- A task header displays cumulative subagent input tokens, output tokens, and cost in the existing tool-usage format.
- Usage appears at the right edge of the terminal-width batch buffer, matching batch-level usage in the normal full-width view.
- The subagent model annotation remains visible.
- Usage updates after each completed subagent turn.
- Cached and cache-creation input tokens are included.
- Cost uses the subagent session model and fast-mode pricing, with the documented compaction approximation.
- Zero-priced models omit cost.
- Concurrent standalone and batched tasks update only their matching task header.
- Restored tasks omit live usage metadata.
