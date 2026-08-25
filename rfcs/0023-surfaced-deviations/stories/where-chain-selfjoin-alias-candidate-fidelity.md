---
title: "where.associated/missing self-join renders association-name alias instead of JoinDependency alias_candidate"
status: ready
updated: 2026-07-08
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

After PR #4790 routed `where.associated` / `where.missing` through JoinDependency
(`joins!` / `left_outer_joins!` + a `:class_name`-keyed predicate), a
`:class_name` self-join renders with the **association name** as its SQL alias
instead of Rails' JoinDependency `alias_candidate` (`{plural}_{owner_table}`):

- `Comment.joins("children")` alone →
  `INNER JOIN "comments" "children_comments" ON "children_comments"."parent_id" = "comments"."id"`
  (correct — matches Rails' `alias_candidate`).
- `Comment.all().where().associated("children")` and
  `Comment.joins("children").where().associated("children")` →
  `INNER JOIN "comments" "children" ON "children"."parent_id" = "comments"."id"
WHERE "children"."id" IS NOT NULL`

So adding the `where`-reference (`self.not(children => {id: nil})`, keyed by the
association name) drives the join alias to `children` rather than the
`children_comments` JoinDependency would otherwise mint. Rows are correct, but
the alias diverges from Rails and is inconsistent with a plain `joins(:children)`
on the same relation. Rails resolves the `association => conditions` hash through
the JoinDependency's aliased table, so both forms land on `children_comments`.

Root cause is in the where-hash association resolution / reference-driven alias
assignment (predicate-builder `associatedTable` block + manager-build alias
selection), not in `whereAssociated`/`whereMissing` themselves — those now
faithfully do `joins! + self.not(association => …)` per query_methods.rb:88-92.

Relevant trails: `packages/activerecord/src/relation/predicate-builder.ts`
(`buildFromHashInternal` associatedTable branch, ~line 147-184),
`packages/activerecord/src/relation.ts` `whereAssociated`/`whereMissing`
(~line 612-652). Rails: `activerecord/lib/active_record/relation/query_methods.rb`
`WhereChain#associated`/`#missing`; `associations/join_dependency.rb`
`table_alias_for` / `alias_candidate`.

## Acceptance criteria

- [ ] A `:class_name` self-join reached via `where.associated`/`where.missing`
      (e.g. `Comment.where.associated(:children)`) renders its target join with
      the JoinDependency `alias_candidate` alias (`children_comments`), matching
      a plain `Comment.joins(:children)` and Rails.
- [ ] The IS NULL / IS NOT NULL predicate lands on that same alias.
- [ ] Existing where-chain tests stay green (test names unchanged); update the
      trails-only SQL-shape assertion in `where-chain.trails.test.ts` (which PR
      #4790 pinned to the `children` alias) back to `children_comments` once the
      alias is converged.
