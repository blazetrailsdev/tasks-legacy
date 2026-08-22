---
title: "stringify_keys/symbolize_keys route through a ported transform_keys"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 150
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6167, which bucketed `core_ext/hash/keys.rb` on `hash-utils.ts`
and measured its bodies for the first time. Two call-mismatch rows were
baselined rather than converged
(`scripts/api-compare/call-mismatches-exclude/activesupport/hash-utils.json`).

Rails routes every key cast through one primitive:

```ruby
# vendor/rails/activesupport/lib/active_support/core_ext/hash/keys.rb:10-27
def stringify_keys
  transform_keys { |k| Symbol === k ? k.name : k.to_s }
end
def stringify_keys!
  transform_keys! { |k| Symbol === k ? k.name : k.to_s }
end
def symbolize_keys
  transform_keys { |k| k.to_sym rescue k }
end
```

`deep_transform_keys` / `deep_transform_keys!` (:39-53) and their
`_deep_transform_keys_in_object` helpers (:63-76) sit on the same primitive.
trails' `stringifyKeys` / `symbolizeKeys`
(`packages/activesupport/src/hash-utils.ts:144-150`) each inline their own
`Object.keys` loop instead, so the shared primitive has no TS counterpart and
the two bodies diverge from Rails independently.

## Converged shape

- Add `transformKeys` (and the bang arm, plus
  `_deepTransformKeysInObject`) to `hash-utils.ts` with the Rails names, taking
  the plain object as the receiver argument the rest of that file already takes.
- `stringifyKeys` / `symbolizeKeys` / `deepTransformKeys` become one-line
  delegations to it, matching Rails' decomposition method-for-method.
- Delete the two rows from
  `call-mismatches-exclude/activesupport/hash-utils.json` (only-shrink; no
  `--write` reseed).

## Acceptance criteria

- `stringifyKeys` and `symbolizeKeys` call `transformKeys`; neither carries its
  own key loop.
- `hash-utils.json`'s two rows are deleted and `pnpm parity:api:calls` is green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
