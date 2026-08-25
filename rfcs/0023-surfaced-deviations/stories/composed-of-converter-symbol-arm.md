---
title: "composed_of's converter drops Rails' Symbol arm (klass.send(converter, part))"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced landing PR #6828 (`composed-of-local-derivations`, RFC 0099).

Rails' `writer_method` accepts a Symbol converter naming a class method
(`vendor/rails/activerecord/lib/active_record/aggregations.rb:264`):

```ruby
part = converter.respond_to?(:call) ? converter.call(part) : klass.send(converter, part)
```

and Rails' own test model uses it —
`vendor/rails/activerecord/test/models/customer.rb:13`,
`composed_of :fullname, ..., converter: :parse`.

trails' `ComposedOfOptions.converter`
(`packages/activerecord/src/aggregations.ts:45`) is function-only, so the
Symbol arm is dropped and `packages/activerecord/src/test-helpers/models/customer.ts:139`
spells it as a lambda wrapping `Fullname.parse`. PR #6828 added the matching
String arm for `:constructor` (`aggregations.ts` `readerMethod`); `converter`
is the remaining half.

Per CLAUDE.md, a Ruby Symbol value is a JS string, and this is the
`respond_to?(:call)`-vs-`send` discriminator, not a lookup-vs-literal one, so
the plain name (`"parse"`) is the spelling — as the `constructor` arm already
does.

## Converged shape

`converter?: ((value: unknown) => unknown) | string`, with the non-callable arm
resolving `klass[converter](part)`, mirroring the `constructor` arm; then
`customer.ts`'s `fullname` converter goes back to Rails' `"parse"`.

## Acceptance criteria

- [ ] `writerMethod` ports both arms of `converter.respond_to?(:call)`.
- [ ] `customer.ts` `fullname` declares `converter: "parse"`, as
      `test/models/customer.rb:13` does.
- [ ] `aggregations.test.ts` "custom converter" / "assigning hash to custom
      converter" stay green.
