---
title: "Port Ruby's reject / delete_if / merge! so compact_blank and deep_merge! call what Rails calls"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6556 ported `Hash#compact_blank`, `Hash#compact_blank!`, `Array#compact_blank!`
and `DeepMergeable#deep_merge!`, and shipped four call-mismatch baseline rows
because each Rails body bottoms out in a Ruby Enumerable primitive trails has no
counterpart for:

| Baseline row                                                      | Rails body                                                                       |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `activesupport hash-utils.ts compact_blank reject`                | `reject { \|_k, v\| v.blank? }` — `core_ext/enumerable.rb:222-224`               |
| `activesupport hash-utils.ts compact_blank! delete_if`            | `delete_if { \|_k, v\| v.blank? }` — `core_ext/enumerable.rb:232-235`            |
| `activesupport core-ext/array/access.ts compact_blank! delete_if` | `delete_if(&:blank?)` — `core_ext/enumerable.rb:263-266`                         |
| `activesupport hash-utils.ts deep_merge! merge!`                  | `merge!(other) { \|key, this_val, other_val\| ... }` — `deep_mergeable.rb:34-44` |

Each TS body open-codes the primitive's effect inline (an `Object.keys` loop, a
reverse `splice`, a per-key conflict loop). That is correct behaviour but it is
four bodies that do not call what Rails calls, and the shape will keep recurring
— `core_ext/enumerable.rb` reaches for `reject`/`delete_if` throughout.

Note Rails picks `delete_if` over `reject!` deliberately (its own comment at
`enumerable.rb:233`, :264): `delete_if` always returns self even when nothing
changed. Any ported primitive has to keep that distinction.

## Converged shape

Port the Ruby Enumerable primitives themselves into
`packages/activesupport/src/enumerable-utils.ts` at their Rails names — `reject`,
`deleteIf`, and a `merge!` with Ruby's conflict-block arity — then rewrite the
four bodies above to call them, so each reads as its Ruby does. Delete the four
rows from `call-mismatches-exclude/activesupport/hash-utils.json` and
`call-mismatches-exclude/activesupport/core-ext/array/access.json` by hand
(only-shrink; do not reseed) and run
`pnpm parity:api:calls:tighten` on the affected shards.

Check `enumerable-utils.ts` first — it already carries several Enumerable
members, so some of these may only need widening rather than adding.

## Acceptance criteria

- [ ] `compact_blank`, `compact_blank!` (Hash and Array) and `deep_merge!` call
      the primitive Rails calls.
- [ ] `delete_if`'s always-returns-self contract is preserved and asserted.
- [ ] The four baseline rows are deleted by hand; `pnpm parity:api:calls` and
      `:args` green.
