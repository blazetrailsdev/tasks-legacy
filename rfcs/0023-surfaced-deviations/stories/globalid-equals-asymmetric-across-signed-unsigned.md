---
title: "GlobalID#equals is not symmetric across signed/unsigned pairs"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "globalid"
deps: []
deps-rfc: []
est-loc: 50
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`SignedGlobalID < GlobalID` in Rails, and `GlobalID#==` compares class-agnostically
by URI (`vendor/globalid/lib/global_id/global_id.rb:88-90`):

```ruby
def ==(other)
  other.is_a?(GlobalID) && @uri == other.uri
end
```

So a signed and an unsigned GID over the same record are `==`. Rails asserts
exactly that in `signed_global_id_test.rb:25-27`:

```ruby
test 'value equality with an unsigned id' do
  assert_equal GlobalID.create(Person.new(5)), SignedGlobalID.create(Person.new(5))
end
```

That test is registered as unported in
`scripts/parity/unported-files/unscoped.ts:733-742`, with the reason that
"cross-class equality across `@blazetrails/globalid` subpath imports hits the
TS private-field nominal-typing trap; the `SignedGlobalID#equals` contract is
symmetric only within its own class."

That register row is a burndown ledger entry, not permission. The behavioral
gap is real: trails' `GlobalID#equals` (`packages/globalid/src/global-id.ts:171-173`)
does test `other instanceof GlobalID`, which a `SignedGlobalID` satisfies, so
the asymmetry is worth re-deriving rather than inheriting — the recorded reason
may predate the current class layout, and the surrounding tests were re-ported
in PR #6651 without revisiting it.

## Converged shape

`GlobalID#equals` should be `other instanceof GlobalID && this.uri === other.uri`
and hold in **both** directions for a GlobalID/SignedGlobalID pair.
`SignedGlobalID#equals` (`signed-global-id.ts:221`) additionally compares
`purpose`, mirroring Rails' override — confirm against the Ruby which of the
two runs for `assert_equal expected, actual` (minitest calls `expected ==
actual`, so the _left_ operand's `==` wins; the Rails test puts the unsigned
GlobalID on the left).

If a genuine TS private-field nominal-typing problem remains after that, the
`#uri` access should route through the public reader rather than a private
field so the cross-subpath case works — and if it truly cannot, block the story
with the specific failing construction rather than restating the register row.

## Acceptance criteria

- `signed_global_id_test.rb › value equality with an unsigned id` is ported and
  passing, with the Rails operand order preserved.
- Its row is removed from `scripts/parity/unported-files/unscoped.ts`.
- `pnpm parity:test --package globalid` does not drop from 131/131.
