---
title: "Tempfile buffers writes in memory because the fs adapter has no descriptor-based write"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Ruby's `Tempfile` is an IO delegate: `initialize` holds the `File` object from
`File.open(tmpname, RDWR|CREAT|EXCL, perm: 0600)` (`tempfile.rb:157-161`), every
write goes through that descriptor, and `_close` (`tempfile.rb:197-199`) closes
it.

`packages/activesupport/src/tempfile.ts` has no descriptor to hold. The
`FsAdapter` is open-write-close per call — `writeFileSync` / `readFileSync`,
with `openSync`/`closeSync` exposed only as a raw fd pair used for the `O_EXCL`
create and for `flockSync`. So the port buffers writes in memory (`buffer`,
`flushed`) and `close()` flushes them, which is documented on the class but is
not what Ruby does:

- `write()` returns the byte count without the bytes reaching the file, so a
  reader that opens the path directly between `write` and `close` sees stale
  content where Ruby's would not (Ruby's own IO buffering is smaller and
  flushable, and `Tempfile#path` is explicitly meant to be handed to other
  processes — `postgresql_database_tasks.rb:143` does exactly that).
- `IO#flush`, `IO#rewind`, `IO#sync=` and friends have nowhere to land.
- `close` cannot mirror `_close` because there is nothing to release.

The convergence is a descriptor-shaped seat on the fs adapter — a `write(fd,
buffer)` beside the existing `readSync(fd, ...)` (`fs-adapter.ts:69-77`) and
`closeSync(fd)` — so `Tempfile` can hold the fd `openSync` already returns and
write through it, the way `tempfile.rb` does. `gzip.ts` carries a similar
buffer-shaped stand-in (`buffer`/`rewind`/`read`/`write` show as novel surface
in `parity:api:extra`), so the seat is likely to pay for itself more than once.

## Acceptance criteria

- [ ] The fs adapter exposes descriptor-based write alongside its existing
      `openSync`/`readSync`/`closeSync`, with the node adapter implementing it.
- [ ] `Tempfile` holds the fd from its `openSync(path, "wx")` create and writes
      through it; the `buffer` / `flushed` fields and the private `flush()` are
      gone, and `close()` closes the descriptor as `_close` does.
- [ ] The class JSDoc paragraph explaining the in-memory buffer is deleted, not
      reworded.
- [ ] `packages/activesupport/src/tempfile.trails.test.ts` gains a case proving
      a write is visible to a reader that opens the path before `close()`.
- [ ] Existing `encrypted-file`, `core-ext/file` and `FileStore` suites stay
      green on all three database lanes.
