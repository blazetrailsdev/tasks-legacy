---
title: "StatementPool's public surface is a JS Map API, not Rails'"
status: draft
updated: 2026-08-18
rfc: "0119-connection-adapter-fidelity"
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

# `StatementPool`'s public surface is a JS Map API, not Rails'

## Context

Surfaced while converging `StatementPool`'s internal `cache` reader in PR #6718
(RFC 0106 wave 4b). That PR fixed the six call-set rows by giving the class the
private `cache` accessor Rails calls throughout
(`activerecord/lib/active_record/connection_adapters/statement_pool.rb:60-62`)
and deleting an invented module-level `cache()` export. It did not touch the
public surface, which is still spelled as a JS Map rather than as Rails'.

Rails (`statement_pool.rb:16-57`) vs
`packages/activerecord/src/connection-adapters/statement-pool.ts`:

| Rails            | trails                                   |
| ---------------- | ---------------------------------------- |
| `[](key)`        | `get(key)`                               |
| `[]=(sql, stmt)` | `set(key, stmt)`                         |
| `key?(key)`      | `isKey(key)` **and** a second `has(key)` |
| `length`         | `length` (matches)                       |
| —                | `maxSize` getter                         |
| —                | `setMaxSize(maxSize)`                    |
| —                | `keys` getter                            |

`isKey` is the conventional rename of `key?`, but `has` is a duplicate of it
under a JS name, and `maxSize` / `setMaxSize` / `keys` have no Rails
counterpart at all — `@statement_limit` is set once in `initialize` and never
reassigned, so `setMaxSize` also carries eviction logic Rails has no site for.
`get` additionally does LRU reordering (delete + re-set) that Rails' `[](key)`
does not: Ruby's `[]=` evicts with `cache.shift`, which is insertion order, so
trails' read path silently changes which statement gets evicted next.

## Converged shape

- `get`/`set` keep Rails' semantics under the trails spelling for `[]`/`[]=`
  (check `docs/ruby-ts-conventions.md` for the settled operator-method
  translation before renaming — do not invent one).
- Drop the LRU reordering from the read path so eviction is insertion-ordered,
  matching `cache.shift` at `statement_pool.rb:32`.
- Delete `has` in favour of the single `isKey`, and delete `keys`.
- Delete `maxSize` / `setMaxSize`, or, if a caller genuinely needs to resize,
  tag it `@noRailsEquivalent` with the reason at the call site — check callers
  first; if there are none, it is dead invented surface.
- Verify with `pnpm parity:api:extra --package activerecord` that
  `connection-adapters/statement-pool.ts` reports no novel names afterwards.

## Acceptance criteria

- [ ] No public member of `StatementPool` lacks a Rails counterpart or a
      reviewed `@noRailsEquivalent` tag at its call site.
- [ ] Eviction order matches `cache.shift` (insertion order), with a test that
      fails on the current LRU behaviour.
- [ ] `pnpm parity:api:extra --package activerecord` shows no novel names for
      this file.
- [ ] `pnpm parity:api:calls` / `:args` green; PG and MySQL lanes green (both
      subclass this pool for prepared statements).
