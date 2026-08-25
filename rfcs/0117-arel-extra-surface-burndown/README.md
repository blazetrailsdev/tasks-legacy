---
rfc: "0117-arel-extra-surface-burndown"
title: "arel extra TS surface burndown to zero"
status: closed
created: 2026-08-22
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "arel"
clusters: []
priority: 1
---

## Summary

`packages/arel` is the only package in the data layer whose
`parity:api:extra` population is **growing**. Measured from CI's
`Rails API/Test Comparison` logs across 440 merged PRs, 2026-08-05 →
2026-08-22:

| package      | novel            | moved             | total              | `@noRailsEquivalent` |
| ------------ | ---------------- | ----------------- | ------------------ | -------------------- |
| arel         | 86 → 89 (**+3**) | 164 → 169 (+5)    | **250 → 258 (+8)** | 1 → 2                |
| activemodel  | 138 → 95 (−43)   | 202 → 180 (−22)   | 340 → 275 (−65)    | 2 → 14               |
| activerecord | 581 → 533 (−48)  | 1417 → 1462 (+45) | 1998 → 1995 (−3)   | 86 → 117             |

arel's total did not decrease on a single day in that window. activemodel had
an active campaign (RFC 0115); arel has had no owner and no RFC.

arel is also the package where **finishing is realistic**. Fresh run
2026-08-22 (`pnpm build && pnpm parity:api && pnpm parity:api:extra --package arel --json`):

- **69 files with drift, 89 novel, 169 moved, 258 total**, 2 `@noRailsEquivalent`,
  16 interface-exempt, and **12 files with no Rails counterpart at all** carrying
  84 extras (23 of them novel).

Against activerecord's 1995, arel is the one package that can reach **zero**.
This RFC is the burndown, and its terminal state is a ratchet that keeps it
there.

## The triage model

Every extra name resolves exactly one of four ways. Stories bucket their names
under these headings by name, not by count:

1. **Delete it.** trails-only surface nothing needs, or a duplicate of a
   Rails-named member. The default and the preferred outcome.
2. **Rename / relocate to the Rails name.** A `moved` name already exists in
   Rails, just in a different `.rb` — usually our _file layout_ diverges, not
   the name. Check `docs/ruby-ts-conventions.md` (`PATH_SEGMENT_ALIASES`,
   `RUBY_FILE_TS_OVERRIDES`) before concluding a name is genuinely misplaced.
3. **Fold into the ported method.** A helper Rails inlines. CLAUDE.md's
   "No extra abstraction": if Rails inlines it, inline it.
4. **Tag `@noRailsEquivalent <reason>`.** LAST resort, only for a genuine
   TypeScript language shortcoming, and only after the settled workaround in
   CLAUDE.md has actually been tried. The tag is "a receipt, not absolution".

### The tag budget is a hard number

arel carries **2** `@noRailsEquivalent` tags today. A burndown that ends with
50 is a failed burndown — it has relabelled the debt, not paid it.

