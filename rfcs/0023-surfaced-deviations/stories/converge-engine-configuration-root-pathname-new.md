---
title: "Engine::Configuration#root= drops Pathname.new behind a @missingRailsCall receipt"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
  - "trailties"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Rails::Engine::Configuration#root=`
(`vendor/rails/railties/lib/rails/engine/configuration.rb:115-117`) is

```ruby
def root=(value)
  @root = paths.path = Pathname.new(value).expand_path
end
```

`packages/trailties/src/engine/configuration.ts:52` (`setRoot`) renders it as
`getPath().resolve(getFs().cwd(), value)` and carries a `@missingRailsCall new`
receipt for the dropped `Pathname.new` (added by PR #6452, which surfaced the
mismatch once a Ruby writer stopped resolving to its reader). No trails package
defines a `Pathname` class — `grep -rn "class Pathname" packages/ --include=*.ts`
is empty — so there is no receiver to construct today.

The receipt is debt, not permission: it is the only `@missingRailsCall` on this
file and it will keep hiding any future divergence in the same body.

Related: `packages/activesupport/src/core-ext/pathname/` exists with
`blank.test.ts` / `existence.test.ts` but no `Pathname` implementation, so the
Ruby stdlib class is expected here eventually.

## Converged shape

Either port enough of Ruby's `Pathname` (`new` + `expand_path` at minimum, at
the AS core_ext path that already has the test files) and have `setRoot` call
it, or establish that the trails path adapter IS the Pathname rendering and
record that once, centrally, rather than per call site. Whichever lands, the
`@missingRailsCall new` receipt at `engine/configuration.ts:50-54` is DELETED by
this story.

## Acceptance criteria

- [ ] `setRoot` no longer carries a `@missingRailsCall` tag.
- [ ] `pnpm parity:api:calls` clean with no new baseline row for `root=`.
- [ ] The stored root still expands a relative value against the working
      directory (existing trailties engine/configuration tests stay green).

## Second consumer: `FileStore`'s cache_dir (added by PR #6652)

`FileStoreTest#setup` builds a third store from a `Pathname` cache dir
specifically to pin that `FileStore` accepts one:

```ruby
# vendor/rails/activesupport/test/cache/stores/file_store_test.rb:21
@cache_with_pathname = lookup_store(cache_dir: Pathname.new(cache_dir), expires_in: 60)
```

which `test_key_transformation_with_pathname` (file_store_test.rb:67-71) then
exercises. `FileStore#initialize` is `@cache_path = cache_path.to_s`
(`vendor/rails/activesupport/lib/active_support/cache/file_store.rb:20-23`), so
the Pathname arm is the whole point of that test.

With no `Pathname` class to construct, PR #6652 ported the test building the
store from the same directory as a String, and said so at the call site in
`packages/activesupport/src/cache/stores/file-store.test.ts`
(`key transformation with pathname`). The assertions converged, so that file is
at 0/0/0 — but the Pathname coverage the Rails test exists for is absent.

When this story lands, rebuild `cacheWithPathname` from a real `Pathname` and
delete that note.

- [ ] `key transformation with pathname` constructs its store from a `Pathname`,
      and the deviation note in `file-store.test.ts` is deleted.
