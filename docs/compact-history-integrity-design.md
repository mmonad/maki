# Compact History Integrity Design

## Problem

OpenAI Responses rejects a `function_call_output` unless the same request contains
the preceding `function_call` with a matching `call_id`.

Session `CdHUnTjUrgtDrvC9Wz1dP` reproduced a violation during `/compact`. Its
history began with:

1. a user request
2. an assistant tool call `call_dMZDTpEfz2JxMvFbqFHua1Zy`
3. the matching user tool result

When the compaction request exceeded the context window,
`truncate_oldest_round` removed the first user message and the following
assistant message. It left the tool result at the front of the retry history.
The OpenAI Responses converter serialized that result as an orphaned
`function_call_output`, and the API returned HTTP 400.

## Goal

Every compaction retry must preserve valid tool-call exchanges. Removing a tool
call must also remove its associated result message. Compaction must still make
progress when histories contain unusual consecutive roles or malformed tool
exchanges.

## Non-goals

- Changing persisted session data.
- Redesigning the provider-independent message model.
- Silently repairing arbitrary malformed histories in provider converters.
- Changing normal conversation or tool execution behavior.

## Design

Extract restored history's existing tool-result validation into a reusable
history helper. For every user message, the helper retains a `ToolResult` only
when its ID appears in the immediately preceding assistant message. It removes
an empty result message and preserves unrelated text. It retains tool-returned
images only when at least one result remains valid, matching current restore
behavior. Compaction runs this validation before replacing images with text and
again after every retry truncation, so contextless tool-returned images remain
identifiable when an exchange is removed.

Truncation then removes the oldest logical round and restores invariants:

- If history starts with a user message, remove it and one following assistant
  response when present.
- If history starts with an assistant message, remove it.
- Validate all remaining tool results after these removals. Results whose calls
  were removed disappear, while unrelated text in a mixed message survives.
- If validation exposes another assistant at the front, remove that assistant
  and validate again. Repeat until history starts with a user message or only
  the compaction prompt remains.

Valid result messages do not need special-case pairing. Removing their assistant
call makes them invalid, and the shared validator removes precisely the result
blocks tied to the removed call. The same rule also removes pre-existing
mismatched results rather than sending them to a provider.

The compaction prompt is always the final message and is never removed.

## Safety

The change reuses the restore path's established ID-level rule instead of
introducing a second definition of a valid tool exchange. Persistence and
provider conversion remain unchanged. Compaction validation removes only
orphaned result blocks and contextless tool-returned images; ordinary text and
chat-pasted images remain. Restored histories keep their existing additional
behavior of closing a final dangling tool call.


## Validation

Regression tests will cover:

- the exact session prefix that produced the orphaned output
- assistant-first tool-call/result exchange removal
- mismatched tool-call and result IDs
- consecutive assistant messages
- single-message and plain-message boundaries

After each tested truncation, no retained tool result may lose the matching
assistant tool call because of that truncation.
