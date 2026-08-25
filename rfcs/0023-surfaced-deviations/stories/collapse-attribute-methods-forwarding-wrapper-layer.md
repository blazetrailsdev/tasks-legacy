---
title: "attribute-methods.ts forwards 24 this-typed wrappers to record-first ports Rails has as plain mixin methods"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 400
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

In Rails, a module's instance methods _are_ the class's instance methods —
`include ActiveRecord::AttributeMethods::Dirty` puts `attribute_in_database`
itself on the record. trails inserts two extra layers Rails does not have:

1. The ported bodies in `packages/activerecord/src/attribute-methods/*.ts` take
   the record as an explicit **first parameter** rather than being `this`-typed:

   ```ts
   // attribute-methods/dirty.ts
   export function attributeInDatabase(record: DirtyRecord, attr: string): unknown;
   ```

2. `packages/activerecord/src/attribute-methods.ts` then carries a band of
   **24** `this`-typed wrappers whose whole body is a forward to layer 1:

   ```ts
   export function attributeInDatabase(this: InstanceMethodHost, attr: string): unknown {
     return _attributeInDatabase(this as any, attr);
   }
   ```

   …and `base.ts` mixes in the wrapper, not the port.

The settled trails idiom for Ruby `include` is a `this`-typed function assigned
to the class, or `include()` / `Included<>` (CLAUDE.md, "Module mixins"). The
record-first shape is neither, and the forwarding band is exactly the "extra
helper, wrapper, indirection layer" CLAUDE.md rules out. Each wrapper is also a
`ref:` the call gates have to see through.

The cost is not hypothetical: because the wrappers are ordinary exported
functions rather than the mixin itself, five of them
(`savedChanges`, `hasChangesToSave`, `changesToSave`,
`changedAttributeNamesToSave`, `attributesInDatabase`) sat in the file fully
wired-looking but imported by nobody — dead for as long as they existed, and
only found by hand-grepping `base.ts` during #6858. A layer that can hold dead
code that looks live is the argument against the layer.

`attribute-methods/primary-key.ts`'s `class PrimaryKey`
(`primary-key.ts:95-160`) is the shape that already works here, including for
accessor properties, and #6858 added `class Dirty` alongside it.

## Converged shape

Collapse both layers: the port becomes `this`-typed (or a class module where
the members are accessor properties) and `base.ts` mixes it in directly.

```ts
// attribute-methods/dirty.ts
export function attributeInDatabase(this: DirtyRecord, attr: string): unknown {
  const change = attributeChangeToBeSaved.call(this, attr);
  return change ? change[0] : this.readAttribute(attr);
}
```

with the `attribute-methods.ts` wrapper deleted and `base.ts` importing from
`attribute-methods/dirty.js`.

## Scope

The 24 wrappers in `attribute-methods.ts` and the 5 remaining record-first
exports across `attribute-methods/{dirty,before-type-cast,query,read,write}.ts`.
Larger than one PR — split by source file, one PR per
`attribute-methods/<file>.ts`, and file the rest as siblings.

## Acceptance criteria

- No exported function in `packages/activerecord/src/attribute-methods/*.ts`
  takes the record as an explicit first parameter; each is `this`-typed or a
  class-module member.
- The corresponding forwarding wrapper in `attribute-methods.ts` is deleted and
  `base.ts` mixes in the port directly.
- Any wrapper found to be dead is deleted rather than rewired.
- `pnpm parity:api:extra --package activerecord` loses the wrapper rows.
- Call gates clean, no reseed; `pnpm parity:api` / `pnpm parity:test` deltas
  non-negative.
