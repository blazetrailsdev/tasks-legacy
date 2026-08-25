---
title: "AbstractMysqlAdapter#strict_mode? is a hardcoded false stub; Rails reads @config.fetch(:strict, true) and the real decision is duplicated inline in _buildInitSql"
status: done
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 90
pr: 7059
claim: "2026-08-25T17:30:31Z"
assignee: "exec-scope-takes-explicit-fn-instead-of-instance-exec"
blocked-by: null
closed-reason: null
---

## Context

Surfaced reading `abstract-mysql-adapter.ts` end-to-end for RFC 0106 wave 3b
(PR #6577). Not a call-set row — the Ruby body makes a call the gate does not
pair — so nothing else will surface it.

`abstract_mysql_adapter.rb:630-632`:

    def strict_mode?
      self.class.type_cast_config_to_boolean(@config.fetch(:strict, true))
    end

trails (`abstract-mysql-adapter.ts:1090`):

    isStrictMode(): boolean {
      return false;
    }

A hardcoded `false` where Rails reads config and DEFAULTS TO TRUE. Note this is
`@config.fetch(:strict, true)`, not `@config[:strict] || true` — a stored
`false` is honoured, and an absent key means strict (see
[[project_ruby_fetch_nil_presence_vs_js_nullish]]).

`strict_mode?` is read by `configure_connection` (rb:928-929) to decide between
`CONCAT(@@sql_mode, ',STRICT_ALL_TABLES')` and the three `REPLACE(...)` calls
that strip strictness. trails does not call `isStrictMode` from anywhere —
`grep -rn isStrictMode packages/activerecord/src` matches only the definition.
The equivalent logic is re-implemented inline in
`Mysql2Adapter#_buildInitSql` (`mysql2-adapter.ts:1817+`) off a destructured
`strict` field, so the observable SET is roughly right while the Rails-named
predicate is a dead stub returning the wrong default.

Two defects, one story: the stub, and the duplicated/relocated decision.

## Converged shape

    isStrictMode(): boolean {
      return AbstractMysqlAdapter.typeCastConfigToBoolean(fetch(this._config, "strict", true));
    }

with `_buildInitSql`'s inline strict handling routed through it, so there is one
source of truth at the Rails name. Coordinate with RFC 0076, which owns
`configure_connection` on this file.

## Acceptance criteria

- [ ] `isStrictMode` mirrors rb:630-632, including the `fetch(:strict, true)`
      default-true semantics (a stored `false` still wins).
- [ ] The sql_mode decision in `_buildInitSql` reads `isStrictMode()` rather
      than re-deriving it.
- [ ] A regression test that FAILS on baseline: an adapter built with no
      `strict` key emits the `STRICT_ALL_TABLES` arm.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
