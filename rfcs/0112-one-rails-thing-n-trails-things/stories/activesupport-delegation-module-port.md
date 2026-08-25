---
title: "Port ActiveSupport::Delegation so Module#delegate fronts it"
status: done
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 350
pr: 6965
claim: "2026-08-24T02:13:27Z"
assignee: "activesupport-delegation-module-port"
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6167, which bucketed `core_ext/module/delegation.rb` and
`core_ext/module/attribute_accessors.rb` on `module-ext.ts` and so measured
their bodies for the first time. Three call-mismatch rows were baselined rather
than converged (`scripts/api-compare/call-mismatches-exclude/activesupport/module-ext.json`);
this story converges them.

Rails' `Module#delegate` is a thin front for a module trails has not ported:

```ruby
# vendor/rails/activesupport/lib/active_support/core_ext/module/delegation.rb:160-170
def delegate(*methods, to: nil, prefix: nil, allow_nil: nil, private: nil)
  ::ActiveSupport::Delegation.generate(
    self, methods, location: caller_locations(1, 1).first,
    to: to, prefix: prefix, allow_nil: allow_nil, private: private,
  )
end
```

`delegate_missing_to` (:218-223) likewise fronts
`ActiveSupport::Delegation.generate_method_missing`, and `mattr_reader` /
`mattr_writer` thread the same `location:` (`caller_locations(1, 1).first`,
`core_ext/module/attribute_accessors.rb:57`) so a raised `NameError` points at
the declaring line.

trails' `delegate` (`packages/activesupport/src/module-ext.ts:16`) reimplements
the method generation inline and plumbs no location, so the delegation module
has no TS home at all. `ActiveSupport::Delegation` itself is
`vendor/rails/activesupport/lib/active_support/delegation.rb`.

## Converged shape

- Port `ActiveSupport::Delegation` to `packages/activesupport/src/delegation.ts`
  (the path `rubyFileToTs` already expects), with `generate` and
  `generate_method_missing` carrying the Rails names and parameter order.
- `delegate`, `delegateMissingTo`, `mattrReader` and `mattrWriter` become the
  thin fronts Rails has, passing the call-site location through.
- Delete the three rows from
  `call-mismatches-exclude/activesupport/module-ext.json` (only-shrink; do not
  reseed with `--write`).

## Acceptance criteria

- `delegate` calls a ported `Delegation.generate` rather than generating inline.
- The `location:` argument is threaded from the call site in `delegate`,
  `mattrReader` and `mattrWriter`.
- The three `module-ext.json` baseline rows are deleted, `pnpm parity:api:calls` green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
