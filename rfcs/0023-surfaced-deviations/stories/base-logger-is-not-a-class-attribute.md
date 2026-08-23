---
title: "Base.logger is a plain accessor pair where Rails uses class_attribute, and three call sites keep a now-unforced level ?."
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 110
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::Base.logger` is `class_attribute :logger, instance_writer: false`
(`activerecord/lib/active_record/core.rb:22`) holding an
`ActiveSupport::Logger`, which answers every level unconditionally. trails
declares it as a plain `static get/set logger` pair over a `_logger` module
field (`packages/activerecord/src/base.ts:1643-1659`) — so reads do not walk
the constructor chain and writes are not class-local, the two semantics
`classAttribute()` from `@blazetrails/activesupport` already gives.

PR #6931 narrowed the property's TYPE onto `BenchmarkLogger` (whose five level
methods are now required), which is what made `benchmark`'s direct
`logger.public_send(level, ...)` dispatch (`benchmarkable.rb:47`) type-check.
The accessor SHAPE was left alone as out of scope.

That same PR also left three call sites carrying a `?.` on the level method
that the narrowed type no longer forces:

- `packages/activerecord/src/migration.ts:2673` — `_base.logger.info?.(...)`,
  with a stale comment above it claiming `Base.logger` "declares every level
  optional (base.ts:1717-1723)". Rails is a bare
  `Base.logger.info "Migrating to ..."` (`migration.rb:1538` region).
- `packages/activesupport/src/cache/store.ts:606` — `Store.logger.debug?.(...)`
  against Rails' `logger.debug "Cache #{operation}#{key}#{options}"`
  (`activesupport/lib/active_support/cache.rb`).
- `packages/trailties/src/rack/logger.ts:73` — `this.logger.info?.(...)`.

## Converged shape

- `Base.logger` becomes a `classAttribute("logger", { instanceWriter: false })`,
  mirroring `core.rb:22`, keeping the `BenchmarkLogger` element type.
- The three `?.` level calls become direct calls, and migration.ts' stale
  comment (which cites line numbers that no longer describe the type) is
  deleted.

## Acceptance criteria

- [ ] `Base.logger` is declared with `classAttribute`, `instanceWriter: false`,
      mirroring `core.rb:22`.
- [ ] No `logger.<level>?.()` optional call remains in non-test source.
- [ ] `pnpm parity:api:calls` / `:args` green with no new rows.
