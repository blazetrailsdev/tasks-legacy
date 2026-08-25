---
title: "SelectCore#froms is a misplaced SelectManager method (arel/select_manager.rb:98)"
status: ready
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while burning down arel's `@internal` tags for RFC 0121 (#7045). The
work was reverted from that PR and filed here instead.

`packages/arel/src/nodes/select-core.ts` declares a `froms` getter/setter pair
that just aliases its own `from`/`from=`:

```ts
get froms(): Node | null {
  return this.from;
}
set froms(value: Node | null) {
  this.from = value;
}
```

Rails has no `froms` on `SelectCore` at all. It declares it ONE level up, on
`SelectManager`, where it means something different — it collects the `from` of
every core in the AST:

```ruby
# vendor/rails/activerecord/lib/arel/select_manager.rb:98
def froms
  @ast.cores.filter_map { |x| x.from }
end
```

`SelectCore` itself has only `from` (`arel/nodes/select_core.rb`). So trails has
the right NAME on the wrong HOST, with the wrong semantics (a single node rather
than a collection), and no `froms=` exists in Rails on either host.

Both halves currently carry `@internal` plus a `@noRailsEquivalent CONVERGEABLE`
receipt pointing here.

## Converged shape

- Delete `froms` / `froms=` from `SelectCore` — `from`/`from=` already mirror
  `select_core.rb`.
- If a `froms` reader is needed, put it on `SelectManager` with Rails'
  semantics: map every core's `from` and drop the nils (`filter_map`), returning
  an array.
- Update the call sites, then delete the two `@noRailsEquivalent` receipts.

Note for whoever takes this: arel is gated by `pnpm parity:api:extra:gate`
(only-shrink, no reseed). `froms` scores as **moved** — the name exists in Rails,
just on another host — and a moved name cannot be resolved with a receipt; it
converges by relocating the port. Removing `@internal` without relocating grows
arel's `total` and reds the gate.

## Acceptance criteria

- `SelectCore` has no `froms` / `froms=`.
- Any `froms` reader lives on `SelectManager` and mirrors
  `select_manager.rb:98` (`@ast.cores.filter_map { |x| x.from }`).
- Both `@noRailsEquivalent` receipts deleted.
- `pnpm parity:api:extra` reports no STALE tag; `pnpm parity:api:extra:gate`
  green with `extra-surface-mark.json` unchanged or tightened.
