---
title: "Unify the eight divergent Ruby class-name message helpers onto one ActiveSupport pair"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

MRI builds the class name in an error message with `rb_builtin_class_name`,
which prints `nil` / `true` / `false` for the special constants and the class
name otherwise — and distinguishes `Integer` from `Float`, which JS's single
`number` type does not.

trails re-derives that spelling in a private helper per file, and the copies do
not agree. At least eight exist, under three different names and two different
answers for `nil`:

- `packages/activemodel/src/naming.ts` — `builtinClassName` (added by #6801:
  `nil`/`true`/`false`, `Integer`/`Float`)
- `packages/activerecord/src/connection-adapters/abstract/database-statements.ts:1153`
  — `rubyClassName` (`nil`, no Integer/Float split)
- `packages/activerecord/src/associations/preloader/branch.ts:359` — `rubyClassName`
- `packages/activerecord/src/relation/query-methods.ts:1471` — `rubyClassNameOf`
  (`NilClass`)
- `packages/activemodel/src/attribute-assignment.ts` — `classOf` (`NilClass`)
- `packages/activemodel/src/validations/comparison.ts:18` (`NilClass`)
- `packages/activesupport/src/transliterate.ts:11`,
  `packages/activesupport/src/cache/store.ts:27`
- `packages/actionpack/src/action-dispatch/http/param-builder.ts:178`,
  `.../request/session.ts:113`, `packages/activerecord/src/nested-attributes.ts:756`

The `nil` vs `NilClass` split is not arbitrary — Ruby genuinely prints both,
depending on whether the message came from `rb_builtin_class_name` (`nil`) or
from `value.class` (`NilClass`) — but nothing in the tree records which a given
site needs, so each new port picks one by copying its nearest neighbour. Message
strings are observable API and `parity:test:assertions` matches on them.

## Converged shape

One helper per Ruby primitive, in ActiveSupport, mirroring the two Ruby
operations rather than merging them: the `rb_builtin_class_name` spelling
(`nil`/`true`/`false`) and the `Object#class`-name spelling (`NilClass`,
`TrueClass`, `FalseClass`). Both split `Integer`/`Float` the way MRI does. Each
existing call site is then re-pointed at whichever one its Rails message
actually uses, checked against the Ruby that raises it (`ruby` is on PATH — run
the raise and read the message rather than deriving it from the C).

Note the helpers are module-private today, so `parity:api:extra` does not score
them; the debt is invisible to the gates and only shows up as a wrong message in
a test.

## Acceptance criteria

- The per-file `rubyClassName` / `rubyClassNameOf` / `classOf` /
  `builtinClassName` copies are deleted in favour of the ActiveSupport pair.
- Every re-pointed site's message is verified against MRI output.
- `pnpm parity:test:assertions` delta non-negative.
