---
title: "Converge Cache::Store's public name parameter from string to unknown"
status: draft
updated: 2026-08-17
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

Surfaced by PR #6652 (assertion parity for `file_store_test.rb`).

`Cache::Store`'s public key-taking readers and writers take `name` as any
object, and `normalize_key` is what converts it:

```ruby
# vendor/rails/activesupport/lib/active_support/cache.rb:498, :662, :703
def read(name, options = nil)
def write(name, value, options = nil)
def exist?(name, options = nil)
```

trails narrows all three to `name: string`
(`packages/activesupport/src/cache/store.ts:390`, `:489`, `:519`), so a Rails
test that writes under a non-String key cannot be ported literally.
`file_store_test.rb:155-160` is exactly that case:

```ruby
def test_write_with_unless_exist
  assert_equal true, @cache.write(1, "aaaaaaaaaa")
  assert_equal false, @cache.write(1, "aaaaaaaaaa", unless_exist: true)
  @cache.write(1, nil)
  assert_equal false, @cache.write(1, "aaaaaaaaaa", unless_exist: true)
end
```

PR #6652 had to spell the Integer key `1` as the String `"1"` in
`packages/activesupport/src/cache/stores/file-store.test.ts` to typecheck. The
assertions themselves converged (that file is at 0/0/0), so this is a signature
gap rather than an open parity row — but every future port of a Rails cache
test that keys on a Symbol, Integer, Array or a `cache_key`-responding object
hits the same wall, and the workaround is a cast at each call site.

## Converged shape

Widen the `name` parameter of `Store`'s public key-taking surface to `unknown`,
matching Rails, and let `normalizeKey` / `expandedKey` do the conversion as
Rails does (`cache.rb` `normalize_key` / `expanded_key`). At minimum `read`,
`write`, `exist`, `delete`, `fetch`, `increment`, `decrement`; check the whole
public surface of `store.ts` against `cache.rb` for others taking `name`.

`FileStore#normalizeKey` already accepts `key: unknown`
(`packages/activesupport/src/cache/file-store.ts:230`), so the private layer is
already Rails-shaped — this is the public signatures only.

Then restore the Rails literal in
`packages/activesupport/src/cache/stores/file-store.test.ts` (`write with
unless exist` → `store.write(1, …)`), and drop any `as unknown as string` casts
elsewhere that only exist to feed a non-String cache key.

## Acceptance criteria

- `Store`'s public key-taking methods take `name: unknown`, as `cache.rb` does.
- `write with unless exist` in
  `packages/activesupport/src/cache/stores/file-store.test.ts` uses the Integer
  key `1` verbatim, matching file_store_test.rb:155-160.
- `pnpm parity:api` / `pnpm parity:test` deltas are non-negative and the
  `parity:api:calls` / `parity:api:calls:args` ratchets stay clean.
