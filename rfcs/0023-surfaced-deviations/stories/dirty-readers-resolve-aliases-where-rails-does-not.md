---
title: "Dirty readers self-send resolve_attribute_name where Rails only does attr_name.to_s"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Every body in the `Dirty` module (`packages/activemodel/src/dirty.ts`, landed by
PR #6990) self-sends `resolve_attribute_name` before touching the tracker:

```ts
attributeChanged(name: string, options?: DirtyOptions): boolean {
  return this._dirty.attributeChanged(
    (this.constructor as unknown as DirtyClass).resolveAttributeName(name),
    options,
  );
}
```

Rails does not. `activemodel/lib/active_model/dirty.rb:300-302` is:

```ruby
def attribute_changed?(attr_name, **options)
  mutations_from_database.changed?(attr_name.to_s, **options)
end
```

`attr_name.to_s` and nothing more — same at `:305` (`attribute_was`), `:310`,
`:315`, `:367`, `:399`, `:404`, `:409`, `:414`. Alias resolution happens once,
at method-definition time: `alias_attribute`
(`activemodel/lib/active_model/attribute_methods.rb:96-119`) generates
`alias_name_changed?` whose body already passes the RESOLVED name, and
`resolve_attribute_name` (`:396-398`) is called from the read/write paths, not
from the dirty readers.

So in Rails `pirate.attribute_changed?("alias_name")` called directly does NOT
resolve the alias, while in trails it does. trails is the more lenient of the
two, which is the direction that hides bugs rather than surfacing them.

The extra call is invisible to both call gates (they flag calls Rails makes
that trails omits, not the reverse), so nothing will catch this but a reader.

The behaviour predates #6990 — the wrappers carried it in `model.ts` and the
fan-out moved them verbatim rather than changing behaviour mid-move.

## Acceptance criteria

- Establish what Rails actually does for a direct
  `attribute_changed?("alias_name")` call — run it under `ruby` against the
  vendored tree rather than deriving it; `pnpm rails:find alias_attribute` and
  `activemodel/test/cases/attribute_methods_test.rb` are the starting points.
- If Rails does not resolve, drop the `resolveAttributeName` self-send from the
  `Dirty` bodies so the two agree, and confirm the generated `*Changed` /
  `*Was` / `restore*!` methods still resolve aliases at definition time (that
  is where the resolution belongs).
- If some arm genuinely needs it, keep only that arm and cite the Rails line at
  the call site.
- `packages/activerecord/src/dirty.test.ts` and
  `packages/activemodel/src/dirty.test.ts` stay green; add coverage for the
  alias arm either way.
