---
title: "Return Rails' AttributeMethod Struct from AttributeMethodPattern#match"
status: claimed
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: "2026-08-25T15:30:56Z"
assignee: "port-attribute-method-pattern-match-struct"
blocked-by: null
closed-reason: null
---

## Context

`AttributeMethodPattern#match` in Rails returns a two-member `Struct`, not a
bare hash:

```ruby
# activemodel/lib/active_model/attribute_methods.rb:474
AttributeMethod = Struct.new(:proxy_target, :attr_name)

# activemodel/lib/active_model/attribute_methods.rb:485-489
def match(method_name)
  if @regex =~ method_name
    AttributeMethod.new(proxy_target, $1)
  end
end
```

trails' `match` (`packages/activemodel/src/attribute-methods.ts:114-120`)
returns the object literal `{ attr }` — it drops `proxy_target` entirely and
spells `attr_name` as `attr`. Callers that want the proxy target read it back
off the pattern instead of off the match result.

This is the one remaining CONVERGEABLE row in
`scripts/api-compare/call-mismatches-exclude/activemodel/attribute-methods.json`
(`match` omits `new`); PR #7034 reviewed its reason and explicitly deferred the
Struct port rather than closing it by rewording.

## Converged shape

- Declare `AttributeMethod` next to `AttributeMethodPattern` in
  `attribute-methods.ts`, with the Rails member names `proxyTarget` and
  `attrName`, mirroring `Struct.new(:proxy_target, :attr_name)`
  (attribute_methods.rb:474).
- `match` returns `new AttributeMethod(this.proxyTarget, m[1])` (or `null`),
  matching attribute_methods.rb:487.
- Update the call sites to read `attrName` / `proxyTarget` off the result
  rather than re-deriving the proxy target from the pattern.
- Hand-delete the `match` -> `new` row from
  `call-mismatches-exclude/activemodel/attribute-methods.json` (sorted position
  preserved, no `--write`), then
  `pnpm parity:api:calls:tighten activemodel/attribute-methods.json`.

## Acceptance criteria

- `attribute-methods.json` is at 2 rows, down from 3; the `match` -> `new` row
  is gone.
- `pnpm parity:api:calls`, `pnpm parity:api:calls:args`,
  `pnpm parity:api:extra --package activemodel` clean; no new baseline rows.
- `pnpm vitest run packages/activemodel` green.
