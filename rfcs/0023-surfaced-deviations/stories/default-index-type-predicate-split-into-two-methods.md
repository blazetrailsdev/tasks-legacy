---
title: "Rails' default_index_type?(index) exists twice in trails as defaultIndexType(using) and isDefaultIndexType(index), with divergent bodies"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced reading `abstract-mysql-adapter.ts` end-to-end for RFC 0106 wave 3b
(PR #6577).

Rails has ONE method, taking the index object
(`abstract_mysql_adapter.rb:634-636`):

    def default_index_type?(index)
      index.using == :btree || super
    end

and the base (`abstract/schema_statements.rb`):

    def default_index_type?(index)
      index.using.nil?
    end

trails has TWO, on both the abstract adapter and the MySQL one:

- `AbstractAdapter#defaultIndexType(using?: string): boolean`
  (`abstract-adapter.ts:1959`), overridden at `abstract-mysql-adapter.ts:354`
  and `postgresql-adapter.ts:3147` as `using === "btree" || super...`
- `AbstractAdapter#isDefaultIndexType(_index: unknown): boolean`
  (`abstract-adapter.ts:2232`), overridden at `abstract-mysql-adapter.ts:1094`
  as `index.using == null || index.using.toUpperCase() === "BTREE"`

So the same Rails predicate exists twice per adapter, under two names, with two
different signatures (a `using` string vs the index object) and two subtly
different bodies — the second folds the base's `using.nil?` in by hand and adds
a case-insensitive compare Rails does not have.

`isDefaultIndexType` is the one matching Rails' arity and receiver;
`defaultIndexType(using)` is the invented split. Per the naming table,
`default_index_type?` maps to `isDefaultIndexType`, so the `defaultIndexType`
spelling is also the wrong name for a Ruby predicate.

## Converged shape

One method per adapter — `isDefaultIndexType(index)` — with Rails' two-line
body (`index.using === "btree" || super.isDefaultIndexType(index)`) over a base
that answers `index.using == null`. `defaultIndexType` and its two overrides are
deleted and their call sites re-pointed.

Check the Ruby Symbol arm while converging: Rails compares `index.using ==
:btree`, i.e. the string `"btree"` in trails, NOT `"BTREE"` — confirm which
casing the MySQL/PG introspection paths actually store before dropping the
`toUpperCase()`.

## Acceptance criteria

- [ ] Exactly one `isDefaultIndexType(index)` per adapter, mirroring
      rb:634-636 and the abstract base.
- [ ] `defaultIndexType` is gone; `pnpm parity:api:extra --package activerecord`
      does not report it.
- [ ] Existing index-dumping tests still pass on all three dialects (the
      predicate gates whether `using:` is emitted into schema.rb).
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
