---
title: "converge-disable-joins-association-scope-test-to-canonical"
status: draft
updated: 2026-08-03
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`packages/activerecord/src/associations/disable-joins-association-scope.test.ts`
builds bespoke `djs_authors` / `djs_posts` / `djs_comments` tables inline
(createTable + dropTable in before/after hooks) rather than going through the
canonical schema. None of those table names exists in
`vendor/rails/activerecord/test/schema/schema.rb`, and the models they imply
have no counterpart in `vendor/rails/activerecord/test/models/`.

Rails' disable_joins coverage runs on the canonical models
(`vendor/rails/activerecord/test/cases/associations/has_many_through_disable_joins_associations_test.rb`
uses `Member` / `Club` / `Sponsor`), which trails already mirrors in
`has-many-through-disable-joins-associations.test.ts`.

Noticed while working `extra-surface-classify-invented-adapter-constants`
(PR #5936), which touched these tests to drop the invented
`DisableJoinsAssociationScope.INSTANCE` override.

## Acceptance criteria

- The `djs_*` inline tables and their createTable/dropTable hooks are gone.
- The file's tests run against canonical tables/models via `fixtures({ ... })`.
- Test names are unchanged.
