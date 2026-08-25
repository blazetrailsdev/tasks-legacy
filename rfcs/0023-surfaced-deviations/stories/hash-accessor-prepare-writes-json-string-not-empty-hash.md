---
title: 'HashAccessor.prepare writes the string "{}" where Rails writes an empty Hash'
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
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

Surfaced by PR #6844, which converged `HashAccessor.read -> prepare`
(`packages/activerecord/src/store.ts`) and in doing so put every store read on
the `prepare` path, making a pre-existing divergence load-bearing.

Rails (`activerecord/lib/active_record/store.rb:227-241`):

```ruby
class HashAccessor # :nodoc:
  def self.read(object, attribute, key)
    prepare(object, attribute)
    object.public_send(attribute)[key]
  end

  def self.write(object, attribute, key, value)
    prepare(object, attribute)
    object.public_send(attribute)[key] = value if value != read(object, attribute, key)
  end

  def self.prepare(object, attribute)
    object.public_send :"#{attribute}=", {} unless object.send(attribute)
  end
end
```

`prepare` writes an empty **Hash**. trails writes the two-character **string**
`"{}"`:

```ts
static prepare(object: Base, attribute: string): void {
  const val = object.readAttribute(attribute);
  if (val === null || val === undefined) {
    object.writeAttribute(attribute, "{}");
  } else if (/* plain object -> promote to HWIA, then write .toHash() */) { ... }
}
```

Two consequences, both visible in the file today:

1. **`StringKeyedHashAccessor` needs an override Rails does not have.** Its
   `prepare` (store.ts) exists solely to undo the string, and says so: "the base
   `HashAccessor.prepare` writes `"{}"` (a JSON string), which the hstore parser
   rejects as invalid hstore format. Write `{}` (plain object) instead. Rails'
   `StringKeyedHashAccessor` does not override prepare". Rails
   (store.rb:243-251) overrides only `read` and `write`, both to `key.to_s`.
2. **The `else if` arm is not Rails.** store.rb:241's guard is a single Ruby
   truthiness test on the current value; trails adds a second branch that
   promotes a plain object to `HashWithIndifferentAccess` and writes
   `.toHash()`. `IndifferentHashAccessor.prepare` (store.rb:253-262) is where
   Rails does that promotion, and trails already has that override too — so the
   base class is doing a subclass's job.

The string write survives today only because `_readHash` / `_writeHash`
`JSON.parse` a string back. Those two helpers are themselves trails-only; Rails
just indexes the Hash the reader returns.

## Converged shape

```ts
static prepare(object: Base, attribute: string): void {
  if (!rubyTruthy(object.readAttribute(attribute))) {
    object.writeAttribute(attribute, {});
  }
}
```

— one branch, an empty object, mirroring store.rb:240-242 (note Ruby's `unless
object.send(attribute)`: false and nil are the only falsy values, so a `""` or
`0` stored value must NOT be replaced). Then delete
`StringKeyedHashAccessor.prepare`, and let `IndifferentHashAccessor.prepare`
(store.rb:253-262) remain the only override, as in Rails. `_readHash` /
`_writeHash` should lose their string arms with it.

## Acceptance criteria

- [ ] `HashAccessor.prepare` writes `{}`, not `"{}"`, in a single Rails-shaped
      branch with Ruby truthiness.
- [ ] `StringKeyedHashAccessor` overrides only `read` and `write`
      (store.rb:243-251); its `prepare` override is deleted.
- [ ] The HWIA-promotion arm lives only in `IndifferentHashAccessor.prepare`.
- [ ] `pnpm parity:api:extra --package activerecord` loses `_readHash` /
      `_writeHash` if their string arms are what justified them.
- [ ] `store.test.ts` green on SQLite, PostgreSQL and MySQL/MariaDB — the hstore
      arm on PG is what the deleted override was protecting.
