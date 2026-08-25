---
title: "table_options is split across the adapter and an invented exported parseTableOptions helper, duplicating the comment-needed predicate"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 130
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

`abstract_mysql_adapter.rb:549-576` keeps `table_options` as ONE method that
mutates a local through a sequence of `sub`/`sub!` calls, using Ruby's pre-match and post-match globals
to splice around the charset match, and calls `table_comment(table_name)` only
when the COMMENT strip actually fired:

    def table_options(table_name)
      create_table_info = create_table_info(table_name)
      raw_table_options = create_table_info.sub(/\A.*\n\) ?/m, "").sub(/\n\/\*!.*\*\/\n\z/m, "").strip
      return if raw_table_options.empty?
      table_options = {}
      if / DEFAULT CHARSET=(?<charset>\w+)(?: COLLATE=(?<collation>\w+))?/ =~ raw_table_options
        raw_table_options = PREMATCH + POSTMATCH   # Ruby: dollar-backtick + dollar-quote
        ...
      end
      raw_table_options.sub!(/(ENGINE=\w+)(?: AUTO_INCREMENT=\d+)/, '\1')
      if raw_table_options.sub!(/ COMMENT='.+'/, "")
        table_options[:comment] = table_comment(table_name)
      end
      table_options[:options] = raw_table_options unless raw_table_options == "ENGINE=InnoDB"
      table_options
    end

trails splits this across `AbstractMysqlAdapter#tableOptions`
(`abstract-mysql-adapter.ts:979`) and an EXPORTED module-level
`parseTableOptions(createInfo, tableComment)` (`:1869`), because the comment
fetch is async and the parse is not. The adapter method pre-decides whether a
comment is needed by testing `/COMMENT='/` against a separately re-derived
"tail", then passes the fetched comment INTO the parser.

Two problems:

1. `parseTableOptions` is a helper Rails does not have, exported from the
   module (extra public surface — it carries an `@internal` tag rather than a
   `@noRailsEquivalent` justification).
2. The comment-needed test is duplicated and NOT the same predicate: the
   adapter matches `/COMMENT='/` against `createInfo.replace(/[\s\S]*\n\) ?/, "")`,
   while the parser separately does the real `sub!(/ COMMENT='.+'/, "")`. Rails
   has one predicate — whether the `sub!` returned non-nil (see
   [[project_rails_callback_halt_throw_abort_only]] for the bang-method arm
   convention). A table whose options tail contains `COMMENT='` in a form the
   parser's stricter regex misses (or vice versa) diverges.

## Converged shape

One `tableOptions` method mirroring rb:549-576 in order, with Ruby's pre-match
plus post-match splice expressed as a slice around the match index, and the
comment fetch awaited at the point Rails calls it — i.e. `tableOptions` stays
async and the `sub!` result drives the single `tableComment()` call.
`parseTableOptions` is deleted and its unit tests re-pointed at `tableOptions`.

## Acceptance criteria

- [ ] `tableOptions` mirrors rb:549-576; one method, same branch order.
- [ ] `parseTableOptions` is gone; `pnpm parity:api:extra --package activerecord`
      does not report it.
- [ ] `tableComment()` is issued exactly when Rails issues it (the `sub!` fired),
      and at most once.
- [ ] Existing MySQL schema-dump tests covering charset/collation/AUTO_INCREMENT/
      COMMENT/ENGINE options still pass.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