**Budget: arel may finish this RFC with at most 8 tags** (2 existing + 6 new
across every story here). A story that wants the 7th and 8th must say so in its
PR body and name what language shortcoming it hit. A story that would need a
9th is **blocked**, not tagged: file the blocker and stop. Two tags are already
pre-allocated by the analysis below (`visitors/ruby-class.ts`'s Ruby-class
naming shim and `visitors/visitor.ts`'s `VisitorCtor`, which already holds one),
leaving roughly four in reserve for the whole package.

## What the fresh run actually found

Three structural facts reshape the problem. Two of them credit large slices in
one change, which is why they are ordered first.

### 1. 36 novel names are Rails methods the convention drops (~40 extras)

`multiply`, `divide`, `subtract`, `bitwiseAnd`, `bitwiseOr`, `bitwiseXor`,
`bitwiseNot`, `bitwiseShiftLeft`, `bitwiseShiftRight` appear as **novel** in
four files — `math.ts`, `attributes/attribute.ts`, `nodes/node-expression.ts`,
`nodes/infix-operation.ts` — 9 names × 4 files.

They are not invented surface. They are `Arel::Math`'s operator methods
(`vendor/rails/activerecord/lib/arel/math.rb:5-40`: `*`, `+`, `-`, `/`, `&`,
`|`, `^`, `<<`, `>>`, `~@`). `scripts/parity/conventions.ts:1478` returns
`null` for any name in `OPERATORS` (`conventions.ts:346-368`), so the Ruby side
never produces an expected TS name and every TS spelling of one lands in the
extra population.

The precedent for the fix is in the same function: `-@` already maps to
`negate` (`conventions.ts:1486`). The remedy is a **file-scoped** operator
spelling table — global is wrong, because `<<` means _append_ on
`SelectManager` and _bitwise shift_ in `Arel::Math`.

This is a `scripts/parity/conventions.ts` change with **zero trails source
change**, and it is ordered first because it deletes ~40 rows of other
stories' work before that work starts.

### 2. Per-node `accept` is invented double dispatch (~36 extras)

Rails' Arel nodes define **no** `accept`. `grep -rn "def accept"` over
`vendor/rails/activerecord/lib/arel/` returns exactly two hits —
`visitors/visitor.rb:10` and `visitors/dot.rb:28`. Dispatch is
`Visitor#visit` → `dispatch[object.class]`.

trails already ports that faithfully: `visitors/visitor.ts:102-121` dispatches
through the constructor-keyed `dispatchCache` and **never consults
`node.accept`**. On top of it, 36 node classes each carry

```ts
accept<T>(visitor: NodeVisitor<T>): T {
  return visitor.visit(this);
}
```

— in several files (`nodes/false.ts`, `nodes/true.ts`, `nodes/terminal.ts`)
that method is the _entire class body_, where Rails has `class False < Node; end`.
It is a second dispatch mechanism parallel to the faithful one.

**Disposition: delete it** (owner decision, 2026-08-22), together with the
`NodeVisitor` type in `nodes/node.ts:8`, rewiring the handful of real call
sites onto `visitor.accept(node, collector)`. Split across two stories because
the deletion spans 36 files.

### 3. 84 extras live in 12 files with no Rails counterpart

`predications-range.ts` alone carries **41** (8 novel, 33 moved) — a trails
extraction of `predications.rb`'s private helpers (`infinity?`,
`unboundable?`, `open_ended?`) plus the `between` / `not_between` decision
tree, in a file Rails does not have. Folding it back into `predications.ts`
retires 16% of the package's total in one story.

The other eleven split into barrels (`index.ts`, `nodes/index.ts`,
`visitors/index.ts`, `arel.ts` — whose extras are re-exports that die with
their sources, so no separate work), a mis-located pair (`nodes/and.ts`,
`nodes/or.ts`: Rails defines both in `nodes/nary.rb:36-37`, so this is a
relocation, category 2), and genuine trails inventions
(`table-ref.ts`, `visitors/ruby-class.ts`, `visitors/connection.ts`,
`visitors/substitute-bound-values.ts`,
`collectors/substitute-bind-collector.ts`).

**No blanket disposition is set for the inventions** (owner decision,
2026-08-22): each is a per-file judgement made in its own story, which must
propose delete / fold / relocate / tag with the Rails citation and get it
reviewed. Only `visitors/ruby-class.ts` is expected to survive as a tag.

## Non-goals

- Porting new arel functionality. This RFC only removes or relocates surface.
- Widening any baseline or allowlist. `parity:api:extra` has no baseline and
  is not getting one; see CLAUDE.md, "A documented deviation is debt, not
  permission".
- Renaming a TS member to something that is not the name
  `docs/ruby-ts-conventions.md` produces from the Ruby name.

## Ordering

1. `arel-operator-spellings-in-conventions` — convention fix, credits ~40 extras.
2. `arel-node-accept-removal-*` — the largest single shape, 36 files.
3. `arel-predications-range-fold` — the largest single file, 41 extras.
4. Per-file stories, largest novel count first.
5. `arel-extra-surface-ratchet` — an only-shrink gate so arel cannot regress
   again, landed once the count is low enough to be worth pinning.

## Acceptance criteria (RFC level)

- `pnpm parity:api:extra --package arel` reports **0 novel**, and total extras
  reduced to the barrel/interface-exempt residue.
- arel's `@noRailsEquivalent` count is **≤ 8**.
- A ratchet exists that fails CI on any increase in arel's extra count.
