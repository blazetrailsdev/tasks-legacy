---
title: "method_defined_within? tests prototype membership where Rails compares instance_method owners"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
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

Surfaced by PR #6371 (`converge-relation-where-clause-writer` bundle). That PR's
gate-2 accessor-pair fix made the `@missingRailsCall owner` tag on
`isMethodDefinedWithin` stale, so the tag was deleted per the only-shrink
contract — which left the DEVIATION it documented untracked. This story tracks
it.

Rails (`activerecord/lib/active_record/attribute_methods.rb:187-197`):

```ruby
def method_defined_within?(name, klass, superklass = klass.superclass) # :nodoc:
  if klass.method_defined?(name) || klass.private_method_defined?(name)
    if superklass.method_defined?(name) || superklass.private_method_defined?(name)
      klass.instance_method(name).owner != superklass.instance_method(name).owner
    else
      true
    end
  else
    false
  end
end
```

trails (`packages/activerecord/src/attribute-methods.ts`, `isMethodDefinedWithin`):

```ts
if (!(name in klass.prototype)) return false;
if (!superklass) return true;
return !(name in superklass.prototype);
```

Three divergences:

1. **`superklass` has no default.** Rails defaults it to `klass.superclass`; the
   port makes it optional and treats absent as "no superclass", so a two-arg
   call takes the `return true` arm where Rails would still compare owners.
2. **`name in klass.prototype` walks the WHOLE prototype chain**, where Ruby's
   `method_defined?` is also chain-walking but the OWNER comparison that follows
   is what actually decides. The port's `!(name in superklass.prototype)` answers
   "the superclass chain does not have it at all", which is a different question
   from Rails' "klass and superklass resolve it to different owners" — a method
   defined on `klass` that SHADOWS one on `superklass` answers `true` in Rails
   and `false` here.
3. **Private methods.** Rails tests `private_method_defined?` too; `in` sees TS
   private members (they are ordinary properties at runtime) but not `#`-private
   fields.

## Converged shape

Reproduce the owner comparison rather than the membership test. `Object.getOwnPropertyDescriptor`
walked up the prototype chain gives the defining prototype — the JS analogue of
`instance_method(name).owner` — so:

```ts
export function isMethodDefinedWithin(
  this: AttributeMethodsHost,
  name: string,
  klass: any,
  superklass: any = Object.getPrototypeOf(klass),
): boolean;
```

with the three Rails branches preserved in order, comparing owning prototypes in
the inner arm.

## Acceptance criteria

- [ ] `superklass` defaults to `klass`'s superclass, as Rails does.
- [ ] The inner arm compares OWNERS, not membership, so a shadowing definition
      answers `true`.
- [ ] A test covering the shadowing case that fails on baseline.
- [ ] `pnpm parity:api:calls` / `:args` green, no new baseline rows.
