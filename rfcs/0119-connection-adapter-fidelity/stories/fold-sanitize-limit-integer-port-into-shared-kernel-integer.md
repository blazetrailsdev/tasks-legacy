---
title: "sanitize_limit's file-local Kernel#Integer port folds into the shared pair (database_statements.rb:508-514)"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: ["consolidate-kernel-integer-and-float-conversions"]
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`sanitize_limit` is a bare `Integer(limit)` call in Rails:

```ruby
# activerecord/lib/active_record/connection_adapters/abstract/database_statements.rb:508-514
def sanitize_limit(limit)
  if limit.is_a?(Integer) || limit.is_a?(Arel::Nodes::SqlLiteral)
    limit
  else
    Integer(limit)
  end
end
```

Ruby gets `Integer()` from Kernel. trails has no ported Kernel, so PR #6610
(`converge-limit-bang-to-bare-assignment`) added a THIRD file-local copy in
`packages/activerecord/src/connection-adapters/abstract/database-statements.ts`:
`integerFromString` (sign, `0x`/`0b`/`0o`/`0d` and bare-`0` octal prefixes, `_`
digit separators), `rubyClassName` (the `can't convert Array into Integer`
message arm, with nil/true/false rendering as themselves), and a module-local
`FloatDomainError` for `Integer(Float::NAN)` / `Integer(Float::INFINITY)`.

That copy became load-bearing in #6610: `limit!` is now the bare assignment
Rails has (`query_methods.rb:1215-1218`), so `sanitize_limit` is the ONLY parser
on the LIMIT path. It is differential-tested against MRI over 37 inputs
(`database-statements.test.ts`, the `sanitize limit with {integer,string
integer,invalid value}` tests) — `Integer("012") # => 10` octal,
`Integer("0x1f") # => 31`, `Integer("1_000") # => 1000`, `Integer("0_1") # => 1`,
raises for `"1__0"` / `"08"` / `"0b2"` / `"1e3"`.

`consolidate-kernel-integer-and-float-conversions` (0023) already specifies the
shared pair and this exact grammar, but its acceptance criteria name only
`activesupport/src/cache/store.ts` and `activemodel/attribute-assignment.ts` as
call sites — this third copy postdates it and would otherwise be missed.

## Converged shape

Fold `integerFromString` / `rubyClassName` / `FloatDomainError` into the shared
`Kernel#Integer` port, and make `sanitizeLimit` the one-line
`Integer(limit)` fallback its Ruby counterpart is
(`database_statements.rb:512`). Carry the MRI differential cases over to the
shared port's tests rather than dropping them.

## Acceptance criteria

- [ ] `database-statements.ts` declares no `Integer()`-shaped helpers; the
      `sanitize_limit` else-arm calls the shared port at its Ruby name.
- [ ] The 37 MRI-verified cases still run against the shared port.
- [ ] `pnpm parity:api:extra --package activerecord` shows no new novel surface.
