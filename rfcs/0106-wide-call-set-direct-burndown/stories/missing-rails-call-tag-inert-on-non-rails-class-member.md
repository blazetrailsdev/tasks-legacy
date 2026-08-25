---
title: "parity:api:build should report a @missingRailsCall tag on a non-Rails class member as INERT"
status: in-progress
updated: 2026-08-25
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 140
priority: 1
pr: 7026
claim: "2026-08-25T09:46:54Z"
assignee: "missing-rails-call-tag-inert-on-non-rails-class-member"
blocked-by: null
closed-reason: null
---

## Context

`parity:api:build --package activerecord --dry-run` reports, on the committed tree:

```text
preserved @missingRailsCall on associations/disable-joins-association-scope.ts
_addConstraintsDj for `add_constraints` — no expectation for that declaration in
the artifact; the tag is left exactly as written.
```

`_addConstraintsDj` is a port-invented private name (the `Dj`-suffixed spelling
of Rails' `DisableJoinsAssociationScope#add_constraints`,
`vendor/rails/activerecord/lib/active_record/associations/disable_joins_association_scope.rb:33`),
so the compare artifact carries no expectation for that declaration and the tag
on it suppresses nothing. It is documentation sitting in a deviation register
that nothing measures and nothing will ever retire — the same failure class as
the already-converged `missing-rails-call-tag-inert-on-top-level-function` and
`top-level-function-missing-rails-call-tag-does-not-suppress`, but for a CLASS
MEMBER with no Rails counterpart rather than a top-level function.

Surfaced by `route-residual-call-set-rows-to-a-live-owner` (PR #7007), which
moved the load-bearing receipt onto `scope` — where the flag is actually raised
(`disable_joins_association_scope.rb:14`) — and left this one in place as
pre-existing prose rather than churning a reviewed decision.

The converged shape: a `@missingRailsCall` tag is load-bearing or it is not
there. `parity:api:build` already knows which tags matched nothing; it should
report them as a distinct **INERT** class (separate from STALE, which means "it
matched and no longer needs to") so the tree can be swept, and the tag on
`_addConstraintsDj` deleted — its content is already carried, verbatim and
correctly sited, by the tag on `scope`.

## Acceptance criteria

- [ ] `parity:api:build` reports a tag on a declaration with no artifact
      expectation as INERT, distinctly from STALE, with the `file:line` of the tag.
- [ ] The inert tag on `disable-joins-association-scope.ts` `_addConstraintsDj`
      is deleted; the live receipts on `scope` and `lastScopeChain` are untouched.
- [ ] Any other inert tag the new report surfaces is deleted or re-sited onto the
      declaration whose flag it is meant to suppress — not re-justified in place.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green; no
      `--write`, no reseed, no new rows.
