---
title: "LockingType adds a cast() override Rails does not have"
status: draft
updated: 2026-08-13
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

Rails' `LockingType` overrides `deserialize` and `serialize` only — `cast` is
inherited from the delegated subtype, so `cast(nil)` is `nil`:

    # vendor/rails/activerecord/lib/active_record/locking/optimistic.rb:206-217
    class LockingType < DelegateClass(Type::Value) # :nodoc:
      def self.new(subtype)
        self === subtype ? subtype : super
      end

      def deserialize(value)
        super.to_i
      end

      def serialize(value)
        super.to_i
      end

trails (`packages/activerecord/src/locking/optimistic.ts`) adds a third
override and says so at the call site:

    // Diverges from Rails: Rails' LockingType has no cast() override
    // (cast(nil) → nil). We coerce null → 0 here so that user-declared locking
    // attributes (via this.attribute("lock_version", "integer")) also return 0
    // for new records, matching the observable behavior Rails gets via
    // from_database initialization.
    override cast(value: unknown): number {
      return (this._subtype.cast(value) as number | null) ?? 0;
    }

The stated reason is that Rails seeds a new record's lock column through
`from_database` (the reflected column default), which trails cannot rely on for
a _user-declared_ `attribute("lock_version", "integer")`. That is a real gap,
but the fix belongs where the seeding happens, not in the type: `_createRecord`
(`packages/activerecord/src/persistence.ts`) already seeds the locking column
from its reflected default before the INSERT for exactly this reason, and the
`cast` override now double-covers it while silently changing `cast` for every
other caller.

Surfaced while converging `_lock_value_for_database` in PR #6473: that method's
`?? 0` fallbacks WERE a real bug — a lock column NULL in the row constrained as
`= 0` and matched nothing — and this `cast` override is the last `nil → 0`
coercion of the same family still standing.

## Converged shape

Drop the `cast` override so `LockingType` is `deserialize` + `serialize` only,
and seed a user-declared locking attribute where Rails' `from_database` seeding
lands instead. Check `_defaultAttributes` / the `_createRecord` seed first — the
declared-attribute path may already reach it.

## Acceptance criteria

- [ ] `LockingType` overrides `deserialize` and `serialize` only, matching
      optimistic.rb:206-217.
- [ ] A user-declared `attribute("lock_version", "integer")` model still reads
      `0` on a new record and still inserts `0`.
- [ ] The `// Diverges from Rails` comment is deleted — it is the prose this
      story converges.
- [ ] locking, custom-locking and the declared-attribute locking coverage green
      on all three adapters.
