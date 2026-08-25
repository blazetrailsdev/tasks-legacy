---
title: "rack: Deflater stores sync but compress() buffers whole body — no per-chunk flush streaming"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "rack"
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

Found while converging fetch-nil-presence semantics (PR #5136). Rails
`Rack::Deflater` (vendor/rack/lib/rack/deflater.rb:66-120) wraps the body in
`GzipStream`/deflate streams and honors `@sync` by flushing after every chunk
(`gzip.flush if @sync`, deflater.rb:107,115). The trails port
(packages/rack/src/deflater.ts) stores `sync` faithfully since #5136 but never
reads it: `compress()` buffers the entire body into one string and
gzips/deflates it in a single shot, so `sync: true` (the default) has no
streaming effect and upstream tests like `flush deflated chunks to the client
as they become ready` / `will honor sync: false to avoid unnecessary flushing`
(vendor/rack/test/spec_deflater.rb:453,487) cannot be ported meaningfully.
The regression test in deflater.trails.test.ts asserts the stored private
field via `as any` precisely because there is no behavioral surface.

## Acceptance criteria

- Deflater compresses the body as a stream (per-chunk write, flush after each
  chunk when `sync` is truthy), mirroring GzipStream/DeflateStream semantics.
- The `sync` field is read on the write path; the trails regression test can
  drop its private-field peek in favor of behavioral assertions.
- Port (or explicitly skip with reason) the upstream sync-flush tests.
