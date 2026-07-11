# LemmaScript Candidates — Flue (Dafny Backend)

Case study: formally verifying pure logic extracted from **Flue**
([`withastro/flue`](https://github.com/withastro/flue)) with LemmaScript's Dafny
backend.

**Relationship to the pi case study.** Flue is *not* a copy of pi. It is Astro's
own agent-harness framework that depends on pi's lower layers
(`@earendil-works/pi-agent-core`, `@earendil-works/pi-ai` — see
`packages/runtime/package.json`) for message types and provider completion, and
builds its own session/conversation harness on top. So the interesting targets
split in two:

- **Echoes of pi** — Flue's context-compaction cut-point selector solves the
  same "never orphan a `toolResult` at the cut" problem the pi case study
  proved, but over a *different* message model and a *different* implementation
  (`role === 'user' || role === 'assistant'` whitelist vs. pi's six-role
  fall-through switch). Same theorem, a second implementation over a different
  model — the property transfers, the proof is new.
- **Flue-specific** — pure logic unique to Flue's harness: the
  interrupted-submission classifier (`classifySubmissionState`), the
  event-sourced conversation reducer and its **no-orphan-at-projection**
  guarantee, and the usage-aggregation algebra. These are the headline targets.

All paths below are in `packages/runtime/src/`.

## Verified so far

- **`countConsecutiveRetryableModelErrors`** (#1b) — proven **in place**,
  function body byte-identical, **equal to a recursive spec** of its backward
  scan (not merely bounded). Full write-up in
  [`README_LemmaScript.md`](README_LemmaScript.md). It drove three LemmaScript
  toolchain features to completion (native Dafny `continue`;
  `noUncheckedIndexedAccess` optional-index modeling; optional-narrowing past an
  `opt?.disc` guard, composing with discriminant narrowing) — so every target
  below now lands on a toolchain that handles this class of harness code.

Everything after this point is the roadmap.

---

## Tier 1 — Excellent Fit

Pure functions with rich verifiable properties that map directly onto
LemmaScript's supported fragment (arrays, numbers, strings, discriminated
unions, loops with invariants, `Set`/`Map`).

### 1. `classifySubmissionState` — Interrupted-Submission Classifier (`submission-state.ts:119`)

**What it does:** Given the active-path entries that follow a persisted
submission input, classifies how far the submission progressed before the
session was last saved, into a discriminated `SubmissionState`
(`absent` | `advanced_past_input` | `completed` | `tool_use_unresolved` |
`terminal_error` | `resume{mode}`). The single source of truth for both
reconciliation and the resume preamble.

**Why it's the flagship candidate:**
- Core crash-recovery correctness — the highest-value safety target in the repo.
- Pure: reads a `readonly CanonicalSubmissionEntry[]` and a numeric context
  window, returns a value. No I/O, no mutation.
- A tall, order-sensitive decision tree — exactly what a machine checker is good
  at pinning down.

**Properties to verify:**
- **Totality.** Always returns a `SubmissionState` (every path returns; the
  final `terminal_error` is the catch-all).
- **Precedence is exactly the source order.** `advanced_past_input` (a later
  `user` message) dominates everything; among the rest, `completed` (stop/length)
  beats `overflow` beats `transient_retry` beats `stream_continuation` beats the
  `toolUse` cases beats the `aborted` cases beats `terminal_error`. Encode as
  mutually-exclusive guards and prove the chosen kind matches the first
  satisfied guard.
- **`resume` invariants.** Every non-`input_only` resume carries an `assistant`;
  `input_only` never does (matches the `SubmissionState` union shape).
- **`overflow` requires the flag.** `kind === 'completed' && overflow` only when
  `isContextOverflow` held on a stop/length response, etc.

**Modeling notes:** `isContextOverflow` and the `AssistantMessage` graph come
from pi-ai — shadow them the way pi's case study shadowed `SessionTreeEntry`
(`//@ declare-type`), and make `isContextOverflow` an `//@ extern`. The regex in
`isRetryableModelError` (#1b) is a trust boundary: model it as a spec-level
`isRetryableMessage(s): bool`.

### 1b. `countConsecutiveRetryableModelErrors` (`submission-state.ts`) — ✅ **Verified in place**

**What it does:** Scanning the persisted entries from the end, counts consecutive
retryable **assistant** errors — skipping `compaction` entries and non-assistant
messages, stopping at the first `user` message (an operation boundary) or first
non-retryable assistant.

**Proven** (Dafny; function body byte-identical, `//@` annotations only):

```
\result === countRetryableSuffix(entries, entries.length)
```

— equal to a recursive spec-level mirror of the scan, **not merely bounded**. The
loop invariant `count + countRetryableSuffix(entries, i+1) === countRetryableSuffix(entries, entries.length)`
discharges automatically. Full write-up: [`README_LemmaScript.md`](README_LemmaScript.md).

**What it took:** three LemmaScript toolchain gaps, all now closed — native Dafny
`continue`; bounds-guarded modeling of `noUncheckedIndexedAccess` optional array
indexing (`const entry = entries[i]`); and optional narrowing past the
`if (entry?.type !== 'message') continue` guard, composing with the
`CanonicalSubmissionEntry` discriminant match. Modeling: `//@ declare-type` shadows
for `AgentMessage` / `AssistantMessage` (`= AgentMessage`), `//@ extern` on
`isRetryableModelError` (its regex is a trust boundary).

### 1c. `findTrailingPartialToolBatch` (`submission-state.ts:244`)

**What it does:** Locates the trailing `toolUse` turn whose persisted
tool-result batch is incomplete (the shape an abort leaves mid-batch). Returns
`undefined` when the batch is complete, a stream continuation exists, or an
unexpected entry breaks the trailing shape.

**Properties to verify:**
- **Result present ⟺ incomplete.** Returns a batch iff the located `toolUse`
  assistant has ≥1 call id with no matching recorded `toolResult`.
- **Conservative shape.** The walk-back only accepts the
  `assistant → toolResult* → [aborted assistant?]` shape.
- **`toolCalls` is exactly the assistant's calls, in order.**

**Modeling notes:** Uses a `Set<string>` of result ids (`resultIds`) and
`toolCalls.every(...)` — both in-fragment (cf. `topologicalSort` in the CharmChat
study). `AssistantMessage.content` shadows down to a `seq` of tool-call structs.

---

### 2. Compaction Cut-Point Selector — the pi echo (`compaction.ts`)

Same theorem family pi proved (no orphan at the cut), ported to Flue's
`AgentMessage[]` model — the same guarantee holding over a different
implementation and message shape.

#### `findValidCutPoints` (`compaction.ts:395`)
Returns the indices in `[start, end)` that are safe suffix starts.
- **In range.** Every returned index is in `[start, end)`.
- **No orphan at the cut.** Every returned index points at a `user` or
  `assistant` message — never a `toolResult`. (Flue whitelists two roles;
  pi blacklisted one. Same guarantee, dual encoding.)

Two loop `//@ invariant`s carry both, exactly as in pi's proof.

#### `findTurnStartIndex` (`compaction.ts:406`)
Returns a `user` index `<= index` (down to `start`), or `-1`.
- **In range or −1.** `\result === -1 || (start <= \result <= index && messages[\result].role === 'user')`.

#### `findCutPoint` (`compaction.ts:419`)
Accumulates a token estimate from the end, snaps to the nearest valid cut point,
computes split-turn metadata. Three `//@ ensures` mirroring pi:
- The chosen `firstKeptIndex` is a valid cut point (in range, not `toolResult`),
  or the degenerate `start` fallback when there are no cut points.
- `isSplitTurn ⟹ turnStartIndex` names a real `user` boundary at/before the cut.
- The snap can't slide the cut onto a `toolResult`.

#### `prepareCompaction` (`compaction.ts:508`)
Pure wrapper (the file marks it "Pure function — no I/O"). Verify the derived
slice bounds are well-formed: `boundaryStart <= historyEnd <= firstKeptIndex`,
so `messages.slice(boundaryStart, historyEnd)` and the turn-prefix slice are
in-range and non-overlapping. `estimateTokens`/`estimateContextTokens` stay
`//@ extern` (they read fields outside the shadow), as pi did with
`estimateTokens`.

---

### 3. `isCompleteToolBatch` — Batch-Match Predicate (`conversation-reducer.ts:1030`)

**What it does:** True iff `results` line up 1:1 with `toolCalls`, in order, by
`(id, name)`, with no duplicate call ids.

**Why it's a great candidate** (small, pure, high-leverage — it is the gate that
keeps orphaned tool results out of the projection, #4):
- **Length agreement:** `\result ⟹ toolCalls.length === results.length`.
- **Positional match:** `\result ⟹ forall k. results[k].toolCallId === toolCalls[k].id && results[k].toolName === toolCalls[k].name`.
- **No duplicate call ids** among matched calls (the `seen` set).
- Clean loop with a `Set<string>` invariant and `decreases`.

---

### 4. `pathToContextEntries` — No-Orphan **at the Projection Layer** (`conversation-reducer.ts:696`)

**What it does:** Folds a linear entry path into the model-facing message list.
A tool-use assistant and its results are emitted **only** when
`isCompleteToolBatch` holds; a bare `toolResult` is never emitted
(`if (message.role !== 'toolResult') messages.push(...)`); error/aborted
assistants are dropped unless resumable.

**Why it's the Flue-specific complement to pi:** pi proves the *cut* never
orphans a tool result; Flue can prove the *projection* never *produces* one.
Same safety goal, a different layer of the harness.

**Property to verify (headline):**
- **No orphaned `toolResult` in the output.** In the returned
  `ReducedContextEntry[]`, every `toolResult` message is immediately preceded by
  the `assistant` tool-use message whose call it answers — there is no
  free-standing `toolResult`. Provable because the only push of a `toolResult`
  is inside the `isCompleteToolBatch` branch, right after its assistant.

**Supporting properties:**
- Every emitted assistant with tool calls is followed by *exactly* its complete,
  in-order result batch.
- The output is a subsequence of the input path (nothing invented).

**Modeling notes:** The index-jumping loop (`index = resultIndex`) needs a
`decreases` on `path.length - index` and an invariant that `index` never
retreats. `resolveMessageAttachments` (attachment I/O) is out of scope — model
the message it returns as the entry's own message (attachments don't affect
roles/ordering).

---

## Tier 2 — Good Fit (Some Modeling Needed)

### 5. `deriveCompactionDefaults` — Reserve Clamping (`compaction.ts:51`)

Pure integer arithmetic (`Math.min`/`Math.max`/`Math.floor`). Like CharmChat's
80%-threshold target.
- `input.maxTokens > 0 ⟹ reserveTokens <= input.maxTokens`.
- After the tiny-window clamp: `input.contextWindow > 0 ⟹ reserveTokens < contextWindow` (compaction can actually fire) **and** `reserveTokens >= 1024`.
- `keepRecentTokens` is passed through unchanged.

### 6. `addUsage` / `emptyUsage` / `fromProviderUsage` — Usage Algebra (`usage.ts`)

A commutative monoid over token/cost tuples — a clean algebraic target.
- **Identity:** `addUsage(u, emptyUsage()) === u` and `addUsage(emptyUsage(), u) === u`.
- **Commutativity:** `addUsage(a, b) === addUsage(b, a)`.
- **Associativity:** `addUsage(addUsage(a, b), c) === addUsage(a, addUsage(b, c))`.
- **No mutation:** neither argument is changed (field-wise fresh object).

Model `PromptUsage` as a Dafny datatype of ints (including the nested `cost`).

### 7. `computeFileLists` — Read/Modified Partition (`compaction.ts:217`)

Set logic (`modified = edited ∪ written`, `readOnly = read \ modified`, both
sorted). Analogue of CharmChat's cited/uncited partition.
- `readFiles` and `modifiedFiles` are disjoint.
- `modifiedFiles === sort(edited ∪ written)`.
- Every `readFiles` element is in `read` and not in `modified`.

### 8. `buildConversationContextEntries` — Post-Compaction Slice (`conversation-reducer.ts:660`)

Finds the latest compaction, resolves `keptStart`, and concatenates
[summary] + kept-before-compaction slice + after-compaction slice.
- `keptStart` and `latestCompactionIndex` are in range; the two slices don't
  overlap and don't include the compaction entry twice.
- No-compaction path returns the full projection unchanged.

### 9. `shouldCompact` + `calculateContextTokens` (`compaction.ts:170`, `:76`)

Tiny composable predicates — good for wiring/first-light.
- `contextWindow <= 0 ⟹ !shouldCompact` (unknown window never threshold-fires).
- `!settings.enabled ⟹ !shouldCompact`.
- `calculateContextTokens` prefers `totalTokens` when non-zero, else the
  component sum.

---

## Tier 3 — Stretch Goals / Modeling Exercises

### 10. `applyConversationRecord` — Event-Sourced Reducer Integrity (`conversation-reducer.ts:243`)

The canonical integrity core: applies one `ConversationRecord` to the reduced
state, enforcing append invariants. High value, heavy modeling (Maps, a wide
discriminated record union). Extract sub-targets rather than the whole:
- **Linear append** (`assertEntryAppend`, `:774`): a new entry's `parentId` must
  equal `activeLeafId`; entry ids are unique; no advance while an assistant is
  in progress. Prove these as pre/post-conditions on the `Map` of entries.
- **Block-index uniqueness** (`startBlock`, `:854`): a block index is never
  reused within a message.
- **Delta monotonicity** (`appendDelta`, `:870`): `record.sequence` must equal
  the current `deltas.length`, so deltas apply in a strict 0,1,2,… order.

### 11. `getActiveConversationPath` / `pathToLeaf` — Acyclic Walk (`conversation-reducer.ts:639`, `:831`)

Parent-link walk with a `visited` set that throws on a cycle.
- **Termination / acyclicity:** with a `visited` guard, the walk visits each
  entry at most once — bounded by `entries.size`.
- **Path shape:** consecutive entries are parent-linked; the result is reversed
  root→leaf. Needs graph modeling + `decreases |entries| - |visited|`.

### 12. `parseSessionStorageKey` — Round-Trip (`session-identity.ts`)

Model `JSON.parse`/`JSON.stringify` as spec-level inverses (cf. CharmChat's
`parseToolOutput`).
- **Round-trip:** `parseSessionStorageKey(createSessionStorageKey(i, h, s)) === { i, h, s }`.
- **Total:** malformed input returns `undefined`, never throws (the `try/catch`
  and the 3-string-array shape check).

### 13. `isPublicSessionName` and prefix predicates (`session-identity.ts`)

Boolean algebra over string prefixes (the `UUID_PATTERN` regex is a trust
boundary — keep `isUuid` opaque).
- `isPublicSessionName(n) === !n.startsWith('task:') && !n.startsWith('action:')`.
- `createTaskSessionName(...)` and `createActionScopeName(...)` outputs are never
  public (`!isPublicSessionName`).

---

## Out of Scope (Unsupported Features / No Verifiable Core)

- **Async I/O:** `compact`, `generateSummary`, `generateTurnPrefixSummary`
  (`compaction.ts`) — provider calls; only their pure inputs (`prepareCompaction`)
  are targets.
- **Prompt/string builders:** `result.ts`, the summarization prompt constants.
- **Channel signature verification** (`packages/slack`, `github`, `stripe`, …) —
  HMAC/crypto, a trust boundary, not a pure-logic target.
- **`encodeCanonicalId`** (`conversation-reducer.ts:1054`) — `TextEncoder`/`btoa`.
- **Stores/adapters** (`sql-*.ts`, `*-store.ts`) — persistence I/O.

---

## Suggested Path

| Step | Candidate | Why |
|------|-----------|-----|
| ✅ | `countConsecutiveRetryableModelErrors` (#1b) | **Done** — proven equal to a recursive spec, in place. |
| → | `isCompleteToolBatch` (#3) | Small predicate; unlocks #4. |
| | `findValidCutPoints` (#2) | Direct pi port — reuse the proven pattern. |
| | `addUsage` monoid (#6) | Clean algebra; no message modeling. |
| | `classifySubmissionState` (#1) | Flagship Flue-specific correctness result. |
| | `pathToContextEntries` no-orphan (#4) | Flue's projection-layer complement to pi. |

---

## Shared Type Modeling

Flue's message roles (from pi-ai, shadowed like pi's case study) map to a Dafny
datatype:

```dafny
datatype Role = User | Assistant | ToolResult | Signal
datatype AgentMessage = AgentMessage(role: Role)  // fields beyond role: opaque

// Submission classification
datatype SubmissionState =
    Absent
  | AdvancedPastInput
  | Completed(overflow: bool)
  | ToolUseUnresolved
  | TerminalError(reason: string)
  | Resume(mode: ResumeMode, consecutiveRetryableErrors: nat)

datatype ResumeMode =
    InputOnly | ToolResults | ToolResultsPartial | StreamContinuation
  | TransientRetry | Overflow | AbortedPartial

// Usage algebra (nested cost flattened for the proof)
datatype PromptUsage = PromptUsage(
  input: int, output: int, cacheRead: int, cacheWrite: int, totalTokens: int,
  costInput: int, costOutput: int, costCacheRead: int, costCacheWrite: int, costTotal: int)
```

`isContextOverflow`, `estimateTokens`, and the retryable-error regex become
`//@ extern` / spec-level functions — present but uninterpreted, so the safety
proofs depend only on message *roles* and *structure*, never on token estimates
or error-string contents.
