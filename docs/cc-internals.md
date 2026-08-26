# Claude Code internals

How CC processes session files. Reference for anyone modifying the bridge's
JSONL writer, tool pairing, attachment handling, or system prompt forwarding.
Derived from CC 2.0.89 source (`~/dev/fork/claude-code/src/`). Correctness
verified in `docs/cc-source-audit.md`.

## The pipeline

```
JSONL on disk
  │
  ├─ loadTranscriptFile        sessionStorage.ts:3472
  │    reads lines, skips non-transcript types, builds Map<UUID, TranscriptMessage>
  │    bridges legacy progress entries via parentUuid rewriting
  │    handles compact boundaries (nullifies parentUuid, resets collapse state)
  │
  ├─ buildConversationChain    sessionStorage.ts:2069
  │    walks parentUuid from the latest leaf backward to root
  │    reverses to chronological order
  │    post-pass: recoverOrphanedParallelToolResults — finds sibling assistants
  │      with same message.id that the single-parent walk orphaned, splices
  │      them + their tool_results after the on-chain anchor
  │      sorts recovered records by timestamp (ties = nondeterministic)
  │
  ├─ normalizeMessages         messages.ts:741
  │    splits multi-block messages into one-per-block for the REPL
  │    (not directly relevant to the bridge — we write one block per record)
  │
  ├─ normalizeMessagesForAPI   messages.ts:1989  ← the big one
  │    13+ transforms, applied in this order:
  │    1. reorderAttachmentsForAPI — bubble attachments up to nearest stop
  │    2. filter isVirtual messages
  │    3. build strip map for PDF/image errors targeting isMeta messages
  │    4. filter progress, non-local-command system, synthetic API errors
  │    5. normalize each message by type:
  │       - system (local_command) → user message
  │       - user → strip tool_references, strip error-targeted blocks,
  │         inject TOOL_REFERENCE_TURN_BOUNDARY (gated on tengu_toolref_defer_j8m),
  │         merge consecutive users
  │       - assistant → normalize tool inputs, merge by message.id,
  │         dedupe tool_use blocks
  │       - attachment → normalizeAttachmentForAPI, optionally
  │         ensureSystemReminderWrap (gated on tengu_chair_sermon),
  │         merge into preceding user
  │    6. relocateToolReferenceSiblings (gated on tengu_toolref_defer_j8m)
  │    7. filterOrphanedThinkingOnlyMessages
  │    8. filterTrailingThinkingFromLastAssistant
  │    9. filterWhitespaceOnlyAssistantMessages
  │   10. ensureNonEmptyAssistantContent
  │   11. smooshSystemReminderSiblings (gated on tengu_chair_sermon)
  │   12. sanitizeErrorToolResultContent
  │   13. append [id:xxxx] tags to non-isMeta user messages
  │       (gated on HISTORY_SNIP feature + isSnipRuntimeEnabled)
  │   14. validateImagesForAPI
  │
  ├─ ensureToolResultPairing   messages.ts:5120
  │    runs AFTER normalizeMessagesForAPI
  │    forward: missing tool_result → synthetic error result
  │    reverse: orphan tool_result → strip
  │    also: dedupe tool_use IDs, dedupe tool_result IDs,
  │    strip orphan server_tool_use/mcp_tool_use
  │
  └─ addCacheBreakpoints       claude.ts:3070
       one cache_control marker on the last message (or second-to-last
       for fire-and-forget forks). system prompt gets its own markers
       via buildSystemPromptBlocks / splitSysPromptPrefix.
```

## JSONL record types CC recognizes

`isTranscriptMessage` (sessionStorage.ts:139) is the gate:

| type | participates in chain? | notes |
|------|----------------------|-------|
| `user` | yes | prompt text or tool_result blocks |
| `assistant` | yes | model response, tool_use blocks |
| `attachment` | yes | `@file` expansions, skill listings, edit notifications |
| `system` | yes | compact boundaries, turn durations, local commands |
| `progress` | **no** (legacy bridged) | ephemeral UI state, stripped on read |
| `summary` | no | metadata, keyed by leafUuid |
| `custom-title` | no | metadata |
| `tag` | no | metadata |
| everything else | no | metadata (agent-name, mode, pr-link, etc.) |

