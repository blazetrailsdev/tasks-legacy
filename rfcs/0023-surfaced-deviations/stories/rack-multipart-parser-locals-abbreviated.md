---
title: "Restore Rails identifiers for Rack::Multipart::Parser locals and ivars"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "rack"
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

Surfaced while converging `read_data` in PR #6500.
`packages/rack/src/multipart/parser.ts` abbreviates nearly every local and
ivar Rails spells out, which CLAUDE.md's locals-and-parameters rule forbids and
which is most of what makes the body unreadable next to `rack/lib/rack/multipart/parser.rb`.

Observed pairs (trails → `rack/lib/rack/multipart/parser.rb`):

- `this.sb` → `@sbuf` (parser.rb:209)
- `this.qp` → `@query_parser` (parser.rb:240)
- `this.col` → `@collector` (parser.rb:239)
- `this.endBSz` → `@end_boundary_size` (parser.rb:280)
- `this.rxMaxSz` → `@rx_max_size` (parser.rb:420)
- `this.bodyRe` / `this.bodyReEnd` → `@body_regex` / the end variant (parser.rb:412)
- `this.headRe` → `@head_regex` (parser.rb:307)
- `Parser.parse`'s `b` → `boundary`, `p` → `parser`, `cl` → `content_length`,
  `ct` → `content_type` (parser.rb:80-100)
- `readData`'s `c` was `content` — already converged in #6500 (parser.rb:259).

## Acceptance criteria

- [ ] Every local, parameter and field in `packages/rack/src/multipart/parser.ts`
      carries the Rails identifier camelCased, per
      `docs/ruby-ts-conventions.md`.
- [ ] No behavior change; `packages/rack/src/multipart*` suites stay green.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green (the
      `naming` dimension of `parity:api:calls:args:report` should shrink).
