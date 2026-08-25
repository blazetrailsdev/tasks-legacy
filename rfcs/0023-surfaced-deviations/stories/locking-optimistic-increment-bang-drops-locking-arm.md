---
title: "Locking::Optimistic#increment! drops the locking-column bump and clear_attribute_change"
status: draft
updated: 2026-08-18
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

# Locking::Optimistic#increment! drops the entire locking arm

## Context

Found while converging the model-core call-set rows in PR #6722
(`wave-4c-ar-core-residue-model`, RFC 0106). The one `locking/optimistic.json`
row — `increment!` → `clear_attribute_change` — is not a naming row. It is a
missing method body.

Rails, `activerecord/lib/active_record/locking/optimistic.rb:63-70`:

```ruby
def increment!(*, **) # :nodoc:
  super.tap do
    if locking_enabled?
      self[self.class.locking_column] += 1
      clear_attribute_change(self.class.locking_column)
    end
  end
end
```

`Locking::Optimistic#increment!` is a _decorator_ over
`Persistence#increment!`: it calls `super`, then — only when
`locking_enabled?` — bumps the in-memory locking column and clears its dirty
entry, so the record's `lock_version` tracks the atomic `UPDATE ... SET c = c + n`
the superclass just issued and does not look dirty afterwards.

trails, `packages/activerecord/src/locking/optimistic.ts:95-107`, instead
re-ports `Persistence#increment!` into the locking file and drops the locking
arm entirely:

```ts
export async function incrementBang(..., attribute: string, by: number = 1) {
  this.increment(attribute, by);
  await this.updateColumn(attribute, this.readAttribute(attribute));
  return this;
}
```

So on a lock-enabled model, `increment!` leaves `lock_version` stale in memory
and (because there is no `clear_attribute_change`) whatever dirty state the
superclass left behind. The next `save` on that instance can then raise a
spurious `StaleObjectError` or write a stale `lock_version`.

Note that `Persistence#increment!` itself
(`packages/activerecord/src/persistence.ts`, `incrementBang`) is the real
superclass body; the optimistic.ts copy is a duplicate of it, which is why the
divergence is invisible to a reader of either file alone.

## Acceptance criteria

- [ ] `locking/optimistic.ts` `incrementBang` is the decorator Rails writes:
      delegate to the Persistence implementation, then, when
      `isLockingEnabled()`, increment the locking column in memory and call
      `clearAttributeChange(lockingColumn)`. The `super.tap` return value is the
      record, as in Rails.
- [ ] The duplicated `Persistence#increment!` body is removed from
      locking/optimistic.ts — one Rails method is one TS method.
- [ ] A regression test that FAILS on the current baseline: increment a column
      on a lock-enabled model, assert `lockVersion` advanced in memory and that
      the record is not dirty, then `save()` without a reload and assert no
      `StaleObjectError`. Test names match the Rails counterparts in
      `vendor/rails/activerecord/test/cases/locking_test.rb`.
- [ ] Delete the `increment!` → `clear_attribute_change` row from
      `scripts/api-compare/call-mismatches-exclude/activerecord/locking/optimistic.json`
      by hand (`serializeBaseline`), then
      `pnpm parity:api:calls:tighten activerecord/locking/optimistic.json`.
      No `--write`, no reseed.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
