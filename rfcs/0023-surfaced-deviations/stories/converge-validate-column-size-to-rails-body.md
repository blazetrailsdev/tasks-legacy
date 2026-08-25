---
title: "Remove validateColumnSize's double-registration guard, converge to Rails' body"
status: draft
updated: 2026-08-12
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

# Remove validateColumnSize's double-registration guard, converge to Rails' body

## Context

Surfaced during review of PR #6406 (RFC 0099), which converted
`validateColumnSize` to a `this`-typed function but deliberately left the body
alone as out of scope.

Rails, `activerecord/lib/active_record/encryption/encryptable_record.rb:138-142`:

```ruby
def validate_column_size(attribute_name)
  if limit = columns_hash[attribute_name.to_s]&.limit
    validates_length_of attribute_name, maximum: limit
  end
end
```

Three terms, no more. trails
`packages/activerecord/src/encryption/encryptable-record.ts` carries two extras:

1. A `typeof this.validatesLengthOf !== "function"` early return Rails has no
   analogue for — Ruby's `validates_length_of` is always defined on the model.
2. A dedup guard that scans `this._validators.get(attributeName)` for an
   existing `LengthValidator` with the same `maximum` and skips registration
   when one is found. Rails registers unconditionally; calling
   `validates_length_of` twice with the same options is idempotent enough in
   Rails that `load_schema!` re-running it is harmless.

The guard exists because trails calls `validateColumnSize` from THREE places
where Rails calls it from one:

- `encryptAttribute` (`encryptable-record.ts`), at declaration time — Rails does
  NOT do this; `encrypt_attribute` (`encryptable_record.rb:84-96`) never calls it.
- `addLengthValidationForEncryptedColumns` from `loadSchemaBang` — this is
  Rails' only call path (`encryptable_record.rb:129-136`).
- `applyPendingEncryptions` (`encryption.ts`), re-running after schema
  reflection — also not a Rails path.

So the guard is a symptom: the real divergence is the two extra call sites. The
declaration-time call was presumably added because trails resolves column limits
later than Rails does.

## Converged shape

`validateColumnSize` reduces to Rails' three terms — read the limit off the
column, call `validatesLengthOf` when present, nothing else. That requires
first establishing that `loadSchemaBang` / `applyPendingEncryptions` is the only
caller, i.e. deleting the `encryptAttribute` call site
(`encryptable-record.ts`, guarded by `Configurable.config.validateColumnSize`)
and the `encryption.ts` re-run, or collapsing the two remaining paths into one
that fires exactly once per class after schema reflection.

If a limit genuinely is not knowable at `encrypts()` time in trails, the
declaration-time call is the thing to delete, not the guard to keep.

Note `_attributeDefinitions.get(name)?.limit` stands in for Rails'
`columns_hash[name]&.limit`; that substitution is separate and not in scope
here unless it turns out to be why the extra call sites exist.

## Acceptance criteria

1. `validateColumnSize`'s body matches `encryptable_record.rb:138-142` — the
   limit read, the `validatesLengthOf` call, no dedup scan and no
   `typeof … !== "function"` guard.
2. `validate_column_size` is reached from Rails' call path only
   (`load_schema!` -> `add_length_validation_for_encrypted_columns`); any extra
   trails call site is deleted, or justified AT THE CALL SITE with a Rails cite
   and a genuine TypeScript language shortcoming.
3. The encryption suites stay green, in particular the column-size tests in
   `packages/activerecord/src/encryption/`.
4. `pnpm parity:api:calls` / `pnpm parity:api:calls:args` stay green, and no
   baseline row is added.
