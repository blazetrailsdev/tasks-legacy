---
title: "Delete Branch#preloadedRecords' invented root guard and its bespoke Error"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing PR #6771, which made `Branch#preloadedRecords` awaitable
but left this arm as it found it.

Rails' `Preloader::Branch#preloaded_records`
(`vendor/rails/activerecord/lib/active_record/associations/preloader/branch.rb:68-70`)
has no root special-case at all:

```ruby
def preloaded_records
  @preloaded_records ||= loaders.flat_map(&:preloaded_records)
end
```

The root branch's `@preloaded_records` is filled by `attr_writer` (`branch.rb:9`)
from `Preloader#initialize`; a root read before that write simply falls through
to `loaders`, which walks `grouped_records` → `source_records` → `@parent.preloaded_records`
on a nil parent and raises `NoMethodError`.

trails raises a hand-written non-Rails error instead
(`packages/activerecord/src/associations/preloader/branch.ts`, `preloadedRecords`):

```ts
if (this.parent == null) {
  throw new Error("Root preloader branch requires preloadedRecords to be set before access");
}
```

That is an invented guard with an invented message — a branch Rails does not
have, and an error class no Rails caller can rescue on. It also masks the real
failure mode: in Rails the root reader is _reachable_ and answers `[]` for a
root whose loaders are empty, where trails throws unconditionally.

## Converged shape

Delete the root guard so `preloadedRecords()` is Rails' single
`@preloaded_records ||= loaders.flat_map(&:preloaded_records)` for every branch.
Whatever a root read before the writer should do, it should do it by walking the
same path Rails walks, not through a bespoke `Error`.

## Acceptance criteria

- [ ] `Branch#preloadedRecords` has no `parent == null` branch and no
      hand-written `Error` message; the body matches `branch.rb:68-70`.
- [ ] A root branch read after `setPreloadedRecords` behaves unchanged; a read
      before it fails the way Rails fails rather than through a trails-only
      error string.
- [ ] `pnpm parity:api:calls` / `:args` green; `pnpm parity:api:extra --package
activerecord` does not grow; the preloader suites pass with no test
      renames.
