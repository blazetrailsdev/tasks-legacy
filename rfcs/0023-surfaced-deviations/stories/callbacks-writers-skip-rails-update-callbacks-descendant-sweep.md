---
title: "set_callback/reset_callbacks bypass __update_callbacks' descendant sweep"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

`ActiveSupport::Callbacks::ClassMethods#set_callback`, `#skip_callback` and
`#reset_callbacks` all route their chain write through `__update_callbacks`
(activesupport/lib/active_support/callbacks.rb:697-704 and its call sites at
:745-748, :790-807, :811-821), which walks `[self, *self.descendants]` and
writes the rebuilt chain back to every one of them. That is how a superclass
registering a callback after a subclass has already touched its own chain still
reaches the subclass.

trails' `Callbacks` namespace (`packages/activesupport/src/callbacks.ts`) has
no descendant list and substitutes copy-on-write: `getCallbackChains(target)`
clones the nearest ancestor's chains onto `target` on first write, and
`peekCallbackChain` reads without cloning. Reachability therefore depends on
_not_ having materialised an own chain yet, which is a different invariant from
Rails'.

PR #6951 hit the seam twice and patched it locally rather than converging it:

- `runCallbacks` now peeks (a read must not materialise), callbacks.ts:1560-1580.
- `skipCallback` peeks and only materialises once a filter actually matches
  (callbacks.ts:1489-1545), so a skip that removed nothing does not sever the
  subclass. Both carry a JSDoc naming `__update_callbacks` as the thing they
  are standing in for.

`setCallback` and `resetCallbacks` still have the gap unaddressed: a subclass
that owns a chain snapshot never sees a callback its superclass registers
afterwards, and `reset_callbacks`' documented descendant sweep
(callbacks.rb:811-821 deletes the parent's callbacks from every descendant's
chain) is not implemented at all — trails' `resetCallbacks` clears only the one
chain it finds.

## Converged shape

Port `__update_callbacks` with the Rails name, and give the callback host the
descendant list it needs (ActiveSupport::DescendantsTracker is the Rails
source; trails already has `descendants` on model classes and uses it in
`clearAttributeNamesMemo`). Every one of `set_callback` / `skip_callback` /
`reset_callbacks` then writes through it, and copy-on-write stops being load-bearing
for reachability — at which point the two peek workarounds above can go back to
the plain `get_callbacks` read Rails does.

## Acceptance criteria

- [ ] `__updateCallbacks` (callbacks.ts:1313 — already present, currently
      unused by the three writers) is what `setCallback`, `skipCallback` and
      `resetCallbacks` write through, walking self + descendants as Rails does.
- [ ] `resetCallbacks` deletes the reset class's callbacks from every
      descendant's chain (callbacks.rb:811-821), with a test.
- [ ] A superclass `set_callback` after a subclass has materialised its own
      chain reaches the subclass, with a test that fails on the baseline.
- [ ] The `skipCallback` lazy-materialisation and `runCallbacks` peek
      workarounds from #6951 are removed or restated as plain Rails reads.
- [ ] Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
