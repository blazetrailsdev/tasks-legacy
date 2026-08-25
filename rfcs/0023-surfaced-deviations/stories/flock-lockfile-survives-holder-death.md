---
title: "Release the fs-adapter flock on holder death, as flock(2) does"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`FileStore#lock_file` takes `File::LOCK_EX` and releases it in an `ensure`
(`vendor/rails/activesupport/lib/active_support/cache/file_store.rb:140-153`).
The kernel releases an `flock(2)` when the holding process dies, so a crash mid
block cannot wedge later lockers.

trails' Node fs adapter (`packages/activesupport/src/fs-adapter.ts`,
`withFlock`) implements `flockSync` with an `O_EXCL` lockfile at
`<entry>.lock` plus a blocking retry loop, because Node exposes no `flock(2)`.
PR #6448 removed the age-based staleness sweep (it broke a legitimate long
hold, which Rails never does), so exclusion is now faithful — but a process
killed while holding the lock leaves the lockfile behind and every later
`increment`/`decrement` on that key blocks forever. Rails has no such failure
mode.

A secondary artifact: the `<entry>.lock` file is transiently visible to
`searchDir`, so `cleanup` / `deleteMatched` can observe a path that decodes to a
bogus key.

## Converged shape

Release the lock on holder death the way the kernel does — e.g. hold the
lockfile's descriptor open and detect a dead holder via an OS-backed signal
rather than a timeout, or move the lock off a plain file onto a primitive with
owner-death semantics — without reintroducing a time-based break. Keep the
lockfile out of `searchDir`'s key space (or out of the cache tree).

## Acceptance criteria

- [ ] A process (or worker) killed while holding the lock does not wedge later
      lockers.
- [ ] No age/timeout heuristic breaks a live holder's exclusion.
- [ ] The lock artifact is invisible to `searchDir` / `cleanup` /
      `deleteMatched`.
- [ ] The concurrent-increment regression test in
      `file-store-atomic-write.trails.test.ts` stays green.
