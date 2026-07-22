# Task token progress implementation

## Live usage data

Add a `ToolLive::Usage(String)` payload containing the fully formatted cumulative usage string. Format it in Rust from cumulative `TokenUsage`, resolved subagent `ModelPricing`, and fast-mode state before crossing into Lua, so the `on_usage` callback receives one string like `12.3k↑ 456↓ $0.123`. Before spawning the session forwarding task in `session()`, clone the callback sink installed by `maki.agent.call_tool` from that function's local `agent_ctx.live_sink` and move the clone into the forwarding task. `call_tool` first clears any inherited sink, then installs this new per-child sink before dispatch, so this does not enable grandchild streaming. It observes each child `TurnComplete` and accumulates its usage with explicit `saturating_add` for every `TokenUsage` field in a forwarding-task-local aggregate; do not use the type's ordinary `AddAssign` implementation. Format total input with widened or saturating arithmetic instead of `TokenUsage::total_input()`, whose ordinary additions can overflow. After each accumulation, send the current totals through that sink when present, then preserve normal forwarding of the same `TurnComplete`. Continue using terminal `Done` only for existing prompt totals and suppression. This prevents compaction `Done` events from causing duplicate progress updates. Usage includes compaction turns. As with the existing aggregate turn display, cost uses the session model's pricing, so a separately priced compaction turn is an accepted approximation.

The live payload remains runtime-only and is not serialized or persisted.

## Standalone task routing

For standalone tasks, add cloned `pricing: ModelPricing` and `fast: bool` runtime fields to `SubagentInfo`, mark both `#[serde(skip)]`, and populate them from `SessionState.params.model.pricing` after either explicit resolution or inherited-model cloning, plus the effective fast state when first creating `SubagentInfo`. Update every constructor and test fixture. Subagent `TurnComplete` events already reach the UI. After adding that event's usage to the child `Chat`, directly format the chat's cumulative usage with these fields and set the string on the parent task tool message by its exact tool ID in the same event handler, without using the child's pending-turn or `ToolResultsSubmitted` path.

Add targeted `Chat` and `MessagesPanel` methods that set usage by tool ID. Do not use `set_turn_usage_on_last_tool`, because concurrent task completions must update their matching parent tool call.

## Batched task routing

Extend `maki.agent.call_tool` options and its generated Lua API documentation with an `on_usage` callback and forward `ToolLive::Usage` to it. Include `on_usage` when deciding whether to create the live-event channel, so usage-only callers work. The batch plugin stores the formatted usage on the matching child and rerenders. Its child header renderer measures every left span, annotation, and usage with one local UTF-8 width helper, requires at least one separating column, and pads usage across the batch buffer width derived from `maki.ui.terminal_size().cols` minus the existing tool-body indent. If the terminal width is zero or both sides do not fit, omit usage. This mirrors `append_right_info` in the normal full-width view; narrower split viewports may clip because Lua buffers do not receive their eventual render width. Batch child headers have no timestamp. Keep the model annotation on the left and use the same compact token and cost format as `format_turn_usage`.

Task usage is not included in session or batch state. Restored standalone and batch-child views omit live usage metadata.

Regenerate user documentation with `just gen-docs` after updating the Rust API doc comment.

## Tests

Add tests proving:

- standalone usage is attached to the matching parent task tool;
- batched task usage updates the matching child header;
- cumulative input, cached input, cache creation, and output are formatted;
- child pricing, zero pricing, and fast pricing produce the expected cost;
- repeated and compaction turns update cumulative usage without double-counting terminal `Done`;
- concurrent tasks do not update each other's headers;
- model annotations remain intact;
- narrow terminal-width batch headers omit usage rather than corrupting layout;
- restored standalone and batch-child views omit live usage metadata.
