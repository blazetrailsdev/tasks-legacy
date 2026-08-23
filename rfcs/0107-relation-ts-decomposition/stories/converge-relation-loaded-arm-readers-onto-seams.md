---
title: "Relation readers branch on _loaded/_records instead of the loaded?/records seams a CollectionProxy overrides"
status: ready
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: 1
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `Relation` readers take their loaded arm through the `loaded?` and
`records` seams, never through the `@loaded` / `@records` ivars directly:

- `empty?` — `relation.rb:362-369` (`if loaded? then records.empty?`)
- `one?` — `relation.rb:404-410` (`return records.one? if loaded?`)
- `many?` — `relation.rb:413-419` (`return records.many? if loaded?`)
- `size` — `relation.rb:349` (`loaded? ? records.length : count(:all)`)
- `records` — `relation.rb:342`

That indirection is load-bearing: `CollectionProxy` overrides exactly those two
members (`collection_proxy.rb:1024-1026` `records` is `load_target`, `:53-55`
`loaded?` is `@association.loaded?`), which is how the inherited `Relation`
bodies read an association's target with no override of their own.

PR #6758 converged `isEmpty` / `isMany` / `isOne` for that reason — deleting the
proxy's `many` / `one` / `exists` overrides made an inherited body run against a
proxy for the first time, and `this._loaded` / `this._records` are empty there.
The rest of `relation.ts` still reads the ivars directly. `grep -n
"this\._loaded\b\|this\._records\b" packages/activerecord/src/relation.ts` is 17
hits; the reader-side ones (as opposed to the writer-side assignments in
`reset` / `loadRecords`, which are correct) are:

- `size()` (`:866`) — `if (this._loaded) return this._records.length;`
- `toArray()` (`:1024`, `:1042`)
- `computeCacheVersion()` (`:2940-2946`) — Rails' `cache_version`
- `inspect()` (`:617-619`) and `prettyPrint()` (`:646`)

Each is a latent version of the same bug: correct on a plain `Relation`, wrong
on a `CollectionProxy` (and on anything else that overrides the seams), and
invisible until an override is deleted.

## Converged shape

Every reader takes its loaded arm through `this.isLoaded` and `await
this.records()`, matching the Ruby cites above. The writer-side
`this._loaded = …` / `this._records = …` assignments in `reset` and the load
path stay as they are — Rails assigns the ivars there too.

## Acceptance criteria

- No reader in `relation.ts` branches on `this._loaded` or reads
  `this._records`; the remaining hits are assignments only.
- `size`, `toArray`, `computeCacheVersion`, `inspect`, `prettyPrint` return the
  target (not a query, not an empty array) when called on a loaded
  `CollectionProxy`.
- `pnpm parity:api:calls` / `:args` add zero rows.
