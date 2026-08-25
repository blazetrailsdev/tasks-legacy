---
title: "Converge the four divergent public_send ports onto one"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Object#public_send` is ported four times in the tree, with four different
semantics for the same Ruby method:

- `packages/activemodel/src/access.ts` (`publicSend`, file-local) — raises
  `NoMethodError` when the member is absent, reads an accessor property,
  invokes a function member. Backs `slice` / `values_at` (access.rb:8, :12).
- `packages/activemodel/src/conversion.ts` (`publicSend`, file-local) — a
  duplicate of the above, backing `to_key`'s `respond_to?(:id) && id`
  (conversion.rb:67-70).
- `packages/activerecord/src/attribute-methods/query.ts:29` (`publicSend`,
  file-local) — walks own descriptors and then the prototype chain by hand,
  and does NOT raise for an absent member (falls through to `obj[name]`, i.e.
  `undefined`). Backs `query_attribute`
  (`activerecord/lib/active_record/attribute_methods/query.rb:11-14`).
- `packages/activemodel/src/secure-password.ts:122` (`publicSend`, file-local)
  — a bare property read that never invokes a function member and never
  raises. Backs the `has_secure_password` validations' `record.public_send(...)`
  (secure_password.rb:104-119).

Ruby has one `public_send`. Three of these four disagree with it in at least one
arm — a missing member answers `undefined` instead of raising `NoMethodError`
in two of them, and a `def`-style member is never invoked in one — so the same
Rails line ports to different behaviour depending on which file it landed in.
`serialization.ts:189-231` is a fifth, deliberate variant (`send`, not
`public_send`, with a documented store-attribute arm) and is NOT in scope.

## Converged shape

One ported `public_send` all four call sites share, with Ruby's semantics: a
member the receiver does not respond to raises `NoMethodError` with `send`'s
message; an accessor property is read (a generated attribute reader ports as a
property, CLAUDE.md § "Generated attribute readers are properties"); a function
member is invoked. `query.ts`'s hand-rolled prototype walk collapses into it —
`name in obj` already answers the same question `respond_to?` does.

Where the shared helper lives is part of the story: activesupport has no
`object/public-send.ts` today, and Ruby's `public_send` is `Object`'s, not
Active Support's, so a `@noRailsEquivalent PERMANENT` carrier may be the honest
home.

## Acceptance criteria

- One `public_send` port; `access.ts`, `conversion.ts`,
  `attribute-methods/query.ts` and `secure-password.ts` all call it, and none
  keeps a file-local copy.
- Its missing-member arm raises `NoMethodError`; its function-member arm
  invokes. Pinned by a test per arm.
- `pnpm vitest run packages/activemodel/src packages/activerecord/src/attribute-methods.test.ts packages/activerecord/src/attribute-methods.trails.test.ts`
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.

## Definition of done

Not done while two files in the tree spell `public_send` differently.
