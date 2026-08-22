---
title: "ActiveModel _assignAttribute writes through writeAttribute and sniffs error classes where Rails only sends the setter"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
packages: []
deps: []
deps-rfc: []
est-loc: 140
pr: 6738
claim: "2026-08-19T12:59:52Z"
assignee: "days-into-week-duplicated-in-date-calculations"
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6216, which converged the ActiveRecord copy of this arm
(`packages/activerecord/src/persistence.ts`'s `_assignAttribute` now calls
`self.attributeWriterMissing(key, value)` directly). The **ActiveModel** copy
was left on the old shape and is now the only one still diverging.

Rails (`vendor/rails/activemodel/lib/active_model/attribute_assignment.rb:67-75`):

```ruby
def _assign_attribute(k, v)
  setter = :"#{k}="
  public_send(setter, v)
rescue NoMethodError
  if respond_to?(setter)
    raise
  else
    attribute_writer_missing(k.to_s, v)
  end
end
```

There is **no `write_attribute` call in that body at all**. The only write is
`public_send(setter, v)`.

trails (`packages/activemodel/src/attribute-assignment.ts:38-63`) instead
attempts `model.writeAttribute(key, value)` in a `try`, then inspects the
thrown error to decide whether the key was writer-less:

```ts
try {
  model.writeAttribute(key, value);
} catch (error) {
  if (error instanceof UnknownAttributeError || error instanceof MissingAttributeError) {
    if (typeof model.attributeWriterMissing === "function") {
      model.attributeWriterMissing(key, value);
    } else {
      attributeWriterMissing(model, key, value);
    }
  } else {
    throw error;
  }
}
```

Three deviations in one body:

1. A `writeAttribute` call Rails does not make. Any key that IS a known
   attribute but has no setter gets silently written, where Rails raises
   `UnknownAttributeError`.
2. The writer-less test is "did `writeAttribute` throw one of two error
   classes", not Rails' `respond_to?(setter)`. A `MissingAttributeError` raised
   from deep inside a legitimate write is misread as "no setter" and swallowed
   into `attribute_writer_missing`.
3. `typeof model.attributeWriterMissing === "function"` guards a hook that
   `ActiveModel::Model` always defines (`packages/activemodel/src/model.ts:2483`),
   so the `else` arm calling the free function is unreachable branch surface.

Rails' `respond_to?(setter)` re-raise arm (`:70-71`) is also absent, same as it
was on the AR side.

## Converged shape

Mirror the Ruby exactly: resolve the setter, send it, and on a missing setter
call `attribute_writer_missing`. No `writeAttribute` call, no error-class
sniffing, no `typeof` guard on the hook.

```ts
export function _assignAttribute(model: AttributeAssignment, key: string, value: unknown): void {
  const setter = findSetter(model, key);
  if (setter) {
    setter.call(model, value);
    return;
  }
  model.attributeWriterMissing(key, value);
}
```

`attributeWriterMissing` becomes a required member of the `AttributeAssignment`
interface (it already is on `Model`), matching what PR #6216 did to
`AttributeIO` in `persistence.ts`.

Note the `respond_to?(setter)` re-raise arm is tracked separately by
[[ar-assign-attribute-bypasses-attribute-writer-missing]] — coordinate, since
that story's AR half was shipped by #6216 and its remaining scope is exactly
this re-raise arm on both copies.

## Acceptance criteria

- [ ] `activemodel`'s `_assignAttribute` makes no `writeAttribute` call.
- [ ] A writer-less key raises `UnknownAttributeError` via
      `attribute_writer_missing`, not via a rescued `MissingAttributeError`.
- [ ] `attributeWriterMissing` is required on `AttributeAssignment`; the
      `typeof ... === "function"` guard and its unreachable `else` are gone.
- [ ] activemodel + activerecord attribute-assignment suites green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
