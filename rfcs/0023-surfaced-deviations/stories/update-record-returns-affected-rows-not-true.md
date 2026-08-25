---
title: "Persistence#_update_record returns true where Rails returns affected_rows"
status: draft
updated: 2026-08-13
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

`Persistence#_update_record` returns the affected-row count; trails' returns a
bare `true`:

    # vendor/rails/activerecord/lib/active_record/persistence.rb:900-916
    def _update_record(attribute_names = self.attribute_names)
      attribute_names = attributes_for_update(attribute_names)

      if attribute_names.empty?
        affected_rows = 0
        @_trigger_update_callback = true
      else
        affected_rows = _update_row(attribute_names)
        @_trigger_update_callback = affected_rows == 1
      end

      @previously_new_record = false

      yield(self) if block_given?

      affected_rows
    end

trails (`packages/activerecord/src/persistence.ts`, `instanceUpdateRecord`)
matches this body line for line after PR #6473 EXCEPT that it declares
`Promise<boolean>`, never binds the empty-branch `affected_rows = 0`, and
`return true`s unconditionally. `AttributeMethods::Dirty#_update_record`
(dirty.rb:233-237) and `Timestamp`/`Callbacks`' layers all forward the value as
`affected_rows`, so every layer above is typed on a lie.

The one consumer is `base.ts:3381`:

    const updateOk = await this._updateRecord(block);

which treats it as a boolean. Rails' caller (`create_or_update`,
persistence.rb:889-895) does the same thing with the count — `result != false`
— so an affected-row count works there unchanged.

## Converged shape

`instanceUpdateRecord` returns `Promise<number>`, binding `affectedRows = 0` in
the empty branch as Rails does, and `createOrUpdate`'s `result != false` reads
the count. The `_updateRecord` declaration on `base.ts:4396`, the
`Dirty`/`Timestamp` super-chain signatures and the `PersistenceInstanceChainHost`
type all move with it.

## Acceptance criteria

- [ ] `_updateRecord` answers the affected-row count, `0` for the
      empty-`attribute_names` branch.
- [ ] `createOrUpdate` reads it as Rails' `result != false` rather than as a
      boolean, and the `_createRecord` sibling (which answers `id`) is
      unaffected.
- [ ] persistence, callbacks, dirty, timestamp and locking suites green on all
      three adapters.
