---
title: "check_constraints_in_create's invalid arm returns a raw array where Rails puts a joined block to a remaining stream"
status: done
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 70
priority: 35
pr: 7026
claim: "2026-08-25T09:46:54Z"
assignee: "missing-rails-call-tag-inert-on-non-rails-class-member"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `check_constraints_in_create`'s `join` separator in
PR #6362 (RFC 0099 `call-args-ar-join-separator`).

Rails' `check_constraints_in_create`
(`vendor/rails/activerecord/lib/active_record/schema_dumper.rb:283-306`) has
two arms. The VALID arm is now ported faithfully — one
`stream.puts stmts.sort.join("\n")` (`schema_dumper.rb:292`), which trails
spells as `lines.push(checkConstraintStatements.sort().join("\n"))` in
`packages/activerecord/src/schema-dumper.ts`.

The INVALID arm (`schema_dumper.rb:295-305`) is NOT converged. Rails builds a
`StringIO` named `remaining`, does
`remaining.puts add_check_constraint_statements.sort.join("\n")`
(`schema_dumper.rb:303`) and returns that stream; trails returns the sorted
`string[]` directly, with no `join("\n")` at all. Same emitted text today
(the caller joins its lines with `"\n"`), but a different shape: there is no
`remaining` analogue, so the two arms of one Rails method are ported to two
different return types.

The same `remaining`-stream shape recurs in `schema_dumper.rb` — converging
one arm in isolation is why this is worth its own story rather than a drive-by.

## Converged shape

Give the invalid arm the same treatment as the valid one — a single joined
block standing in for `remaining.puts stmts.sort.join("\n")` — so both arms of
`check_constraints_in_create` return the same shape and the body reads
line-for-line against `schema_dumper.rb:283-306`. Decide at the same time
whether trails wants a `remaining`-stream analogue at all or whether the
one-joined-block form IS the settled port of `StringIO` + `puts` (the caller
already treats a pushed entry containing newlines as one block).

## Acceptance criteria

- [ ] Both arms of `checkConstraintsInCreate` produce their block through one
      `sort().join("\n")`, mirroring `schema_dumper.rb:292` and `:303`.
- [ ] Emitted dump text is byte-identical (regression-checked against
      `packages/activerecord/src/schema-dumper.test.ts`).
- [ ] The chosen `remaining`-stream disposition is stated at the call site if
      it deviates, or converged if it does not.
- [ ] `pnpm parity:api:calls` / `:args` stay green.
