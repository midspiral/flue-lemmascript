# Flue — Verified with LemmaScript

[![LemmaScript verified](https://img.shields.io/github/actions/workflow/status/midspiral/flue-lemmascript/lemmascript.yml?branch=lemmascript&label=LemmaScript%20verified)](https://github.com/midspiral/flue-lemmascript/actions/workflows/lemmascript.yml)

Fork of **Flue** ([`withastro/flue`](https://github.com/withastro/flue)), Astro's
agent-harness framework, applying [LemmaScript](https://github.com/midspiral/LemmaScript)
to the pure logic in its crash-recovery harness. Verified with the **Dafny**
backend, from the same `//@`-annotated TypeScript. Annotations are added
**in-place**: the verified function bodies are **byte-identical** to the shipped
ones — everything else goes through `//@` comments (type shadows for the
cross-package message types, an `//@ extern` for the retryable-error regex, and
the `//@ verify` filter that isolates the target so the surrounding out-of-subset
functions are ignored).

Flue depends on pi's lower layers for message types and provider completion but
builds its own session/conversation harness on top; this case study targets that
harness.

## Coverage

| File | Function | Property | Dafny |
|---|---|---|---|
| [`submission-state.ts`](packages/runtime/src/submission-state.ts) | `countConsecutiveRetryableModelErrors` | functional: equals a recursive spec of the backward scan (+ bounded output, termination) | ✓ |
| [`conversation-reducer.ts`](packages/runtime/src/conversation-reducer.ts) | `isCompleteToolBatch` | length agreement + positional `(id, name)` match between tool calls and results | ✓ |
| [`usage.ts`](packages/runtime/src/usage.ts) | `addUsage` / `emptyUsage` | commutative monoid — left/right identity, commutativity, associativity | ✓ |
| [`compaction.ts`](packages/runtime/src/compaction.ts) | `findValidCutPoints` | every returned cut index is in range and never a `toolResult` (no orphan at the cut) | ✓ |

## What's Verified

### `countConsecutiveRetryableModelErrors` — [`submission-state.ts`](packages/runtime/src/submission-state.ts)

On crash recovery, Flue counts the trailing retryable model errors of an
interrupted submission to decide how long to back off. Scanning the persisted
entries from the end, it counts consecutive **retryable assistant errors**,
transparently skipping `compaction` entries and non-assistant messages, and
stopping at the first **user** message (an operation boundary) or the first
**non-retryable** assistant.

We prove the shipped function computes **exactly** that, against a recursive
spec-level mirror `countRetryableSuffix` (itself `//@ verify`-checked for
termination):

```
//@ ensures \result === countRetryableSuffix(entries, entries.length)
```

The proof rests on one loop invariant relating the running `count` to the spec
over the unprocessed prefix,

```
count + countRetryableSuffix(entries, i + 1) === countRetryableSuffix(entries, entries.length)
```

which Dafny discharges automatically — the invariant unfolds `countRetryableSuffix`
at `entries[i]` and matches the imperative body case-for-case. Two supporting
`//@ ensures` also carry the sanity bound `0 <= \result <= entries.length`.

The generated Dafny composes four things the shipped source stacks on one line —
`noUncheckedIndexedAccess` optional indexing (`entries[i]` bounds-guarded to
`Some/None`), optional narrowing past the `entry?.type` guard, the
`CanonicalSubmissionEntry` discriminated-union match, and a mid-loop `continue` —
each of which required LemmaScript toolchain support to express in place.

### `isCompleteToolBatch` — [`conversation-reducer.ts`](packages/runtime/src/conversation-reducer.ts)

Gates whether a persisted tool-use turn's results form a complete batch — the
predicate that keeps orphaned tool results out of the model-facing projection.
We prove that when it returns `true`, the lists agree in length and each result
matches its call positionally by `(id, name)`:

```
//@ ensures \result ==> toolCalls.length === results.length
//@ ensures \result ==> forall k. results[k].toolCallId === toolCalls[k].id
                                && results[k].toolName === toolCalls[k].name
```

Body byte-identical; the tool-call/result element types are `//@ declare-type`
shadows and the inline `Extract<…>` parameter type is redirected with a
`//@ type` override (no signature change). It drove one toolchain addition —
truthiness of a non-optional object (`!obj → false`), which proves the shipped
`!call || !result` bounds-guards are dead under the length invariant.

### `addUsage` / `emptyUsage` — [`usage.ts`](packages/runtime/src/usage.ts)

The token-and-cost aggregator is a **commutative monoid**. Over the shipped
functions (bodies byte-identical), we prove `emptyUsage()` is a left and right
identity for `addUsage`, and that `addUsage` is commutative and associative —
each as a spec-only lemma, discharged automatically from the field-wise integer
sums. No-mutation is inherent: `//@ pure` functions can't mutate their arguments.

### `findValidCutPoints` — [`compaction.ts`](packages/runtime/src/compaction.ts)

The context-compaction cut-point selector — the same "never orphan a `toolResult`
at the cut" property pi proved, over Flue's `AgentMessage[]` model (a `user`/
`assistant` role whitelist rather than pi's role fall-through). We prove every
returned index is in `[start, end)` and points at a `user` or `assistant`
message — never a `toolResult` — so a retained suffix can't *begin* with an
orphaned tool result. Two loop invariants carry both, exactly as in pi's proof;
the `messages[i]?.role` optional-index access needed no new toolchain support.

## Running the verification

The generated `.dfy`/`.dfy.gen` artifacts are committed and CI fails if they are
stale. Needs Node ≥ 18, [Dafny](https://github.com/dafny-lang/dafny) ≥ 4.x, and a
LemmaScript clone as a sibling directory:

```sh
git clone https://github.com/midspiral/LemmaScript.git ../LemmaScript
(cd ../LemmaScript/tools && npm ci)
../LemmaScript/tools/check.sh dafny   # batches over LemmaScript-files.txt
```

## Trust boundary

The proof holds relative to assumptions made explicit in the annotations:

- **`AgentMessage` / `AssistantMessage`** are shadowed (`//@ declare-type`) as
  `{ role: string }` — the proof depends only on message *roles*, not on any
  other field. `AssistantMessage` is an alias of `AgentMessage`, so the shipped
  `as AssistantMessage` cast is identity in the model.
- **`isRetryableModelError`** is `//@ extern` — its regex over the error string
  is a trust boundary, uninterpreted in the proof. The functional result holds
  for *any* retryable predicate; the theorem is about the scan's control flow,
  not which errors are retryable.
- **`CanonicalSubmissionEntry`** is modeled as its shipped discriminated union
  (`message | compaction`); the other, out-of-subset functions in the file are
  excluded from the model by the `//@ verify` filter and are not part of the
  claim.
