---
title: "activerecord/lib/arel.rb is outside every package's libPath, so Arel.sql/star score as extra surface"
status: draft
updated: 2026-08-25
rfc: "0025-fidelity-verification-tooling"
cluster: null
packages: []
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

Surfaced in PR #7054 (RFC 0124, `arel-star-is-a-shared-const-not-a-per-call-method`).

`vendor/sources.ts` gives the arel package `libPath: "activerecord/lib/arel"`,
so the sibling file `activerecord/lib/arel.rb` — which defines the whole `Arel`
module surface — is extracted by nobody:

- `Arel.sql` (`arel.rb:51-57`)
- `Arel.star` (`arel.rb:59-61`)
- `Arel.fetch_attribute` (`arel.rb:63-72`)
- `Arel::VERSION` (`arel.rb:29`)

`rails-api.json` proves it: `packages.arel.modules.Arel` has empty
`classMethods` / `instanceMethods`, and the only trace of the file is a
`fileConstants["../arel.rb"] = { VERSION }` entry.

Two consequences, both measured today:

1. `packages/arel/src/arel.ts` — the faithful port of those very methods — is
   scored as a file **no Rails file maps onto**, so every public function in it
   is extra surface. `parity:api:extra --package arel` lists `Arel`, `sql`,
   `star` there (plus the `index.ts` re-exports).
2. Converging `Arel.star` from a module constant to Rails' per-call method
   (`arel.rb:59-61`) therefore _raised_ arel's extra-surface total by one, even
   though the change made the port strictly more faithful — constants are not
   scored, functions are. #7054 had to offset that by deleting the invented
   `Table#star` to keep the only-shrink gate green.

The file-level `@noRailsEquivalent` escape hatch does not apply: it is refused
when any name in the file scores as `moved`, and `sql`/`star` do.

## Converged shape

Let a package name extra lib files (or a second lib root) alongside `libPath`,
so `activerecord/lib/arel.rb` is extracted into the `arel` package and its
module functions match `packages/arel/src/arel.ts`. Expect arel's method
denominator to grow by the four names above and its extra-surface total to drop
by ~3 (`Arel`, `sql`, `star`), with `parity:api:extra:tighten` narrowing the
mark afterwards.

Check the same gap for other packages whose gem root file sits beside the lib
dir before generalizing the mechanism.

## Acceptance criteria

- `activerecord/lib/arel.rb` is extracted into the `arel` package; `Arel.sql`,
  `Arel.star`, `Arel.fetch_attribute` appear in `rails-api.json` as class
  methods of `Arel` and match their `arel.ts` ports.
- `packages/arel/src/arel.ts` is no longer reported as "[no Rails counterpart]".
- `pnpm parity:api` delta non-negative; extra-surface mark narrowed with
  `parity:api:extra:tighten` (never raised).
