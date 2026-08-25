---
title: "Converge Cache::Store#logger to Rails' instance_writer cattr_accessor"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 100
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6652 (assertion parity for `file_store_test.rb`).

Rails declares the cache logger with an instance writer:

```ruby
# vendor/rails/activesupport/lib/active_support/cache.rb:189
cattr_accessor :logger, instance_writer: true
```

so a test can point one store's logger at its own buffer without touching the
class-wide default — which is what `FileStoreTest#setup` does:

```ruby
# vendor/rails/activesupport/test/cache/stores/file_store_test.rb:23-24
@buffer = StringIO.new
@cache.logger = ActiveSupport::Logger.new(@buffer)
```

trails has the reader/writer as a plain static
(`packages/activesupport/src/cache/store.ts:214`, `static logger: CacheLogger |
null = null`) with no instance writer, so PR #6652's port of
`test_log_exception_when_cache_read_fails` had to save `Store.logger`, assign
the class-wide slot, and restore it in a `finally` — noted at
`packages/activesupport/src/cache/stores/file-store.test.ts` in the
`log exception when cache read fails` body. That is a global mutation standing
in for a per-instance one: it works only because the test restores it, and it
would cross-talk between concurrently-running store tests.

## Converged shape

Give `Store` the instance writer `cattr_accessor :logger, instance_writer:
true` implies — a per-instance `logger` slot that shadows the class-wide value
when set, with the class-level reader/writer unchanged. Note that a TS `set`
accessor is fine here (the write is synchronous), so the Rails name `logger` is
reachable directly and no `setLogger()` workaround is needed.

Every internal read of the logger must then go through the instance
(`this.logger`), not `Store.logger`, so the shadowing actually takes effect —
`cache.rb:896` (`logger.error(...)`), `cache.rb:1011-1021`
(`logger.debug?` / `logger.debug`), and in trails
`packages/activesupport/src/cache/store.ts:590`, `:603`, `:757` plus
`packages/activesupport/src/cache/file-store.ts:170`.

Then rewrite `log exception when cache read fails` in
`packages/activesupport/src/cache/stores/file-store.test.ts` to assign
`store.logger` rather than saving and restoring the class-wide slot, and delete
the note explaining the workaround.

## Acceptance criteria

- `Store` has a per-instance `logger` writer shadowing the class-wide value,
  matching `cattr_accessor :logger, instance_writer: true` (cache.rb:189).
- All internal logger reads go through the instance, so a per-store logger is
  actually used.
- `log exception when cache read fails` assigns `store.logger` and no longer
  mutates or restores `Store.logger`; the workaround note is deleted.
- `file_store_test.rb` stays at 0 assertion-count / 0 assertion-kind / 0
  assertion-value mismatches, and the `parity:api:calls` /
  `parity:api:calls:args` ratchets stay clean.
