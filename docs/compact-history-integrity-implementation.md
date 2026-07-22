# Compact History Integrity Implementation

## Scope

Implement the design in `compact-history-integrity-design.md` within
`maki-agent`. No provider wire format or persisted session format changes.

## History validation

In `maki-agent/src/agent/history.rs`, extract the result-filtering portion of
`sanitize_restored` into a `pub(super)` helper that accepts
`&mut Vec<Message>`.

The helper will:

1. inspect each user message
2. collect tool-use IDs from the immediately preceding assistant message
3. remove result blocks whose IDs are absent from that set
4. remove tool-returned images when the message had results but retained none
5. remove messages that become empty

`sanitize_restored` will call this helper, then retain its existing
restore-only call to `close_dangling_tool_calls` and warning behavior. The helper
will return whether message count or content changed so restore logging remains
accurate even when only blocks are removed.

## Compaction preparation

In `maki-agent/src/agent/compaction.rs`, call the shared helper immediately
after cloning live history and before `strip_images`. This ensures malformed
live history is normalized while tool-returned images are still typed as
images.

## Retry truncation

Replace role-only pair sizing in `truncate_oldest_round` with this sequence:

1. Return when only the final compaction prompt remains.
2. Remove the first message.
3. If that message was user-role and the new first message is assistant-role,
   remove that assistant response too.
4. Run shared result validation, removing results orphaned by those removals.
5. While more than one message remains and the first role is assistant, remove
   it and run validation again.

This preserves truncation progress, keeps the final prompt, and ensures each
removal is followed by ID-aware repair. Validation preserves text in mixed user
messages.

## Tests

Add unit tests in `maki-agent/src/agent/compaction.rs` for:

- `[User, Assistant(ToolUse), User(ToolResult), prompt]`, asserting the exact
  reproduced result is removed with its call
- mismatched result IDs, asserting no orphan result survives
- mixed text and orphan result, asserting text survives
- existing assistant-first and consecutive-assistant behavior
- compaction preparation with an orphan result and tool-returned image,
  capturing provider input to prove both are removed before image replacement
- a chat-pasted image, proving it still becomes `IMAGE_PLACEHOLDER`

Use a shared assertion over every truncation fixture to verify each retained
`ToolResult` is in a user message immediately following an assistant message
with a matching `ToolUse` ID.

Add unit tests in `maki-agent/src/agent/history.rs` for the extracted helper's
change result and block-level removal if existing restore tests do not cover
those contracts.

Use shared constants for test IDs and expected text where repeated.

## Verification

Run:

1. `cargo fmt --all -- --check`
2. targeted `maki-agent` tests while iterating
3. `cargo clippy --all --tests -- -D warnings`
4. `cargo nextest run --workspace`
5. `cargo test --workspace --doc`

Review the final code diff twice after the last fix and rerun checks before
commit.
