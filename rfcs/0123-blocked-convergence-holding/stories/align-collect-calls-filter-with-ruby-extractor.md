---
title: "collectCalls records _private()/Klass() names the Ruby extractor drops"
status: blocked
updated: 2026-08-25
rfc: "0123-blocked-convergence-holding"
cluster: api-compare
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: "The filter does not only delete call-set entries: `calls` is also the edge set reachedSameFileMethods/SAME_FILE_CLOSURE_DEPTH walks (compare.ts:538-585), so dropping `_`-prefixed names severs every this._helper() closure edge. Measured with the filter applied: 92 NEW call-mismatch rows (44 STALE), and the NEW rows are closure false positives, not divergences — e.g. activemodel callbacks.ts _define_after_model_callback/new, actioncontroller metal/strong-parameters.ts convert_value_to_parameters/new, activerecord type/type-map.ts perform_fetch/call. Landing AC1 as written means baselining ~92 rows wholesale. Needs a companion design that keeps the closure sound: apply the filter in compare.ts after the same-file closure is computed, or emit closure edges separately from the compared call set."
closed-reason: null
---

> Re-filed from RFC 0084 on 2026-08-14 when 0084 was superseded by RFC 0106
> (direct burndown). Body preserved verbatim below; the original is closed as superseded.

## Context

`callSiteName` (`scripts/api-compare/extract-ts-api.ts`, added by #6304) now
applies the Ruby extractor's own name filter — `extract-ruby-api.rb`'s
`call_site_name` drops a name starting with `_` or with anything other than a
lowercase letter — so the `callArgs` stream cannot manufacture TS-only sites.

`collectCalls`, feeding `calls` / `callSeq`, does NOT apply it: it records
`this._privateHelper()` and `Klass(...)` as call names. Ruby never emits those,
so every such name is an unpairable TS-only entry in the call set. Left
unaligned in #6304 because changing it moves the `parity:api:calls` population, which
needs its own measured PR.

Reference: `extract-ruby-api.rb#call_site_name` (the `!name.start_with?("_") &&
name =~ /\A[a-z]/` guard, shared with `walk_for_calls`), against
`collectCalls`'s identifier / property-access / super branches.

Note this is the reverse direction of the usual concern: the filter REMOVES
names, so it can only delete call-set entries. Verify it does not silently drop
a name that was legitimately pairing — a Ruby method genuinely named with a
leading underscore is dropped on the Ruby side too, so the pair was already
impossible.

## Acceptance criteria

1. `collectCalls` applies the same name filter as `callSiteName`, citing the
   Ruby guard.
2. The `parity:api:calls` artifact is regenerated and the row movement reported; stale
   baseline rows for now-dropped names are deleted by hand (only-shrink — no
   `--write` reseed).
3. Tests pin that `this._helper()` and `Klass()` are not credited, while
   `constructor` and `super` still are.
