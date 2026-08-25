---
title: "Cache payload byte sizes mix latin1 and UTF-8 units in the compression decision"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6446. Rails compares like with like when deciding to compress:
`Coder#dump_compressed` (activesupport/lib/active_support/cache/coder.rb:37-51)
and `Marshal70WithFallback#dump_compressed`
(activesupport/lib/active_support/cache/serializer_with_fallback.rb:77-86) both
weigh `compressed.bytesize` against the serialized payload's `bytesize`, and
both strings are byte strings.

trails mixes two units for the same comparison, because a JS string carries no
encoding:

- `packages/activesupport/src/cache/coder.ts` `Coder#tryCompress` compares
  `compressed.length` (a latin1 string out of `gzip.ts`'s `deflate`, one char
  per byte) against `Buffer.byteLength(string)` (UTF-8 bytes).
- `serializer-with-fallback.ts` `marshal70WithFallback.dumpCompressed` does the
  same.
- `entry.ts` `Entry#bytesize` and the `sizeOf` helper in
  `cache/behaviors/cache-store-compression-behavior.ts` each discriminate the
  two shapes by hand.
- `file-store.ts:175` then writes the payload as `"utf-8"`, so a compressed
  payload's bytes on disk are ~1.5x the `.length` the decision was made on.

So a payload can be "compressed" into something that occupies more storage than
the uncompressed form, and every reader of a payload size needs to know which
shape it holds.

## Converged shape

One byte-string representation for cache payloads — latin1 throughout, so
`.length` IS `#bytesize` for every payload shape and every comparison and
`Entry#bytesize` collapse to a single arm, mirroring Ruby's binary String. That
means `coder.dump` emitting its UTF-8 bytes as a latin1 string, `FileStore`
reading/writing `"latin1"`, and dropping the per-shape discrimination in
`Entry#bytesize` and the compression behavior helper.

## Acceptance criteria

- [ ] The compress/don't-compress comparison weighs two counts in the same unit.
- [ ] `Entry#bytesize` has no compressed-vs-uncompressed unit branch.
- [ ] `sizeOf` in `cache-store-compression-behavior.ts` needs no shape
      discrimination.
- [ ] `CacheStoreCompressionBehavior` stays green for FileStore and MemoryStore.
