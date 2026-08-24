---
title: "extra-surface-allow-reopened-module-method-files"
status: claimed
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-24T12:03:42Z"
assignee: "extra-surface-allow-reopened-module-method-files"
blocked-by: null
closed-reason: null
---

## Context

`parity:api:extra` scores a faithfully-placed port as `moved` when its Ruby
method lives in a **reopened** module whose primary declaration site is a
different `.rb`.

`extra-surface.ts:1515-1524` builds `rubyFiles` by each entity's _primary_
`info.file`. `ActiveModel::Validations` and `ActiveModel::Validations::ClassMethods`
are both first declared in `validations.rb`, so the allow-set for
`validations/with.rb` contains only `WithValidator` — even though
`ActiveModel::Validations#validates_with` and
`ActiveModel::Validations::ClassMethods#validates_with` both carry
`file: "validations/with.rb"` on their `MethodInfo`.

Consequence, measured on the RFC 0115 fan-out of `validates_with` (PR for
`fan-out-model-validates-with-to-validations-with`): moving both arms from
`model.ts` into `validations/with.ts` — exactly where Rails puts them — takes
`validations/with.ts` from `0 novel, 1 moved` to `0 novel, 3 moved`
(`validatesWith`, credited by `--verbose` to
`activemodel validations/with.rb ActiveModel::Validations#validates_with`, i.e.
the same file, plus the `ClassMethods` const). The tool penalises the correct
placement, which is backwards pressure on every remaining fan-out story.

`collectAllowedNames` already takes a `methodFile` and filters
`m.file !== methodFile` (`extra-surface.ts:1188`), so the fix is on the
registration side: also register an entity under each distinct file its
methods declare, method-file-filtered — never widening the allow-set beyond
the methods that Ruby file actually declares.

## Acceptance criteria

- A method whose Ruby `MethodInfo.file` is the counterpart `.rb` of a TS file
  is not scored `extra` in that TS file, even when its owning module's primary
  declaration site is another `.rb`.
- `validations/with.ts` (activemodel) returns to `0 novel, 1 moved` (the
  pre-existing `checkValidityBang` row) after the change.
- The `arel` extra-surface gate stays green; if the change lowers arel's
  numbers, tighten with `pnpm parity:api:extra:tighten` (never raise a mark).
- Unit tests for `extra-surface.ts` cover the reopened-module case.
