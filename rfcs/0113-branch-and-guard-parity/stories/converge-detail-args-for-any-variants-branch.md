---
title: "detailArgsForAny drops Rails' variants: :any loop arm, which also blocks the details_cache_key call"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: arm-order
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`LookupContext#detailArgsForAny` collapses Rails' two-armed loop into one, so
the details map it builds carries `variants: []` where Rails carries `:any`.
That dropped branch is also what makes the following
`DetailsKey.details_cache_key(details)` call impossible, so one fix retires
both baseline rows for this method.

Rails, `actionview/lib/action_view/lookup_context.rb:188-205`:

```ruby
LookupContext.registered_details.each do |k|
  if k == :variants
    details[k] = :any
  else
    details[k] = Accessors::DEFAULT_PROCS[k].call
  end
end

if @cache
  [details, DetailsKey.details_cache_key(details)]
else
  [details, nil]
end
```

Trails, `packages/actionview/src/lookup-context.ts`:

```ts
for (const k of REGISTERED_DETAILS) {
  details[k] = DEFAULT_PROCS[k]();
}
// ...
const key = this._detailsCache
  ? new Requested({ locale: ..., handlers: ..., formats: ..., variants: "any" })
  : null;
```

Two divergences, one cause:

1. **The `k === "variants"` arm is gone.** Every key takes the
   `DEFAULT_PROCS[k]()` path, so `details.variants` is `[]` — the registered
   default (`registerDetail("variants", () => [])`) — not the wildcard. Rails
   returns a details hash whose `variants` says "any". CLAUDE.md: "Same
   branches, in the same order, with the same guards."
2. **`DetailsKey.detailsCacheKey` is never called**, because a `DetailsMap`
   value is `ReadonlyArray<string | symbol>` and the scalar sentinel has no
   representation in it. The `Requested` is constructed inline instead, so the
   `any?` tuple is not memoized through `DetailsKey._detailsKeys` the way
   every other lookup's is — `isAny` builds a fresh `Requested` per call.

These are the two rows in
`scripts/api-compare/call-mismatches-exclude/actionview/lookup-context.json`
(`call` and `details_cache_key`), both seeded by RFC 0047.

Note for whoever picks this up: the `details_cache_key` row currently reports
as STALE on some branches without the code changing, which is a separate
extractor defect —
`0025-fidelity-verification-tooling/extractor-missing-set-perturbed-by-unrelated-edits`.
Do not retire the row on the strength of that; retire it by making this call.

## Converged shape

`DetailsMap` gains a representation for the `:any` variants sentinel — the
`Requested` constructor already models it
(`template-details.ts:32,43`, `variantsIdx: ReadonlyMap | "any"`) — so the
loop can restore Rails' two arms and the tuple can be built by
`DetailsKey.detailsCacheKey(details)`, memoized like every other.

## Acceptance criteria

- The loop has Rails' two arms; `details.variants` is the sentinel, not `[]`.
- `detailArgsForAny` calls `DetailsKey.detailsCacheKey(details)` under
  `_detailsCache`, matching `lookup_context.rb:198-202`.
- Repeated `isAny` calls reuse one canonical `Requested` from
  `DetailsKey._detailsKeys`.
- Both `detail_args_for_any` rows are deleted from the exclude shard, and the
  mark tightened, _because the calls are now made_.
- The inline-`Requested` bypass comment is deleted, not reworded.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
