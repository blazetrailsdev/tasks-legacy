---
title: "in_batches validates its options on first iteration, not at the call site (batches.rb:250)"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

## Context

Surfaced while moving `inBatches` into
`packages/activerecord/src/relation/batches.ts` in PR #6594. The deviation is
pre-existing and was moved verbatim, not introduced there.

Rails calls the validation EAGERLY, as the second statement of the method:

    # vendor/rails/activerecord/lib/active_record/relation/batches.rb:249-250
    def in_batches(of: 1000, ..., use_ranges: nil, &block)
      cursor = Array(cursor).map(&:to_s)
      ensure_valid_options_for_batching!(cursor, start, finish, order)

so `Person.in_batches(cursor: :nickname)` raises from the `in_batches` call
itself, before any BatchEnumerator is returned and before any row is read.
`ensure_valid_options_for_batching!` (batches.rb:306-330) raises
`ArgumentError` for a start/finish arity mismatch, for a cursor with no
covering unique index, and for an `:order` that is not `:asc`/`:desc`.

trails defers the whole call into the generator:

    // packages/activerecord/src/relation/batches.ts (inBatches)
    const ensureValidOptions = () =>
      ensureValidOptionsForBatchingBang(self, cursor, start, finish, (order ?? "asc") as any);
    ...
    generator = async function* () {
      await ensureValidOptions();
      ...

so the error surfaces on first iteration rather than at call time. The reason
is that the unique-index arm reaches `model.schemaCache().indexes(...)`, which
is async in trails and sync in Rails, making the whole helper a promise that
the synchronous `inBatches` signature cannot await.

Consequences: `expect(() => rel.inBatches({ cursor: "x" })).toThrow()` cannot
be written the way the Rails test writes it; a caller that builds a
BatchEnumerator and never iterates never learns its options are invalid; and
the `:order` arm (batches.rb:326-330), which needs no database access at all,
is deferred along with the arm that does.

## Converged shape

Split the helper along the sync/async seam rather than deferring all of it.
The arity checks (batches.rb:306-312) and the `:order` validation
(:326-330) are pure and can run synchronously at the top of `inBatches`,
matching Rails' raise site. Only the unique-index arm (:314-324) needs the
schema cache; keep that one deferred, with the deviation justified at that
call site with the `batches.rb:316` cite.

If `schemaCache().indexes` can be served from an already-warmed cache
synchronously, prefer converging the whole helper to Rails' eager call and
delete the deferral entirely.

## Acceptance criteria

- [ ] The arity and `:order` arms of `ensureValidOptionsForBatchingBang` raise
      from the `inBatches` call itself, matching batches.rb:250.
- [ ] Any residual deferral is narrowed to the schema-cache arm and carries a
      one-line justification at the call site citing `batches.rb:316`.
- [ ] A test covers `inBatches` raising without iterating, mirroring the Rails
      test in `vendor/rails/activerecord/test/cases/batches_test.rb`.
- [ ] `pnpm parity:api:calls` / `:args` green; no new baseline rows.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