The bridge writes only `user`, `assistant`, `attachment`. This is correct —
system records are CC-internal.

## Record fields that matter

**Load-bearing** (wrong value = broken):
- `uuid` — chain identity. `buildConversationChain` walks these.
- `parentUuid` — chain link. null = root. cycle = partial transcript.
- `timestamp` — sort key for orphan recovery. ties = nondeterministic sort.
- `message.id` (assistant) — merge key. same id = same API response = merged.
- `isSidechain` — false for main conversation. true = filtered from /resume.

**Affects normalization output** (wrong value = different API bytes):
- `isMeta` (user) — skips [id:] tag injection, changes merge UUID selection,
  targets message for error-block stripping. Bridge correctly omits (= undefined
  = not meta).
- `message.content` — obviously.

**Inert** (never read by normalization):
- `userType` — written, never read back
- `requestId` — on assistant records, never consulted by normalization
- `slug` — CC overwrites on next write
- `entrypoint` — analytics only
- `version` — LogOption display only (today)
- `gitBranch` — LogOption display only

## [id:xxxx] tags

`deriveShortMessageId(uuid)` (messages.ts:200): takes first 10 hex chars of
the UUID (dashes removed), parseInt as hex, toString(36), slice 6 chars.
Deterministic from UUID.

Appended as `\n[id:xxxx]` to the last text block of every non-isMeta user
message. Only when `HISTORY_SNIP` feature is enabled AND `isSnipRuntimeEnabled()`
returns true. Skipped in `NODE_ENV=test`.

The bridge uses deterministic UUIDs (SHA-256 of sessionId + record index) so
the [id:] tags are stable across rebuilds → cache-compatible prefix bytes.

## Attachment reordering

`reorderAttachmentsForAPI` (messages.ts:1481) scans bottom-up:
- Attachment records are collected
- On hitting a **stopping point** (assistant message OR user message whose
  first content block is tool_result), pending attachments are flushed
  **after** the stopping point
- Remaining attachments bubble to the top

The bridge places attachments after their parent user record. The reorder
either leaves them in place (parent is a text message → it's the stopping
point) or bubbles them to a semantically equivalent position.

## Feature flags that affect normalization

| Flag | Type | What it changes |
|------|------|----------------|
| `tengu_toolref_defer_j8m` | GrowthBook | Where tool_reference turn-boundary text siblings go |
| `tengu_chair_sermon` | GrowthBook | Whether attachments get `<system-reminder>` wrap, smoosh pass |
| `HISTORY_SNIP` | Compile-time | Whether [id:] tags are injected on user messages |
| `tengu_anti_distill_fake_tool_injection` | GrowthBook | Injects fake tool calls (distillation defense) |

GrowthBook flags can flip mid-session → different normalization → cache miss.
This is the dominant cause of the ~25% cache miss baseline.

## System prompt caching

`buildSystemPromptBlocks` (claude.ts:3213) splits on `\n\n---\n\n` boundaries
via `splitSysPromptPrefix`. Each block gets a `cache_control` marker with
`{type: "ephemeral", ttl: "1h"}` (for subscribers). The bridge's forwarded
prompt has no such delimiters → single block → single marker → fine.

Message-level caching: one `cache_control` marker on the last message
(`addCacheBreakpoints`, claude.ts:3070). Everything before that marker is the
cached prefix. [id:] tags on earlier messages land in this prefix, which is
why deterministic UUIDs matter.

## Tool pairing repair

CC has two repair mechanisms:

1. **`recoverOrphanedParallelToolResults`** (sessionStorage.ts:2097) — read-time.
   Recovers sibling assistant blocks and their tool_results that the
   single-parent chain walk orphaned. Groups by `message.id`. Only matters for
   CC's own streaming records (one assistant per content_block_stop).

2. **`ensureToolResultPairing`** (messages.ts:5120) — post-normalization.
   Forward: missing result → synthetic error. Reverse: orphan result → strip.
   Also dedupes tool_use IDs and tool_result IDs across messages.

The bridge's own `repairToolPairing` (cc-session/repair.ts) runs before
writing JSONL and is compatible with both: its synthetic results look validly
paired to CC's repair, and `importMessages` applies it again idempotently.
